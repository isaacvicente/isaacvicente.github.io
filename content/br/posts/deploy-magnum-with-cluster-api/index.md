---
title: Magnum + Cluster API
date: 2025-09-15
summary: Nesse post detalho a instalação do Magnum junto ao Cluster API para criação de clusters no Openstack
toc: true
readTime: true
---

# Magnum + Cluster API

![Magnum shark with Cluster API turtles](./clusterapi-magnum.png#small "Magnum shark with Cluster API turtles")

O Magnum é o serviço nativo do Openstack para orquestração de containers e
oferece suporte ao Kubernetes como orquestrador de containers.

O [Cluster API (CAPI)](https://cluster-api.sigs.k8s.io/) é um projeto do
Kubernetes para simplificar o deploy e gerência de clusters k8s, e oferece
suporte a vários _cloud providers_, incluindo o Openstack.

É possível utilizar o CAPI junto ao Openstack sem a necessidade do Magnum.
Porém, nesse post irei introduzir o uso do CAPI junto ao Magnum. Dessa forma,
podemos criar vários clusters baseados em templates definidos sem nos
procuparmos sobre como as coisas estão funcionando de baixo dos panos. Com isso
teremos uma oferta de Cluster as a Service (CaaS).

## k3s

Para o cluster de gerência, isto é, o cluster do CAPI, iremos utilizar o
[k3s](https://docs.k3s.io/).

Siga a [documetação oficial](https://docs.k3s.io/quick-start)
do k3s para instalação.

### Instalação do k3s

```bash
curl -sfL https://get.k3s.io | sh -
```

### Acessando cluster k3s

Por padrão, o arquivo de configuração do k3s fica em
`/etc/rancher/k3s/k3s.yaml`. Nesse sentido, precisamos fazer uma mudança na
configuração.

Primeiro, faça uma cópia do arquivo para onde você está e o edite:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml .
# Mude as permissões do arquivo para pode acessá-lo como usuário atual
sudo chown <user>:<group> k3s.yaml
vim k3s.yaml
```

Edite o arquivo e mude o endereço de servidor para o IP do nó de cluster de gerenciamento.
Por exemplo, mude de `127.0.0.1` (`localhost`) para `10.0.0.11`.

```diff
-    server: 127.0.0.1:6443
+    server: 10.0.0.11:6443
```

Agora, use o arquivo para acessar o cluster.

```bash
export KUBECONFIG=k3s.yml
# Liste todos os pods para testar a conectividade
kubectl get pods -A
```

## Instalação do CAPI no cluster

O Cluster API fornece uma ótima
[documetação](https://cluster-api.sigs.k8s.io/user/quick-start) sobre como
instalar e testar o CAPI.

Instale o `clusterctl` no nó de deploy.

```bash
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.11.1/clusterctl-linux-amd64 -o clusterctl
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
```

Para verificar que o clusterctl está na última versão rode

```bash
clusterctl version
```

Antes de inicializar o `clusterctl` para o provider Openstack, exporte essa
variável.

```bash
export CLUSTER_TOPOLOGY=true
```

Agora, inicialize o provider do Openstack

```bash
# Install ORC (needed for CAPO >=v0.12)
kubectl apply -f https://github.com/k-orc/openstack-resource-controller/releases/latest/download/install.yaml
# Initialize the management cluster
clusterctl init --infrastructure openstack
```

## Configurando o Magnum

### Deploy do Magnum usando `kolla`

Edite o arquivo `globals.yaml` em `/etc/kolla/`.

```yaml
# globals.yaml
enable_magnum: true
enable_cluster_user_trust: true
```

```bash
kolla-ansible deploy -i <inventory> --tags common,horizon,magnum
```

Agora, precisamos configurar o Magnum para utilizar o Cluster API para criar os clusters.

```bash
# Crie o diretório de configuração do Magnum
mkdir -p /etc/kolla/config/magnum
# Copie o k3s.yaml como arquivo de kubeconfig para o Magnum utilizar
cp k3s.yaml /etc/kolla/config/magnum/kubeconfig
```

Reconfigure o Magnum para utilizar desse arquivo e se autenticar com o cluster
de gerência do Cluster API.

```bash
kolla-ansible reconfigure -i <inventory> --tags magnum
```

### Download de imagem compatível com o CAPI

É necessário ter pelo menos uma imagem compatível com o Cluster API no Openstack.

Para isso, teremos que baixar a imagem e subí-la no Openstack.

A seguir, baixamos uma imagem compatível Ubuntu 22.04 e a enviamos ao Openstack.
A versão do Kubernetes utilizada aqui é a `v1.27.4`.

```bash
export OS_DISTRO=ubuntu # you can change this to "flatcar" if you want to use Flatcar
for version in v1.27.4; do \
  [[ "${OS_DISTRO}" == "ubuntu" ]] && IMAGE_NAME="ubuntu-2204-kube-${version}" || IMAGE_NAME="flatcar-kube-${version}"; \
  curl -LO https://object-storage.public.mtl1.vexxhost.net/swift/v1/a91f106f55e64246babde7402c21b87a/magnum-capi/${IMAGE_NAME}.qcow2; \
  openstack image create ${IMAGE_NAME} --disk-format=qcow2 --container-format=bare --property os_distro=${OS_DISTRO} --file=${IMAGE_NAME}.qcow2;
done;
```

## Criação de template

> Requisitos:
>
> 1. Carregar as variáveis de ambiente de admin do Openstack
> 2. Ter o client do Magnum baixado no ambiente (`pip install python-magnumclient`)
> 3. Para a criação de um template do Magnum, é necessário ter uma rede
>    externa, aqui como `external`

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

## Criando um cluster

```bash
openstack coe cluster create \
  --cluster-template k8s-v1.27.4 \
  --master-count 1 \
  --node-count 1 \
  k8s-v1.27.4
```

Esse comando cria um cluster de nome `k8s-v1.27.4` com 1 nó de controle e 1 nó
de trabalho.

## Conclusão

Com isso, temos um cluster k8s funcionando e acessível. Podemos então expor
esse template cluster como público para os usuários da cloud poderem, de forma
transparente, criar clusters baseados nos templates definidos.

## Referências

* <https://cluster-api.sigs.k8s.io/>
* <https://www.roksblog.de/openstack-magnum-cluster-api-driver/>
* <https://vexxhost.github.io/magnum-cluster-api/user/getting-started/>
* <https://vexxhost.github.io/magnum-cluster-api/admin/troubleshooting/>
* <https://www.stackhpc.com/magnum-clusterapi.html>
* <https://www.stackhpc.com/magnum-cluster-api-helm-deep-dive.html>
* <https://satishdotpatel.github.io/openstack-magnum-capi/>
