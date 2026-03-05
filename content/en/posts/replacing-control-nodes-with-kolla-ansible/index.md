---
title: Replacing Control Nodes with Kolla Ansible
date: 2026-03-04
toc: true
summary: How to add and remove control nodes in an OpenStack cloud using `kolla-ansible`
tags: [openstack, cloud, kolla-ansible]
readTime: true
---

Maintaining an OpenStack cloud in production requires constant care, and one of
the tasks that may arise is the replacement or addition of new control nodes.
Whether due to expansion, hardware failure, or infrastructure upgrade, the
process needs to be secure and well-planned to avoid downtime.

We went through this experience in our environment, using [**kolla-ansible**](https://opendev.org/openstack/kolla-ansible)
as a deployment tool. We followed the [official
documentation](https://docs.openstack.org/kolla-ansible/latest/user/adding-and-removing-hosts.html),
but like any real-world environment, we encountered some challenges. In this post,
I share the step-by-step process we followed, the checks performed, and an
unexpected problem with RabbitMQ – and how we solved it. Additionally, I present the
complete process for removing control nodes, based on the documentation and
lessons learned.

---

## Adding a new control node

### Prerequisites

The new control node needs to be properly prepared. In our case, all servers use **bond** for high availability of network interfaces. Therefore, it was necessary to:

- Install **Ubuntu 22.04** on the machine.
- Configure a bond (link aggregation) and, on top of it, create **two VLANs**:
  - One for internal communication of cloud services.
  - Another for external communication (external services, API).
- Assign an IP to each VLAN and ensure that the new node is accessible by both addresses.

This step is important because kolla-ansible uses these IPs to configure services and communication between nodes.

### Adding the node to the cluster with Kolla-Ansible

We used a `multinode` file with a list of all nodes (including the new one).
The addition was done in stages, always limiting the execution to the `control`
group (or to the new host specifically).

```bash
# Prepares the node: installs dependencies, configures repositories, etc.
kolla-ansible bootstrap-servers -i multinode --limit control

# Downloads the container images to the new node
kolla-ansible pull -i multinode --limit <new-host>

# Executes the deployment only on the control nodes (updates the configuration)
kolla-ansible deploy -i multinode --limit control
```

After deployment, the new node should already be participating in the cluster. However, it needs to be verified.

### Post-deploy checks

We validate the main services running on the control nodes: MariaDB (Galera), RabbitMQ, Neutron, and Nova.

#### **MariaDB (Galera)**

The MariaDB (Galera) cluster needs to be synchronized and have the correct size.

```bash
# Gets the MariaDB root password from the configuration file
ROOT_PASS=$(cat /etc/kolla/mariadb/galera.cnf | grep wsrep_sst_auth | cut -f 2 -d:)

# Checks the number of nodes in the cluster
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_cluster_size';"

# Checks if the cluster is primary (accepts read/write access)
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_cluster_status';"

# Checks if the current node is synchronized (synced)
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"

# Lists the addresses of the cluster members
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_incoming_addresses';"
```

#### **RabbitMQ**

RabbitMQ is the OpenStack message bus. We checked the cluster status and connectivity:

```bash
# RabbitMQ cluster status
docker exec -it rabbitmq rabbitmqctl cluster_status

# Test connectivity between listener ports
docker exec -it rabbitmq rabbitmq-diagnostics check_port_connectivity
```

The new node appeared in the cluster and connectivity was OK.

#### **Neutron and Nova**

Finally, we checked if the agents and compute services were present:

```bash
openstack network agent list --host <new-host>
openstack compute service list --host <new-host>
```

Up to that point, everything indicated that the addition had been successful. However, when trying to create an instance, the error appeared.

### Error: "missing queue" in RabbitMQ

When creating a VM, the operation failed with the following message in the `nova-conductor` log:

```
The reply XXXX failed to send after 60 seconds due to a missing queue
oslo_messaging.exceptions.MessageUndeliverable
```

Upon further investigation, we realized that RabbitMQ was in HA (high availability) mode, but the **queues** had not been created on the new node. The queues existing on the other control nodes were not automatically replicated to the new cluster member. When `nova-conductor` attempted to send a message to a queue that only existed on the old nodes, the new node failed to deliver it, resulting in the error.

### Solution: Restart RabbitMQ on all nodes

After some investigation, the solution found was to restart the RabbitMQ service
**on all control nodes**. This forced the reconciliation of the queues and the
complete synchronization of the cluster.

```bash
# On each control node
docker restart rabbitmq
```

After the restart, the cluster recreated the queues and VM creation returned to
normal.

---

## Removing a control node

Just like adding a control node, removing one needs to be done carefully to
avoid affecting the availability of the cluster. The official Kolla-Ansible
documentation recommends a step-by-step process, which includes reallocating
resources, stopping services, and cleaning up the records.

> [!WARNING]
> Before removing any node, verify that the remaining nodes
> still have sufficient quorum. In a cluster with 3 controllers, remove only
> one at a time.

### Step 1: Move Neutron resources

Before removing the node, it is necessary to relocate the routers and DHCP networks managed by the L3 and DHCP agents that are on the host to be removed.

**For L3 agents (routers):**

```bash
# Identifies the L3 agent ID on the host to be removed
l3_id=$(openstack network agent list --host <host> --agent-type l3 -f value -c ID)

# Identifies the ID of an L3 agent on another node (target)
target_l3_id=$(openstack network agent list --host <target-host> --agent-type l3 -f value -c ID)

# Moves each router from the old agent to the new one
openstack router list --agent $l3_id -f value -c ID | while read router; do
  openstack network agent remove router $l3_id $router --l3
  openstack network agent add router $target_l3_id $router --l3
done

# Disable the old L3 agent
openstack network agent set $l3_id --disable
```

**For DHCP agents:**

```bash
# Identifies the DHCP agent ID on the host to be removed
dhcp_id=$(openstack network agent list --host <host> --agent-type dhcp -f value -c ID)

# Identifies the ID of a DHCP agent on another node (target)
target_dhcp_id=$(openstack network agent list --host <target-host> --agent-type dhcp -f value -c ID)

# Move each network from the old agent to the new one
openstack network list --agent $dhcp_id -f value -c ID | while read network; do
  openstack network agent remove network $dhcp_id $network --dhcp
  openstack network agent add network $target_dhcp_id $network --dhcp
done
```

### Step 2: Stop services on the removed host

With the resources reallocated, we can stop all containers on the host to be removed:

```bash
kolla-ansible stop -i multinode --yes-i-really-really-mean-it --limit <host>
```

### Step 3: Remove the host from the Ansible inventory

Edit your inventory file (`multinode`) and remove or comment out the lines referring to the host being removed.

### Step 4: Reconfigure the remaining controllers

#### **Extra care with RabbitMQ**

Just like with the addition, RabbitMQ deserves special attention. There are [indications](https://bugs.launchpad.net/kayobe/+bug/2085608) that, even after removing the node from the inventory and reconfiguring, RabbitMQ may continue to list the removed node in the cluster. This can cause connectivity problems.

Check if the host to be removed is still listed:

```bash
docker exec -it rabbitmq rabbitmqctl cluster_status
```

If it is still listed in some way (in our case the host still appeared in `Disk Nodes`), remove the host:

```bash
docker exec -it rabbitmq rabbitmqctl forget_cluster_node <node-name>
```

> [!TIP]
> `node-name` here is usually `rabbit@<host>`.

Verify again that the host has actually been removed, then proceed.

#### **Deploy**

Now it is necessary to reconfigure the remaining control nodes so that they update the cluster state (MariaDB, RabbitMQ, etc.).

```bash
kolla-ansible deploy -i multinode --limit control
```

### Step 5: Clean up services on the removed host

Finally, remove any OpenStack agent and service records that may still be visible:

```bash
# Remove network agents
openstack network agent list --host <host> -f value -c ID | while read id; do
  openstack network agent delete $id
done

# Remove compute services
openstack compute service list --os-compute-api-version 2.53 --host <host> -f value -c ID | while read id; do
  openstack compute service delete --os-compute-api-version 2.53 $id
done
```

## Conclusion

Replacing control nodes in an Openstack environment with kolla is a well-documented process, but it's not always free of surprises. The main lesson we learned was: **always check the status of RabbitMQ queues after adding a new node**, and when removing, **make sure the node has been completely forgotten by the messaging cluster**.

A simple queue listing (`rabbitmqctl list_queues`) or cluster check (`rabbitmqctl cluster_status`) before and after could have indicated the problem earlier.
