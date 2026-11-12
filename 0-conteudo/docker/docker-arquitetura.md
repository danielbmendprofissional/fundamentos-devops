# Docker: arquitetura

## Client

É uma das formas de interação com o Docker: o programa `docker` em que eu digito os comandos na mão, em qualquer terminal (PowerShell, WSL...). O Docker Desktop é outra forma.

## Daemon

É o gerenciador do Docker. Tanto o client quanto qualquer outra forma se comunica com o daemon por meio da API do Docker (REST, por socket Unix ou rede), e é ele que executa tudo e gerencia os objetos (imagens, containers, redes, volumes). Por exemplo, um `docker run` do client chega ao daemon para que ele rode.

## Registry

É o hub onde ficam salvas as imagens, fora da minha máquina. Pode ser público (Docker Hub, com as imagens da internet que posso usar) ou privado (as minhas). As imagens que já baixei ficam na máquina onde o daemon roda.

## Plugin

É igual extensão do Visual Studio. Existem dois tipos:

- **Plugins do daemon** (Engine): volume, rede e autorização, instalados com `docker plugin install`. O daemon chama o plugin.
- **Plugins do client** (CLI plugins): executáveis `docker-<nome>` que o `docker` encontra em pastas específicas e roda como `docker <nome>`. Exemplos: buildx e Compose.

## Passo a passo do `docker run`

Para `docker run -i -t ubuntu /bin/bash`:

1. Se a imagem não está local, o Docker a puxa do registry (como um `docker pull`).
2. Cria o container (como um `docker container create`).
3. Aloca um filesystem de leitura e escrita como camada final do container.
4. Cria uma interface de rede ligando o container à rede padrão.
5. Inicia o container e executa o comando.

Exemplo para ver a separação client e daemon, e os plugins:

```bash
docker version        # mostra duas seções: Client e Server (o daemon)
docker buildx version # buildx: plugin do client
docker compose version # Compose: plugin do client
docker plugin ls      # plugins do daemon (Engine)
```

Fontes: [Docker overview](https://docs.docker.com/get-started/docker-overview/), [plugins do Engine](https://docs.docker.com/engine/extend/), [API de plugins](https://docs.docker.com/engine/extend/plugin_api/). Sobre CLI plugins: [README do buildx](https://github.com/docker/buildx) (não é a doc oficial do Docker).
