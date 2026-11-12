# Linux: processos, serviços, rede, SSH e logs

Entendimento por alto. Tudo neste arquivo é de memória, exceto o que vem da doc do systemd e da doc do SSH (marcado). Os comandos se pesquisam na hora; o importante é o conceito.

## Processo

Processo é um programa em execução, com um número de identificação, o PID. No Windows é o que o Gerenciador de Tarefas lista.

Ligação com o Docker: o container é, no fundo, um processo isolado. O processo que o `ENTRYPOINT` inicia é o PID 1 do container. Se ele termina, o container termina. Por isso o container só vive enquanto a API roda, e daí vem a ideia de um processo por container.

## Serviço (systemd)

Num servidor sem container, quem mantém a API de pé depois de o servidor reiniciar ou de ela cair é o systemd, o gerenciador de serviços do Linux. O programa é descrito num arquivo `.service` (a estrutura é da doc do systemd).

Exemplo (adaptado da doc), para a API .NET:

```ini
[Unit]
Description=API Galo

[Service]
User=app                               # roda como usuário comum
ExecStart=/usr/bin/dotnet /opt/api/Api.dll
Restart=on-failure                     # caiu? reinicia
RestartSec=5s

[Install]
WantedBy=multi-user.target             # sobe quando o servidor liga
```

| systemd | Docker / Kubernetes |
|---|---|
| `ExecStart=` | `ENTRYPOINT` |
| `User=` | `USER` |
| `Restart=on-failure` | restart policy / o ReplicaSet recriando o Pod |
| `[Install]` (sobe com o servidor) | o cluster recriando o Pod ao subir |

O systemd só é usado quando o programa roda direto no Linux. Com Docker ou Kubernetes, quem supervisiona e reinicia é o Docker ou o cluster, e não se escreve `.service`. A tabela mostra apenas que a ideia é a mesma.

## Rede

### IP e porta

```text
IP    = o endereço da máquina        (o prédio)
porta = a porta de um programa nela (o apartamento)
```

A porta é onde o programa escuta, isto é, espera ser chamado. A API escuta a 8080. Quem chama usa IP e porta: `192.168.0.10:8080`. Quem chama também usa uma porta, temporária e aleatória, da qual ninguém cuida.

### A mesma porta para várias aplicações

Numa mesma máquina, um programa só por porta. Dois programas não escutam a 8080 ao mesmo tempo.

O esquema de "servidor:8080 com várias aplicações publicadas, cada uma com seu DNS" funciona assim: um programa na frente (IIS, Nginx) escuta a porta e distribui para as aplicações conforme o nome do site (ou a pasta da URL). É o mesmo princípio do Ingress do Kubernetes. As aplicações atrás dele escutam portas internas diferentes, e só o da frente aparece para fora.

```text
api.galo.com    ┐
admin.galo.com  ├──> servidor:443 ──> [Nginx / IIS] ──┬─> aplicação da API
front.galo.com  ┘                    (distribui)      ├─> aplicação do admin
                                                      └─> aplicação do front
```

Dentro de containers diferentes, cada um tem a sua rede própria: duas APIs podem escutar a 8080 cada uma no seu container sem conflito.

### DNS

O DNS traduz o nome (`api.galo.com`) em IP. Não traz a porta: o navegador usa `80` para HTTP e `443` para HTTPS sozinho.

```text
1. DNS: api.galo.com  ->  203.0.113.7
2. o navegador conecta em 203.0.113.7:443
3. o programa que escuta a 443 (Nginx) vê o nome "api.galo.com" e escolhe a aplicação
```

### Firewall

É um porteiro com uma lista de regras: qual origem pode chegar em qual porta. "De fora" é fora da máquina ou da rede, conforme a regra.

```text
Regra A: porta 443  <- qualquer origem (a internet inteira)   ABERTA
Regra B: porta 5432 <- só o IP da API                          RESTRITA
Regra C: porta 22   <- só o IP da empresa (SSH)                RESTRITA
```

