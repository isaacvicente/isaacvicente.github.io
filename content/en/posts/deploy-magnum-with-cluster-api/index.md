---
title: Magnum + Cluster API
date: 2025-09-15
summary: In this post, I detail how to install Magnum with the Cluster API to create clusters in Openstack.
toc: true
readTime: true
---

# Magnum + Cluster API

![Magnum shark with Cluster API turtles](./clusterapi-magnum.png#small "Magnum shark with Cluster API turtles")

Magnum is Openstack's native service for container orchestration and
supports Kubernetes as a container orchestrator.

The [Cluster API (CAPI)](https://cluster-api.sigs.k8s.io/) is a Kubernetes
project to simplify the deployment and management of k8s clusters, and supports
several cloud providers, including Openstack.

It is possible to use CAPI with Openstack without needing Magnum. However, in
this post, I will introduce the use of CAPI with Magnum. This way, we can
create multiple clusters based on defined templates without having to worry
about how things are working under the hood. This will provide a Cluster as a
Service (CaaS) offering.

## k3s

For the management cluster, that is, the CAPI cluster, we will use
[k3s](https://docs.k3s.io/).

Follow the official [k3s documentation](<https://docs.k3s.io/quick-start>)
for installation.

### Installing k3s

```bash
curl -sfL https://get.k3s.io | sh -
```

### Accessing the k3s cluster

By default, the k3s configuration file is located at
`/etc/rancher/k3s/k3s.yaml`. Therefore, we need to make a configuration change.

First, make a copy of the file to your current location and edit it:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml .
# Change the file permissions so you can access it as the current user
sudo chown <user>:<group> k3s.yaml
vim k3s.yaml
```

Edit the file and change the server address to the IP of the management cluster node.
For example, change it from `127.0.0.1` (`localhost`) to `10.0.0.11`.

```diff
- server: 127.0.0.1:6443
+ server: 10.0.0.11:6443
```

Now, use the file to access the cluster.

```bash
export KUBECONFIG=k3s.yml
# List all pods to test connectivity
kubectl get pods -A
```

## Installing CAPI on the cluster

The Cluster API provides great
[documentation](https://cluster-api.sigs.k8s.io/user/quick-start) on how to
install and test CAPI.

Install `clusterctl` on the deploy node.

```bash
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.11.1/clusterctl-linux-amd64 -o clusterctl
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
```

To verify that clusterctl is the latest version, run

```bash
clusterctl version
```

Before initializing `clusterctl` for the Openstack provider, export this
variable.

```bash
export CLUSTER_TOPOLOGY=true
```

Now, initialize the Openstack provider.

```bash
# Install ORC (needed for CAPO >=v0.12)
kubectl apply -f https://github.com/k-orc/openstack-resource-controller/releases/latest/download/install.yaml
# Initialize the management cluster
clusterctl init --infrastructure openstack
```

## Configuring Magnum

### Deploy Magnum using `kolla`

Edit the `globals.yaml` file in `/etc/kolla/`.

```yaml
# globals.yaml
enable_magnum: true
enable_cluster_user_trust: true
```

```bash
kolla-ansible deploy -i <inventory> --tags common,horizon,magnum
```

Now, we need to configure Magnum to use the Cluster API to create the clusters.

```bash
# Create the Magnum configuration directory
mkdir -p /etc/kolla/config/magnum
# Copy k3s.yaml as the kubeconfig file for Magnum to use
cp k3s.yaml /etc/kolla/config/magnum/kubeconfig
```

Reconfigure Magnum to use this file and authenticate with the Cluster API
management cluster.

```bash
kolla-ansible reconfigure -i <inventory> --tags magnum
```

### Downloading a CAPI-compatible image

You must have at least one Cluster API-compatible image in Openstack.

To do this, we'll need to download the image and upload it to Openstack.

Next, we download a compatible Ubuntu 22.04 image and upload it to Openstack.
The Kubernetes version used here is `v1.27.4`.

```bash
export OS_DISTRO=ubuntu # you can change this to "flatcar" if you want to use Flatcar
for version in v1.27.4; do \
[[ "${OS_DISTRO}" == "ubuntu" ]] && IMAGE_NAME="ubuntu-22.04-kube-${version}" || IMAGE_NAME="flatcar-kube-${version}"; \ 
curl -LO https://object-storage.public.mtl1.vexxhost.net/swift/v1/a91f106f55e64246babde7402c21b87a/magnum-capi/${IMAGE_NAME}.qcow2; \ 
openstack image create ${IMAGE_NAME} --disk-format=qcow2 --container-format=bare --property os_distro=${OS_DISTRO} --file=${IMAGE_NAME}.qcow2;
done;
```

## Creating a template

> Requirements:
>
> 1. Load the Openstack admin environment variables
> 2. Have the Magnum client downloaded to the environment (`pip install python-magnumclient`)
> 3. To create a Magnum template, you must have an external network, here as `external`

```bash
export version=v1.27.4
openstack coe cluster template create \
    --image $(openstack image show ${IMAGE_NAME} -c id -f value) \
    --external-network external \
    --dns-nameserver 8.8.8.8 \
    --master-lb-enabled \
    --master-flavor general.medium \
    --flavor general.small \
    --network-driver calico \
    --docker-storage-driver overlay2 \
    --coe kubernetes \
    --label kube_tag=${version},availibilty_zone=nova \
    k8s-${version};
```

## Creating a cluster

```bash
openstack coe cluster create \
    --cluster-template k8s-v1.27.4 \
    --master-count 1 \
    --node-count 1 \
    k8s-v1.27.4
```

This command creates a cluster named `k8s-v1.27.4` with 1 control node and 1
worker node.

## Conclusion

With this, we have a working and accessible k8s cluster. We can then expose
this cluster template as public so that cloud users can transparently create
clusters based on the defined templates.

## References

* <https://cluster-api.sigs.k8s.io/>
* <https://www.roksblog.de/openstack-magnum-cluster-api-driver/>
* <https://vexxhost.github.io/magnum-cluster-api/user/getting-started/>
* <https://vexxhost.github.io/magnum-cluster-api/admin/troubleshooting/>
* <https://www.stackhpc.com/magnum-clusterapi.html>
* <https://www.stackhpc.com/magnum-cluster-api-helm-deep-dive.html>
* <https://satishdotpatel.github.io/openstack-magnum-capi/>
