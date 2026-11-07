# Docker: camadas e cache

## Imagem e camadas

Uma imagem é a junção de N camadas de um produto que estamos tentando subir. Numa aplicação .NET, por exemplo: o `.csproj`, os pacotes NuGet, as dlls, o resultado do publish.

Cada camada criada é imutável. Se algo muda em qualquer camada, ele gera uma nova, e todas as camadas que vêm depois dela no processo de build também são refeitas, mesmo que não tenham mudado.

Para ver as camadas de uma imagem: `docker image history <imagem>`.

## Cache de build

O cache, no processo de build da imagem, analisa cada camada para ver se mudou ou se manteve. Se manteve, ele usa a que já está no cache.

## A ordem importa

A ordem em que as instruções estão no Dockerfile importa. Se tenho uma camada que muda MUITO, o ideal é que ela venha **depois** (nas linhas de baixo do Dockerfile) das camadas que mudam pouco, para que o cache seja bem aproveitado.

O Dockerfile é executado de cima para baixo. Quando uma linha muda, ela e todas as linhas abaixo dela são refeitas. As linhas acima continuam no cache.

Exemplo ilustrativo em .NET, com a ordem **certa**:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 # 1  acima: cache
WORKDIR /src                           # 2  acima: cache
COPY Api.csproj .                      # 3  muda raramente: lista de pacotes
RUN dotnet restore                     # 4  lento; só é refeito se o .csproj mudar
COPY . .                               # 5  muda toda hora: o código
RUN dotnet publish -c Release -o /app  # 6  refeito sempre que o código mudar
```

- Se eu editar só um Controller (muda a linha 5): a 5 e a 6 são refeitas, e as linhas 1 a 4 (inclusive o `restore`) vêm do cache.
- Se eu adicionar um pacote NuGet (o `.csproj` muda, linha 3): da linha 3 até a 6 são refeitas, e as linhas 1 e 2 vêm do cache.

Regra para bater o olho: o que muda pouco fica nas linhas de cima, o que muda muito fica nas linhas de baixo.

Com a ordem **errada** (`COPY . .` antes do `restore`), qualquer mudança de código refaz o `restore` e baixa todos os pacotes de novo.

## Cache externo

Existe também o cache externo, salvo no registry, caso necessário. Serve para compartilhar o cache entre máquinas e execuções de CI, que costumam começar sem cache nenhum (`--cache-to` e `--cache-from`).

## .dockerignore

É uma espécie de `.gitignore` para o build do Docker: ele lista os objetos e recursos que não servem para a imagem, para que não se gaste recursos sem necessidade. Os arquivos listados são excluídos do **contexto de build** (o conjunto de arquivos que o build pode acessar), então o `COPY` nem os enxerga.

Exemplo para .NET:

```text
bin/
obj/
.git
.vs
```

## Cache mount

Não tem relação com bind mount. É um esquema dentro do Dockerfile onde eu registro determinada pasta que terá o cache persistido, independente do esquema de camadas. Por exemplo, o `.nuget/packages`:

- Sem o cache mount: mudou 1 pacote = restaura todos.
- Com o cache mount: mudou 1 pacote = restaura ele, e o restante vem do cache mount.

Exemplo da doc para .NET:

```dockerfile
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore
```

Aplicado ao Dockerfile do exemplo:

```dockerfile
COPY Api.csproj .
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app
```

A ordem das linhas evita que o `restore` rode (usa a camada em cache). O cache mount faz o `restore` ser rápido quando ele precisa rodar.

## O container e as camadas

As camadas da imagem são compartilhadas entre todos os containers criados dela. Cada container recebe um diretório próprio para as suas mudanças, e as camadas da imagem ficam intocadas.

Fontes: [camadas de imagem](https://docs.docker.com/get-started/docker-concepts/building-images/understanding-image-layers/), [cache de build](https://docs.docker.com/build/cache/) [otimização do cache](https://docs.docker.com/build/cache/optimize/) e [contexto de build](https://docs.docker.com/build/concepts/context/).