O Postgres na 5432 não deve ser aberto para a internet. É a mesma ideia da NetworkPolicy.

Há firewall em dois lugares: no próprio servidor e na rede (na nuvem, o security group ou NSG). As duas camadas precisam permitir. Para saber o que está aberto, não se adivinha: é o que foi configurado. Para ver quais portas os programas escutam na máquina, existe o comando `ss`. A porta de uma aplicação está no `EXPOSE` do Dockerfile, na doc da imagem ou na configuração dela.

### O `8080:80` do Docker

Lê-se `porta-do-servidor : porta-dentro-do-container`:

```text
docker run -p 8080:80 nginx

quem chama ──> servidor:8080 ──> container:80
               (esquerda)        (direita: onde o programa escuta)
```

A esquerda é a porta que o servidor abre para fora, e a direita é a porta em que o programa dentro do container escuta. Podem ser diferentes. Se a API escuta a 8080 no container, fica `-p 8080:8080` (ou `-p 80:8080` para o servidor atender na 80).

No Kubernetes é o mesmo desenho, com outros nomes no Service: `port` é a esquerda (a do Service) e `targetPort` é a direita (a do container).

## SSH

Abre um terminal numa máquina remota, com conexão criptografada (doc do SSH). É a porta de entrada de administração do servidor, e a porta 22 é testada por robôs da internet o tempo inteiro.

| Forma | Como funciona |
|---|---|
| **Senha** | a senha vai criptografada ao servidor (doc); pode ser adivinhada por tentativa e erro |
| **Chave pública** | o par tem a pública, que fica no servidor (`~/.ssh/authorized_keys`), e a privada, que fica só na máquina e nunca sai dela; prova-se que a tem sem revelá-la |

A chave não pode ser adivinhada, mas o risco se desloca para o arquivo da chave privada. A boa prática é desligar a senha no servidor e proteger a chave:

- passphrase na chave privada;
- nunca colocar a chave no Git, no log do pipeline nem na imagem;
- uma chave por pessoa e por máquina, para revogar uma sem afetar as outras;
- revogar = remover a chave pública do `authorized_keys` do servidor;
- o pipeline usa uma chave própria, com poder limitado, e não a de uma pessoa.

Aviso da doc: o SSH guarda a identificação de cada servidor visitado (`~/.ssh/known_hosts`). Se ela mudar, ele avisa, e isso pode ser um ataque do tipo homem no meio. Investigar antes de seguir.

```bash
ssh usuario@192.168.0.10
```

## Logs

Num servidor com systemd, os logs do serviço vão para o journal, consultado com o `journalctl`. Os logs de sistema ficam em `/var/log`. Em container, a API escreve em stdout/stderr e a plataforma coleta (`docker logs`, `kubectl logs`).

## Resumo

| Assunto | No servidor | Em Docker / Kubernetes |
|---|---|---|
| Programa em execução | processo | o container é um processo |
| Mantê-lo de pé | systemd (`.service`) | restart policy, ReplicaSet |
| Rede | portas + firewall | `-p`, Service, NetworkPolicy |
| Acesso remoto | SSH | `docker exec`, `kubectl exec` |
| Logs | journal, `/var/log` | stdout, `docker logs`, `kubectl logs` |

---

# Grande exemplo: 3 aplicações num servidor, com DNS, firewall e Nginx

Os nomes, os IPs e os domínios são inventados, só para ilustrar (`203.0.113.7` e `198.51.100.10` são faixas reservadas para documentação). Os arquivos de configuração são de memória: confira a sintaxe antes de usar.

## O cenário

Três aplicações num servidor só, cada uma com 4 componentes: **api**, **banco**, **frontend** e **site** (o site institucional). Sem IIS: o Nginx distribui no servidor com Docker, e depois a mesma ideia no Kubernetes.

