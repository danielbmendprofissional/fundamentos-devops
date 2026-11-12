# Linux: usuários e permissões (o básico, no contexto de Docker e Kubernetes)

Entendimento por alto: o objetivo é saber como funciona; os comandos se pesquisam na hora. Tudo neste arquivo é de memória, nenhuma doc foi consultada.

## O container é um Linux pequeno

A imagem traz as pastas de um Linux (`/etc`, `/var`, `/home`, `/usr`, `/tmp`, `/run`), e o container roda em cima do kernel do host. Não existe `C:\`: tudo parte da raiz `/`.

```text
/            raiz de tudo
├─ etc/      arquivos de configuração
├─ var/      dados que mudam (logs em /var/log, dados em /var/lib)
├─ home/     pastas dos usuários
├─ usr/      programas instalados
├─ tmp/      arquivos temporários
└─ run/      dados de execução (como /run/secrets)
```

## Cada sistema tem a sua lista de usuários

```text
1. Windows
2. WSL (o Linux do Windows)
3. container da API       (lista vinda da imagem)
4. container do Postgres  (lista vinda da imagem)
```

O Docker em si não é um mundo com usuários: é um programa que roda no WSL, como root. Os containers compartilham o kernel do Linux do WSL, mas cada um tem a sua lista de nomes. O que vale de verdade é o número (UID para usuário, GID para grupo): o nome é só um rótulo, e o kernel enxerga só o número.

Os usuários do container vêm da imagem (a do .NET traz o `app`, a do Postgres traz o `postgres`). Um usuário novo se cria no Dockerfile, na construção da imagem.

## Como funciona a permissão

Todo arquivo e toda pasta tem um dono (usuário), um grupo e permissões para três públicos: o dono, o grupo e o resto. É como uma pasta compartilhada: o dono manda, o grupo é quem trabalha junto, e o resto é o mundo.

- `r` ler, `w` escrever, `x` executar (numa pasta, `x` é poder entrar nela).
- Cada letra vale um número (r=4, w=2, x=1), e a soma forma o número do `chmod` (`755`, `644`).
- A permissão é de cada arquivo e de cada pasta; não existe "uma permissão do container". Dá para configurar a raiz e qualquer pasta, porque na construção da imagem você é root.
- Na prática se cuida de poucas pastas: onde a API grava (uma pasta de dados, `/tmp`, um volume).

Grupos: não existe "grupo do container". Os grupos pertencem ao sistema (à lista de usuários daquele container). Todo usuário já nasce com um grupo próprio e pode pertencer a outros. Quando uma pasta diz "o grupo X pode gravar", vale para quem pertence ao X.

## Por que `/etc` e `/usr` pertencem ao root

Não é risco: é a proteção. Se pertencessem ao usuário da API, uma API comprometida poderia alterar os programas e as configurações do sistema. Com o dono sendo o root e a API só lendo, ela não consegue alterar nada disso. O usuário da API deve ser dono só do que precisa gravar. Colocar tudo em um usuário é o contrário do menor privilégio.

Reforço: sistema de arquivos somente leitura (a flag `--read-only` do Docker, já vista em segurança; no Kubernetes, `readOnlyRootFilesystem`, de memória).

## Como descobrir o usuário de uma imagem

Só é necessário quando você precisa do número: por exemplo, para `runAsNonRoot` ou `fsGroup` no Kubernetes, ou para acertar a permissão de um volume. Se você usa a imagem como ela vem, ela já funciona como foi desenhada.

Nem toda imagem roda como usuário comum: algumas rodam como root por padrão.

Formas de descobrir (comandos de memória, confira):

- A página da imagem no Docker Hub ou a doc dela costuma dizer.
- Ver qual usuário a imagem declara: `docker inspect <imagem>` e procurar o campo `User`.
- Ver quem é o usuário rodando a imagem: `docker run --rm <imagem> id` (mostra o UID e o GID).
- Ver a lista de usuários da imagem: `docker run --rm <imagem> cat /etc/passwd`.

Exemplos (de memória, confirme na imagem que usar): o usuário `app` das imagens recentes do .NET costuma ser o 1654; o `postgres` costuma ser o 999 nas imagens baseadas em Debian.

## Na minha própria imagem

Eu escolho o número. Convenção (de memória): usar um número de 1000 para cima, para não colidir com os usuários do sistema; criar o usuário no Dockerfile e usar o número no `USER`.

```dockerfile
RUN useradd --uid 10001 --no-create-home app
USER 10001
```

Usar o número no `USER` (e não o nome) permite ao Kubernetes verificar que não é root.

## Números diferentes entre apps

O `fsGroup` é um número que se escolhe: não precisa existir antes. Pode repetir o mesmo número em containers de imagens diferentes sem erro, porque as listas de nomes são separadas. Mas o kernel enxerga só o número, e dois Pods que compartilham o mesmo volume precisam combinar os números. Recomendação: usar números diferentes por aplicação, para que uma app não leia os arquivos da outra se os volumes se misturarem.

## Alinhando com Docker e Kubernetes

**Docker:** o `USER` do Dockerfile define com quem a aplicação roda. O problema aparece nos volumes: o volume é criado com um dono (um número) e a API roda com outro, então ela não consegue gravar ("permission denied"). A solução é alinhar os números.

**Kubernetes:** declara-se no YAML do Pod, no `securityContext`:

```yaml
spec:
  securityContext:
    runAsUser: 1654          # o número do usuário com que o processo roda
    runAsNonRoot: true       # recusa subir se for root
    fsGroup: 2000            # os volumes montados ficam com este grupo
  containers:
  - name: api
    image: minha-api:abc123
```

O `fsGroup` ajusta o grupo dos arquivos montados e coloca o processo da API como membro dele, então o Pod consegue gravar sem que se "descubra" nada antes. O 2000 é um número escolhido (de exemplo), não de um grupo existente.

**Quem vale, Dockerfile ou Kubernetes?** O YAML do Pod sobrescreve o Dockerfile, quando declara. Se não declara, vale o `USER` da imagem.

```text
Dockerfile:  USER 1654               <- o padrão da imagem
YAML do Pod: runAsUser: 1654         <- se declarado, é este que vale
```

Organização recomendada: o Dockerfile traz o padrão sensato (usuário comum); o YAML do Pod traz a exigência explícita (`runAsNonRoot`, `fsGroup`), versionada no Git. Assim a imagem funciona sozinha, e o cluster ainda recusa se alguém trocar por root.

Ajustar permissão com comando dentro de um container em execução não é o caminho: o container é efêmero, e o que foi feito à mão se perde quando o Pod é recriado. O certo é declarar no Dockerfile e no YAML.

## Exemplo: a API que não consegue gravar no volume

```text
O volume foi criado por root:  drwxr-xr-x  root root  /dados
A API (usuário app) tenta gravar em /dados  ->  "permission denied"
Solução: acertar o dono do volume para o usuário da API (ou usar o fsGroup no Kubernetes).
```

É um dos erros mais comuns ao subir APIs e Postgres em container.
