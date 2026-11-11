# GitLab CI/CD: arquiteturas de pipeline

Fontes: [Pipeline architecture](https://docs.gitlab.com/ci/pipelines/pipeline_architectures/) e [Downstream pipelines](https://docs.gitlab.com/ci/pipelines/downstream_pipelines/).

A doc cita 4 arquiteturas, que podem ser combinadas: **básica**, **com `needs` (DAG)**, **pai-filho** e **multi-projeto**.

## 1. Básica

Para projetos simples, com toda a configuração em um lugar. Os jobs de um stage rodam em paralelo, e o seguinte só começa quando o anterior termina.

- Vantagem: fácil de manter.
- Desvantagem: um job lento atrasa todo o stage seguinte; com muitos passos a configuração fica complexa.

```yaml
stages: [build, test, deploy]

build_a:
  stage: build
  script: echo "Constrói algo."
build_b:
  stage: build
  script: echo "Constrói outra coisa."
test_a:
  stage: test
  script: echo "Só roda quando todo o stage build terminar."
deploy_a:
  stage: deploy
  script: echo "Só roda após o stage test."
  environment: production
```

## 2. Com `needs` (DAG)

Para projetos grandes que precisam de execução rápida. Define dependências explícitas entre jobs, e um job começa assim que suas dependências terminam, mesmo que outros jobs do mesmo stage ainda rodem.

```yaml
stages: [build, test, deploy]

build_a:
  stage: build
  script: echo "Constrói rapidamente."
build_b:
  stage: build
  script: echo "Constrói devagar."
test_a:
  stage: test
  needs: [build_a]       # começa assim que build_a termina, sem esperar build_b
  script: echo "Testa A."
test_b:
  stage: test
  needs: [build_b]
  script: echo "Testa B."
deploy_a:
  stage: deploy
  needs: [test_a]        # pode rodar cedo, sem esperar build_b nem test_b
  script: echo "Faz deploy de A."
  environment: production
```

```text
sem needs:   [build_a build_b] ──> [test_a test_b] ──> [deploy_a deploy_b]
com needs:   build_a ──> test_a ──> deploy_a
             build_b ──> test_b ──> deploy_b      (cada trilha anda sozinha)
```

## 3. Pai-filho (mesmo projeto)

Para monorepos e projetos com muitos componentes definidos de forma independente, ou quando o YAML é gerado dinamicamente. Separa a configuração em vários arquivos, reduz a espera entre stages, evita colisão de nomes de `include` e permite disparar sub-pipelines só quando há mudança na área (`rules:changes`).

Pai:

```yaml
stages: [triggers]

trigger_a:
  stage: triggers
  trigger:
    include: a/.gitlab-ci.yml
  rules:
    - changes:
        - a/*

trigger_b:
  stage: triggers
  trigger:
    include: b/.gitlab-ci.yml
  rules:
    - changes:
        - b/*
```

Filho (`a/.gitlab-ci.yml`, com `needs`):

```yaml
stages: [build, test, deploy]

build_a:
  stage: build
  script: echo "Constrói algo."
test_a:
  stage: test
  needs: [build_a]
  script: echo "Testa algo."
deploy_a:
  stage: deploy
  needs: [test_a]
  script: echo "Faz deploy de algo."
  environment: production
```

### Variantes de `trigger` (downstream)

Um pipeline downstream roda de forma independente e concorrente ao pipeline que o disparou.

```yaml
# arquivo local no mesmo projeto
trigger_job:
  trigger:
    include:
      - local: path/to/child-pipeline.yml

# arquivo de outro projeto
microservice_a:
  trigger:
    include:
      - project: 'my-group/my-pipeline-library'
        ref: 'main'
        file: '/path/to/child-pipeline.yml'

# até três arquivos mesclados no filho (local, template, project)

# com inputs
staging:
  trigger:
    include:
      - local: path/to/child-pipeline.yml
        inputs:
          environment: staging
```

### Pipeline filho dinâmico (gerado por artifact)

```yaml
generate-config:
  stage: build
  script: generate-ci-config > generated-config.yml
  artifacts:
    paths: [generated-config.yml]

child-pipeline:
  stage: test
  trigger:
    include:
      - artifact: generated-config.yml
        job: generate-config
```

O caminho é interpretado pelo GitLab, não pelo runner. Não dá para usar variáveis de CI/CD na seção `include` do filho dinâmico.

### Profundidade: pai, filho e neto

- **Pai-filho:** no máximo **dois níveis de filhos**. Pai → filho → neto (o "avô, pai e filho") é possível, mas o neto não pode disparar outro nível.
- **Multi-projeto:** sem limite de aninhamento.
- A hierarquia total aceita até 1000 downstream por padrão.

```text
pipeline avô (projeto)
  └─ trigger ──> pipeline pai (filho do avô)
                    └─ trigger ──> pipeline filho (neto do avô)   <- daqui não desce mais
```

## 4. Multi-projeto

Para produtos que dependem de vários projetos (microsserviços): dispara um pipeline em outro projeto, cada um com seu próprio build, teste e deploy, e mostra as dependências entre projetos num lugar só.

```yaml
trigger_job:
  trigger:
    project: project-group/my-downstream-project

staging:
  trigger:
    project: my/deployment
    branch: stable-11-2
```

Também pode ser disparado pela API com `CI_JOB_TOKEN`; o pipeline criado é tratado como downstream do job que fez a chamada. A página de arquiteturas não traz exemplo nem desvantagens para esta (e para a DAG).

## O pai espera o filho? (`strategy`)

| Valor | Comportamento |
|---|---|
| (padrão) | o job de trigger vira `passed` assim que o downstream é criado; o pai não espera |
| `strategy: depend` | o job de trigger acompanha o filho (a doc não recomenda: o status nem sempre corresponde) |
| `strategy: mirror` | o job de trigger sempre tem o mesmo status do downstream |

Para que os relatórios do filho (testes, segurança...) apareçam no merge request, use `depend` ou `mirror`; sem isso o pai termina antes do filho. O auto-cancelamento do pai por falha do filho exige `workflow: auto_cancel: on_job_failure` nos dois e `strategy: mirror`.

## Passar variáveis

```yaml
variables:
  VERSION: "1.0.0"

staging:
  variables:
    ENVIRONMENT: staging
  trigger:
    include:
      - local: path/to/child-pipeline.yml
```

- O downstream herda as `variables`, incluindo as globais; para bloquear: `inherit: variables: false`.
- Variável predefinida (como `$CI_COMMIT_REF_NAME`): salve numa variável do job de trigger e use no downstream.
- Variáveis mascaradas não são repassadas em multi-projeto.
- Em multi-projeto, se o mesmo nome existir nos dois lados, vale a do pipeline pai.
- `trigger:forward` controla quais tipos de variável são repassados (detalhes em outra página).

## Passar artifacts

Pai-filho (`needs:pipeline:job`):

```yaml
build_artifacts:
  stage: build
  script: echo "teste" >> artifact.txt
  artifacts:
    paths: [artifact.txt]

deploy:
  stage: deploy
  trigger:
    include:
      - local: path/to/child-pipeline.yml
  variables:
    PARENT_PIPELINE_ID: $CI_PIPELINE_ID
```

No filho:

```yaml
test:
  needs:
    - pipeline: $PARENT_PIPELINE_ID
      job: build_artifacts
```

Multi-projeto (`needs:project`): o projeto downstream precisa estar na allowlist do job token do upstream, e o usuário precisa ser Reporter no upstream.

```yaml
test:
  needs:
    - project: my/upstream_project
      job: build_artifacts
      ref: main
      artifacts: true
```

Em merge request, use `ref` com `$CI_MERGE_REQUEST_REF_PATH` (não o nome da branch); não funciona com merged results pipelines.

## Environment com trigger

`environment` junto com `trigger` permite ver o ambiente no projeto de origem. Os relatórios do filho (testes JUnit, code quality, Terraform, métricas, segurança) aparecem no widget do MR (segurança a partir da 18.9).

## Quando usar cada uma (resumo meu)

| Situação | Arquitetura |
|---|---|
| projeto pequeno | básica |
| pipeline lento por esperar o stage inteiro | DAG com `needs` |
| monorepo ou YAML gigante | pai-filho |
| vários serviços/repositórios que se disparam | multi-projeto |
| precisa gerar o pipeline em tempo de execução | filho dinâmico por artifact |

Não foram lidos: os vídeos da página, a referência YAML completa (`trigger:forward`, `trigger:strategy`, `interruptible`), os limites de administração e a página de troubleshooting.
