# Docker: segurança

## Namespaces

São o recurso do kernel que dá a cada container a sua própria visão das coisas. Há um kernel só, o do host, e cada container recebe os seus namespaces. É o que impede um container de enxergar os processos de outro (o isolamento de processos é o que a doc cita; que rede e arquivos também sejam isolados é conhecimento meu).

## Cgroups

Limitam e contabilizam os recursos, e impedem que um container esgote o host. Os limites estão em [docker-limites-de-recurso.md](docker-limites-de-recurso.md).

## O daemon é um ponto sensível

Quem controla o daemon controla o host: a doc mostra que dá para subir um container com o `/` do host montado dentro dele. Por padrão o daemon roda como root e escuta num socket Unix local; quem pode usar esse socket é definido pelo sistema operacional (as permissões do arquivo). O grupo `docker` e os comandos abaixo são de memória, não da doc consultada.

```text
SERVIDOR LINUX
 ├─ dockerd (daemon, roda como root)
 │    └─ socket  /var/run/docker.sock
 ├─ usuário ana    (no grupo "docker")  -> fala com o socket
 └─ usuário bruno  (fora do grupo)      -> "permission denied"
```

```bash
ls -l /var/run/docker.sock       # srw-rw---- root docker
sudo usermod -aG docker ana      # a ana passa a controlar o daemon
docker run -it -v /:/host alpine sh   # dentro: /host é o disco do servidor, como root
```

Se a API for exposta por TCP, o TLS é obrigatório (sem TLS "is not permitted"). O melhor é não expor.

## Capabilities

Gerenciam as capacidades que o container tem. Por padrão o Docker inicia o container com um conjunto restrito. Remover as desnecessárias reduz o risco.

## Imagens assinadas (Content Trust)

O Engine pode ser configurado para rodar só imagens assinadas.

## Rootless

O rootless roda o daemon (e os containers) como um usuário comum do host, em vez de root. É diferente do `USER`, que define quem é o processo dentro do container. Se o daemon for explorado, o invasor assume só o usuário comum.

## Outros recursos

Outros recursos do kernel citados pela doc: AppArmor, SELinux, TOMOYO, GRSEC e user namespaces.

## Exemplo: execução endurecida da API

As flags são de memória, não conferidas na doc: confirme antes de usar.

```bash
docker run -d \
  --cap-drop ALL \                         # remove todas as capabilities
  --security-opt no-new-privileges \       # impede ganhar privilégio
  --read-only \                            # sistema de arquivos só leitura
  --user 1000 \                            # não roda como root
  minha-api
```

Fontes: [Docker Engine security](https://docs.docker.com/engine/security/) e [Rootless mode](https://docs.docker.com/engine/security/rootless/).
