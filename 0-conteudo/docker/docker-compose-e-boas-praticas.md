# Docker: configuração, Compose e boas práticas

## Variáveis de ambiente

Não tem nem o que dissertar: é um conceito para não ter segredos hardcoded em código ou container. As chaves ficam com os valores em variáveis de ambiente.

Ponto de atenção: variável de ambiente evita segredo no código, mas **não é cofre**. A doc do Docker diz que, ao injetar senhas e chaves por variável de ambiente, "you risk unintentional information exposure", e que elas podem aparecer em logs de debug. Para segredo de verdade, o Compose entrega *secrets* como arquivo em `/run/secrets/<nome>`. E `ARG`/`ENV` usados no build "persist in the final image", ou seja, ficam gravados na imagem.

Exemplo (meu) no Compose, com um `.env` ao lado do arquivo:

```yaml
services:
  api:
    image: minha-api
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__Default: ${DB_CONNECTION}   # o valor vem do .env ou do ambiente
```

```text
# .env (não vai para o Git)
DB_CONNECTION=Host=postgres;Database=app;Username=app;Password=exemplo
```

### Variável de ambiente ou secret?

| Tipo de dado | Onde vai | Exemplo na API |
|---|---|---|
| Configuração não sensível | variável de ambiente (`.env` / `environment:`) | `ASPNETCORE_ENVIRONMENT`, nível de log, host do banco, portas, feature flags |
| Dado sensível de verdade | `secret` | senha do Postgres, token do NuGet privado, chave do RabbitMQ, certificado |

Critério: se vazar num log ou numa tela de depuração dá problema, é secret. Se não dá, é variável.

Exemplo (meu) com os dois lado a lado:

```yaml
services:
  api:
    image: minha-api
    environment:
      ASPNETCORE_ENVIRONMENT: Production         # configuração: variável
      Logging__LogLevel__Default: Information    # configuração: variável
    secrets:
      - db_password                              # sensível: secret
  postgres:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
secrets:
  db_password:
    file: ./db_password.txt                      # fora do Git
```

O secret só adianta se a aplicação sabe ler o arquivo. O Postgres lê por causa do `POSTGRES_PASSWORD_FILE`. No ASP.NET Core existe o provedor **Key-Per-File**, que a doc da Microsoft diz ser usado em cenários Docker: ele usa os arquivos de uma pasta como configuração (nome do arquivo = chave, conteúdo = valor; `__` no nome vira `:`).

Exemplo (meu) no `Program.cs` da API:

```csharp
builder.Configuration.AddKeyPerFile(directoryPath: "/run/secrets", optional: true);

var senha = builder.Configuration["db_password"];   // conteúdo do arquivo /run/secrets/db_password
```

Ressalva (minha, não verificada na doc): no laboratório local, variável com senha de mentira basta; em servidor ou pipeline, o ideal costuma ser o gerenciador de segredos da plataforma.

Fontes: [Microsoft, configuração do ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/).

Fontes: [Compose, secrets](https://docs.docker.com/compose/how-tos/use-secrets/) e [Docker, build secrets](https://docs.docker.com/build/building/secrets/).

## Docker Compose

O Dockerfile é sobre uma aplicação: ele **constrói uma imagem**. O Compose é o arquivo em que manuseamos a **execução** de N serviços diferentes: healthcheck, dependência, configuração dos volumes, redes, etc. Não é um "Dockerfile maior", são ferramentas diferentes. O Compose pode apontar para um Dockerfile (`build:`) ou para uma imagem pronta (`image:`).

Exemplo (meu): a API é construída pelo Dockerfile (`build:`) e o banco vem de uma imagem pronta (`image:`).

```yaml
services:
  api:
    build: .                  # usa o Dockerfile desta pasta
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
  postgres:
    image: postgres:17        # imagem pronta do registry
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

## Boas práticas

1. **Imagem com apenas o necessário.** Multi-stage build e imagem base enxuta: menos tamanho, menos superfície de ataque.
2. **Um processo por container.** "Each container should have only one concern." É uma regra prática, não absoluta. Cada container (API, banco, broker) escala, reinicia e atualiza sozinho, e os logs não se misturam.
3. **Usuário não-root.** "If a service can run without privileges, use `USER` to change to a non-root user." Se alguém explorar uma falha na aplicação, o processo não entra com privilégio máximo.
4. **Healthcheck.** O Docker só sabe se o processo está vivo, não se a aplicação está funcionando. O `HEALTHCHECK` "tells Docker how to test a container to check that it's still working", e o status passa de `starting` para `healthy` ou `unhealthy` (padrão: intervalo 30s, timeout 30s, 3 tentativas). No Compose, `depends_on` sozinho só espera o container estar rodando ("Compose does not wait until a container is ready"); para esperar a saúde de verdade se usa `condition: service_healthy`.
5. **Tag fixa na imagem base.** "Image tags are mutable": uma tag pode passar a apontar para outra imagem, e então o build deixa de ser reprodutível. Para fixar de verdade, usa-se o digest.
6. **`.dockerignore`.** Exclui do contexto de build o que não é relevante (`bin/`, `obj/`, `.git`, arquivos de segredo): build mais rápido, melhor cache e nada de segredo indo para dentro da imagem.
7. **Logs em stdout/stderr.** Por padrão o `docker logs` mostra o `STDOUT` e o `STDERR` do container. A aplicação escreve na saída padrão e a plataforma coleta; gravar em arquivo dentro do container perde tudo quando ele morre.

### Exemplos das boas práticas (meus)

Healthcheck no Compose, com a API esperando o banco ficar saudável:

```yaml
services:
  api:
    build: .
    depends_on:
      postgres:
        condition: service_healthy
  postgres:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

Healthcheck no Dockerfile (pressupõe `curl` na imagem e um endpoint `/health`: confirme):

```dockerfile
HEALTHCHECK --interval=30s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

Tag fixa e digest:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet                     # sem tag = latest (evitar)
FROM mcr.microsoft.com/dotnet/aspnet:10.0                # tag fixa
FROM mcr.microsoft.com/dotnet/aspnet:10.0@sha256:<hash>  # digest: fixa de verdade
```

Usuário não-root (confirme se a imagem base traz esse usuário):

```dockerfile
USER app
```

Logs em stdout/stderr: a API escreve no console e eu leio por fora.

```bash
docker logs -f --tail 100 api
```

Fontes: [Docker, build best practices](https://docs.docker.com/build/building/best-practices/), [Dockerfile reference](https://docs.docker.com/reference/dockerfile/), [Compose, startup order](https://docs.docker.com/compose/how-tos/startup-order/) e [Docker, logging](https://docs.docker.com/engine/logging/).
