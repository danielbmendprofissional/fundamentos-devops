# GitLab CI/CD: artifacts, cache, variáveis e segredos

## Artifacts

Arquivos ou diretórios que um job gera e salva ao final da execução, como saídas de build ou relatórios. Por padrão, os jobs dos estágios seguintes baixam os artifacts dos jobs dos estágios anteriores.

Exemplo: o build publica a API uma vez, e os jobs seguintes usam os mesmos arquivos.

```yaml
build:
  stage: build
  image: mcr.microsoft.com/dotnet/sdk:10.0
  script:
    - dotnet publish -c Release -o publish
  artifacts:
    paths:
      - publish/
    expire_in: 1 day          # some depois de 1 dia

package:
  stage: package
  script:
    - ls publish/             # os arquivos do build já estão aqui
```

Fonte: [Job artifacts](https://docs.gitlab.com/ci/jobs/job_artifacts/).

### Opções

| Opção | O que faz |
|---|---|
| `artifacts:paths` | arquivos e diretórios do artifact, relativos ao repositório; aceita curingas |
| `expire_in` | por quanto tempo é mantido. Sem ele, vale a configuração "Default artifacts expiration" da instância. A doc usa o exemplo `1 week`; o valor padrão dessa configuração não aparece na página, então não se confirma o padrão de 30 dias |
| `artifacts:when` | quando fazer o upload: `on_success` (padrão), `on_failure` ou `always` |
| `artifacts:reports` | relatórios de teste e análise, como JUnit. Não entram no arquivo baixado pelo nome do job; baixa-se pelo ID do job com `file_type` (por exemplo `file_type=junit`) |
| `artifacts:exclude` | exclui arquivos; não é recursivo (para excluir o conteúdo de um diretório, casar os arquivos) |
| `artifacts:untracked` | inclui todos os arquivos não rastreados pelo Git |
| `artifacts:name` | nome do arquivo; aceita variáveis. Se a branch tem barras, usar `$CI_COMMIT_REF_SLUG` e não `$CI_COMMIT_REF_NAME` |
| `artifacts:access` | quem pode baixar (por exemplo `maintainer`) |

Um artifact com expiração definida pode ser mantido manualmente pelo botão Keep na página do job.

### Exemplos de `reports`, `exclude`, `name` e `access`

```yaml
test:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:10.0
  script:
    - dotnet test --logger "junit;LogFilePath=test-results/results.xml"   # exige o pacote JunitXml.TestLogger no projeto de testes
  artifacts:
    reports:
      junit: test-results/results.xml        # os testes aparecem no MR e no pipeline
      coverage_report:                       # (doc) cobertura linha a linha no diff do MR
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

build:
  stage: build
  script:
    - dotnet publish -c Release -o publish
  artifacts:
    name: "api-$CI_COMMIT_REF_SLUG"          # nome do arquivo; com barras na branch, use _SLUG
    paths:
      - publish/
    exclude:
      - publish/**/*.pdb                     # exclusão explícita; não é recursiva
    access: maintainer                       # (doc) só Maintainer baixa
    expire_in: 1 day
```

Os relatórios (`reports`) são enviados sempre, com sucesso ou falha do job; os achados de segurança só são ingeridos quando o job tem sucesso. O `coverage_report` não mostra o percentual: para isso, configure o `coverage` à parte (ver [gitlab-ci-fundamentos.md](gitlab-ci-fundamentos.md)). Os valores de `access` além de `maintainer` (como `developer`, `all`, `none`) são de memória.

### Tipos de `artifacts:reports` (doc)

| Tipo | Para quê |
|---|---|
| `junit` | resultados de teste em XML JUnit, no MR e no pipeline |
| `coverage_report` | cobertura linha a linha no diff do MR (Cobertura ou JaCoCo); não mostra percentual |
| `codequality` | problemas de qualidade de código no MR (expira em 1 semana) |
| `dotenv` | variáveis de um arquivo que viram variáveis de CI/CD dos jobs seguintes |
| `accessibility` | impacto de acessibilidade das mudanças no MR (pa11y) |
| `annotations` | dados auxiliares, como links externos, na página do job |
| `metrics` (Premium, Ultimate) | métricas no widget de métricas do MR |
| `browser_performance` (Premium, Ultimate) | métricas de navegador (sitespeed.io) |
| `load_performance` (Premium, Ultimate) | métricas de teste de carga no MR |
| `sast` | vulnerabilidades do SAST |
| `secret_detection` | segredos detectados no código |
| `dependency_scanning` (Ultimate) | vulnerabilidades em dependências |
| `container_scanning` (Ultimate) | vulnerabilidades em containers |
| `dast` (Ultimate) | vulnerabilidades do DAST |
| `api_fuzzing`, `coverage_fuzzing` (Ultimate) | bugs achados por fuzzing |
| `cyclonedx` (Ultimate) | SBOM no padrão CycloneDX |
| `sarif` (Ultimate) | achados de qualquer scanner compatível com SARIF 2.1.0 |
| `requirements` (Ultimate) | marca requisitos do projeto como atendidos |
| `terraform` | plano do OpenTofu (`tfplan.json`) no widget do MR; exige tratar com jq para remover credenciais |

Fonte: [Artifacts reports](https://docs.gitlab.com/ci/yaml/artifacts_reports/). O exemplo do `dotnet test` não foi verificado.

### Controlar quem recebe os artifacts

```yaml
job:
  stage: test
  script: make build
  dependencies: []        # não baixa artifact nenhum
```

- `dependencies`: lista os jobs cujos artifacts serão baixados.
- `needs:artifacts`: alternativa que também controla o download (a página só cita, sem exemplo).
- Quando um deles é usado, só os jobs indicados fornecem artifacts.

### Manter o último artifact (keep latest)

Por padrão, para cada branch ou tag, o GitLab mantém os artifacts do pipeline mais recente (mesmo que tenha falhado) e do pipeline bem-sucedido mais recente. Esses ignoram o `expire_in`. Para economizar espaço, desative "Keep artifacts from most recent successful jobs" em Configurações > CI/CD > Artifacts.

### Limite de tamanho

No GitLab.com, 1 GB por arquivo de artifact compactado; no self-managed, o administrador define. A página também cita 100 MB como padrão de configuração, o que não alinha com o 1 GB, então confira a doc de limites. Para pacotes e versões finais, a página recomenda o package registry, e não os artifacts.

### Baixar pela API

```text
https://gitlab.com/api/v4/projects/<id>/jobs/artifacts/main/download?job=build
```

Com o token do job (Premium ou Ultimate) um job busca artifacts de outro pipeline: `...&job_token=$CI_JOB_TOKEN`.

## Cache

Arquivos que um job baixa e guarda para reutilizar, para que os jobs seguintes não baixem tudo de novo. Serve para dependências, como pacotes da internet.

Fonte: [Caching](https://docs.gitlab.com/ci/caching/).

| Opção | O que faz |
|---|---|
| `cache:key` | a chave do cache; por exemplo `$CI_COMMIT_REF_SLUG` (um cache por branch) |
| `cache:key:files` | a chave vem do conteúdo de arquivos (por exemplo um `lock`); muda quando as dependências mudam |
| `cache:key:files_commits` | a chave vem do último commit desses arquivos |
| `cache:paths` | o que guardar; relativo ao diretório do projeto, nunca fora dele |
| `cache:policy` | `pull-push` (baixa e atualiza), `pull` (só baixa); o `push` não é detalhado na página |
| `cache:fallback_keys` | até 5 chaves alternativas, testadas em ordem; também existe a variável `CACHE_FALLBACK_KEY` |

Exemplo: cache do NuGet para a API .NET. O cache só funciona com caminhos dentro do projeto, por isso a variável `NUGET_PACKAGES`.

```yaml
variables:
  NUGET_PACKAGES: "$CI_PROJECT_DIR/.nuget/packages"

build:
  image: mcr.microsoft.com/dotnet/sdk:10.0
  cache:
    key:
      files:
        - packages.lock.json
    paths:
      - .nuget/packages/
  script:
    - dotnet restore
    - dotnet build --no-restore
```

### Onde fica

Por padrão, **na máquina do runner** (executor Docker: volumes do Docker; shell: pasta do usuário do runner). O cache de um job vira um único `cache.zip`, então dois jobs com a mesma chave e caminhos diferentes podem se sobrescrever.

O problema do cache local: ele é de **um runner**. Se o job cair em outro runner (ou o runner for um container/pod que some, como no executor Kubernetes), não há cache.

### Cache distribuído (o "servidor de cache")

Para todos os runners consumirem o mesmo cache existe o **cache distribuído**. Um esclarecimento: não é um servidor do GitLab. É um **armazenamento de objetos** (S3, Google Cloud Storage, Azure Blob, ou um MinIO próprio) que você configura nos runners, e o cache passa a ir e vir dele:

```text
cache local:        runner A [cache]     runner B [cache]     (cada um com o seu)

cache distribuído:  runner A ──┐
                    runner B ──┼──> bucket S3 / GCS / Azure / MinIO   (um só para todos)
                    runner C ──┘
```

O que a doc confirma: a página de cache cita o S3 e a recomendação de usar "vários runners com cache distribuído (S3)"; a configuração do runner diz que o nível de cache local (`disable_cache`) é separado e que `disable_cache` não desativa o cache quando o runner está em modo distribuído, e que existe também o nível global.

A seção `[runners.cache]` do `config.toml` não pôde ser lida (a página veio cortada). Pelo que lembro, é algo assim, e deve ser conferido na doc do runner:

```toml
[[runners]]
  name = "runner-k8s"
  executor = "kubernetes"
  [runners.cache]
    Type = "s3"                      # s3, gcs ou azure
    Shared = true                    # compartilha entre runners
    [runners.cache.s3]
      ServerAddress = "minio.exemplo.com:9000"   # ou s3.amazonaws.com
      BucketName = "runner-cache"
      Insecure = false
```

Faz sentido quando há muitos runners ou runners efêmeros: sem isso, cada job começa do zero.

### Boas práticas (doc)

- Marcar os runners com tags e usar a mesma tag nos jobs que dividem cache.
- Chave adequada ao fluxo; não usar a mesma chave para caminhos diferentes nem para jobs diferentes.
- Fixar as versões das dependências, porque limpar o cache pode trazer versões novas não revisadas.
- O cache é uma otimização e não é garantido: o job precisa conseguir rodar sem ele.

### Artifacts x cache

| | Cache | Artifacts |
|---|---|---|
| Uso | dependências (pacotes baixados) | resultados de build entre estágios, e relatórios |
| Onde fica | na máquina do runner (ou S3) | armazenado no GitLab, baixável |
| Compartilhamento | entre jobs e entre pipelines futuros | entre jobs do mesmo pipeline |
| Expira | não há prazo fixo descrito | `expire_in` (o padrão de 30 dias aparece na página do cache, mas não foi confirmado na dos artifacts) |

## Variáveis de CI/CD

Fonte: [CI/CD variables](https://docs.gitlab.com/ci/variables/).

### Tipos

| Tipo | Onde se define | Observação |
|---|---|---|
| **Predefinidas** | fornecidas pelo GitLab | informações do job, pipeline e projeto |
| **Projeto** | Configurações > CI/CD > Variáveis | limite de 8000 por projeto |
| **Grupo** | no grupo | valem para os projetos do grupo e subgrupos; limite de 30000; papel Owner |
| **Instância** | administração | só self-managed e Dedicated |
| **YAML** | `variables:` no `.gitlab-ci.yml` | ficam visíveis a quem acessa o repositório: só configuração não sensível |
| **Pipeline** | ao criar o pipeline (manual, agendamento, API, trigger) | limite de 1200 por pipeline no GitLab.com; a doc recomenda os inputs de pipeline a partir da 17.7 |

Desativar os padrões globais num job: `variables: {}`.

### Flags

| Flag | O que faz |
|---|---|
| **masked** | oculta o valor nos logs (`[MASKED]`). Requisitos: uma linha, sem espaços, mínimo de 8 caracteres. Não é garantia contra acesso malicioso |
| **protected** | só em pipelines de branches ou tags protegidas |
| **masked and hidden** | além de mascarar, o valor não é exibido de novo na interface; só se escolhe ao criar |
| **expand** | permite que o valor referencie outra variável com `$`; desativado por padrão nas da interface desde a 18.6; não funciona com masked |
| **tipo File** | o valor vai para um arquivo temporário e a variável recebe o caminho; útil para `kubectl` e AWS CLI; no YAML não existe tipo file |

### Precedência (do maior para o menor)

1. políticas de pipeline ou de varredura
2. variáveis de job manual
3. variáveis de pipeline
4. variáveis de projeto
5. variáveis de grupo (o subgrupo mais próximo vence)
6. variáveis de instância
7. variáveis de relatórios dotenv (jobs em `dependencies` ou `needs`)
8. variáveis do YAML: `rules:variables`, depois `variables` do job, depois `workflow:rules:variables`, depois o `variables` global
9. variáveis de implantação
10. variáveis predefinidas

Os valores de `secrets` não entram nessa ordem: vêm do gerenciador externo na execução. Inputs não são variáveis.

### Exemplo: a mesma variável em vários lugares

Digamos que `API_URL` esteja definida em todos estes lugares:

| Onde | Valor | Posição na ordem |
|---|---|---|
| variável de job manual (ao rodar o job) | `manual.exemplo.com` | 2 |
| variável de pipeline (ao rodar, agendar ou via API) | `pipeline.exemplo.com` | 3 |
| variável do projeto (Configurações > CI/CD) | `projeto.exemplo.com` | 4 |
| variável do grupo | `grupo.exemplo.com` | 5 |
| variável da instância | `instancia.exemplo.com` | 6 |
| `variables:` do job no YAML | `job.exemplo.com` | 8 |
| `variables:` global no YAML | `global.exemplo.com` | 8 (a mais baixa dentro do YAML) |

O job enxerga **o primeiro da lista que existir**. Então:

- Se existir só o do projeto e o do grupo: vale o do **projeto**.
- Se alguém rodar o pipeline manualmente informando `API_URL`: vale o do **pipeline**, por cima do projeto.
- Se alguém rodar um job manual e informar o valor: vale o do **job manual**, o mais alto.
- Dentro do YAML, o `variables` do job vence o `variables` global.

O ponto que surpreende: **uma variável da interface (projeto, grupo, instância) vence a do `.gitlab-ci.yml`**. Quem define `API_URL: x` no YAML e também a tem em Configurações > CI/CD acaba usando a da interface. Por isso, o valor "padrão" costuma ficar no YAML e o "oficial" na interface.

Dois efeitos laterais, de memória: quando a mesma variável existe com escopos de ambiente diferentes, o escopo mais específico vence; e no grupo, a do subgrupo mais próximo do projeto vence (a doc confirma este último).

A ordem completa é a lista numerada acima (políticas de pipeline ou de varredura, no topo, valem só nos jobs que a política adiciona).

### Predefinidas mais usadas

A lista oficial está na página de variáveis predefinidas. Fora `CI_JOB_STAGE`, `CI_COMMIT_SHA`, `CI_ENVIRONMENT_ID`, `CI_ENVIRONMENT_SLUG`, `CI_ENVIRONMENT_URL` e `CI_PAGES_URL`, os nomes abaixo são de memória:

| Variável | O que é |
|---|---|
| `CI_COMMIT_SHA` | o hash do commit (a tag da imagem, na regra de ouro) |
| `CI_COMMIT_REF_SLUG` | o nome da branch ou tag, próprio para nomes e URLs |
| `CI_REGISTRY_IMAGE` | o endereço do registro de imagens do projeto |
| `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` | credenciais para fazer login no registro |
| `CI_JOB_TOKEN` | token do job, para chamar a API do GitLab |
| `CI_PROJECT_DIR` | a pasta do projeto no runner |
| `CI_DEFAULT_BRANCH` | o nome da branch padrão |
| `CI_PIPELINE_SOURCE` | a origem do pipeline (`push`, `merge_request_event`, `schedule`...) |

### Valores pré-preenchidos em pipeline manual

`value` e `description` no bloco de `variables` definem o valor sugerido e o texto ao disparar o pipeline manualmente.

### Escopo por ambiente

Cada variável de projeto (e de grupo, em Premium e Ultimate) pode valer só para um ambiente (`production`, `review/*`) ou para todos (`*`). Serve para limitar variáveis sensíveis. Não use variáveis com escopo de ambiente em `rules` ou `include`, porque podem estar indefinidas na validação do pipeline.

## Segredos externos

Fonte: [External secrets](https://docs.gitlab.com/ci/secrets/). Premium e Ultimate.

- Variáveis de CI/CD ficam sempre disponíveis nos jobs; os segredos precisam ser pedidos explicitamente por um job (`secrets:`).
- Provedores suportados: **HashiCorp Vault, Google Cloud Secret Manager, Azure Key Vault e AWS Secrets Manager**.
- A autenticação usa **ID tokens** (JWT); com qualquer provedor que aceite OIDC, é possível autenticar manualmente.

Ver o critério de quando usar variável ou segredo em [../docker/docker-compose-e-boas-praticas.md](../docker/docker-compose-e-boas-praticas.md) e o aviso de que nenhum segredo é 100% secreto em [../kubernetes/kubernetes-configmap-e-secret.md](../kubernetes/kubernetes-configmap-e-secret.md).
