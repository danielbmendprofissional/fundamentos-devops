# GitLab CI/CD: reaproveitamento (include, extends, !reference, componentes)

São 4 formas de reduzir duplicação dentro do YAML, mais os componentes. Em geral, o `extends` é a preferida por ser mais legível.

## include

Fonte: [include](https://docs.gitlab.com/ci/yaml/includes/).

Tipos: `local`, `project`, `remote`, `template` e `component`. Sem tipo explícito, o GitLab assume `local` para caminhos do repositório e `remote` para URLs.

```yaml
include:
  - local: 'templates/.before-script-template.yml'
  - project: 'meu-grupo/meu-projeto'
    ref: main
    file: 'templates/.gitlab-ci-template.yml'
  - remote: 'https://exemplo.com/raw/main/.before-script-template.yml'
  - template: Auto-DevOps.gitlab-ci.yml
```

- `remote`: a doc recomenda evitar hospedar o arquivo no GitLab Pages (o limite de requisições pode causar falhas); prefira `project`.
- Vários arquivos: é uma lista, misturando tipos. Aceita curingas: `include: 'configs/*.yml'` (só a pasta) e `'configs/**.yml'` (com subpastas).
- `ref` indica a branch, tag ou commit de onde o arquivo é lido. Os exemplos sempre o informam; a página não define o padrão se omitido.
- Arquivos incluídos podem ter seus próprios `include`: o limite é de **150 includes aninhados** por pipeline. Cuidado com loops (A inclui B, que inclui A).

### Incluir só se um arquivo existir

```yaml
include:
  - local: builds.yml
    rules:
      - exists:
          - file.md
  - local: builds.yml
    rules:
      - exists:
          - exception-file.md
        when: never
```

Em includes de outro projeto, a busca é no projeto e na ref do arquivo incluído; para mudar, use `exists:paths`, `project` e `ref`.

### Variáveis permitidas no include

Podem ser usadas: variáveis de projeto, grupo e instância; predefinidas `CI_PROJECT_*`; de trigger, agendamento e execução manual; `CI_PIPELINE_SOURCE`, `CI_PIPELINE_TRIGGERED` e `CI_COMMIT_REF_NAME`. Não podem: as definidas em jobs ou no `variables` global, porque os includes são avaliados antes dos jobs.

### O que prevalece (merge)

- Os arquivos incluídos são mesclados na ordem; em conflito, o último incluído vence.
- O `.gitlab-ci.yml` principal é mesclado por último e prevalece sobre os incluídos.
- A mesclagem é profunda para hashes. Valores simples são substituídos.
- **Arrays não são mesclados por item**: `script` e `rules`, por exemplo, são substituídos inteiros. Para estender um `script`, repita os comandos originais.

### Exemplo: um template central usado por vários projetos

Num projeto central `meu-grupo/templates`, o arquivo `templates/template.yml`:

```yaml
default:
  before_script:
    - echo "Preparando o ambiente"

stages:
  - build
  - test

build-job:
  stage: build
  script:
    - dotnet build -c Release
```

Cada projeto consumidor inclui o template e sobrescreve o que precisar:

```yaml
include:
  - project: 'meu-grupo/templates'
    ref: main
    file: 'templates/template.yml'

test-job:
  stage: test
  script:
    - dotnet test
```

Quando o template central muda, todos os projetos com `ref: main` recebem a alteração na próxima execução. Para evitar mudanças inesperadas, fixe uma tag ou commit em `ref`.

## extends

Fonte: [Optimize GitLab CI/CD configuration files](https://docs.gitlab.com/ci/yaml/yaml_optimization/).

Reaproveita configuração de forma legível e funciona com `include`. Jobs ocultos, com nome começando por ponto (`.tests`), servem de template porque não são executados.

```yaml
.tests:
  rules:
    - if: $CI_PIPELINE_SOURCE == "push"

.rspec:
  extends: .tests          # herança em dois níveis
  script: rake rspec

rspec 1:
  variables:
    RSPEC_SUITE: '1'
  extends: .rspec
```

- **Vários `extends`:** é uma lista. Hashes são mesclados; listas (`tags`, `script`) são substituídas; em conflito, vence o último da lista.
- **Excluir uma chave:** atribuir `null`. `{}` só zera o conteúdo.
- Recomendação: no máximo 3 níveis de herança (o limite técnico é 11).

## Anchors e merge keys (YAML nativo)

```yaml
.job_template: &job_configuration
  image: mcr.microsoft.com/dotnet/sdk:10.0
  services:
    - postgres

test1:
  <<: *job_configuration   # copia o conteúdo da âncora
  script:
    - dotnet test

.some-script: &some-script
  - echo "Executa no meio"

job1:
  script:
    - *some-script
    - echo "Específico deste job"
```

Limitações: só funcionam dentro do mesmo arquivo (não atravessam `include`); a âncora precisa ser definida antes do alias; copiam hashes mas não substituem listas de forma granular.

## `!reference`

Seleciona uma palavra-chave específica de outro job e a reutiliza no atual, inclusive de arquivos incluídos. Sintaxe: `!reference [job, palavra-chave]` ou `!reference [job, palavra-chave, subchave]`.

```yaml
.setup:
  script:
    - echo "preparando"

.teardown:
  after_script:
    - echo "limpando"

test:
  script:
    - !reference [.setup, script]
    - dotnet test
  after_script:
    - !reference [.teardown, after_script]
```

Variáveis (tudo ou uma só):

```yaml
.vars:
  variables:
    URL: "http://my-url.internal"
    IMPORTANT_VAR: "the details"

test-vars-1:
  variables: !reference [.vars, variables]
  script: [printenv]

test-vars-2:
  variables:
    MY_VAR: !reference [.vars, variables, IMPORTANT_VAR]
  script: [printenv]
```

Regras (monta a lista de `rules` a partir de pedaços):

```yaml
.rules_prod:
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  - if: $CI_PIPELINE_SOURCE == "schedule"

.rules_staging:
  - if: $CI_COMMIT_BRANCH =~ /^wip-.*/
  - if: $CI_PIPELINE_SOURCE == "push"

deploy_job:
  script: echo test
  rules:
    - !reference [.rules_prod]
    - !reference [.rules_staging]
```

Aninhado (até 10 níveis dentro de `script`, `before_script` e `after_script`):

```yaml
.snippets:
  one:
    - echo "ONE!"
  two:
    - !reference [.snippets, one]
    - echo "TWO!"

nested-references:
  before_script:
    - !reference [.snippets, two]
  script:
    - echo "job principal"
```

Limitações: fora de `script`, `before_script`, `after_script`, `rules` (e dos exemplos de `variables`), pode dar o erro de validação "config should be an array of"; não dá para usar inputs no caminho do `!reference`; os editores podem marcar a tag como inválida (no VS Code, configure `yaml.customTags` com `"!reference sequence"`).

### Comparação

| Recurso | Escopo | Granularidade | Principal uso | Limitação principal |
|---|---|---|---|---|
| Anchors (`&`, `*`) | mesmo arquivo | bloco inteiro | duplicar blocos e listas de script | não atravessa `include` |
| Merge key (`<<`) | mesmo arquivo | hash inteiro | copiar um mapa para outro job | depende de âncora |
| `extends` | inclui `include` | chaves do job | templates e herança legível | até 3 níveis; listas são substituídas |
| `!reference` | inclui `include` | palavra-chave ou subchave | pegar partes específicas | restrito a certas palavras-chave |

Prefira `extends` para templates gerais; use `!reference` para pegar um trecho (principalmente de arquivos incluídos); anchors só dentro do mesmo arquivo. Para criar jobs parecidos com variáveis diferentes, a doc aponta o `parallel:matrix`.

## Componentes de CI/CD (CI/CD Catalog)

Fonte: [CI/CD components](https://docs.gitlab.com/ci/components/).

Uma unidade reutilizável de configuração, com **inputs** (parâmetros) para torná-la configurável. Os componentes publicados ficam no **CI/CD Catalog**, que os usuários pesquisam.

Estrutura do projeto de componentes:

```text
README.md                    documentação (descrição do projeto é obrigatória na publicação)
LICENSE.md                   recomendado
.gitlab-ci.yml               testa os componentes e cria os releases
templates/
  meu-componente.yml         um arquivo
  outro-componente/
    template.yml             ou uma pasta (só o template.yml é usado por outros projetos)
```

Limite de 100 componentes por projeto. Os inputs são declarados no cabeçalho `spec`:

```yaml
spec:
  inputs:
    stage:
      default: test          # sem default, o input é obrigatório
      description: etapa do job
---
meu-job:
  stage: $[[ inputs.stage ]]
  script: dotnet test
```

(O `$[[ inputs.stage ]]` é de memória.)

Uso:

```yaml
include:
  - component: $CI_SERVER_FQDN/grupo/projeto/nome@1.0.0
    inputs:
      stage: build
```

- Só componentes da mesma instância GitLab podem ser referenciados.
- Jobs com o mesmo nome no pipeline e no componente se fundem.
- Input obrigatório ausente gera erro; variável indefinida não gera.

### Versões (prioridade decrescente)

1. SHA do commit
2. tag, como `1.0.0` (tem precedência sobre uma branch com o mesmo nome)
3. branch, como `main`
4. `~latest`: a última versão publicada no catálogo, sem pré-releases
5. versão parcial: `1` seleciona `1.*.*`, `1.2` seleciona `1.2.*` (só para componentes publicados)

Entre `1.0.0`, `1.1.1`, `1.2.0` e `2.0.0`, a versão `1` resolve para `1.2.0`.

### Publicar

Pré-requisitos: o projeto marcado como "CI/CD Catalog project" (papel Owner), descrição, README, pelo menos um componente em `templates/`, e papel Maintainer ou Owner. O release é criado com a palavra-chave `release` em um job disparado por tag (versionamento semântico), e não pela API de Releases. Testar os componentes antes do job de release.

### Componente x template comum

Os dois são incluídos com `include` e mesclados ao pipeline. O componente: aparece no catálogo, tem versões (tags e releases), vários componentes podem ser versionados juntos no mesmo projeto e usa inputs com validação em vez de variáveis. Restrições: componentes não usam `spec:include` e devem ser autocontidos.
