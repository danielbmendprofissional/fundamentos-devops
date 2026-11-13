# Fundamentos DevOps

Laboratório de estudos de **DevOps**: CI/CD, Docker, Kubernetes, servidores Linux e Windows, pipelines no GitLab e workflows no GitHub, com material teórico em português ancorado na documentação oficial.

> Este repositório existe em dois lugares: o **GitLab** é a fonte (aqui entram commits e issues) e o **GitHub** é um
> espelho somente leitura, atualizado sozinho a cada push. Visão geral do conjunto na
> [vitrine fundamentos](https://github.com/danielbmendprofissional/fundamentos).

## Sobre

- **Um repositório só.** O conteúdo vive junto, e os links entre os arquivos são relativos.
- **Fonte oficial primeiro.** Os conceitos vêm da documentação oficial, com link; o que é de memória está dito como tal.
- **Exemplos reais.** Os exemplos usam uma API .NET (imagem, Compose, pipeline, manifests) para o conceito não ficar abstrato.

## Conteúdo

A teoria fica em [0-conteudo](0-conteudo/README.md), organizada por tema:

| Tema | O que cobre | Arquivos |
|---|---|---|
| [DevOps e CI/CD](0-conteudo/README.md#devops-e-cicd) | Entrega contínua, estratégias de deploy, métricas DORA, segurança no pipeline | 4 |
| [Docker](0-conteudo/README.md#docker) | Arquitetura, Dockerfile, camadas e cache, volumes e redes, Compose, limites, segurança | 8 |
| [Kubernetes](0-conteudo/README.md#kubernetes) | Pod ao cluster, workloads, Service, Ingress, storage, probes, RBAC, arquitetura por ambiente | 14 |
| [Servidores](0-conteudo/README.md#servidores) | Linux (usuários, permissões, rede, SSH) e Windows Server por alto | 3 |
| [GitLab CI/CD](0-conteudo/README.md#gitlab-cicd) | Palavras-chave, artifacts, reaproveitamento, arquiteturas, rules, ambientes | 6 |
| [GitHub Actions](0-conteudo/README.md#github-actions) | Workflows, gatilhos, jobs, secrets, artifacts e a equivalência com o GitLab | 6 |

### Ordem sugerida de estudo

1. [DevOps e CI/CD](0-conteudo/README.md#devops-e-cicd): o que se quer entregar e como medir.
2. [Docker](0-conteudo/README.md#docker): empacotar a aplicação.
3. [Kubernetes](0-conteudo/README.md#kubernetes): rodar os containers em escala. Comece pela
   [visão geral](0-conteudo/kubernetes/kubernetes-visao-geral.md).
4. [Servidores](0-conteudo/README.md#servidores): o sistema por baixo dos containers.
5. [GitLab CI/CD](0-conteudo/README.md#gitlab-cicd): automatizar o caminho do código até o ambiente.
6. [GitHub Actions](0-conteudo/README.md#github-actions): o mesmo caminho na outra plataforma.

## Estrutura

```text
0-conteudo/
├── devops-cicd/         # CI/CD, estratégias de deploy, DORA, segurança no pipeline
├── docker/
├── kubernetes/
├── servidores/          # Linux e Windows Server
├── gitlab-pipelines/
└── github-workflows/
```