| Aplicação | O que é |
|---|---|
| `galo` | a aplicação de estudos (DDD) |
| `loja` | uma loja virtual |
| `agenda` | um sistema de agendamento |

Servidor: IP público `203.0.113.7`, Linux com Docker. IP da empresa (para SSH): `198.51.100.10`.

## 1. DNS: os nomes

Um registro do tipo A (nome para IP) por nome público. Os 9 nomes apontam para o mesmo IP, porque quem escolhe a aplicação é o Nginx.

| Nome | Tipo | Aponta para | Componente |
|---|---|---|---|
| `galo.exemplo.com` | A | `203.0.113.7` | site do galo |
| `app.galo.exemplo.com` | A | `203.0.113.7` | frontend do galo |
| `api.galo.exemplo.com` | A | `203.0.113.7` | api do galo |
| `loja.exemplo.com` | A | `203.0.113.7` | site da loja |
| `app.loja.exemplo.com` | A | `203.0.113.7` | frontend da loja |
| `api.loja.exemplo.com` | A | `203.0.113.7` | api da loja |
| `agenda.exemplo.com` | A | `203.0.113.7` | site da agenda |
| `app.agenda.exemplo.com` | A | `203.0.113.7` | frontend da agenda |
| `api.agenda.exemplo.com` | A | `203.0.113.7` | api da agenda |
| (nenhum) | | | os 3 bancos |

Os bancos não têm DNS público: ninguém de fora precisa chamá-los. Dentro da rede do Docker eles têm um nome interno (`galo-db`, `loja-db`, `agenda-db`), que o próprio Docker resolve.

## 2. Quem escuta qual porta

| Componente | Escuta (dentro do container) | Aparece na porta do servidor? |
|---|---|---|
| **Nginx** | 80 e 443 | **sim**, é o único |
| api (das 3) | 8080 | não |
| frontend (os 3) | 80 | não |
| site (os 3) | 80 | não |
| banco (os 3) | 5432 | não |

Repare: as 3 APIs escutam a 8080, os 3 frontends e os 3 sites escutam a 80 e os 3 bancos a 5432. Não há conflito, porque cada um está no seu container, com a sua rede. Só o Nginx usa portas do servidor (80 e 443). É a resposta à pergunta da mesma porta para várias aplicações.

## 3. O caminho de um pedido

```text
navegador: https://api.galo.exemplo.com/pedidos
 1. DNS     api.galo.exemplo.com -> 203.0.113.7
 2. conecta em 203.0.113.7:443            (firewall: 443 aberto, passa)
 3. Nginx lê o nome "api.galo.exemplo.com"
 4. Nginx repassa para galo-api:8080      (rede interna do Docker)
 5. a API consulta galo-db:5432           (rede interna; o Nginx nem enxerga o banco)
 6. a resposta volta pelo mesmo caminho
```

## 4. Firewall: um por componente

### No servidor (a borda)

Política padrão: tudo negado de entrada. Só abre o necessário.

| Porta | Origem permitida | Para quê |
|---|---|---|
| 443 | qualquer origem | HTTPS (o Nginx) |
| 80 | qualquer origem | só redireciona para o HTTPS |
| 22 | só `198.51.100.10` (a empresa) | SSH |
| todas as outras | ninguém | fechadas |

Na nuvem, as mesmas regras vão no security group ou NSG, além do firewall do servidor.

### Por componente

| Componente | Quem pode chegar nele | Como é garantido |
|---|---|---|
| Nginx | a internet, nas portas 80 e 443 | é o único com `ports:` publicado |
| site (os 3) | só o Nginx | está só na rede interna da aplicação |
| frontend (os 3) | só o Nginx | está só na rede interna da aplicação |
| api (as 3) | só o Nginx (de entrada); ela mesma chama o banco | rede interna, sem `ports:` |
| banco (os 3) | só a api da mesma aplicação | rede de dados separada, onde só a api e o banco estão |

Isolamento entre aplicações: a `galo-api` não alcança a `loja-db`, porque cada aplicação tem a sua rede de dados.

