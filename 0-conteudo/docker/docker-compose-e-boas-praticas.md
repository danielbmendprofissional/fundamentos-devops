# Docker: configuração, Compose e boas práticas

## Variáveis de ambiente

Não tem nem o que dissertar: é um conceito para não ter segredos hardcoded em código ou container. As chaves ficam com os valores em variáveis de ambiente.

Ponto de atenção: variável de ambiente evita segredo no código, mas **não é cofre**. A doc do Docker diz que, ao injetar senhas e chaves por variável de ambiente, "you risk unintentional information exposure", e que elas podem aparecer em logs de debug. Para segredo de verdade, o Compose entrega *secrets* como arquivo em `/run/secrets/<nome>`. E `ARG`/`ENV` usados no build "persist in the final image", ou seja, ficam gravados na imagem.

Fontes: [Compose, secrets](https://docs.docker.com/compose/how-tos/use-secrets/) e [Docker, build secrets](https://docs.docker.com/build/building/secrets/).

## Docker Compose

O Dockerfile é sobre uma aplicação: ele **constrói uma imagem**. O Compose é o arquivo em que manuseamos a **execução** de N serviços diferentes: healthcheck, dependência, configuração dos volumes, redes, etc. Não é um "Dockerfile maior", são ferramentas diferentes. O Compose pode apontar para um Dockerfile (`build:`) ou para uma imagem pronta (`image:`).

## Boas práticas

1. **Imagem com apenas o necessário.** Multi-stage build e imagem base enxuta: menos tamanho, menos superfície de ataque.
2. **Um processo por container.** "Each container should have only one concern." É uma regra prática, não absoluta. Cada container (API, banco, broker) escala, reinicia e atualiza sozinho, e os logs não se misturam.
3. **Usuário não-root.** "If a service can run without privileges, use `USER` to change to a non-root user." Se alguém explorar uma falha na aplicação, o processo não entra com privilégio máximo.
4. **Healthcheck.** O Docker só sabe se o processo está vivo, não se a aplicação está funcionando. O `HEALTHCHECK` "tells Docker how to test a container to check that it's still working", e o status passa de `starting` para `healthy` ou `unhealthy` (padrão: intervalo 30s, timeout 30s, 3 tentativas). No Compose, `depends_on` sozinho só espera o container estar rodando ("Compose does not wait until a container is ready"); para esperar a saúde de verdade se usa `condition: service_healthy`.
5. **Tag fixa na imagem base.** "Image tags are mutable": uma tag pode passar a apontar para outra imagem, e então o build deixa de ser reprodutível. Para fixar de verdade, usa-se o digest.
6. **`.dockerignore`.** Exclui do contexto de build o que não é relevante (`bin/`, `obj/`, `.git`, arquivos de segredo): build mais rápido, melhor cache e nada de segredo indo para dentro da imagem.
7. **Logs em stdout/stderr.** Por padrão o `docker logs` mostra o `STDOUT` e o `STDERR` do container. A aplicação escreve na saída padrão e a plataforma coleta; gravar em arquivo dentro do container perde tudo quando ele morre.

Fontes: [Docker, build best practices](https://docs.docker.com/build/building/best-practices/), [Dockerfile reference](https://docs.docker.com/reference/dockerfile/), [Compose, startup order](https://docs.docker.com/compose/how-tos/startup-order/) e [Docker, logging](https://docs.docker.com/engine/logging/).
