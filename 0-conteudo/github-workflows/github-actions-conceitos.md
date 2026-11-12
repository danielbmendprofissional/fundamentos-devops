# GitHub Actions: o que é e os conceitos básicos

Fonte: [Understanding GitHub Actions](https://docs.github.com/en/actions/get-started/understand-github-actions). A página não traz YAML; os exemplos e as comparações com o GitLab são de memória.

## O que é

O GitHub Actions é tanto uma plataforma de CI/CD quanto uma forma de realizar tarefas disparadas por eventos do repositório, não necessariamente ligadas ao fluxo de CI/CD (por exemplo, rotular uma issue nova). Nisso ele tem um "plus" em relação ao GitLab.

## Os conceitos

- **Workflow:** é o processo de pipeline. Fica num arquivo YAML em `.github/workflows`.
- **Evento:** é a trigger do workflow (push, pull request, issue criada, agendamento, chamada de API, disparo manual).
- **Job:** o conceito é bem parecido com o do GitLab. É um conjunto de passos que roda no mesmo runner. Por padrão os jobs rodam em paralelo, a menos que dependam de outros.
- **Step:** cada passo de um job. Pode ser um comando (`run`) ou uma action pronta (`uses`).
- **Action:** uma peça reutilizável que faz uma tarefa específica (instalar o .NET, baixar o código). É um template de **um passo**, não de um job. Fica mais perto de um componente do GitLab do que de um `.build`. Pode ser criada por você ou pega pronta no GitHub Marketplace.
- **Runner:** é o mesmo esquema do GitLab: o servidor que roda o job, em máquinas Linux, Windows ou macOS (do GitHub ou próprias).

Resumo: um evento dispara um workflow que tem N jobs, cada job roda em 1 runner e tem N steps, e cada step é um `run` ou um `uses`.

```text
EVENTO (push, pull request, agendamento, manual)
   │
   ▼
WORKFLOW (um arquivo YAML)
   ├─ JOB build ──> RUNNER 1: step 1, step 2, step 3
   └─ JOB test  ──> RUNNER 2: step 1, step 2   (só começa depois do build, se tiver needs)
```

## Exemplo: o menor workflow da API .NET

```yaml
# .github/workflows/ci.yml
name: CI

on: push                          # o evento que dispara

jobs:
  build:                          # nome do job
    runs-on: ubuntu-latest        # em qual runner
    steps:
      - uses: actions/checkout@v4          # action pronta: baixa o código
      - uses: actions/setup-dotnet@v4      # action pronta: instala o .NET
        with:
          dotnet-version: '10.0.x'
      - run: dotnet build -c Release       # comando de shell
      - run: dotnet test
```

## run e uses

- `run` é uma lista de comandos de shell.
- `uses` chama uma action. Uma action pode ter comandos de shell dentro (de memória, chama-se composite action; existem também actions em JavaScript e em container Docker).
- GitLab e GitHub fazem o mesmo processo, cada um do seu jeito: no GitLab o `script` fica no job e o reaproveitamento vem de `include`/`extends`/componentes; no GitHub o reaproveitamento de um passo vem do `uses`.

## Diferenças em relação ao GitLab

| | GitLab CI/CD | GitHub Actions |
|---|---|---|
| Arquivo | um `.gitlab-ci.yml` na raiz | vários arquivos, um por workflow, em `.github/workflows/` |
| Arquivo central | `include` chama os demais | não é automático (veja abaixo) |
| Ordem | stages | sem stages: paralelo por padrão, ordem com `needs` (de memória) |
| Unidade pequena | `script` | step (`run` ou `uses`) |
| Reaproveitar | `include`, `extends`, componentes | actions; para job ou pipeline inteiro, workflows reutilizáveis (de memória, passo 5) |
| Disparo | commit, MR, agendamento, manual | eventos, mais variados (issue, rótulo, release...) |

### Arquivo central que chama os demais

O GitHub não faz isso sozinho (de memória):

- todo arquivo em `.github/workflows/` com `on:` roda por conta própria, quando o evento dele acontece; não existe arquivo raiz;
- um arquivo só chama outro se o chamado for declarado como reutilizável (gatilho `workflow_call`) e o central usar `uses:` apontando para ele.

## Espelho do GitLab

O GitHub é um espelho de leitura do GitLab. O pipeline oficial é o do GitLab.
