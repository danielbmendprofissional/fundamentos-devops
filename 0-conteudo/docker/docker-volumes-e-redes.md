# Docker: volumes e redes

## Persistência

Um container não persiste dados: quando desligado, tudo que estava com ele some. Por isso existem os volumes, que independem do ciclo de vida do container.

Quem grava no volume é o container **do banco** (o Postgres, por exemplo), não o da API. A API é *stateless*: se ela cair, não se perde nada. O volume pertence ao banco.

## Volume

Fica dentro do servidor que executa o Docker, controlado pelo próprio Docker.

- **Volume vazio:** o Docker copia para ele o conteúdo que a imagem tem naquele caminho (populate inicial). Basta a imagem ter arquivos ali.
- **Volume com dados:** o conteúdo do volume vence e o que a imagem tinha naquele caminho fica escondido. Não existe junção dos dois.

Exemplo com o Postgres da API, num Compose:

```yaml
services:
  postgres:
    image: postgres:17
    volumes:
      - pgdata:/var/lib/postgresql/data   # volume nomeado, gerenciado pelo Docker
volumes:
  pgdata:
```

O mesmo pela linha de comando: `docker run -v pgdata:/var/lib/postgresql/data postgres:17`. Se eu apagar o container e subir outro com o mesmo `pgdata`, os dados continuam lá. (O caminho de dados do Postgres é de memória: confirme na página da imagem.)

## Bind mount

O apontamento fica no próprio SO do host, controlado por ele. Se a imagem tem dados naquele caminho, os do host prevalecem: os arquivos da imagem ficam **escondidos** pelo mount, não apagados. O bind mount nunca copia nada.

Exemplo: pôr no container da API um `appsettings.Development.json` que está na minha máquina.

```yaml
services:
  api:
    image: minha-api
    volumes:
      - ./appsettings.Development.json:/app/appsettings.Development.json
```

Se a imagem já tinha um arquivo nesse caminho, o do host o esconde (a imagem não é alterada). Pela linha de comando:

```bash
docker run --mount type=bind,source="$(pwd)"/appsettings.Development.json,target=/app/appsettings.Development.json minha-api
```

Fontes: [Docker, volumes](https://docs.docker.com/engine/storage/volumes/) ("the pre-existing files are obscured by the mount") e [Docker, bind mounts](https://docs.docker.com/engine/storage/bind-mounts/).

## Redes

São as formas (ou não formas) de comunicação entre os containers.

- **bridge:** os containers dentro da rede se comunicam entre si, e ficam isolados de fora (privacidade). É o driver padrão. Há duas variações:
  - **bridge padrão:** os containers se acham só por IP, não por nome.
  - **bridge criada por mim** (o Compose cria uma por projeto): os containers se acham pelo **nome**, por um DNS interno do Docker. É por isso que a API conecta no banco com `Host=postgres`.
- **host:** o container usa a rede do host diretamente, sem isolamento de rede. Se o container escuta na porta 8080, o host está escutando na 8080. É arriscado.
- **none:** isolamento total, sem rede nenhuma. Ninguém se comunica.

Existem outros drivers para cenários avançados: **overlay** (liga containers de vários hosts, usado no Swarm), **macvlan** (o container aparece como um dispositivo na rede do host) e **ipvlan** (liga containers a VLANs externas).

Exemplo das três redes principais, pela linha de comando:

```bash
# bridge criada por mim: os containers se acham pelo nome
docker network create minha-rede
docker run -d --name postgres --network minha-rede -e POSTGRES_PASSWORD=exemplo postgres:17
docker run -d --name api --network minha-rede minha-api   # na API: Host=postgres

# host: usa a rede do host, sem isolamento
docker run --network host minha-api

# none: sem rede nenhuma
docker run --network none minha-api
```

No Compose, não preciso criar a rede: ele cria uma por projeto, e os serviços já se acham pelo nome.

```yaml
services:
  api:
    image: minha-api
    ports:
      - "8080:8080"      # só a API é publicada para fora
    environment:
      ConnectionStrings__Default: Host=postgres;Database=app;Username=app;Password=${DB_PASSWORD}
  postgres:
    image: postgres:17   # sem "ports": só a API enxerga o banco
```

### Publicação de porta

Em uma bridge, os containers conversam entre si. Para o mundo de fora (navegador, outras máquinas) o container só é alcançável se a porta for publicada, com `-p 8080:80` (porta do host : porta do container).

```bash
docker run -d -p 8080:80 minha-api   # http://localhost:8080 chega na porta 80 do container
```

Fonte: [Docker, networking](https://docs.docker.com/engine/network/).
