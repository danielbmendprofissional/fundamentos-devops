# GitHub Actions: artifacts, cache e reaproveitamento

Fontes: [Artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts), [Cache](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching) e [Workflows reutilizáveis](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows).

## Artifacts

- São arquivos gerados por um workflow (binários, resultados de teste, logs, cobertura) que continuam existindo depois que o job termina.
- Servem para ver ou baixar depois da execução e para passar arquivos de um job para outro do mesmo workflow, já que cada job roda num runner limpo.
- Duas actions: `actions/upload-artifact` e `actions/download-artifact`.
- Apagar uma execução apaga os artifacts dela.
- Também dá para gerar attestations: declarações assinadas de proveniência do build (workflow, repositório, commit, evento), que podem incluir SBOM.

Exemplo (sintaxe de memória; a doc não traz YAML):

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: dotnet publish -c Release -o publicado
      - uses: actions/upload-artifact@v4
        with:
          name: api-publicada
          path: publicado/
          retention-days: 7

  deploy:
    needs: build                       # o build precisa ter terminado
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: api-publicada
          path: publicado/
      - run: ./deploy.sh publicado/
```

## Cache

Reaproveita arquivos que mudam pouco entre execuções (pacotes NuGet, npm); o job ainda precisa funcionar sem o cache. Artifact é o que o job produziu. Cache é o que ele baixa de novo se faltar.

`actions/cache`:

| Campo | Para que serve |
|---|---|
| `path` | o que guardar e restaurar (obrigatório) |
| `key` | identifica o cache (obrigatório, até 512 caracteres) |
| `restore-keys` | chaves de reserva, tentadas em ordem se a `key` exata não existir |
| saída `cache-hit` | `true` só se a `key` bateu exata |

- O cache é imutável: para atualizar, cria-se outro com `key` nova. Por isso se usa `hashFiles`, que gera a chave a partir do conteúdo de um arquivo: mudou o arquivo de dependências, mudou a chave, cache novo.
- **Hit:** a `key` bate exata e os arquivos são restaurados. **Miss:** tenta as `restore-keys` em ordem (a mais recente que casar por prefixo); no fim do job, se ele passou, salva um cache novo.
- **Quem enxerga qual cache:** a própria branch, a padrão e, em PR, a base. Não enxerga branch irmã, filha ou tag. Cache de `pull_request` fica preso ao PR.
- **Escrita:** só eventos de confiança (`push`, `workflow_dispatch`, `schedule` e outros) escrevem no cache da branch padrão. `pull_request_target`, `issue_comment` e `workflow_run` só leem, para evitar envenenamento de cache.
- **Limites:** cache sem uso por mais de 7 dias é removido; o padrão é 10 GB por repositório; cheio, apaga o menos usado recentemente.
- Os `setup-*` têm cache embutido; a doc lista o `setup-dotnet` para NuGet, sem exemplo.

Exemplo para a API (de memória):

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.nuget/packages
    key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
    restore-keys: |
      ${{ runner.os }}-nuget-

# alternativa, embutida no setup-dotnet:
- uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '10.0.x'
    cache: true
```

## Workflow reutilizável

Criar (arquivo em `.github/workflows/`; subpastas não são suportadas):

```yaml
# .github/workflows/ci-dotnet.yml
on:
  workflow_call:
    inputs:
      configuracao:
        required: true
        type: string                 # boolean, number ou string
    secrets:
      token_nuget:
        required: false
    outputs:
      versao:
        value: ${{ jobs.build.outputs.versao }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      versao: ${{ steps.v.outputs.versao }}
    steps:
      - uses: actions/checkout@v4
      - id: v
        run: echo "versao=1.0.${{ github.run_number }}" >> $GITHUB_OUTPUT
      - run: dotnet build -c ${{ inputs.configuracao }}
```

Chamar (o `uses` fica no nível do job, nunca dentro de `steps`):

```yaml
jobs:
  ci:
    uses: ./.github/workflows/ci-dotnet.yml      # ou dono/repo/.github/workflows/x.yml@v1
    with:
      configuracao: Release
    secrets:
      token_nuget: ${{ secrets.NUGET }}          # ou: secrets: inherit

  deploy:
    needs: ci
    runs-on: ubuntu-latest
    steps:
      - run: echo "Versão ${{ needs.ci.outputs.versao }}"
```

Regras:

- `uses` aceita `./.github/workflows/x.yml` (mesmo repositório), `dono/repo/.github/workflows/x.yml@ref` (outro repositório) e `$/.github/workflows/x.yml` (mesmo repositório e mesmo commit; indisponível no GitHub Enterprise Server). A forma local não aceita `@ref` nem expressões. Em outro repositório, o `@ref` pode ser SHA, tag ou branch; o SHA é o mais seguro.
- `secrets: inherit` repassa todos os secrets, dentro da mesma organização ou empresa.
- Secret de environment: quem define o `environment` é o workflow chamado; o chamador não pode usar essa chave, mas ainda precisa passar o secret. Se não passar, a referência vira string vazia, sem erro.
- Outputs: a saída do step vira saída do job, que vira saída do workflow (`on.workflow_call.outputs`), e o chamador lê em `needs.<job>.outputs.<nome>`.
- Matriz: um job com matriz pode chamar o reutilizável, gerando uma execução por valor (por exemplo `target: [dev, stage, prod]`). Em matriz, o output vem do último job bem-sucedido que definiu valor.
- Limites: até 10 níveis (o chamador do topo mais 9), sem ciclos. As permissões só se mantêm ou diminuem ao longo da cadeia. Os secrets só vão ao workflow chamado diretamente: em A > B > C, o C só recebe os de A se o B repassar.

## Action composta (de memória)

Uma composite action é uma action cujo conteúdo são vários steps (`runs: using: composite`), chamada com `uses` dentro de um step. O workflow reutilizável é chamado no job e traz jobs inteiros.

| | Action composta | Workflow reutilizável |
|---|---|---|
| Chamada | `uses` num step | `uses` num job |
| O que traz | uma sequência de steps | jobs completos, com runner, services, environment |

## Equivalência com o GitLab (de memória)

| GitLab | GitHub |
|---|---|
| `artifacts:` + `needs`/`dependencies` (o arquivo passa sozinho) | `upload-artifact` e `download-artifact` (passa explicitamente) |
| `cache:` com `key:files` | `actions/cache` com `hashFiles` |
| `include:` de um template central | `uses:` de workflow reutilizável (job) |
| componente com `spec:inputs` | `workflow_call` com `inputs` |
| `extends:` / `!reference` / âncoras | sem equivalente direto no YAML do workflow; o reaproveitamento é por workflow reutilizável ou action composta |
