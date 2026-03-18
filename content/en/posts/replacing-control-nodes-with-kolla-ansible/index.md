---
title: Replacing control nodes with Kolla Ansible
date: 2026-03-04
toc: true
summary: How to add and remove control nodes in an OpenStack cloud using `kolla-ansible`
tags: [openstack, cloud, kolla-ansible]
readTime: true
---

Maintaining an OpenStack cloud in production requires constant care, and one of
the tasks that may arise is replacing or adding new control nodes. Whether due
to expansion, hardware failure, or infrastructure upgrades, the process needs
to be secure and well-planned to avoid downtime.

We went through this experience in our environment, using [**kolla-ansible**](https://opendev.org/openstack/kolla-ansible)
as the deployment tool. We followed the [official
documentation](https://docs.openstack.org/kolla-ansible/latest/user/adding-and-removing-hosts.html),
but as with any real-world environment, we encountered some challenges. In this
post, I share the step-by-step process we executed, the checks we performed, and
an unexpected issue with RabbitMQ – and how we resolved it. Additionally, I
present the complete process for removing control nodes, based on the
documentation and lessons learned.

---

## Adding a new control node

### Prerequisites

The new control node needs to be properly prepared. In our case, all servers
use **bonding** for high availability of network interfaces. Therefore, it was
necessary to:

- Install **Ubuntu 22.04** on the machine.
- Configure bonding (with two interfaces) and, on top of it, create **two VLANs**:
  - One for internal communication of cloud services.
  - Another for external communication (external services, API).
- Assign an IP to each VLAN and ensure the new node was reachable via both addresses.

This step is important because kolla-ansible uses these IPs to configure
services and communication between nodes.

### Adding the node to the cluster with Kolla-Ansible

We used a `multinode` inventory file listing all nodes (including the new
one). The addition was done in stages, always limiting execution to the
`control` group (or to the new host specifically).

```bash
# Prepares the node: installs dependencies, configures repositories, etc.
kolla-ansible bootstrap-servers -i multinode --limit control

# Pulls container images for the new node
kolla-ansible pull -i multinode --limit <new-host>

# Runs the deployment only on control nodes (updates configuration)
kolla-ansible deploy -i multinode --limit control
```

After the deployment, the new node should already be participating in the
cluster. However, verification is necessary.

### Post-deployment checks

We validated the main services running on the control nodes: MariaDB (Galera),
RabbitMQ, Neutron, and Nova.

#### **MariaDB (Galera)**

The MariaDB (Galera) cluster needs to be synchronized and have the correct size.

```bash
# Retrieves the MariaDB root password from the configuration file
ROOT_PASS=$(cat /etc/kolla/mariadb/galera.cnf | grep wsrep_sst_auth | cut -f 2 -d:)

# Checks the number of nodes in the cluster
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_cluster_size';"

# Confirms the cluster is primary (accepts read/write)
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_cluster_status';"

# Verifies if the current node is synchronized (synced)
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"

# Lists the addresses of cluster members
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_incoming_addresses';"
```

#### **RabbitMQ**

RabbitMQ is OpenStack's message bus. We checked the cluster status and connectivity:

```bash
# RabbitMQ cluster status
docker exec -it rabbitmq rabbitmqctl cluster_status

# Tests connectivity between listener ports
docker exec -it rabbitmq rabbitmq-diagnostics check_port_connectivity
```

The new node appeared in the cluster, and connectivity was ok.

#### **Neutron and Nova**

Finally, we checked if agents and compute services were present:

```bash
openstack network agent list --host <new-host>
openstack compute service list --host <new-host>
```

Up to this point, everything indicated that the addition had been a success.
However, when attempting to create an instance, the error appeared.

### Error: "missing queue" in RabbitMQ

When creating a VM, the operation failed with the following message in the `nova-conductor` log:

```
The reply XXXX failed to send after 60 seconds due to a missing queue
oslo_messaging.exceptions.MessageUndeliverable
```

Analyzing further, we realized that RabbitMQ was in HA (high availability), but
the **queues** had not been created on the new node. The queues existing on the
other control nodes were not automatically replicated to the new cluster
member. When `nova-conductor` tried to send a message to a queue that only
existed on the old nodes, the new node couldn't deliver it, resulting in the
error.

### Solution: restart RabbitMQ on all nodes

After some investigation, the solution found was to restart the RabbitMQ
service on all control nodes. This forced queue reconciliation and complete
cluster synchronization. For this, it is necessary to perform a rolling restart
to ensure that a subsequent container on another node can only be restarted
when the previous one is functional.

```bash
# Restart RabbitMQ on the first node
docker restart rabbitmq

# Check if the container is HEALTHY
watch 'docker ps -a --filter name=rabbitmq'
```

When RabbitMQ is working correctly on the first node, repeat the same
procedure for the next ones.

After the restart, the cluster recreated the queues, and VM creation returned to
normal.

---

## Removing a control node

Like addition, removing a control node needs to be done carefully to avoid
affecting cluster availability. The official Kolla-Ansible documentation
recommends a step-by-step process, which includes reallocating resources,
stopping services, and cleaning up registries.

> [!WARNING]
> Before removing any node, verify that the remaining nodes
> still have sufficient quorum. In a cluster with 3 controllers, remove only
> one at a time.

### Step 1: Move Neutron resources

Before removing the node, it is necessary to reallocate routers and DHCP
networks managed by the L3 and DHCP agents residing on the host to be removed.

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

# Disables the old L3 agent
openstack network agent set $l3_id --disable
```

**For DHCP agents:**

```bash
# Identifies the DHCP agent ID on the host to be removed
dhcp_id=$(openstack network agent list --host <host> --agent-type dhcp -f value -c ID)

# Identifies the ID of a DHCP agent on another node (target)
target_dhcp_id=$(openstack network agent list --host <target-host> --agent-type dhcp -f value -c ID)

# Moves each network from the old agent to the new one
openstack network list --agent $dhcp_id -f value -c ID | while read network; do
  openstack network agent remove network $dhcp_id $network --dhcp
  openstack network agent add network $target_dhcp_id $network --dhcp
done
```

### Step 2: Moving Cinder Resources

> [!WARNING]
> We use external Ceph for Cinder pools. Because of this, the following
> solution logically updates (internally in the database) the `host` attribute
> of the volumes, which does not interfere with the Cinder pools because they
> are outside the host (in Ceph). If you use any internal backend, such as LVM,
> this solution may not work.

To update the host reference for the volumes, access the host where the volumes
will be migrated:

```bash
docker exec -it -u0 cinder_volume cinder-manage volume update_host --currenthost <old_host> --newhost <new_host>
```

To update the reference to the service_uuid column in the Cinder database, run
the following command from the host where the volumes were updated
(`<new_host>` in the commands above):

```bash
docker exec -it -u0 cinder_volume cinder-manage volume update_service
```

To confirm, list the volumes associated with the backend of the host to be removed:

```bash
cinder list --filters host=<host> --all-tenants 1
```

After confirming that there are no associated volumes from the host's backend
to be removed, the next step is to disable the cinder-volume service. Thus,
determine the binary and host to be removed:

```bash
openstack volume service list
```

Then disable the service:

```bash
openstack volume service set --disable HOST_NAME BINARY_NAME
```

Similarly, to completely remove the service, go to one of the
storage hosts (that have the cinder-volume service) and run:

```bash
cinder-manage service remove BINARY_NAME HOST_NAME
```

### Step 3: Stop services on the removed host

With resources reallocated, we can stop all containers on the host to be removed:

```bash
kolla-ansible stop -i multinode --yes-i-really-really-mean-it --limit <host>
```

### Step 4: Remove the host from the Ansible inventory

Edit your inventory file (`multinode`) and remove or comment out the lines
referring to the host being retired.

### Step 5: Reconfigure the remaining controllers

#### **Extra care with RabbitMQ**

As with addition, RabbitMQ deserves special attention. There are
[indications](https://bugs.launchpad.net/kayobe/+bug/2085608) that, even after
removing the node from the inventory and reconfiguring, RabbitMQ might still
list the removed node in the cluster. This can cause connectivity issues.

Check if the host to be removed is still listed:

```bash
docker exec -it rabbitmq rabbitmqctl cluster_status
```

If it is still listed in any way (in our case, the host still appeared under
`Disk Nodes`), remove the host:

```bash
docker exec -it rabbitmq rabbitmqctl forget_cluster_node <node-name>
```

> [!TIP]
> `node-name` here is generally `rabbit@<host>`.

Verify again that the host has indeed been removed, then proceed.

#### **Deploy**

Now it's necessary to reconfigure the remaining control nodes so they update
the cluster state (MariaDB, RabbitMQ, etc.).

```bash
kolla-ansible deploy -i multinode --limit control
```

### Step 6: Clean up services from the removed host

Finally, remove the registries of OpenStack agents and services that might
still be visible:

```bash
# Removes network agents
openstack network agent list --host <host> -f value -c ID | while read id; do
  openstack network agent delete $id
done

# Removes compute services
openstack compute service list --os-compute-api-version 2.53 --host <host> -f value -c ID | while read id; do
  openstack compute service delete --os-compute-api-version 2.53 $id
done
```

## Conclusion

Replacing control nodes in an OpenStack environment with kolla is a
well-documented process, but it's not always free of surprises. The main lesson
we learned was: **always check the state of RabbitMQ queues after adding a new
node** and, when removing, **ensure the node has been completely forgotten by
the messaging cluster**.

A simple queue listing (`rabbitmqctl list_queues`) or cluster check
(`rabbitmqctl cluster_status`) before and after could have indicated the
problem earlier.

Fortunately, the solutions were simple (although they involve restarting
critical services). In production environments, it's advisable to plan a
maintenance window for this type of operation.

In the end, the entire process was much smoother than we thought, and this
really shows how fascinating the
[kolla-ansible](https://opendev.org/openstack/kolla-ansible) project is.
