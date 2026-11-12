# Kubernetes: visão geral e índice

## O que é

Plataforma portável, extensível e open source para gerenciar cargas de trabalho e serviços em containers, com configuração declarativa e automação. Os controllers fazem o estado atual caminhar para o estado desejado (ver [kubernetes-objetos-e-controllers.md](kubernetes-objetos-e-controllers.md)).

## O que o Kubernetes oferece (doc, 10 itens)

1. Descoberta de serviços e balanceamento de carga (Service discovery and load balancing)
2. Orquestração de armazenamento (Storage orchestration)
3. Rollouts e rollbacks automáticos
4. Empacotamento automático (bin packing): encaixa os containers nos nodes conforme os recursos
5. Autocorreção (self-healing)
6. Gerenciamento de segredos e configuração
7. Execução em lote (batch)
8. Escala horizontal
9. IPv4/IPv6 dual-stack
10. Projetado para ser extensível

## O que o Kubernetes não é (doc)

- Não é uma PaaS tradicional e completa.
- Não limita os tipos de aplicação.
- Não faz deploy de código-fonte nem builda aplicações (o build é do pipeline).
- Não traz serviços de aplicação prontos (middleware, bancos, caches).
- Não dita como fazer logging, monitoramento e alertas.
- Não obriga uma linguagem de configuração.
- Não é só um sistema de orquestração: tem processos de controle que levam o estado atual ao desejado.

## Do menor para o maior

```text
Container  ->  Pod  ->  Node (servidor/VM)  ->  Cluster (control plane + worker nodes)
```

## Índice do que foi coberto

| Assunto | Arquivo |
|---|---|
| Pod | [kubernetes-pod.md](kubernetes-pod.md) |
| Node, Cluster e componentes | [kubernetes-node-e-cluster.md](kubernetes-node-e-cluster.md) |
| Objetos, controllers, imperativo x declarativo | [kubernetes-objetos-e-controllers.md](kubernetes-objetos-e-controllers.md) |
| ReplicaSet, Deployment, ReplicationController, StatefulSet, DaemonSet, Job, CronJob | [kubernetes-workloads.md](kubernetes-workloads.md) |
| Service | [kubernetes-service.md](kubernetes-service.md) |
| Ingress e Ingress controller | [kubernetes-ingress.md](kubernetes-ingress.md) |
| ConfigMap e Secret | [kubernetes-configmap-e-secret.md](kubernetes-configmap-e-secret.md) |
| Storage (Volume, PV, PVC, StorageClass) e Kubernetes x Compose | [kubernetes-storage.md](kubernetes-storage.md) |
| Probes e resources | [kubernetes-probes-e-resources.md](kubernetes-probes-e-resources.md) |
| Namespaces e kubectl | [kubernetes-namespaces-e-kubectl.md](kubernetes-namespaces-e-kubectl.md) |
| HPA e Helm | [kubernetes-hpa-e-helm.md](kubernetes-hpa-e-helm.md) |
| RBAC e NetworkPolicy | [kubernetes-seguranca-basica.md](kubernetes-seguranca-basica.md) |

## Mapa da documentação (14 seções) e o que foi coberto

| # | Seção da doc | Situação |
|---|---|---|
| 1 | Overview | coberto (este arquivo e os componentes) |
| 2 | Cluster Architecture | coberto: nodes e componentes; controllers |
| 3 | Containers | parcial: resources e probes |
| 4 | Workloads | coberto por completo (os 7) |
| 5 | Services, Load Balancing and Networking | essencial: Service, Ingress, NetworkPolicy |
| 6 | Storage | essencial: Volume, PV, PVC, StorageClass |
| 7 | Configuration | essencial: ConfigMap e Secret |
| 8 | Security | essencial: RBAC e NetworkPolicy |
| 9 | Policies | fora (resource quotas e limit ranges) |
| 10 | Scheduling, Preemption and Eviction | fora (taints, tolerations, affinity) |
| 11 | Resource Management | parcial: requests e limits |
| 12 | Cluster Administration | fora |
| 13 | Windows in Kubernetes | fora |
| 14 | Extending Kubernetes | fora (CRDs, operators) |

## Tópicos não cobertos

- Rede: Gateway API, EndpointSlices, DNS for Services and Pods em detalhe, dual-stack, Topology Aware Routing, Networking on Windows, Service ClusterIP allocation, Service Internal Traffic Policy.
- Pod em detalhe: multi-container, init containers, static Pods, ciclo de vida.
- Workloads extras citados pela doc: recursos de terceiros via CRD e a Workload API (PodGroup, Beta).
- Volumes: o restante da lista de tipos (nfs, iscsi, fc, csi, local, projected, downwardAPI, image e os depreciados).
- Segurança: Pod Security Standards, ServiceAccount em detalhes, auditoria, criptografia em repouso (apenas citada).
