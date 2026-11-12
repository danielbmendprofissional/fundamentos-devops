# GitHub Actions: variáveis, secrets e environments

Fontes: [Variáveis](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-variables), [Secrets](https://docs.github.com/en/actions/concepts/security/secrets) e [Deployments e environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments).

## Variáveis: dois tipos

### `env`: definida no próprio arquivo, em três níveis

O mais específico vence.

```yaml
env:                              # workflow inteiro
  DAY_OF_WEEK: Monday

jobs:
  greeting_job:
    runs-on: ubuntu-latest
    env:                          # só este job
      Greeting: Hello
    steps:
      - run: echo "$Greeting $First_Name. Hoje é $DAY_OF_WEEK"
        env:                      # só este step
          First_Name: Mona
```

### `vars`: variável de configuração, criada na interface do GitHub

Serve para definir uma vez e reaproveitar em vários workflows.

| Escopo | Onde criar |
|---|---|
| Organização | pelos donos da organização, com uma política de quais repositórios enxergam (não chega a repositórios privados no GitHub Free) |
| Repositório | Settings > Secrets and variables > Actions > Variables |
| Environment | Settings > Environments > o ambiente |

Lida com o contexto `vars`; se não existir, vem string vazia:

```yaml
env:
  URL_API: ${{ vars.URL_API }}
```

## `$NOME` ou `${{ }}`

- Dentro de um `run`, a variável é do shell: `$NOME` (bash) ou `$env:NOME` (PowerShell).
- Fora do shell (`if:`, `name:`, `runs-on:`, `with:`), usa-se contexto: `${{ env.NOME }}`, `${{ vars.NOME }}`.

```yaml
- if: ${{ env.DAY_OF_WEEK == 'Monday' }}      # contexto
  run: echo "Hoje é $DAY_OF_WEEK"             # shell
```

O GitHub define variáveis padrão em cada execução. A doc cita `RUNNER_OS` (e o contexto `runner.os`); o resto da lista (`GITHUB_SHA`, `GITHUB_REF` e outras) é de memória.

## Passar valor entre steps e jobs

- Entre steps do mesmo job: gravar no `GITHUB_ENV`.
- Entre jobs: definir `outputs` no job e ler via `needs`.

## Secrets

- Guardam informação sensível, em três escopos: organização, repositório e environment. O valor é criptografado antes de chegar ao GitHub.
- Só são lidos se forem incluídos explicitamente no workflow, como entrada de action ou variável de ambiente:

```yaml
steps:
  - run: ./deploy.sh
    env:
      TOKEN: ${{ secrets.TOKEN_DEPLOY }}      # sintaxe de memória
```

- **Mascaramento:** o GitHub esconde o valor se ele for impresso no log, mas não é garantido (o valor pode ser transformado), e o runner só mascara os segredos usados no job atual.
- **Boas práticas:**
  - dar o mínimo de permissão;
  - preferir deploy key ou conta de serviço, em vez de credencial pessoal;
  - preferir somente leitura quando bastar;
  - considerar um GitHub App, com permissões finas e token de vida curta, que não depende de um usuário.
- De memória: o `GITHUB_TOKEN` é um token gerado por execução, com as permissões do bloco `permissions`. Os segredos não são passados a workflows disparados por fork.

## Environments

Um environment é o alvo de deploy (hml, prod) que um job referencia. Nele ficam regras de proteção, secrets e variáveis.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: prod                 # ou: name: prod  e  url: https://api.exemplo.com
    steps:
      - run: ./deploy.sh
```

Regras de proteção, que precisam passar antes de o job rodar:

| Regra | O que faz |
|---|---|
| Revisores obrigatórios | até 6 pessoas ou times; uma aprovação basta; dá para impedir que quem iniciou aprove |
| Wait timer | atraso de 1 minuto a 30 dias; não conta como tempo cobrado |
| Branches e tags de deploy | sem restrição, só branches protegidas, ou padrões escolhidos (o `*` não casa `/`, então `release/*/*` para nomes com barra extra) |
| Regras personalizadas (prévia pública) | apps de terceiros que liberam o deploy; máximo de 6 por ambiente |
| Bypass de admin | ligado por padrão; pode ser desligado por ambiente |

Os secrets e as variáveis do environment só chegam aos jobs que referenciam aquele environment. Se há revisor obrigatório, o job não enxerga o secret até alguém aprovar.

Limites por plano: em repositório privado, revisores obrigatórios e wait timer exigem plano pago; no Free só valem em repositório público. Runners auto-hospedados não são isolados, então os secrets do environment pedem o mesmo cuidado dos demais.

## Equivalência com o GitLab (de memória)

| GitLab | GitHub |
|---|---|
| `variables:` no YAML | `env:` |
| variável de CI/CD na UI (projeto, grupo) | `vars` (repositório, organização) |
| variável mascarada ou protegida | secret |
| variável com escopo de environment | variável e secret do environment |
| `environment:` do job | `environment:` do job |
| ambiente protegido e aprovação de deploy | revisores obrigatórios e wait timer (plano pago em privado) |
| `secrets:` de cofre externo, `id_tokens` | OIDC com o provedor de nuvem |

No GitHub variável e secret são dois recursos separados (`vars` e `secrets`). A precedência entre organização, repositório e environment não está nessas páginas; de memória, o mais específico vence.
