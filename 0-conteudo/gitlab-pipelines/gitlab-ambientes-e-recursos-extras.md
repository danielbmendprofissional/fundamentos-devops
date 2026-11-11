# GitLab: ambientes, Auto DevOps, Pages, feature flags e cluster

## Ambientes (environments)

Fonte: [Environments](https://docs.gitlab.com/ci/environments/).

Um ambiente é o destino de implantação (desenvolvimento, homologação, produção). Serve para manter configurações separadas e registrar o que foi implantado em cada lugar.

```yaml
deploy-hml:
  stage: deploy
  script: ./deploy.sh hml
  environment:
    name: staging
    url: https://hml.exemplo.com
```

- **`name`:** se o ambiente ainda não existe quando o pipeline roda, é criado. Pode ser dinâmico (com variáveis como `CI_COMMIT_REF_SLUG`), com um prefixo estático para agrupar na interface.
- **`url`:** aparece nos merge requests, na lista de ambientes e nas implantações. Pode ser preenchido dinamicamente a partir de um relatório dotenv gerado pelo job.
- **Deployment:** o registro criado por um job de implantação; só aparece na lista do ambiente depois que o job o cria.

### Estáticos e dinâmicos

| | Estático | Dinâmico |
|---|---|---|
| Uso | reutilizado por implantações sucessivas | criado no pipeline para uma implantação e depois parado ou excluído |
| Nome | fixo (`staging`, `production`) | baseado em variáveis, geralmente da branch |
| Exemplo | homologação, produção | review apps |

Estados: disponível, parando (o job `on_stop` começou) e parado.

### Review app (exemplo meu)

Um ambiente por merge request, que sobe a versão da branch e é apagado depois.

```yaml
deploy-review:
  stage: deploy
  script: ./deploy-review.sh $CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.hml.exemplo.com
    on_stop: stop-review
    auto_stop_in: 1 week          # para sozinho em 1 semana
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

stop-review:
  stage: deploy
  script: ./destroy-review.sh $CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop                  # marca o job como o que para o ambiente
  when: manual
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
```

### Quem hospeda a aplicação?

O GitLab **não hospeda** a aplicação do review app. O que ele faz:

- **Registra** o ambiente e as implantações (histórico, URL, quem fez, de qual commit).
- **Mostra** o link no merge request e na lista de ambientes.
- **Dispara** os jobs: o `script` de implantação e o job que para o ambiente (`on_stop`, `auto_stop_in`).

Quem sobe a aplicação de verdade é o **seu script**, num lugar que é seu: um cluster Kubernetes, uma VM por SSH, um serviço de nuvem. Tudo aquilo que o `./deploy-review.sh` faz (Helm, `kubectl apply`, `docker compose`) é com você, e o job de parada precisa **destruir** o que foi criado: o `auto_stop_in` só dispara esse job, ele não apaga nada por conta própria.

```text
MR aberto ──> job deploy-review ──(seu script)──> cluster/VM: sobe a versão da branch
                       │
                       └─ GitLab registra o ambiente review/branch e põe o link no MR
auto_stop_in vence ──> GitLab dispara stop-review ──(seu script)──> destrói o que foi criado
```

"É tão fácil assim, subir o deploy da API ou do front completinho, com HTTPS?" A parte do GitLab é fácil. O difícil é a **infraestrutura por baixo**: um cluster, um Ingress com DNS curinga (`*.hml.exemplo.com`) e certificado curinga (ou cert-manager), para cada branch ganhar `https://branch.hml.exemplo.com`. Com o Auto DevOps sobre um cluster Kubernetes, essa montagem vem pronta (próxima seção). Para sites estáticos, o Pages (mais abaixo) faz o HTTPS sozinho, sem infraestrutura sua.

- **`on_stop`:** no job de implantação, aponta para o job que encerra o ambiente. Vários `on_stop` rodam em paralelo, sem ordem, e só os de jobs de implantação bem-sucedidos.
- **`action: stop`:** o job de parada precisa estar num stage (ou com `needs`) que permita sua execução; senão o ambiente não é parado.
- **Parada forçada:** para o ambiente sem rodar o `on_stop`; não cancela o job pendente nem limpa recursos criados pelo job.
- **`auto_stop_in`:** para o ambiente depois de um período. Um worker roda por volta de uma vez por hora, então a parada pode atrasar. `access` e `prepare` reiniciam o prazo, `verify` não. `never` cancela a expiração.

### Tier e proteção

- **`deployment_tier`:** `development`, `testing`, `staging`, `production` e `other`. Sem a palavra-chave, o GitLab deduz pelo nome por expressão regular; a doc recomenda declará-la. Não se define pela interface. Permite proteções de ambiente em nível de grupo entre projetos com nomes diferentes.
- **Ambiente protegido:** restringe quem pode implantar ou alterar. Quem não tem acesso não pode pará-lo nem excluí-lo. O fluxo de aprovação não foi lido (a página só remete).

### Rollback

- **Manual:** em Operate > Environments, escolhe-se o ambiente, e ao lado de cada implantação há **Re-deploy** (repete a implantação atual) e **Rollback** (volta a uma implantação anterior bem-sucedida). O rollback **cria uma nova implantação apontando para o commit anterior** e executa de novo o job de implantação daquele commit (com um ID de job próprio). Portanto:
  - **não** é um `git revert`: o código no repositório não muda;
  - **não** roda o pipeline inteiro: só os jobs de implantação. Se a implantação depende de artifacts de jobs anteriores (por exemplo um `plan` do Terraform separado do `apply`), é preciso executar esses jobs à mão na página de pipelines;
  - o processo de implantação precisa estar no `script` do job (um rollback que depende de passos externos não funciona);
  - se a proteção contra jobs de implantação desatualizados estiver ativa, os botões podem ficar ocultos ou desabilitados (seção de segurança de implantação, não lida).
- **Automático** (Ultimate, desligado por padrão, segundo a página de ambientes): um alerta crítico com a chave `gitlab_environment_name` reimplanta a implantação bem-sucedida mais recente; não ocorre se já houver uma em execução, e no máximo um rollback a cada 3 minutos. Esse assunto não apareceu na página de implantações.

**Está no plano gratuito?** A página de implantações lista Free, Premium e Ultimate para o conjunto de recursos, mas não detalha o rollback por plano; então não dá para afirmar pela doc. De memória, o rollback manual está disponível em todos os planos e o automático é do Ultimate. As **aprovações de implantação** (*deployment approvals*) só apareceram como link e não foram lidas.

### Variáveis por ambiente

A variável pode valer só para `production` ou `review/*` (ver [gitlab-artifacts-cache-variaveis.md](gitlab-artifacts-cache-variaveis.md)). A variável `$CI_ENVIRONMENT_NAME` não foi encontrada na página lida; ela cita `CI_ENVIRONMENT_SLUG`.

Não foram lidos: ambientes protegidos, implantações, review apps (páginas), a API de ambientes, os painéis do Kubernetes, o Environments Dashboard e os terminais web.

## Auto DevOps

Fonte: [Auto DevOps](https://docs.gitlab.com/topics/autodevops/).

Um conjunto de pipelines de CI/CD pré-configurados, que detecta a linguagem e o framework e leva o código a uma aplicação pronta para produção. Funciona com qualquer cluster Kubernetes, sem custo adicional.

| Stage | Recursos |
|---|---|
| Build | Auto build; Auto dependency scanning |
| Test | Auto test; Auto browser performance testing; Auto code quality; Auto container scanning; Auto code intelligence (descontinuado) |
| Deploy | Auto review apps; Auto deploy |
| Secure | Auto DAST; Auto SAST; Auto secret detection |

A página não menciona um "Auto Monitoring" (o monitoramento aparece só como tema relacionado, o Prometheus).

**Requisitos:** um Dockerfile ou buildpack correspondente; Kubernetes para o deploy (sem ele, o Auto DevOps faz build e teste, mas não deploy); domínio base e estratégia de deployment (recomendados). O runner só é citado em links.

**Habilitar (por projeto):** ser Maintainer ou Owner; o projeto **não** pode ter `.gitlab-ci.yml` (se existir, ele tem precedência); em Configurações > CI/CD > Auto DevOps, marcar o pipeline Auto DevOps como padrão, informar o domínio base e a estratégia; conferir em Build > Pipelines. Também por grupo (Owner) e por instância (administrador).

Vale a pena: começar rápido; ter segurança e qualidade já integradas; usar Kubernetes; usar o registro do GitLab. Não vale: dependência de registro privado de terceiros; Helm atrás de proxy; já ter `.gitlab-ci.yml`; sem Dockerfile nem buildpack compatível.

### Exemplo de Auto DevOps

Para uma API .NET com `Dockerfile` e um cluster conectado (agent):

**Opção 1: ligar pela interface, sem `.gitlab-ci.yml`.** Em Configurações > CI/CD > Auto DevOps, marcar o pipeline Auto DevOps como padrão, informar o domínio base (por exemplo `apps.exemplo.com`) e a estratégia. O resultado, no estilo da tabela de stages acima (os nomes exatos dos jobs são de memória):

```text
build ──> test ──> [code quality, SAST, secret detection, dependency scanning, container scanning]
   │
   ├─ em merge request:  review   ──> https://<branch>.apps.exemplo.com   (review app)
   └─ na branch padrão:  production (deploy contínuo)
```

**Opção 2: já tendo um `.gitlab-ci.yml`, incluir o template** e ajustar com variáveis:

```yaml
include:
  - template: Auto-DevOps.gitlab-ci.yml     # (visto no include)

variables:
  KUBE_INGRESS_BASE_DOMAIN: apps.exemplo.com   # domínio base do Ingress
  STAGING_ENABLED: "true"                      # vai primeiro para staging; produção vira job manual (production_manual)
  TEST_DISABLED: "true"                        # pula o job de testes
  SAST_DISABLED: "true"                        # pula os jobs de SAST
```

Variáveis do Auto DevOps (página de variáveis):

| Variável | O que faz |
|---|---|
| `KUBE_INGRESS_BASE_DOMAIN` | domínio base do Ingress; pode ser configurado por cluster |
| `STAGING_ENABLED` | deploy automático em staging; produção vira job manual |
| `CANARY_ENABLED` | cria o job manual `canary` e mantém o `production_manual` (Premium, Ultimate) |
| `INCREMENTAL_ROLLOUT_MODE` | `manual` (jobs de 10%, 25%, 50% e 100%) ou `timed` (5 minutos entre as etapas); Premium, Ultimate |
| `POSTGRES_ENABLED` | `true` ativa o deploy automático do PostgreSQL; não é mais definida por padrão |
| `TEST_DISABLED` | se presente, o job de testes não é criado |
| `SAST_DISABLED` | `true` não cria os jobs de SAST |
| `AUTO_DEVOPS_BUILD_IMAGE_CNB_BUILDER` | builder do Cloud Native Buildpacks (padrão `heroku/buildpacks:22`) |
| `AUTO_DEVOPS_BUILD_IMAGE_EXTRA_ARGS` | argumentos extras para o `docker build` |
| `AUTO_DEVOPS_BUILD_IMAGE_FORWARDED_CI_VARIABLES` | variáveis de CI/CD repassadas ao build |
| `AUTO_DEVOPS_BUILD_IMAGE_CNB_PORT` | porta exposta pela imagem (padrão 5000; `false` não expõe porta) |
| `AUTO_DEVOPS_BUILD_IMAGE_CONTEXT` | diretório de contexto do build (padrão a raiz) |

Por padrão, o Auto DevOps faz **deploy contínuo**: cada pipeline na branch padrão vai para `production`. A variável `AUTO_DEVOPS_PLATFORM_TARGET` não apareceu na página. Fonte: [Auto DevOps, variáveis](https://docs.gitlab.com/topics/autodevops/cicd_variables/).

## GitLab Pages

Fonte: [GitLab Pages](https://docs.gitlab.com/user/project/pages/).

Publica sites estáticos direto de um repositório (Hugo, Jekyll, Gatsby, HTML/CSS/JS/Wasm). Não faz processamento dinâmico no servidor (PHP, ASP). Infraestrutura do GitLab, sem custo adicional; deploy via CI/CD.

```yaml
pages:
  stage: deploy
  script:
    - mkdir -p public
    - cp index.html public/
  pages: true                  # marca o job como deploy do Pages
```

- A pasta padrão é `public`. A doc diz que dá para definir outra, mas essa seção não foi lida; o jeito antigo, com `artifacts: paths: [public]`, é de memória (a página não o mostrou, nem o `pages:publish`).
- Domínio padrão `*.gitlab.io` (únicos por projeto, evitam cookies compartilhados entre projetos do mesmo grupo). Domínios customizados exigem acesso ao DNS.
- HTTPS automático no domínio padrão; nos customizados, com certificado (Let's Encrypt obtido e renovado automaticamente).
- No GitLab.com o site é público por padrão; restringe-se com o Pages Access Control.
- **Parallel deployments** (review apps de site):

```yaml
deploy-pages-review-app:
  stage: deploy
  script:
    - npm run build
  pages:
    path_prefix: '_staging'
```

Se vários jobs usam o mesmo `path_prefix`, o último concluído é o publicado.

- **Expiração:** `pages.expire_in`; tarefas a cada 10 minutos param e depois excluem os deployments expirados.

Para quê, no seu contexto (meu): documentação, o site estático do projeto ou o relatório de cobertura publicado.

### Um painel de publicação que se atualiza a cada pipeline (meu)

Sim, dá para ter um HTML que o próprio pipeline regenera a cada execução e publica no Pages. O job escreve o `index.html` com as informações do pipeline (as variáveis `CI_COMMIT_SHORT_SHA`, `CI_COMMIT_TITLE`, `CI_PIPELINE_URL`, `GITLAB_USER_NAME` e `CI_COMMIT_REF_NAME` são de memória):

```yaml
pages:
  stage: deploy
  script:
    - mkdir -p public
    - |
      cat > public/index.html <<EOF
      <!doctype html>
      <html lang="pt-BR">
      <head><meta charset="utf-8"><title>Última publicação</title></head>
      <body>
        <h1>Última publicação</h1>
        <ul>
          <li>Commit: $CI_COMMIT_SHORT_SHA - $CI_COMMIT_TITLE</li>
          <li>Autor: $GITLAB_USER_NAME</li>
          <li>Branch: $CI_COMMIT_REF_NAME</li>
          <li>Pipeline: <a href="$CI_PIPELINE_URL">#$CI_PIPELINE_ID</a></li>
          <li>Data (UTC): $(date -u +%Y-%m-%dT%H:%M:%SZ)</li>
        </ul>
        <h2>Últimos commits</h2>
        <pre>$(git log --oneline -10)</pre>
      </body>
      </html>
      EOF
  pages: true
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Dá para copiar para `public/` também o relatório de cobertura em HTML ou o resultado dos testes (usando `needs` e `artifacts` do job que os gerou) e linkar da página.

Cuidados:

- Cada publicação **substitui** a anterior (de memória); o "histórico" que a página mostra é o que você escreve nela (o `git log`, por exemplo), e não um registro das execuções.
- No GitLab.com o site é **público** por padrão: não publique nada sensível, ou ative o Pages Access Control.
- Para uma página por MR (um painel de review), os parallel deployments com `path_prefix` acima.

## Feature flags

Fonte: [Feature flags](https://docs.gitlab.com/operations/feature_flags/).

Interruptores para ligar e desligar uma funcionalidade sem novo deploy: entrega em lotes menores, teste para subconjuntos de usuários, separa a entrega técnica do lançamento (ver feature flags em [../devops-cicd/ci-cd-fundamentos.md](../devops-cicd/ci-cd-fundamentos.md)).

**Como funciona:** o GitLab oferece uma API compatível com o **Unleash** (projeto open source). Você cria as flags no GitLab; a aplicação usa um **cliente Unleash** (há SDKs para várias linguagens) para consultar a lista. Cada chamada pode ter um valor padrão, usado quando a flag não existe.

**Estratégias de rollout:**

| Estratégia | O que faz |
|---|---|
| All users | ativa para todos |
| Percent Rollout | ativa para uma fração das visualizações; a base de consistência pode ser ID do usuário, ID de sessão, aleatória ou "disponível"; a aleatória não garante comportamento consistente |
| Percent of Users | ativa para uma fração de usuários autenticados (a doc recomenda o Percent Rollout por ser mais flexível) |
| User IDs | lista de usuários-alvo por ID separado por vírgula; não precisam ser usuários do GitLab |
| User List | listas reutilizáveis, pela interface ou pela API |

Uma flag pode ter várias estratégias, cada uma com os ambientes onde vale (por padrão, todos).

### É a mesma ideia de feature flag do CI/CD?

Sim: é a mesma ideia de [../devops-cicd/ci-cd-fundamentos.md](../devops-cicd/ci-cd-fundamentos.md), com um painel pronto. Um ponto importante: **o GitLab não marca código como flag**. A flag é um interruptor guardado no GitLab, e **o seu código precisa perguntar** se ele está ligado. Exemplo (meu, ilustrativo):

```csharp
// Program.cs da API: a rota nova só existe quando a flag está ligada
app.MapGet("/pedidos/v2", (IUnleash unleash) =>
    unleash.IsEnabled("pedidos_v2")
        ? Results.Ok(PedidosV2.Listar())
        : Results.NotFound());                       // flag desligada: a rota "não existe"
```

```csharp
// alterar um método existente: o caminho antigo continua
public Pedido Calcular(Pedido p) =>
    _unleash.IsEnabled("novo_calculo_frete") ? CalcularNovo(p) : CalcularAntigo(p);
```

Fluxo: você faz o deploy com a flag **desligada**, o código novo vai para produção sem ser executado, e depois liga a flag no GitLab (Deploy > Feature flags), sem novo deploy. Quem pode ligar é definido pelo papel mínimo do projeto (Developer, Maintainer, Owner ou ninguém). A página lida **não** mostra um fluxo de aprovação para ligar uma flag; o controle é por permissão.

### Estratégias de rollout, em palavras simples

A estratégia responde: **para quem a funcionalidade está ligada?**

| Estratégia | Em linguagem simples | Exemplo (meu) |
|---|---|---|
| All users | para todo mundo | lançamento final da funcionalidade |
| Percent Rollout | para uma porcentagem das visualizações | 10% das requisições veem o novo cálculo de frete |
| Percent of Users | para uma porcentagem dos usuários logados | 10% dos clientes logados |
| User IDs | só para estes usuários | a equipe interna (IDs 1, 2, 3) testa em produção |
| User List | só para os de uma lista reutilizável | a lista "beta testers" |

A "base de consistência" do Percent Rollout decide **como o sorteio é feito**: por ID do usuário (o mesmo usuário vê sempre a mesma coisa), por ID de sessão, "disponível" (usa o ID do usuário se estiver logado, senão o da sessão, senão o aleatório) ou aleatória (a mesma pessoa pode ver ligado numa requisição e desligado na seguinte; a doc avisa que não é consistente).

Como é usado na prática, em ondas, ligando cada vez mais gente:

```text
dia 1:  User IDs = equipe interna                 (testa em produção)
dia 2:  Percent Rollout 5%  em production         (se der erro, desliga)
dia 4:  Percent Rollout 25%
dia 7:  All users
```

Por ambiente: ligada para todos em `staging` e a 10% em `production`, na mesma flag.

### Unleash e GitLab: casam?

Casam, e é a forma de usar. **Unleash** é um projeto open source de feature flags, com um protocolo e SDKs para várias linguagens. O GitLab **implementa essa API do Unleash dentro dele**: você cria as flags no GitLab, e a sua aplicação usa um **cliente (SDK) Unleash** apontando para o GitLab, como se o GitLab fosse o servidor do Unleash.

```text
GitLab (cria e liga as flags)
   ▲
   │  consulta periódica da lista de flags (API compatível com Unleash)
   │
SDK Unleash dentro da sua API .NET ──> guarda a lista em memória ──> IsEnabled("pedidos_v2")
```

Configuração (os três dados de Deploy > Feature flags > Configure): a URL da API (`.../api/v4/feature_flags/unleash/<project_id>`), o Instance ID e o Application name (o nome do ambiente, como `production`). O que "não casa" são duas peças auxiliares: o **Unleash Proxy** está descontinuado (fim de vida em 26/11/2026) e o **Unleash Edge**, o substituto, não é compatível com o GitLab. Isso só pesa em escala grande: sem proxy, cada instância da aplicação consulta o GitLab direto, e a API tem limite por IP (429).

**Criar:** Deploy > Feature flags > nova flag. Nome começando por letra, com minúsculas, números, `_` ou `-`, sem terminar em `-` ou `_`. Limite por projeto: 50 (Free), 150 (Premium), 200 (Ultimate).

**A aplicação precisa de três dados** (Deploy > Feature flags > Configure): a **URL da API**, o **Instance ID** e o **Application name** (o nome do ambiente, como `production`, e não o do app).

Exemplo (da doc, em Ruby, dados fictícios):

```ruby
client = Unleash::Client.new(
  url: 'https://gitlab.example.com/api/v4/feature_flags/unleash/<project_id>',
  app_name: 'production',
  instance_id: '<instance_id>'
)
contexto = Unleash::Context.new
contexto.user_id = '123'
puts 'Funcionalidade ativa' if client.is_enabled?('minha_feature', contexto)
```

Em .NET existe um cliente Unleash para a aplicação (de memória; a página lida só mostrou Ruby).

Atenções: o Unleash Proxy está descontinuado (fim de vida previsto para 26/11/2026); o Unleash Edge, o substituto, não é compatível com o GitLab; a API tem limite de requisições por IP (muitos clientes atrás do mesmo IP podem receber 429).

## Cluster: o agent do GitLab para Kubernetes

Fonte: [GitLab agent for Kubernetes](https://docs.gitlab.com/user/clusters/agent/).

É a solução do GitLab para conectar clusters Kubernetes ao GitLab. Dois componentes:

- **agentk:** roda dentro do cluster e mantém a conexão segura com o GitLab.
- **KAS** (GitLab agent server): o lado do GitLab que gerencia a conexão com os clusters.

Permite: comunicar-se com clusters atrás de firewall ou NAT; acessar a API do cluster em tempo real; receber eventos do cluster; manter um cache de objetos Kubernetes com baixa latência.

**Modelos de deploy:**

| Modelo | Como funciona | Observação da doc |
|---|---|---|
| **GitOps (pull-based)** | o **Flux** busca mudanças num repositório Git e as aplica no cluster | o recomendado |
| **CI/CD (push-based)** | os pipelines enviam comandos ao cluster pela API do Kubernetes | modelo de segurança mais fraco; não usar em produção |

O agente é instalado no cluster e sempre registrado em um projeto do GitLab; depois a conexão pode ser compartilhada com outros projetos, grupos e usuários. Recomenda-se um agente por cluster. Os *receptive agents* (Ultimate, self-managed, desde a 17.4) deixam o GitLab se conectar ao agente por uma URL, para quando o cluster não consegue abrir conexão até o GitLab.

De memória (a página lida não cobre; confira na doc de instalação e do fluxo de CI/CD): a configuração do agente fica em `.gitlab/agents/<nome>/config.yaml`, com `ci_access` listando os projetos e grupos autorizados; no pipeline, o contexto do `kubectl` vem do agente.

```yaml
deploy:
  image: bitnami/kubectl
  script:
    - kubectl config use-context meu-grupo/meu-projeto:meu-agente   # contexto do agente (de memória)
    - kubectl apply -f k8s/
```

Ligação com o Kubernetes: o `kubectl apply` do pipeline é o deploy declarativo de [../kubernetes/kubernetes-objetos-e-controllers.md](../kubernetes/kubernetes-objetos-e-controllers.md), e o RBAC de [../kubernetes/kubernetes-seguranca-basica.md](../kubernetes/kubernetes-seguranca-basica.md) limita o que o agente pode fazer.

Não foram lidos: a instalação passo a passo, o `config.yaml`, o `ci_access`, o uso no pipeline e a migração da integração antiga baseada em certificado. Versões do Kubernetes suportadas citadas pela página: 1.34 a 1.36.
