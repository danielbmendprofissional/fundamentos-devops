# GitHub Actions: o arquivo do workflow e os gatilhos `on:`

Fontes: [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) e [Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows).

## O arquivo

Fica em `.github/workflows/`, com extensão `.yml` ou `.yaml`. Pode haver vários, um por workflow.

Chaves do topo do arquivo:

| Chave | Para que serve |
|---|---|
| `name` | nome na aba Actions; se faltar, aparece o caminho do arquivo |
| `run-name` | nome de cada execução; pode usar dados do evento (`${{ github.actor }}`) |
| `on` | o que dispara o workflow |
| `permissions` | o que o token automático (`GITHUB_TOKEN`) pode fazer: `read`, `write` ou `none` por escopo; o que não for listado vira `none` |
| `env` | variáveis para todos os jobs e steps |
| `defaults` | por exemplo `shell` e `working-directory` dos `run` |
| `concurrency` | uma execução por grupo ao mesmo tempo; `cancel-in-progress: true` cancela a anterior |
| `jobs` | os jobs |

## O `on:` em três formas

```yaml
on: push                      # um evento

on: [push, fork]              # lista: basta um acontecer

on:                           # com configuração
  push:
    branches: [main]
```

Evento com configuração leva dois-pontos. Se vários eventos acontecem juntos, cada um dispara uma execução.

## Filtros

```yaml
on:
  push:
    branches: [main, 'releases/**']   # glob; branches-ignore é o oposto (não use os dois)
    tags: ['v*']                      # só no push
    paths-ignore: ['docs/**']         # por arquivos alterados
  pull_request:
    types: [opened, reopened]         # tipos de atividade do evento
```

- `!` exclui dentro de uma lista (`'releases/**'` e `'!releases/**-alpha'`) e precisa vir depois de um padrão positivo.
- Com `branches` e `paths` juntos, os dois precisam casar.
- `paths` não vale para push de tag.
- Se o filtro barra o workflow, os checks dele ficam Pending; se forem obrigatórios, travam o merge.

## Os eventos

A doc lista 33 eventos.

| Grupo | Eventos |
|---|---|
| Código | `push`, `pull_request`, `create`, `delete` |
| Atividade do repositório | `issues`, `issue_comment`, `label`, `release`, `milestone`, `discussion`, `fork`, `watch` |
| Tempo | `schedule` (cron) |
| Manual e externo | `workflow_dispatch`, `repository_dispatch` (chamada de API) |
| Encadeamento | `workflow_call` (workflow reutilizável), `workflow_run` (reage a outro workflow, no máximo 3 níveis) |

Outros eventos: `check_run`, `check_suite`, `deployment`, `deployment_status`, `gollum`, `image_version`, `merge_group`, `page_build`, `public`, `registry_package`, `status`, `branch_protection_rule`, `discussion_comment`, `pull_request_review`, `pull_request_review_comment`, `pull_request_target`.

## push, pull_request, schedule e workflow_dispatch

```yaml
# push e pull_request: pipeline de branch e de merge
on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

# schedule: cron de 5 campos, em UTC por padrão
on:
  schedule:
    - cron: '30 5 * * 1-5'      # 05:30 UTC, de segunda a sexta (mínimo: 5 minutos)

# workflow_dispatch: botão manual, com campos de entrada
on:
  workflow_dispatch:
    inputs:
      ambiente:
        type: choice
        options: [hml, prod]
        required: true
```

Tipos de input: `boolean`, `choice`, `number`, `environment`, `string`. O valor chega em `${{ inputs.ambiente }}`.

## Pegadinhas

- `pull_request` dispara por padrão só em `opened`, `synchronize` (commit novo na branch) e `reopened`. Roda sobre o resultado do merge com a base e não roda se houver conflito.
- `schedule` roda só na branch padrão. O botão do `workflow_dispatch` também exige o arquivo nela.
- Em fork, os secrets não são passados e o `GITHUB_TOKEN` fica só leitura.
- `pull_request_target` roda com as permissões da base; rodar código não confiável nele é risco de segurança.

## Equivalência com o GitLab (de memória)

| GitLab | GitHub |
|---|---|
| `rules: if: $CI_PIPELINE_SOURCE == "schedule"` | `on: schedule` |
| `rules: if: $CI_COMMIT_TAG` | `on: push: tags:` |
| `rules: changes:` | `paths:` / `paths-ignore:` |
| "Run pipeline" manual com variáveis | `workflow_dispatch` com `inputs` |
| pipeline de MR (`merge_request_event`) | `pull_request` |
| `workflow: rules` contra pipeline duplicado | o filtro fica em cada `on:`; o mesmo problema aparece se `push` e `pull_request` pegam a mesma branch |

No GitLab as `rules` ficam em cada job. No GitHub o filtro fica no workflow inteiro, no `on:`, e depois o job ainda pode ter um `if`.
