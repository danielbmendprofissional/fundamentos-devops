# GitHub Actions: jobs, steps e actions

Fonte: [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax). Os exemplos .NET são de memória.

## O job

```yaml
jobs:
  test:                         # id do job (letra ou _, depois letras, números, - ou _)
    name: Testes da API         # nome exibido (opcional)
    runs-on: ubuntu-latest      # onde roda
    needs: build                # só depois do job build
    if: github.ref == 'refs/heads/main'
    timeout-minutes: 15
    steps: [...]
```

| Chave | O que faz |
|---|---|
| `runs-on` | o runner: um nome (`ubuntu-latest`), uma lista de labels (o runner precisa ter todas) ou um grupo |
| `needs` | um id ou lista de ids. Se o job necessário falha ou é pulado, este é pulado, a menos que tenha `if: ${{ always() }}` |
| `if` | condição do job, avaliada antes da matriz |
| `outputs` | valores que o job entrega aos seguintes via `needs` (limite de 1 MB por job; saída com segredo não é enviada) |
| `env`, `defaults` | variáveis e `shell`/`working-directory` só deste job |
| `environment` | ambiente de deploy |
| `timeout-minutes` | padrão 360; passou, o job é cancelado |
| `continue-on-error` | `true`: o workflow passa mesmo se este job falhar |

## A matriz (`strategy.matrix`)

```yaml
test:
  runs-on: ${{ matrix.os }}
  strategy:
    fail-fast: false              # padrão true: um falhou, cancela os outros da matriz
    matrix:
      os: [ubuntu-latest, windows-latest]
      dotnet: ['8.0.x', '10.0.x']   # 2 x 2 = 4 jobs
      include:
        - os: ubuntu-latest         # acrescenta/ajusta uma combinação
          dotnet: '10.0.x'
          cobertura: true
      exclude:
        - os: windows-latest        # tira uma combinação
          dotnet: '8.0.x'
  steps:
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ matrix.dotnet }}
```

O máximo é 256 jobs por execução. O `include` é processado depois do `exclude`.

## Job com banco: `services` e `container`

```yaml
test-integracao:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: postgres:17
      env:
        POSTGRES_PASSWORD: exemplo
      ports:
        - 5432:5432               # porta do host : porta do container
  steps:
    - uses: actions/checkout@v4
    - run: dotnet test            # a API fala com localhost:5432
```

- `services` sobe containers de apoio (banco, Redis) durante o job.
- `container:` roda os steps dentro de uma imagem. Exige runner Linux, e o shell padrão dentro dele é `sh`, não `bash`.
- Se o job roda no runner (sem `container`), acessa o serviço por `localhost` e pela porta mapeada. Se roda em `container`, acessa pelo nome do serviço.

## O step

```yaml
steps:
  - id: versao                          # para referenciar depois
    name: Calcular versão
    run: echo "v=1.2.${{ github.run_number }}" >> $GITHUB_OUTPUT
    shell: bash
    working-directory: ./src
    env:
      DOTNET_CLI_TELEMETRY_OPTOUT: 1

  - uses: actions/checkout@v4
    with:
      fetch-depth: 0                    # entradas da action
```

| Chave | O que faz |
|---|---|
| `uses` | chama uma action |
| `run` | comandos de shell; as linhas de um bloco compartilham o mesmo shell |
| `with` | entradas da action |
| `env` | variáveis só do step |
| `shell` | `bash`, `pwsh`, `python`, `sh`, `cmd`, `powershell` |
| `if` | condição do step; não dá para usar `secrets` direto aqui, copie para o `env` do job antes |
| `continue-on-error`, `timeout-minutes` | como no job (máximo de 360) |

Cada step roda em um processo separado: variável de ambiente criada em um step não passa para o seguinte. Para passar valor entre steps usa-se `$GITHUB_OUTPUT` (de memória).

## As formas do `uses` num step

| Forma | Exemplo | O que é |
|---|---|---|
| `{dono}/{repo}@{versão}` | `actions/checkout@v4` | action de um repositório |
| `./caminho` | `./.github/actions/minha` | action dentro do próprio repositório |
| `docker://imagem` | `docker://alpine:3.20` | roda uma imagem Docker |

A doc recomenda fixar o `@` num SHA de commit, porque tag pode ser movida.

## O `uses` num job: workflow reutilizável

O `uses` fica no nível do job, sem `steps`:

```yaml
jobs:
  chama-ci:
    uses: ./.github/workflows/ci-dotnet.yml      # ou dono/repo/.github/workflows/x.yml@v1
    with:
      configuracao: Release                       # entradas do workflow chamado
    secrets: inherit                              # repassa todos os segredos do chamador
```

- O chamado precisa ter `on: workflow_call`.
- O `with` vira `inputs` no chamado; não vira variável de ambiente, como acontece no `with` de um step.
- Não aceita expressões `${{ }}` no próprio `uses`.
- `secrets: inherit` passa todos os segredos que o chamador enxerga. Também é possível passar um por um.

## Equivalência com o GitLab (de memória)

| GitLab | GitHub |
|---|---|
| `script:` | `steps` (`run`) |
| `image:` do job | `container:` |
| `services:` | `services:` |
| `needs:` | `needs:` |
| `parallel:matrix` | `strategy.matrix` |
| `timeout:` | `timeout-minutes:` |
| `allow_failure: true` | `continue-on-error: true` |
| `include:` / componente | `uses:` de step (action) ou de job (workflow reutilizável) |
| `stages:` | não existe; a ordem vem só de `needs` |

No GitLab o job herda os artifacts do estágio anterior. No GitHub cada job roda num runner limpo: passar arquivo de um job para outro exige artifacts ou cache.
