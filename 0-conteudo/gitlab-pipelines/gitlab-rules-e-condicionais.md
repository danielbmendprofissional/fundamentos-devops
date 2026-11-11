# GitLab CI/CD: rules, condicionais `if` e pipelines de merge request

Fontes: [Job rules](https://docs.gitlab.com/ci/jobs/job_rules/) e [Merge request pipelines](https://docs.gitlab.com/ci/pipelines/merge_request_pipelines/).

## Como as regras são avaliadas

- Em ordem; **a primeira que casa vence**.
- Se casa, o job entra ou sai do pipeline conforme a regra.
- Se nenhuma casa, o job **não entra** no pipeline.
- Variáveis `dotenv` criadas no script de um job não valem em `rules`: as regras são avaliadas antes de qualquer job rodar.

## Palavras-chave de uma regra

```yaml
job:
  script: echo "exemplo"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:             # só casa se estes arquivos mudaram
        - Dockerfile
        - docker/**/*
      exists:              # só casa se este arquivo existir
        - config/app.yml
      when: manual         # on_success é o padrão; never exclui o job
      allow_failure: true  # o pipeline segue mesmo sem rodar o job
```

| Chave | O que faz |
|---|---|
| `if` | condição com variáveis; se omitida, a regra casa sempre |
| `changes` | casa se os arquivos indicados mudaram; com `compare_to`, compara a branch com outra referência (por exemplo a padrão) |
| `exists` | casa se o arquivo existe (com `when: never`, exclui o job quando ele existe) |
| `when` | `on_success` (padrão), `manual`, `always`, `never` |
| `allow_failure` | `false` por padrão |
| `variables`, `needs` | existem, mas a página lida não os descreve |

## Condicionais `if`: variáveis e operadores

Variáveis comuns:

| Variável | O que é |
|---|---|
| `$CI_PIPELINE_SOURCE` | a origem do pipeline: `push`, `merge_request_event`, `schedule`, `web`, `api`, `trigger`... |
| `$CI_COMMIT_BRANCH` | definida em pipelines de branch |
| `$CI_COMMIT_TAG` | definida em pipelines de tag |
| `$CI_MERGE_REQUEST_IID` | definida em pipelines de merge request |
| `$CI_DEFAULT_BRANCH` | o nome da branch padrão |

Operadores:

| Operador | Significado |
|---|---|
| `==`, `!=` | igualdade com string ou outra variável |
| `=~`, `!~` | casa com expressão regular entre barras (`/^feature-.*/`); diferencia maiúsculas, `/padrao/i` ignora |
| `&&`, `\|\|` | e, ou; parênteses agrupam |
| `$VAR` | definida e não vazia |
| `== null`, `== ""` | não definida; vazia |
| `!` | negação, introduzida na 18.11: `!($VAR == "x")`. Cuidado: `!"false"` é falso, porque a string não é vazia |

## workflow:rules

Controla quais tipos de pipeline podem rodar; é a forma recomendada de evitar pipelines duplicados. A página de `rules` só a menciona (detalhes em outra página, não lida).

## A armadilha do pipeline duplicado (branch + MR)

Um único push na branch de um merge request pode disparar dois pipelines: um de branch (`push`) e um de merge request (`merge_request_event`). Acontece quando um job tem uma regra final sem condição (como `when: always`) e nada filtra os tipos.

```yaml
# 1) Restringir o job a um tipo de pipeline
job:
  script: echo "só em MR"
  rules:
    - if: $CUSTOM_VARIABLE == "true" && $CI_PIPELINE_SOURCE == "merge_request_event"

# 2) Excluir o tipo que duplica (sem workflow:rules, gera aviso)
job:
  script: echo "sem duplicar"
  rules:
    - if: $CI_PIPELINE_SOURCE == "push"
      when: never
    - when: always
```

Não misture jobs com `only/except` e jobs com `rules` no mesmo pipeline.

## Exemplos práticos (doc)

```yaml
# só em pipelines agendados
job-agendado:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script: make mundo

# pula se a branch não tem mudanças em relação à padrão
job-com-mudancas:
  script: echo "rodou"
  rules:
    - if: $CI_COMMIT_BRANCH
      changes:
        compare_to: 'refs/heads/main'
        paths:
          - '**/*'

# branches diferentes da padrão e tags
job2:
  script: echo "rodou"
  rules:
    - if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

Reaproveitando regras com `!reference` (ver [gitlab-reaproveitamento.md](gitlab-reaproveitamento.md)):

```yaml
.regras_padrao:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
      when: never
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

job1:
  rules:
    - !reference [.regras_padrao, rules]
  script: echo "branch padrão, sem agendamentos"
```

Migração de `only/except`: em vez de excluir, defina quando o job deve rodar (por exemplo `if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH`).

## Exemplo (meu): rules para a API .NET

O padrão de `workflow` é o conhecido da doc de workflow (a variável `$CI_OPEN_MERGE_REQUESTS` é de memória); confira antes de usar.

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never                         # evita o pipeline duplicado
    - if: $CI_COMMIT_BRANCH
    - if: $CI_COMMIT_TAG

test:
  script: dotnet test                     # roda em MR, branch e tag

deploy-hml:
  script: ./deploy.sh hml
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH     # só na principal

deploy-prod:
  script: ./deploy.sh prod
  rules:
    - if: $CI_COMMIT_TAG                              # só em tag
      when: manual                                    # com um clique
```

## Pipelines de merge request

Pipelines executados sempre que a source branch de um MR muda. Disparos: criar o MR com pelo menos um commit, enviar um commit novo ou clicar em "Run pipeline" na aba Pipelines do MR. Usam apenas o conteúdo da source branch e aparecem com o rótulo `merge request`.

Requisitos:
- `rules` que testem `$CI_PIPELINE_SOURCE == "merge_request_event"` no próprio `.gitlab-ci.yml` (regras vindas de `include`, como `include:component`, não bastam);
- papel Developer, Maintainer ou Owner no projeto de origem;
- repositório GitLab (não externo).

```yaml
# por job
job1:
  script: echo "Executa em pipelines de merge request"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# no pipeline inteiro
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

Existem variáveis predefinidas exclusivas de MR (as `CI_MERGE_REQUEST_*`), cuja lista está na página de variáveis predefinidas (não lida).

**Merged results pipelines** testam o resultado da junção entre a source e a target branch, e as **merge trains** enfileiram merges para testar cada um sobre o resultado dos anteriores (de memória). A página lida só remete a ambos; a explicação não foi lida.

Não foram lidos: `rules:variables`, `rules:needs`, a página de `workflow` e os exemplos linkados sobre como evitar pipelines duplicados em detalhe.
