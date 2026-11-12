# Kubernetes: Ingress e Ingress Controller

## O problema

Cada aplicação HTTP exposta com um Service `LoadBalancer` gera um balanceador na nuvem, um IP público e uma cobrança. Com a API, o front-end e o painel admin, seriam 3.

## Ingress

Objeto que gerencia o acesso externo aos Services, normalmente HTTP e HTTPS. É um único ponto de entrada que roteia para vários Services por host (o nome do site) e por caminho (a parte `/algo` da URL). Dá também terminação TLS e balanceamento de carga. Não expõe portas nem protocolos quaisquer: para isso continuam NodePort e LoadBalancer.

```text
internet ──> UM balanceador ──> Ingress ──┬─ /api    ──> Service api
                                          ├─ /       ──> Service front
                                          └─ admin.. ──> Service admin
```

## Ingress controller

O Ingress sozinho é só um conjunto de regras e não faz nada. Quem as executa é o Ingress controller, um programa que se instala no cluster, lê os objetos Ingress e configura um proxy ou balanceador (por exemplo, o NGINX).

```text
Ingress            = a regra ("/api vai para o Service api")
Ingress controller = quem cumpre a regra (NGINX, Traefik, etc.)
```

- O Kubernetes mantém dois controllers: AWS e GCE. A doc lista ainda mais de 30 de terceiros, entre eles NGINX, Traefik, HAProxy, Kong, Istio, Contour, Cilium, Gloo e APISIX.
- Pode haver vários controllers no mesmo cluster. O campo `ingressClassName` diz qual cuida de cada Ingress.

## Campos principais

| Campo | Função |
|---|---|
| `ingressClassName` | qual controller processa; omitido, vale o padrão do cluster |
| `rules` | a lista de regras |
| `host` | (opcional) o nome do site; sem ele, vale para todo o tráfego HTTP |
| `path` | o caminho da URL, como `/api` |
| `pathType` | `Prefix`, `Exact` ou `ImplementationSpecific` |
| `backend.service` | o Service de destino (`name`) e a porta (`port`) |

`Prefix` casa tudo que começa com o caminho e `Exact` só o caminho exato (de memória; a doc consultada só mostrava `Prefix`).

## Exemplo (da doc, adaptado por mim): um host, caminhos diferentes

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: galo
spec:
  ingressClassName: nginx
  rules:
  - host: galo.exemplo.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: front
            port:
              number: 80
```

Os Services de trás podem ser ClusterIP: só o controller precisa ser alcançado de fora (de memória).

## TLS (HTTPS)

O certificado vai num Secret do tipo `kubernetes.io/tls` (com `tls.crt` e `tls.key`), referenciado no Ingress. Estrutura padrão da API; a página consultada veio cortada antes dessa parte, então confira antes de usar.

```yaml
spec:
  tls:
  - hosts:
      - galo.exemplo.com
    secretName: galo-tls
```

## Aviso da doc

O Kubernetes recomenda a Gateway API no lugar do Ingress. A API do Ingress está congelada: continua disponível e estável, sem funcionalidades novas, e não há plano de removê-la.

Fontes: [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) e [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).
