# Kubernetes: segurança básica (RBAC e NetworkPolicy)

## RBAC

Controla quem pode fazer o quê no cluster, como uma role de banco de dados: define quem tem acesso a determinado contexto. O conceito é simples; o que complica é a quantidade de objetos dentro dele. Os 4 objetos se resumem a uma tabela 2x2:

| | Vale em um namespace | Vale no cluster inteiro |
|---|---|---|
| **Define o que é permitido** | Role | ClusterRole |
| **Liga a permissão a alguém** | RoleBinding | ClusterRoleBinding |

Duas perguntas em sequência: o que? (Role) e a quem? (Binding). Só muda o alcance. De memória: o ClusterRole serve também para permissões em todos os namespaces, não só para recursos de cluster.

A permissão é um verbo sobre um recurso: `get` (ler um), `list` (listar), `watch` (acompanhar mudanças), `create`, `update`, `patch` (alterar em parte) e `delete`.

O RBAC só concede, nunca nega: o que nenhuma regra permite fica proibido por padrão. Em banco de dados existe `DENY`; aqui não.

Um **ServiceAccount** é a identidade de um Pod (um processo), em vez de uma pessoa, e recebe permissões da mesma forma.

Exemplo (da doc): um Role que só lê Pods e o RoleBinding que o dá à Jane.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]               # "" = o grupo principal da API
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl auth can-i list pods --namespace default --as jane      # yes
kubectl auth can-i get secrets --namespace default --as jane    # no
```

Ligação com os segredos ([kubernetes-configmap-e-secret.md](kubernetes-configmap-e-secret.md)): `get secrets` é o verbo que expõe o Secret. Quem só tem `pod-reader` lê Pods e não lê Secrets. Exemplo (meu): o desenvolvedor recebe o papel de ver Pods e logs em `dev`, e ninguém recebe `get secrets` nem `exec` em `prod`.

## NetworkPolicy

Controla quais conexões de rede os Pods podem fazer, por IP e porta (camadas 3 e 4). Por padrão, todo Pod aceita qualquer conexão e pode conectar em qualquer lugar. Se o Pod do painel admin for invadido, ele alcança o Postgres direto.

Requisito: o plugin de rede do cluster (CNI) precisa suportar NetworkPolicy, como Calico, Cilium ou Antrea. Sem suporte, o objeto é criado e não tem efeito.

Quando uma política seleciona um Pod, ele fica isolado naquela direção e só aceita o que as regras permitirem. Várias políticas se somam (vale a união).

A política protege Pods (`podSelector`), nas duas direções:

| Direção | Pergunta | Campo |
|---|---|---|
| **Ingress** | quem pode falar com o Pod? | `ingress.from` |
| **Egress** | com quem o Pod pode falar? | `egress.to` |

O "de onde" e o "para onde" aceitam 3 seletores:

| Seletor | Cobre |
|---|---|
| `podSelector` | outros Pods, por label |
| `namespaceSelector` | todos os Pods de um namespace, por label |
| `ipBlock` | faixas de IP, inclusive fora do cluster |

Ela cobre Pods, namespaces e o mundo externo, mas só tráfego de Pod: não há seletor de node nem de cluster. De memória (não está na página consultada): o tráfego entre o Pod e o próprio node dele costuma ser sempre permitido, e a política trabalha com Pods e IPs, não com Services.

Campos: `podSelector` (`{}` = todos os Pods do namespace), `policyTypes` (`Ingress`, `Egress`), `ingress.from`, `egress.to` e `ports`.

Exemplo 1 (da doc): negar toda entrada, o ponto de partida seguro.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: negar-entrada
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

Exemplo 2 (da doc, adaptado por mim): só a API acessa o Postgres, na porta 5432.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-so-api
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
```

```text
Pod api   ──> Pod postgres:5432     PERMITIDO
Pod admin ──> Pod postgres:5432     BLOQUEADO
```

Exemplo 3 (meu), de saída: a API só pode falar com o Postgres e o RabbitMQ.

```yaml
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
  - to:
    - podSelector:
        matchLabels:
          app: rabbitmq
```

No Docker, esse papel era da rede isolada do Compose (API e Postgres numa rede privada).

Deixado de fora: Pod Security Standards, ServiceAccount em detalhes, auditoria e os demais itens da seção de segurança da doc.

Fontes: [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) e [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/). As duas páginas vieram cortadas depois de 100 mil caracteres.
