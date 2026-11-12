# Kubernetes: Node, Cluster e componentes

## Node

É onde os Pods rodam. O node é o servidor: uma VM ou uma máquina física.

## Worker node

É um node. "Worker" é só o papel dele: rodar os Pods da aplicação. Não é uma super máquina.

## Control plane

É o que decide e gerencia todo o esquema do cluster. Também roda em máquinas; normalmente são separadas das workers (de memória, não está na doc consultada).

## Cluster

É o nome de tudo isso: o control plane mais os worker nodes.

```text
CLUSTER
 ├─ CONTROL PLANE   (gerencia o estado do cluster)
 └─ WORKER NODES
      ├─ node 1 (VM) ── Pod api, Pod api
      ├─ node 2 (VM) ── Pod api, Pod worker
      └─ node 3 (VM) ── Pod rabbitmq
```

## Componentes do control plane

1. **kube-apiserver:** é a API do Kubernetes, onde o `kubectl` fala com ele. É a única porta de entrada do cluster: o `kubectl`, o scheduler, os controllers e os kubelets conversam com ele, e ninguém mexe no etcd direto (isso é de memória; a doc só diz "exposes the Kubernetes HTTP API").
2. **etcd:** é o banco de dados do cluster.
3. **kube-scheduler:** é quem decide em qual node o Pod roda. "Schedule" aqui não é agendar no tempo, é alocar o Pod em um node. Pelo que li na doc, ele filtra os nodes que servem (têm CPU e memória livres) e depois pontua os que sobraram.
4. **kube-controller-manager:** roda os controllers. Um controller compara o que foi pedido com o que existe e corrige a diferença. Exemplo: pedi 3 Pods da API, um node caiu e restaram 2; o controller manda criar mais 1. Os controllers serão vistos em detalhe adiante.
5. **cloud-controller-manager (opcional):** faz o cluster conversar com o provedor de nuvem (por exemplo, pedir um balanceador de carga da nuvem; exemplo de memória). É opcional porque um cluster local não tem nuvem por trás.

## Componentes do node

1. **kubelet:** garante que o Pod esteja rodando com seus containers. Cuida só dos Pods atribuídos ao seu node e fala com o apiserver (esse detalhe é de memória).
2. **kube-proxy (opcional):** o IP de um Pod muda quando ele é recriado, então o Kubernetes cria um endereço fixo na frente dos Pods (o Service, visto adiante). O kube-proxy, em cada node, mantém as regras de rede para esse endereço fixo chegar a um Pod que existe. É opcional porque alguns plugins de rede fazem isso sozinhos (de memória).
3. **Container runtime:** é ele que é responsável por rodar os containers.

## Addons

Ampliam o Kubernetes: DNS, Web UI (Dashboard), Container Resource Monitoring e Cluster-level Logging.

## Exemplo: o que acontece num `kubectl apply -f pod.yaml`

```text
kubectl ──> kube-apiserver ──> grava no etcd ("existe um Pod sem node")
              kube-scheduler ──> escolhe o node 2
              kubelet (node 2) ──> manda o container runtime subir o container
```

Fonte: [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/).