Aviso de memória: o Docker, ao publicar uma porta com `ports:`, costuma passar por cima das regras do firewall do servidor (o `ufw`, por exemplo). Por isso a regra é publicar só o que precisa (80 e 443), e o resto nunca recebe `ports:`.

## 5. Versão Docker Compose (um servidor)

Cada aplicação tem 2 redes: `X-web` (nginx, api, frontend, site) e `X-dados` (api e banco). O Nginx entra só nas redes `-web`.

```yaml
services:
  nginx:
    image: nginx:1.27
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./ssl.conf:/etc/nginx/ssl.conf:ro
      - ./certs:/etc/nginx/certs:ro
    networks: [galo-web, loja-web, agenda-web]

  # ---------- galo ----------
  galo-site:
    image: registro/galo-site:1.0
    networks: [galo-web]
  galo-front:
    image: registro/galo-front:1.0
    networks: [galo-web]
  galo-api:
    image: registro/galo-api:1.0
    networks: [galo-web, galo-dados]
  galo-db:
    image: postgres:17
    networks: [galo-dados]

  # ---------- loja ----------
  loja-site:
    image: registro/loja-site:1.0
    networks: [loja-web]
  loja-front:
    image: registro/loja-front:1.0
    networks: [loja-web]
  loja-api:
    image: registro/loja-api:1.0
    networks: [loja-web, loja-dados]
  loja-db:
    image: postgres:17
    networks: [loja-dados]

  # ---------- agenda ----------
  agenda-site:
    image: registro/agenda-site:1.0
    networks: [agenda-web]
  agenda-front:
    image: registro/agenda-front:1.0
    networks: [agenda-web]
  agenda-api:
    image: registro/agenda-api:1.0
    networks: [agenda-web, agenda-dados]
  agenda-db:
    image: postgres:17
    networks: [agenda-dados]

networks:
  galo-web:
  galo-dados:
  loja-web:
  loja-dados:
  agenda-web:
  agenda-dados:
```

Só o `nginx` tem `ports:`. Os volumes, a senha dos bancos e o healthcheck foram omitidos de propósito, para o exemplo ficar legível (ver [../docker/docker-compose-e-boas-praticas.md](../docker/docker-compose-e-boas-praticas.md)).

### O `nginx.conf`: quem distribui

```nginx
server {                                  # tudo que chega na 80 vai para o HTTPS
  listen 80 default_server;
  return 301 https://$host$request_uri;
}

# ---------- galo ----------
server {
  listen 443 ssl; server_name galo.exemplo.com;
  include /etc/nginx/ssl.conf;
  location / { proxy_pass http://galo-site:80; }
}
server {
  listen 443 ssl; server_name app.galo.exemplo.com;
  include /etc/nginx/ssl.conf;
  location / { proxy_pass http://galo-front:80; }
}
server {
  listen 443 ssl; server_name api.galo.exemplo.com;
  include /etc/nginx/ssl.conf;
  location / { proxy_pass http://galo-api:8080; }
}

# ---------- loja ----------
server {
  listen 443 ssl; server_name loja.exemplo.com;
  include /etc/nginx/ssl.conf;
  location / { proxy_pass http://loja-site:80; }
}
server {
  listen 443 ssl; server_name app.loja.exemplo.com;
  include /etc/nginx/ssl.conf;
  location / { proxy_pass http://loja-front:80; }
}
server {
  listen 443 ssl; server_name api.loja.exemplo.com;
  include /etc/nginx/ssl.conf;
  location / { proxy_pass http://loja-api:8080; }
}

# ---------- agenda ----------
server {
  listen 443 ssl; server_name agenda.exemplo.com;
  include /etc/nginx/ssl.conf;
  location / { proxy_pass http://agenda-site:80; }
}
server {
  listen 443 ssl; server_name app.agenda.exemplo.com;
  include /etc/nginx/ssl.conf;
  location / { proxy_pass http://agenda-front:80; }
}
server {
  listen 443 ssl; server_name api.agenda.exemplo.com;
  include /etc/nginx/ssl.conf;
  location / { proxy_pass http://agenda-api:8080; }
}
```

