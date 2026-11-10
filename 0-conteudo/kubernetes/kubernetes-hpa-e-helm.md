# Kubernetes: HPA e Helm

## HPA (HorizontalPodAutoscaler)

Faz a escala horizontal automaticamente: ajusta o número de réplicas de um Deployment (ou StatefulSet) conforme a carga, em vez de alguém rodar `kubectl scale`. Não se aplica a DaemonSet (um Pod por node).

É mais um controller: o mesmo laço "desejado x real", rodando no control plane a cada 15 segundos (padrão da doc).

### Dois pré-requisitos

1. **Metrics Server** instalado no cluster: é ele que fornece o uso de CPU e memória. Sem ele, o HPA não lê nada.
2. **`requests` definido** nos containers: o HPA calcula "50% de CPU" em cima do request (ver [kubernetes-probes-e-resources.md](kubernetes-probes-e-resources.md)). Sem request, ele não age.

### A conta (fórmula da doc)

```text
réplicas = ceil( réplicas atuais × (uso atual / uso alvo) )

Exemplo: 2 réplicas, uso médio 80%, alvo 50%
         ceil( 2 × 80/50 ) = 4 réplicas
```

### Exemplo (da doc, adaptado por mim): a API

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
spec:
  scaleTargetRef:                 # quem será escalado
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2                  # nunca menos que 2
  maxReplicas: 10                 # nunca mais que 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50    # mantém a CPU média em 50% do request
```

```bash
kubectl autoscale deployment api --cpu-percent=50 --min=2 --max=10   # jeito imperativo
kubectl get hpa                                                      # alvo, métrica atual, réplicas
kubectl describe hpa api                                             # eventos e decisões
```

### Estabilização ao reduzir

Para não ficar subindo e descendo (flapping), na redução o HPA usa uma janela de 300 segundos (padrão); ao subir, o padrão é 0. Ajuste em `behavior.scaleDown.stabilizationWindowSeconds`.

### HPA x VPA

O HPA muda o número de Pods. O VPA (Vertical Pod Autoscaler) muda os recursos de cada Pod (CPU e memória). Usar os dois sobre a mesma métrica pode causar conflito, segundo a doc.

Fonte: [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/). A página veio cortada depois dos primeiros 100 mil caracteres.

## Helm

Gerenciador de pacotes do Kubernetes. Resolve o problema de subir uma aplicação com muitos recursos (Deployment, Service, ConfigMap, Secret, Ingress, HPA, PVC...), repetidos para dev, hml e prod com valores diferentes. O Helm agrupa tudo num pacote e deixa mudar só os valores.

| Conceito | O que é | Analogia (minha) |
|---|---|---|
| **Chart** | o pacote que descreve uma aplicação e seus recursos | o instalador |
| **Repository** | onde os charts ficam para busca e download | a loja de instaladores |
| **Release** | uma instalação de um chart no cluster, com nome e número de revisão que sobe a cada install, upgrade ou rollback | o programa instalado |

O mesmo chart pode virar várias releases (a API em dev e a API em prod).

### Comandos essenciais (doc)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami   # adiciona um repositório (exemplo meu)
helm search repo postgresql                                 # busca nos repositórios adicionados
helm install meu-banco bitnami/postgresql                   # instala: cria a release "meu-banco"
helm list                                                   # releases instaladas
helm upgrade meu-banco bitnami/postgresql -f values.yaml    # atualiza com novos valores
helm rollback meu-banco 1                                   # volta à revisão 1
helm uninstall meu-banco                                    # remove
```

### Valores

O `-f values.yaml` entrega um arquivo com os valores. O `--set` define direto na linha de comando e tem precedência sobre o arquivo.

Exemplo (meu): a mesma API em dois ambientes.

```bash
helm install api-dev  ./charts/api -f values-dev.yaml  -n dev
helm install api-prod ./charts/api -f values-prod.yaml -n prod --set image.tag=abc123
```

```yaml
# values-prod.yaml
replicaCount: 5
image:
  repository: registro/minha-api
ingress:
  host: galo.exemplo.com
```

De memória (a página consultada não detalha): o Helm pega os templates do chart, substitui os valores e aplica no cluster, atualizando só o que mudou. A estrutura interna de um chart (pasta `templates/`, `Chart.yaml`, `values.yaml`) também é de memória.

Helm é um dos itens que mais aparecem em vaga de Kubernetes e DevOps. Casa com o pipeline: ele roda `helm upgrade` com a tag do commit, em vez de editar YAML.

Fonte: [Helm, using Helm](https://helm.sh/docs/intro/using_helm/).
