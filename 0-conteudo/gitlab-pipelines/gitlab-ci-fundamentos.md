# GitLab CI/CD: fundamentos e palavras-chave

O GitLab CI/CD constrói, testa e implanta o código a cada mudança, para que os erros apareçam cedo e o código que vai para produção siga os padrões do time.

## Conceitos

| Conceito | O que é |
|---|---|
| **Pipeline** | o fluxo completo de automação. Nasce quando algo o dispara: um commit, um merge request, um agendamento ou uma execução manual |
| **Stage** (estágio) | a ordem de execução: construir, testar, implantar |
| **Job** | uma ação dentro de um estágio, como compilar ou testar |
| **Runner** | o agente que executa os jobs; pode ser uma máquina física ou virtual. No GitLab.com já existem runners para Linux, Windows e macOS |
| **`.gitlab-ci.yml`** | o arquivo na raiz do projeto que descreve estágios, jobs, variáveis e quando cada job roda. O nome é esse por padrão e diferencia maiúsculas de minúsculas |

```text
commit / merge request / agendamento / manual
        │
        ▼
     PIPELINE
        ├─ stage build  ── job build
        ├─ stage test   ── job test-unit, job test-integracao   (rodam ao mesmo tempo)
        └─ stage deploy ── job deploy-hml
                              │
                              └─ cada job é executado por um RUNNER
```

Os jobs do mesmo stage rodam em paralelo, e o stage seguinte só começa quando todos os do anterior terminam com sucesso. O `needs` quebra essa regra (ver [gitlab-arquiteturas-de-pipeline.md](gitlab-arquiteturas-de-pipeline.md)).

## Exemplo: o menor pipeline da API .NET

```yaml
stages:
  - build
  - test

build:
  stage: build
  image: mcr.microsoft.com/dotnet/sdk:10.0     # imagem Docker onde o job roda
  script:
    - dotnet build -c Release

test:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:10.0
  script:
    - dotnet test
```

`image` vale para runners com o executor Docker, o mais comum (de memória). Se um comando do `script` falha, o job falha e o pipeline para.

## Palavras-chave do `.gitlab-ci.yml` (índice da doc)