O `ssl.conf` carrega o certificado (um por nome ou um coringa, de memória). O Nginx escolhe o bloco pelo `server_name`, ou seja, pelo nome do site que o navegador pediu. É a "mesma porta para várias aplicações".

## 6. Versão Kubernetes

A mesma ideia, com outros nomes:

| Servidor com Docker | Kubernetes |
|---|---|
| Nginx distribuindo por `server_name` | **Ingress** + Ingress controller (por `host`) |
| rede `X-dados` (só a api e o banco) | **NetworkPolicy** no banco |
| `galo-db` (nome na rede do Docker) | Service, com DNS interno `galo-db.galo.svc.cluster.local` |
| firewall do servidor (443, 80, 22) | security group / NSG na entrada do cluster |
| um `docker compose` | um **namespace** por aplicação: `galo`, `loja`, `agenda` |

Os Services de api, frontend e site, e o do banco, são do tipo ClusterIP: só o Ingress controller é alcançado de fora. O Ingress é de namespace, então são 3 (um por aplicação). O do galo:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: galo
  namespace: galo
spec:
  ingressClassName: nginx
  rules:
  - host: galo.exemplo.com                  # site
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: site, port: { number: 80 } } }
  - host: app.galo.exemplo.com              # frontend
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: front, port: { number: 80 } } }
  - host: api.galo.exemplo.com              # api
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: api, port: { number: 80 } } }
```

O do `loja` e o do `agenda` têm o mesmo molde, mudando apenas o `namespace` e os nomes:

| Namespace | Hosts do Ingress |
|---|---|
| `loja` | `loja.exemplo.com`, `app.loja.exemplo.com`, `api.loja.exemplo.com` |
| `agenda` | `agenda.exemplo.com`, `app.agenda.exemplo.com`, `api.agenda.exemplo.com` |

O firewall do banco no Kubernetes é a NetworkPolicy. Uma por namespace; a do galo:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-so-api
  namespace: galo
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api            # só a api do próprio namespace
    ports:
    - protocol: TCP
      port: 5432
```

Como as NetworkPolicies são por namespace e esta só aceita um Pod com `app: api` do próprio namespace, a `galo-api` não alcança o banco da `loja`. A NetworkPolicy exige um plugin de rede que a suporte (Calico, Cilium ou Antrea).

## 7. O firewall de cada componente, nas duas versões

| Componente | Servidor com Docker | Kubernetes |
|---|---|---|
| Entrada | firewall do servidor: 443 e 80 abertos, 22 só da empresa | security group / NSG: 443 e 80 para o Ingress controller |
| Nginx / Ingress controller | única porta publicada (80 e 443) | o único acessível de fora |
| site | só o Nginx (rede `-web`) | Service ClusterIP; só o Ingress chega |
| frontend | só o Nginx (rede `-web`) | Service ClusterIP; só o Ingress chega |
| api | só o Nginx (rede `-web`) e o banco (rede `-dados`) | Service ClusterIP; Ingress de entrada; NetworkPolicy opcional de saída só para o banco |
| banco | só a api, pela rede `-dados` | NetworkPolicy: só o Pod `app: api` do mesmo namespace, na 5432 |
| SSH | porta 22, só do IP da empresa | acesso aos nodes pelo mesmo SSH; para o cluster, `kubectl` com RBAC |

Liga com o que você já viu: o Ingress e a NetworkPolicy estão em [../kubernetes/kubernetes-ingress.md](../kubernetes/kubernetes-ingress.md) e [../kubernetes/kubernetes-seguranca-basica.md](../kubernetes/kubernetes-seguranca-basica.md); o RBAC controla quem consegue mexer em tudo isso.
