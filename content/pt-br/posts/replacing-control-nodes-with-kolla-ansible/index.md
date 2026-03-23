---
title: Substituindo nós de controle com Kolla Ansible
date: 2026-03-04
toc: true
summary: Como adicionar e remover nós de controle numa cloud Openstack utilizando `kolla-ansible`
tags: [openstack, cloud, kolla-ansible]
readTime: true
---

Manter uma nuvem OpenStack em produção exige cuidados constantes, e uma das
tarefas que pode surgir é a substituição ou adição de novos nós de controle.
Seja por expansão, falha de hardware ou atualização de infraestrutura, o
processo precisa ser seguro e bem planejado para evitar indisponibilidade.

Passamos por essa experiência em nosso ambiente, utilizando o [**kolla-ansible**](https://opendev.org/openstack/kolla-ansible)
como ferramenta de deploy. Seguimos a [documentação
oficial](https://docs.openstack.org/kolla-ansible/latest/user/adding-and-removing-hosts.html),
mas como todo ambiente real, encontramos alguns desafios. Neste post,
compartilho o passo a passo que executamos, as verificações realizadas e um
problema inesperado com o RabbitMQ – e como resolvemos. Além disso, apresento o
processo completo de remoção de nós de controle, baseado na documentação e em
lições aprendidas.

---

## Adicionando um novo nó de controle

### Pré-requisitos

O novo nó de controle precisa estar devidamente preparado. Em nosso caso, todos
os servidores utilizam **bond** para alta disponibilidade das interfaces de
rede. Por isso, foi necessário:

- Instalar **Ubuntu 22.04** na máquina.
- Configurar um bond (com duas interfaces) e, sobre ele, criar **duas VLANs**:
  - Uma para a comunicação interna dos serviços da nuvem.
  - Outra para a comunicação externa (serviços externos, API).
- Atribuir um IP a cada VLAN e garantir que o novo nó fosse acessível por ambos
os endereços.

Essa etapa é importante pois o kolla-ansible utiliza esses IPs para configurar
os serviços e a comunicação entre os nós.

### Adicionando o nó ao cluster com Kolla-Ansible

Utilizamos um arquivo `multinode` com a lista de todos os nós (incluindo o
novo). A adição foi feita em etapas, sempre limitando a execução ao grupo
`control` (ou ao novo host especificamente) .

```bash
# Prepara o nó: instala dependências, configura repositórios, etc.
kolla-ansible bootstrap-servers -i multinode --limit control

# Baixa as imagens dos containers para o novo nó
kolla-ansible pull -i multinode --limit <novo-host>

# Executa o deploy apenas nos nós de controle (atualiza a configuração)
kolla-ansible deploy -i multinode --limit control
```

Após o deploy, o novo nó já deveria estar participando do cluster. Mas, é
preciso verificar.

### Verificações pós-deploy

Validamos os principais serviços que rodam nos nós de controle: MariaDB
(Galera), RabbitMQ, Neutron e Nova.

#### **MariaDB (Galera)**

O cluster MariaDB (Galera) precisa estar sincronizado e com tamanho correto.

```bash
# Obtém a senha root do MariaDB a partir do arquivo de configuração
ROOT_PASS=$(cat /etc/kolla/mariadb/galera.cnf | grep wsrep_sst_auth | cut -f 2 -d:)

# Verifica o número de nós no cluster
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_cluster_size';"

# Confere se o cluster está primário (aceita leitura/escrita)
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_cluster_status';"

# Verifica se o nó atual está sincronizado (synced)
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"

# Lista os endereços dos membros do cluster
docker exec -t mariadb mysql --user root --password=$ROOT_PASS -e "SHOW STATUS LIKE 'wsrep_incoming_addresses';"
```

#### **RabbitMQ**

O RabbitMQ é o barramento de mensagens do OpenStack. Verificamos o status do
cluster e a conectividade:

```bash
# Status do cluster RabbitMQ
docker exec -it rabbitmq rabbitmqctl cluster_status

# Testa conectividade entre as portas de listeners
docker exec -it rabbitmq rabbitmq-diagnostics check_port_connectivity
```

O novo nó aparecia no cluster e a conectividade estava ok.

#### **Neutron e Nova**

Por fim, checamos se os agentes e serviços de computação estavam presentes:

```bash
openstack network agent list --host <novo-host>
openstack compute service list --host <novo-host>
```

Até aí, tudo indicava que a adição havia sido um sucesso. Porém, ao tentar
criar uma instância, o erro apareceu.

### Erro: "missing queue" no RabbitMQ

Ao criar uma VM, a operação falhou com a seguinte mensagem no log do `nova-conductor`:

```
The reply XXXX failed to send after 60 seconds due to a missing queue
oslo_messaging.exceptions.MessageUndeliverable
```

Analisando mais a fundo, percebemos que o RabbitMQ estava em HA (alta
disponibilidade), mas as **filas** não haviam sido criadas no novo nó. As filas
existentes nos outros nós de controle não foram replicadas automaticamente para
o novo membro do cluster. Quando o `nova-conductor` tentou enviar uma mensagem
para uma fila que só existia nos nós antigos, o novo nó não conseguiu
entregá-la, resultando no erro.

### Solução: restart do RabbitMQ em todos os nós

Após alguma investigação, a solução encontrada foi reiniciar o serviço RabbitMQ
**em todos os nós de controle**. Isso forçou a reconciliação das filas e a
sincronização completa do cluster. Para isso, é necessário realizar um
rolling-restart para garantir que um próximo container em outro nó só possa ser
reiniciando quando o anterior estiver funcional.

```bash
# Reinicie o RabbitMQ no primeiro nó
docker restart rabbitmq

# Verifique se o container fica HEALTHY
watch 'docker ps -a --filter name=rabbitmq'
```

Quando o RabbitMQ estiver ok no primeiro nó, repita o mesmo procedimento pros próximos.

Após o restart, o cluster recriou as filas e a criação de VMs voltou a
funcionar normalmente.

---

## Removendo um nó de controle

Assim como a adição, a remoção de um nó de controle precisa ser feita com
cuidado para não afetar a disponibilidade do cluster. A documentação oficial do
Kolla-Ansible  recomenda um processo em etapas, que inclui realocar recursos,
parar serviços e limpar os registros.

> [!WARNING]
> Antes de remover qualquer nó, verifique se os nós restantes
> ainda têm quorum suficiente. Em um cluster com 3 controladores, remova apenas
> um por vez.

### Passo 1: Mover recursos do Neutron

Antes de remover o nó, é necessário realocar os roteadores e redes DHCP
gerenciados pelos agentes L3 e DHCP que estão no host a ser removido.

**Para agentes L3 (roteadores):**

```bash
# Identifica o ID do agente L3 no host a ser removido
l3_id=$(openstack network agent list --host <host> --agent-type l3 -f value -c ID)

# Identifica o ID de um agente L3 em outro nó (target)
target_l3_id=$(openstack network agent list --host <target-host> --agent-type l3 -f value -c ID)

# Move cada roteador do agente antigo para o novo
openstack router list --agent $l3_id -f value -c ID | while read router; do
  openstack network agent remove router $l3_id $router --l3
  openstack network agent add router $target_l3_id $router --l3
done

# Desabilita o agente L3 antigo
openstack network agent set $l3_id --disable
```

**Para agentes DHCP:**

```bash
# Identifica o ID do agente DHCP no host a ser removido
dhcp_id=$(openstack network agent list --host <host> --agent-type dhcp -f value -c ID)

# Identifica o ID de um agente DHCP em outro nó (target)
target_dhcp_id=$(openstack network agent list --host <target-host> --agent-type dhcp -f value -c ID)

# Move cada rede do agente antigo para o novo
openstack network list --agent $dhcp_id -f value -c ID | while read network; do
  openstack network agent remove network $dhcp_id $network --dhcp
  openstack network agent add network $target_dhcp_id $network --dhcp
done
```

### Passo 2: Mover os recursos do Cinder

> [!WARNING]
> Nós utilizamos Ceph externo para as pools do Cinder. Por conta disso, a
> solução a seguir atualiza logicamente (internamente no banco de dados) o
> atributo de `host` dos volumes, o que não interfere nas pools do Cinder pois
> estão fora do host (no Ceph). Se você usa qualquer backend interno, como o
> LVM, essa solução pode não funcionar.

Para atualizar a referência de host os volumes, acesse o host que terá os
volumes migrados:

```bash
docker exec -it -u0 cinder_volume cinder-manage volume update_host --currenthost <old_host> --newhost <new_host>
```

Para atualizar a referência sobre a coluna service_uuid no banco de dados do
cinder, é necessário rodar o seguinte comando, a partir do host para onde os
volume foram atualizados (`<new_host>`, nos comandos acima):

```bash
docker exec -it -u0 cinder_volume cinder-manage volume update_service
```

Para confirmar, liste os volumes associados ao backend do host a ser removido:

```bash
cinder list --filters host=<host> --all-tenants 1
```

Depois de confirmado que não há nenhum volume associado do backend do host a
ser removido, o próximo passo é desabilitar o serviço do cinder-volume. Assim,
determine o binário e o host a ser removido:

```bash
openstack volume service list
```

Depois desabilite o serviço:

```bash
openstack volume service set --disable HOST_NAME BINARY_NAME
```

### Passo 3: Parar os serviços no host removido

Com os recursos realocados, podemos parar todos os containers no host que será removido:

```bash
kolla-ansible stop -i multinode --yes-i-really-really-mean-it --limit <host>
```

### Passo 4: Remover o host do inventário Ansible

Edite seu arquivo de inventário (`multinode`) e remova ou comente as linhas
referentes ao host que está sendo retirado.

### Passo 5: Reconfigurar os controladores restantes

#### **Um cuidado extra com o RabbitMQ**

Assim como na adição, o RabbitMQ merece atenção especial. Há
[indícios](https://bugs.launchpad.net/kayobe/+bug/2085608)  de que, mesmo após
a remoção do nó do inventário e a reconfiguração, o RabbitMQ pode continuar
listando o nó removido no cluster. Isso pode causar problemas de conectividade.

Verifique se o host a ser removido ainda é listado:

```bash
docker exec -it rabbitmq rabbitmqctl cluster_status
```

Se ainda for listado de alguma forma (no nosso caso o host ainda aparecia em
`Disk Nodes`), remova o host:

```bash
docker exec -it rabbitmq rabbitmqctl forget_cluster_node <node-name>
```

> [!TIP]
> `node-name` aqui geralmente é `rabbit@<host>`.

Verifique novamente se o host foi de fato removido, pra então prosseguir.

#### **Deploy**

Agora é necessário reconfigurar os nós de controle restantes para que eles
atualizem o estado dos clusters (MariaDB, RabbitMQ, etc.).

```bash
kolla-ansible deploy -i multinode --limit control
```

### Passo 6: Limpar os serviços do host removido

Remova os registros dos agentes e serviços do OpenStack que ainda
possam estar visíveis:

```bash
# Remove agentes de rede
openstack network agent list --host <host> -f value -c ID | while read id; do
  openstack network agent delete $id
done

# Remove serviços de compute
openstack compute service list --os-compute-api-version 2.53 --host <host> -f value -c ID | while read id; do
  openstack compute service delete --os-compute-api-version 2.53 $id
done
```

Liste os serviços do Cinder a serem removidos:

```bash
openstack volume service list --host <host>
```

Entre em um dos nós de storage (que contém o serviço `cinder-volume`) e rode:

```bash
cinder-manage service remove BINARY_NAME HOST_NAME
```

## Conclusão

A substituição de nós de controle em um ambiente Openstack com kolla é um
processo bem documentado, mas nem sempre é livre de surpresas. A principal
lição que tiramos foi: **sempre verifique o estado das filas do RabbitMQ após
adicionar um novo nó** e, ao remover, **certifique-se de que o nó foi
completamente esquecido pelo cluster de mensageria**.

Uma simples listagem de filas (`rabbitmqctl list_queues`) ou verificação do
cluster (`rabbitmqctl cluster_status`) antes e depois poderia ter indicado o
problema mais cedo.

Felizmente, as soluções foram simples (embora envolvam reinicialização de
serviços críticos). Em ambientes de produção, é recomendável planejar uma
janela de manutenção para esse tipo de operação.

No fim, todo o processo foi bem tranquilo do que pensávamos, e isso realmente mostra o quão fascinante é o projeto [kolla-ansible](https://opendev.org/openstack/kolla-ansible).