Fonte: [referência de sintaxe](https://docs.gitlab.com/ci/yaml/). A página tem cerca de 275 mil caracteres e foram lidos os primeiros 100 mil, então o que há abaixo é o índice com a descrição curta de cada uma.

### Globais

| Palavra-chave | O que faz |
|---|---|
| `default` | valores padrão para palavras-chave de job, copiados para os jobs que não as definem |
| `include` | importa configurações de outros arquivos YAML |
| `stages` | nomes e ordem dos estágios |
| `variables` | variáveis de CI/CD padrão para todos os jobs |
| `workflow` | controla quais tipos de pipeline são executados |
| `spec` (cabeçalho) | especificações para arquivos de configuração externos (inputs) |

### Por job

| Palavra-chave | O que faz |
|---|---|
| `script` | comandos de shell executados pelo runner |
| `before_script` / `after_script` | comandos antes do `script` / depois do job, mesmo se falhar ou for cancelado |
| `stage` | o estágio do job |
| `image` / `services` | imagem Docker do job / imagens de serviços (como um Postgres) usados pelo job |
| `rules` | condições que determinam atributos do job e se ele é criado |
| `when` | quando o job roda |
| `needs` | roda jobs antes da ordem dos estágios |
| `dependencies` | restringe de quais jobs os artifacts são baixados |
| `artifacts` | arquivos anexados ao job quando ele tem sucesso |
| `cache` | arquivos mantidos entre execuções |
| `environment` | o ambiente em que o job faz deploy |
| `variables` | variáveis do job |
| `extends` / `inherit` | configurações herdadas / quais padrões globais o job herda |
| `tags` | tags para escolher um runner |
| `trigger` | dispara um pipeline downstream |
| `allow_failure` | o job pode falhar sem interromper o pipeline |
| `retry` | quando e quantas vezes o job é repetido após falha |
| `timeout` | tempo limite do job |
| `parallel` | quantas instâncias do job rodam em paralelo |
| `interruptible` | o job pode ser cancelado por uma execução mais nova |
| `resource_group` | limita a concorrência do job |
| `start_in` | atrasa a execução (requer `when: delayed`) |
| `manual_confirmation` | mensagem de confirmação para jobs manuais |
| `coverage` | extrai a cobertura de código da saída do job |
| `release` | faz o runner gerar um release |
| `pages` | envia o resultado do job para o GitLab Pages |
| `secrets` | segredos de CI/CD que o job precisa |
| `identity` | autentica em serviços de terceiros por federação de identidade |
| `description` | texto curto sobre a finalidade do job |
| `dast_configuration` | usa perfis de DAST no job |
| `run` | configuração de execução consumida pelo runner |

`only` e `except` não estão no índice (são as palavras-chave antigas, substituídas por `rules`);.

## Exemplos das palavras-chave

### tags: escolher o runner

O job só roda num runner que tenha **todas** as tags listadas (doc). Serve para mandar o job ao runner certo, por exemplo o que tem acesso à rede do cluster.

```yaml
build:
  script: dotnet build
  tags:
    - docker
    - linux

deploy-hml:
  script: kubectl apply -f k8s/
  tags:
    - k8s-deploy        # runner com acesso ao cluster
```

As tags são definidas quando o runner é registrado (de memória). Sem `tags`, o job vai para um runner que aceite jobs sem tag.

### services (e image)

Imagens Docker adicionais que o job precisa, como um banco, ligadas à imagem principal.

```yaml
teste:                      # (doc)
  image: ruby:2.6
  services:
    - name: postgres:11.7
      alias: db
  script:
    - bundle exec rspec
```

Exemplo: testes de integração da API contra um Postgres. O host do banco é o `alias`.

```yaml
test-integracao:
  image: mcr.microsoft.com/dotnet/sdk:10.0
  services:
    - name: postgres:17
      alias: db
  variables:
    POSTGRES_PASSWORD: exemplo
    ConnectionStrings__Default: "Host=db;Database=postgres;Username=postgres;Password=exemplo"
  script:
    - dotnet test --filter Category=Integration
```

### spec (inputs)

O trecho da doc lido não trouxe a seção `spec`. Pelo que ele mostra, existem dois usos de `inputs`:

1. Inputs de um **job**, referenciados com `${{ job.inputs.NOME }}`, que podem ser alterados ao executar ou repetir o job manualmente (doc):

```yaml
deploy_job:
  inputs:
    ambiente:
      default: staging
      options: [staging, production]
  script:
    - echo "Implantando em ${{ job.inputs.ambiente }}"
```

2. Inputs de um **arquivo** incluído ou componente, no cabeçalho `spec` (de memória):

```yaml
spec:
  inputs:
    ambiente:
      default: staging
      options: [staging, production]
---
deploy:
  script: echo "Implantando em $[[ inputs.ambiente ]]"
```

```yaml
include:
  - local: deploy.yml
    inputs:
      ambiente: production
```

Os tipos de input (string, number, boolean, array) são de memória.

### extends

Herda de um job-modelo; as chaves são mescladas, os valores não (doc). Mais exemplos em [gitlab-reaproveitamento.md](gitlab-reaproveitamento.md).

```yaml
.base_testes:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:10.0

unitarios:
  extends: .base_testes
  script: dotnet test
```

### inherit

Controla quais padrões globais o job herda (doc).

```yaml
default:
  retry: 2
  image: mcr.microsoft.com/dotnet/sdk:10.0

job_isolado:
  script: echo "Sem herança de padrões"
  inherit:
    default: false
```

### parallel e parallel:matrix

`parallel: N` roda o job N vezes em paralelo (nomes `nome 1/N`); `parallel:matrix` gera uma instância por combinação de variáveis (doc).

```yaml
testes:
  script: dotnet test
  parallel: 3

deploy:
  script: ./deploy.sh $AMBIENTE $REGIAO
  parallel:
    matrix:
      - AMBIENTE: [hml, prod]
        REGIAO: [brasil, eua]       # 4 instâncias: hml/brasil, hml/eua, prod/brasil, prod/eua
```

### manual_confirmation

Mensagem de confirmação antes de executar um job manual; funciona com `when: manual` (doc).

```yaml
apagar-ambiente:
  script: ./limpa.sh
  when: manual
  manual_confirmation: "Confirma a exclusão do ambiente?"
```

### coverage

Expressão regular que extrai o percentual de cobertura da saída do job; aparece no widget do MR, na lista de jobs e nos gráficos (doc).

```yaml
job1:                       # (doc)
  script: rspec
  coverage: '/Code coverage: \d+(?:\.\d+)?/'

test:                       # regex não testada: depende do formato da saída do coverlet
  script: dotnet test /p:CollectCoverage=true
  coverage: '/Total\s*\|\s*(\d+(?:\.\d+)?)%/'
```

Para mostrar a cobertura linha a linha no diff do MR, usa-se `artifacts:reports:coverage_report` (ver [gitlab-artifacts-cache-variaveis.md](gitlab-artifacts-cache-variaveis.md)).

### release

Cria um release do projeto, com tag, descrição e links de ativos. Exige a CLI `glab` (doc).

```yaml
release_job:
  stage: release
  image: registry.gitlab.com/gitlab-org/cli:latest     # (doc) fixe uma versão em vez de latest
  rules:
    - if: $CI_COMMIT_TAG
  script:
    - echo "Criando a release"
  release:
    tag_name: $CI_COMMIT_TAG
    description: 'Descrição da release'
```

### pages

Define o job que publica conteúdo estático no GitLab Pages; a pasta padrão é `public` (doc). Mais em [gitlab-ambientes-e-recursos-extras.md](gitlab-ambientes-e-recursos-extras.md).

```yaml
create-pages:
  stage: deploy
  script:
    - mv meu-site public
  pages: true
```

### identity

Autentica em serviços de terceiros por federação de identidade (sem segredo guardado). Está em beta e hoje suporta o Google Cloud (doc).

```yaml
job_gcp:
  identity: google_cloud
  script:
    - gcloud compute instances list
```

### description

Texto curto, de até 128 caracteres, com a finalidade do job (doc).

```yaml
rspec:
  script: bundle exec rspec
  description: Testes unitários da aplicação
```

### dast_configuration

Associa a um job de DAST um perfil de site e um perfil de scanner já criados no projeto. O job precisa estar no stage `dast`. Exclusivo do Ultimate (doc).

```yaml
dast:
  dast_configuration:
    site_profile: "Example Co"
    scanner_profile: "Quick Passive Test"
```

### Outras que apareceram nos trechos

```yaml
test:
  script: dotnet test
  retry:                       # tentativas extras se falhar (0, 1 ou 2)
    max: 2
    when: runner_system_failure
  timeout: 1 hour 30 minutes   # tempo máximo; passou, o job falha

deploy-producao:
  script: ./deploy.sh
  resource_group: producao     # jobs do mesmo grupo não rodam ao mesmo tempo, nem em pipelines diferentes

testes_linux:
  stage: test
  needs: ["build_linux"]       # roda fora da ordem dos stages (no máximo 50)
  script: echo "Testando"

limpeza:
  script: ./limpa-build.sh
  when: on_failure             # on_success (padrão), on_failure, always, never, manual, delayed

job:
  secrets:                     # segredo de provedor externo, entregue como variável
    SENHA_BANCO:
      vault: producao/db/password

dispara-outro-projeto:
  trigger:
    project: grupo/meu-projeto

job-passos:
  run:                         # sequência de passos; experimental
    - name: saudacao
      script: echo "olá"
```

## Tópicos da seção de CI/CD da doc

Fonte: [GitLab CI/CD](https://docs.gitlab.com/ci/). São 20 tópicos: primeiros passos, tutoriais, referência de sintaxe, runners, pipelines, jobs, componentes, inputs, variáveis, segurança de pipelines, gerenciador de segredos, segredos externos, depuração, Auto DevOps, testes, sustentabilidade, integração com Google Cloud, migração, integrações com repositórios externos e DevOps para apps móveis.

Neste conteúdo estão cobertos:

| Assunto | Arquivo |
|---|---|
| Artifacts, cache, variáveis, segredos | [gitlab-artifacts-cache-variaveis.md](gitlab-artifacts-cache-variaveis.md) |
| include, extends, `!reference`, componentes | [gitlab-reaproveitamento.md](gitlab-reaproveitamento.md) |
| Arquiteturas de pipeline (pai-filho, multi-projeto, DAG) | [gitlab-arquiteturas-de-pipeline.md](gitlab-arquiteturas-de-pipeline.md) |
| `rules`, `if`, `workflow`, pipelines de MR | [gitlab-rules-e-condicionais.md](gitlab-rules-e-condicionais.md) |
| Environments, Auto DevOps, Pages, feature flags, agent do Kubernetes | [gitlab-ambientes-e-recursos-extras.md](gitlab-ambientes-e-recursos-extras.md) |

Não cobertos: tutoriais, depuração, sustentabilidade, integração com Google Cloud, migração, integrações com repositórios externos e DevOps para apps móveis.

A segurança no pipeline (secret detection, SCA, SAST, container scan, DAST) está em [../devops-cicd/seguranca-no-pipeline.md](../devops-cicd/seguranca-no-pipeline.md).
