# Docker: fundamentos

## Container x VM

Uma VM precisa de todo um SO e de uma infra para rodar. Um container pega tudo o que precisa para existir: é leve, portátil e de fácil gerenciamento. Consigo subir e apagar em segundos, e se roda no container, roda no servidor. Seria muito trampo fazer isso em uma Virtual Machine.

## Imagem x container

A imagem é a representação pronta do que a aplicação precisa para rodar: runtime, dependências e a aplicação já compilada. A compilação acontece na hora de construir a imagem.

O container é uma instância dessa imagem em execução: a imagem mais uma camada fina gravável por cima, rodando como processo. Posso subir vários containers da mesma imagem.

## Dockerfile

São as instruções (a receita) para construir a imagem que será executada dentro de um container. Compilar o código é só uma das coisas que ele pode fazer.

## Multi-stage build

Ao invés de usar uma imagem grande para todo o processo, uso a imagem grande somente no processo em que ela é realmente necessária (compilar) e depois uso uma menor para executar. O primeiro estágio é descartado: só o resultado copiado com `COPY --from` vai para a imagem final.

Exemplo ilustrativo para .NET:

```dockerfile
# Estágio 1: compila (imagem grande, com SDK). Será descartado.
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

# Estágio 2: executa (imagem pequena, só runtime). É a imagem final.
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "Api.dll"]
```

O `*.csproj` é copiado antes do restante do código para o `restore` ficar em cache enquanto as dependências não mudarem.

Fonte: [Docker, build best practices](https://docs.docker.com/build/building/best-practices/).

## Registry e tags

O registry é o hub das imagens que escolho publicar ou que estão publicadas na web para uso. Em um processo robusto de CI eu nunca usaria o `latest`, porque a última imagem pode conter alterações que quebram a esteira.
