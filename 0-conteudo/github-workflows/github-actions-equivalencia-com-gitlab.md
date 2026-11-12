# GitHub Actions × GitLab CI/CD: equivalência

Resumo dos arquivos deste tema. As linhas marcadas com (m) são de memória; as demais vêm da doc lida em cada arquivo.

## Estrutura e execução

| GitLab | GitHub |
|---|---|
| `.gitlab-ci.yml` (um arquivo) | `.github/workflows/*.yml` (um por workflow) |
| pipeline | workflow (cada execução é uma run) |
| stage | não existe; a ordem vem de `needs` |
| job | job |
| `script:` | `steps` com `run` |
| runner | runner (GitHub-hosted ou self-hosted) |
| `tags:` do job | `runs-on:` com labels |
| `image:` do job | `container:` |
| `services:` | `services:` |
| `parallel:matrix` | `strategy.matrix` |
| `timeout:` | `timeout-minutes:` |
| `allow_failure: true` | `continue-on-error: true` |
| `resource_group` | `concurrency` (m) |

## Gatilhos e condições

| GitLab | GitHub |
|---|---|
| `rules:` por job | `on:` no workflow inteiro, mais `if:` por job ou step |
| `$CI_PIPELINE_SOURCE == "schedule"` | `on: schedule` |
| `$CI_COMMIT_TAG` | `on: push: tags:` |
| `rules: changes:` | `paths:` / `paths-ignore:` |
| pipeline de MR | `pull_request` |
| "Run pipeline" manual com variáveis | `workflow_dispatch` com `inputs` |
| `trigger:` para outro projeto | `repository_dispatch` ou chamada de API (m) |
| `workflow: rules` | filtros no `on:` |

## Variáveis, segredos e ambientes

| GitLab | GitHub |
|---|---|
| `variables:` | `env:` |
| variável na UI (projeto, grupo) | `vars` (repositório, organização) |
| variável mascarada ou protegida | secret |
| `environment:` | `environment:` |
| ambiente protegido e aprovação | revisores obrigatórios e wait timer |
| `id_tokens` / `secrets:` de cofre | OIDC com a nuvem (m) |

## Arquivos, cache e reaproveitamento

| GitLab | GitHub |
|---|---|
| `artifacts:` (passa sozinho entre estágios) | `upload-artifact` + `download-artifact` |
| `cache:` com `key:files` | `actions/cache` com `hashFiles` |
| `include:` de template | `uses:` de workflow reutilizável |
| componente com `spec:inputs` | `workflow_call` com `inputs` |
| `extends:`, `!reference`, âncoras | sem equivalente direto; usa-se workflow reutilizável ou action composta |
| `needs:` (DAG) | `needs:` |
| parent-child | workflow reutilizável chamado por outro (até 10 níveis) |
| multi-projeto | workflow reutilizável de outro repositório, ou `repository_dispatch` (m) |

## Recursos do GitLab sem leitura equivalente (m)

| GitLab | GitHub |
|---|---|
| Auto DevOps | sem equivalente pronto; existem starter workflows sugeridos ao criar o workflow |
| GitLab Pages | GitHub Pages, publicado por workflow |
| feature flags | sem recurso nativo; usa-se serviço externo |
| review apps | sem recurso nativo; monta-se com `environment` + `pull_request` |
| agente para Kubernetes | sem equivalente nativo; usa-se Flux ou Argo CD, ou `kubectl` no workflow |

## Diferenças que mais mudam o raciocínio

1. **Filtro no workflow, não no job.** No GitLab as `rules` ficam em cada job; no GitHub o `on:` filtra o workflow inteiro.
2. **Runner limpo por job.** Passar arquivo de um job para outro exige artifact; instalar de novo exige cache.
3. **Reaproveitar tem dois níveis.** Action (um step) e workflow reutilizável (um ou mais jobs). O GitLab concentra isso em `include`, `extends` e componentes.
4. **Eventos além do código.** O GitHub reage a issue, comentário, rótulo e release, não só a commit e MR.
5. **Variável e secret são recursos separados** (`vars` e `secrets`).
6. **Plano importa em privado.** Revisores obrigatórios e wait timer em environment exigem plano pago.

## Arquivos do tema

- [github-actions-conceitos.md](github-actions-conceitos.md)
- [github-actions-arquivo-e-gatilhos.md](github-actions-arquivo-e-gatilhos.md)
- [github-actions-jobs-steps-e-actions.md](github-actions-jobs-steps-e-actions.md)
- [github-actions-variaveis-secrets-e-environments.md](github-actions-variaveis-secrets-e-environments.md)
- [github-actions-artifacts-cache-e-reaproveitamento.md](github-actions-artifacts-cache-e-reaproveitamento.md)
