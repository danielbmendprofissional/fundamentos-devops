# Docker: instruções do Dockerfile

## Bloco 1: construir a base (FROM, WORKDIR, COPY, ADD)

### FROM

Define a imagem base do Dockerfile e abre um estágio de build. A melhor prática é usar uma imagem oficial, com uma versão atual e fixa, sem `latest` (sem tag nem digest, o builder assume `latest`).

No multi-stage usam-se dois ou mais `FROM`. Cada um começa um estágio novo, e o `AS` dá nome a ele. O `COPY --from=<nome>` busca arquivos de um estágio anterior.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build       # estágio 1: SDK, compila
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:10.0              # estágio 2: só o runtime
WORKDIR /app
COPY --from=build /app .                               # pega só o publish do estágio 1
```

Detalhes da doc: `ARG` é a única instrução que pode vir antes do `FROM`, e existe a flag `--platform`.

### WORKDIR

É a pasta de trabalho para as instruções seguintes do Dockerfile (`RUN`, `CMD`, `COPY`...). Se a pasta não existe, ela é criada. Caminho relativo é relativo ao `WORKDIR` anterior. Boa prática: usar caminho absoluto e preferir `WORKDIR` a `RUN cd ... && ...`.

### COPY

Copia arquivos para dentro da imagem. A origem pode ser:

- o **contexto de build** (filtrado pelo `.dockerignore`): o `.csproj` e o código;
- **outro estágio** (`COPY --from=build /app .`): as dlls do publish;
- uma imagem.

Os pacotes NuGet não são copiados: o `dotnet restore` os baixa dentro da imagem. Também aceita `--chown` para definir o dono dos arquivos.

### ADD

Parecido com o COPY, mas a origem também pode ser uma URL remota ou um repositório Git. Um `.tar` **local** é extraído automaticamente (um tar remoto não, por padrão).

### COPY ou ADD?

Preferir **COPY**. O ADD é o melhor para baixar um artefato remoto durante o build, e nesse caso tem verificação de checksum.

## Bloco 2: executar durante o build (RUN, ARG, SHELL)

### RUN

É o comando de execução do Dockerfile. Ele roda quando a imagem é construída (o `CMD` roda quando o container sobe).

Com o `--mount` posso dar algo "extra" ao `RUN`, somente naquele comando:

- **cache:** a pasta de cache, como a do NuGet no exemplo de camadas e cache.
- **secret:** um segredo de build, como o token do NuGet. Não são os segredos da API.
- **ssh:** útil quando a operação precisa de acesso SSH, como um `git clone`.

Os outros tipos de `--mount` são `bind` e `tmpfs`.

### ARG

É uma variável que quem for rodar o build passa no parâmetro (`--build-arg`). Pode ter valor padrão. Vale a partir da linha em que é declarada e é herdada pelos estágios baseados naquele estágio.

**ARG x ENV:**

- `ENV` vai para a imagem e para o container final.
- `ARG` é usado somente na montagem (não existe no container final).

Segredo não entra em nenhum dos dois: para isso existe o `secret` do `RUN --mount`.

### SHELL

Define qual shell executa as instruções na forma shell de `RUN`, `CMD` e `ENTRYPOINT`. Padrão: `/bin/sh -c` no Linux e `cmd /S /C` no Windows. Pode ser trocado (`bash`, `powershell`).

## Bloco 3a: como o container roda (CMD, ENTRYPOINT, ENV, USER)

### CMD

É o comando padrão do container ao subir, mas qualquer um o substitui (o que vem depois da imagem no `docker run` troca o `CMD`). Se houver mais de um, só o último vale.

### ENTRYPOINT

É o programa que o container executa ao subir. Numa API .NET, `ENTRYPOINT ["dotnet", "Api.dll"]` (o `dotnet` é o programa e a dll é o argumento), ou o próprio executável, se for publicado como apphost.

Ele é fixo: a cada `docker run` ele roda, e o que vier depois da imagem é acrescentado como argumento. Só é trocado com `docker run --entrypoint`.

Uma combinação comum é usar os dois: o `ENTRYPOINT` é o programa e o `CMD` são os argumentos padrão dele, que podem ser trocados.

```dockerfile
ENTRYPOINT ["dotnet", "Api.dll"]
CMD ["--urls", "http://+:8080"]
```

### ENV

Define variável de ambiente (`ENV chave=valor`). Vai para a imagem e para o container final, e vale para as instruções seguintes. Se `ENV` e `ARG` têm o mesmo nome, o `ENV` vence. Não é lugar para segredos.

### USER

Define o usuário (e grupo) **dentro do container** com o qual o processo roda. É um usuário do sistema da imagem (Linux, nas imagens .NET), não o do meu Windows nem uma configuração do Docker.

Serve para segurança: se a aplicação for explorada, o invasor assume o processo só com os poderes desse usuário. Por isso a regra é rodar sem privilégio quando não for preciso (não-root).

## Bloco 3b: EXPOSE, VOLUME, STOPSIGNAL, HEALTHCHECK

### EXPOSE

Diz ao Docker em qual porta o container "escuta". Não publica porta nenhuma, só documenta. Quem publica é o `-p`.

### VOLUME

Informa que aquela imagem tem um volume naquele caminho. No Docker Compose é onde eu digo o que fica montado ali e onde ele guarda os dados. O `VOLUME` do Dockerfile não aceita caminho do host.

### STOPSIGNAL

Define o sinal que encerra o container (padrão `SIGTERM`).

### HEALTHCHECK

Está nas boas práticas. No Dockerfile é o teste de saúde que vem na imagem; o Compose pode sobrescrever os valores (ou desligar com `disable: true`).

## Bloco 4: metadados e outros (LABEL, MAINTAINER, ONBUILD)

- **LABEL:** adiciona metadados à imagem, em pares `chave=valor` (valor com espaço entre aspas duplas). As labels da imagem base são herdadas. Para ver: `docker image inspect`.
- **MAINTAINER:** define o autor da imagem. Está **deprecated**; use `LABEL org.opencontainers.image.authors="..."`.
- **ONBUILD:** guarda um gatilho na imagem, que não executa agora: ele roda no build de outra imagem que use esta como base (`FROM`), logo depois do `FROM` dela. Depois de executado, o gatilho é apagado.

## Instruções cobertas

FROM, WORKDIR, COPY, ADD, RUN, ARG, SHELL, CMD, ENTRYPOINT, ENV, USER, EXPOSE, VOLUME, STOPSIGNAL, HEALTHCHECK, LABEL, MAINTAINER, ONBUILD (18 da lista montada na sessão; não conferida contra a tabela oficial).

Fontes: [Dockerfile reference](https://docs.docker.com/reference/dockerfile/), [Compose, services](https://docs.docker.com/reference/compose-file/services/) e [build best practices](https://docs.docker.com/build/building/best-practices/).
