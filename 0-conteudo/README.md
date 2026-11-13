# Conteúdo de estudo

Material teórico do repositório, escrito em português. Cada arquivo tem um tema curto e cita a fonte oficial; o que não vem da documentação está marcado como "de memória".

[← Voltar à prateleira](../README.md)

## DevOps e CI/CD

| Arquivo | Assunto |
|---|---|
| [CI/CD: fundamentos](devops-cicd/ci-cd-fundamentos.md) | Integração, entrega e implantação contínuas, caminho do artefato, branch por ambiente e feature flags |
| [Estratégias de deploy](devops-cicd/estrategias-de-deploy.md) | Recreate, Rolling, Blue/Green, Canary e Canary x feature flag |
| [Métricas DORA](devops-cicd/metricas-dora.md) | As cinco métricas, throughput e instabilidade |
| [Segurança no pipeline](devops-cicd/seguranca-no-pipeline.md) | Secret detection, SCA, SAST, container scan e DAST, e onde cada um roda |

## Docker

| Arquivo | Assunto |
|---|---|
| [Fundamentos](docker/docker-fundamentos.md) | Container x VM, imagem x container, Dockerfile, multi-stage, registry e tags |
| [Arquitetura](docker/docker-arquitetura.md) | Client, daemon, registry, plugins e o passo a passo do `docker run` |
| [Instruções do Dockerfile](docker/docker-instrucoes-dockerfile.md) | FROM, WORKDIR, COPY, RUN, CMD, ENTRYPOINT, ENV, USER e as demais |
| [Camadas e cache](docker/docker-camadas-e-cache.md) | Camadas, cache de build, ordem das instruções, `.dockerignore` e cache mount |
| [Volumes e redes](docker/docker-volumes-e-redes.md) | Persistência, volume, bind mount, redes e publicação de porta |
| [Configuração, Compose e boas práticas](docker/docker-compose-e-boas-praticas.md) | Variáveis de ambiente, secrets, Docker Compose e boas práticas |
| [Limites de recurso](docker/docker-limites-de-recurso.md) | Memória, CPU e cgroups |
| [Segurança](docker/docker-seguranca.md) | Namespaces, cgroups, capabilities, rootless e execução endurecida |

## Kubernetes

Comece pela [visão geral](kubernetes/kubernetes-visao-geral.md), que é o índice e o mapa do que foi coberto.

| Arquivo | Assunto |
|---|---|
| [Visão geral e índice](kubernetes/kubernetes-visao-geral.md) | O que é, o que não é, do menor para o maior e mapa da documentação |
| [Pod](kubernetes/kubernetes-pod.md) | A menor unidade do Kubernetes |
| [Node, cluster e componentes](kubernetes/kubernetes-node-e-cluster.md) | Control plane, kubelet, kube-proxy e o que acontece num `kubectl apply` |
| [Objetos, controllers e imperativo x declarativo](kubernetes/kubernetes-objetos-e-controllers.md) | Spec, status e o laço de controle |
| [Workloads](kubernetes/kubernetes-workloads.md) | ReplicaSet, Deployment, StatefulSet, DaemonSet, Job e CronJob |
| [Service](kubernetes/kubernetes-service.md) | ClusterIP, NodePort, LoadBalancer, ExternalName e Headless |
| [Ingress e Ingress Controller](kubernetes/kubernetes-ingress.md) | Entrada HTTP no cluster |
| [ConfigMap e Secret](kubernetes/kubernetes-configmap-e-secret.md) | Configuração e segredos do Pod |
| [Storage](kubernetes/kubernetes-storage.md) | Volumes, PV, PVC e StorageClass |
| [Probes e resources](kubernetes/kubernetes-probes-e-resources.md) | Liveness, readiness, startup, requests, limits e QoS |
| [Namespaces e kubectl](kubernetes/kubernetes-namespaces-e-kubectl.md) | Separação lógica e os comandos do dia a dia |
| [HPA e Helm](kubernetes/kubernetes-hpa-e-helm.md) | Escala automática e gerenciador de pacotes |
| [Segurança básica](kubernetes/kubernetes-seguranca-basica.md) | RBAC e NetworkPolicy |
| [Arquitetura de clusters por ambiente](kubernetes/kubernetes-arquitetura-de-ambientes.md) | 1, 2, 3 ou mais clusters para dev, qa, hml, produção e review apps |

## Servidores

| Arquivo | Assunto |
|---|---|
| [Linux: usuários e permissões](servidores/linux-usuarios-e-permissoes.md) | UID/GID, rwx, `chmod` e o alinhamento com Docker e Kubernetes |
| [Linux: processos, serviços, rede, SSH e logs](servidores/linux-processos-servicos-e-rede.md) | Processos, systemd, IP, porta, DNS, firewall, SSH e um exemplo com três aplicações |
| [Windows Server](servidores/windows-server.md) | Server Core, IIS, Windows containers e links para pesquisa |

## GitLab CI/CD

| Arquivo | Assunto |
|---|---|
| [Fundamentos e palavras-chave](gitlab-pipelines/gitlab-ci-fundamentos.md) | Pipeline, stage, job, runner e as palavras-chave com exemplos |
| [Artifacts, cache, variáveis e segredos](gitlab-pipelines/gitlab-artifacts-cache-variaveis.md) | Reports, `expire_in`, cache distribuído, precedência de variáveis e cofres externos |
| [Reaproveitamento](gitlab-pipelines/gitlab-reaproveitamento.md) | `include`, `extends`, âncoras, `!reference` e componentes |
| [Arquiteturas de pipeline](gitlab-pipelines/gitlab-arquiteturas-de-pipeline.md) | Básica, DAG, pai-filho, multi-projeto e neto |
| [Rules e condicionais](gitlab-pipelines/gitlab-rules-e-condicionais.md) | `rules`, `if`, `workflow` e pipelines de merge request |
| [Ambientes e recursos extras](gitlab-pipelines/gitlab-ambientes-e-recursos-extras.md) | Environments, review apps, Auto DevOps, Pages, feature flags e agente do Kubernetes |

## GitHub Actions

| Arquivo | Assunto |
|---|---|
| [O que é e os conceitos](github-workflows/github-actions-conceitos.md) | Workflow, evento, job, step, action e runner |
| [O arquivo e os gatilhos `on:`](github-workflows/github-actions-arquivo-e-gatilhos.md) | `.github/workflows`, filtros, schedule e `workflow_dispatch` |
| [Jobs, steps e actions](github-workflows/github-actions-jobs-steps-e-actions.md) | `needs`, matriz, `services`, `uses` e workflow reutilizável |
| [Variáveis, secrets e environments](github-workflows/github-actions-variaveis-secrets-e-environments.md) | `env`, `vars`, `secrets` e regras de proteção |
| [Artifacts, cache e reaproveitamento](github-workflows/github-actions-artifacts-cache-e-reaproveitamento.md) | `upload-artifact`, `actions/cache`, `workflow_call` e action composta |
| [Equivalência com o GitLab](github-workflows/github-actions-equivalencia-com-gitlab.md) | Tabela GitLab × GitHub |
