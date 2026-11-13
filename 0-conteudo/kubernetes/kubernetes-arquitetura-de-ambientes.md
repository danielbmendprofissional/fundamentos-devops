# Kubernetes: arquitetura de clusters por ambiente

Fonte: [Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/). A doc trata só de isolar por namespace ou por cluster separado; as arquiteturas por ambiente são prática de mercado (de memória).

## Namespace ou cluster separado

| | Namespace (mesmo cluster) | Cluster separado |
|---|---|---|
| Isola | nomes dos objetos, RBAC, NetworkPolicy, quotas | tudo: control plane, nodes, rede |
| Não isola | o cluster em si: nodes compartilhados, um erro no control plane afeta todos | |
| Custo | baixo | alto: cada cluster tem control plane, upgrades e manutenção próprios |

- O isolamento forte pesa contra o custo e a complexidade de gerenciar vários clusters.
- O namespace só isola se RBAC, rede e quotas forem configurados.

## Arquiteturas (de memória)

Cenário: dev, qa, hml, produção e review apps (um por feature).

### A) 1 cluster, 1 namespace por ambiente

```text
cluster único
├─ ns dev
├─ ns qa
├─ ns hml
├─ ns prod
└─ ns review-feature-123 (um por MR)
```

- Barato e simples; serve para estudo ou empresa muito pequena.
- Um deploy ou upgrade errado derruba prod junto, e um dev com permissão larga alcança prod.

### B) 2 clusters: não-produção e produção

```text
cluster NÃO-PROD                    cluster PROD
├─ ns dev                           └─ ns prod
├─ ns qa
├─ ns hml
└─ ns review-*
```

- É a mais comum. Prod fica isolado, e o upgrade do Kubernetes é testado primeiro no não-prod.
- Dev, qa e hml compartilham nodes: um teste de carga em hml pode atrapalhar o dev. Mitigação: quotas por namespace e pools de nodes separados.

### C) 3 clusters: dev/review, pré-prod (qa + hml), prod

- Separa o ambiente instável (dev e review apps, que sobem e caem o tempo todo) do ambiente estável de homologação. Útil com muitos review apps.

### D) 4 ou mais clusters, um por ambiente

- Isolamento máximo: cada ambiente com seu upgrade, RBAC e rede.
- O mais caro e o que dá mais manutenção (4 control planes, 4 conjuntos de credenciais, 4 deploys). Aparece em empresa grande ou com exigência de compliance.

### E) Cluster de prod por região ou cliente

- Outro eixo: não separa ambientes, e sim disponibilidade e tenants.

## Review apps

- Em qualquer arquitetura, o review app é um namespace temporário, um por MR, criado no pipeline e apagado ao fechar o MR (`on_stop` e `auto_stop_in` do GitLab).
- Nunca vai para o cluster de prod.
- Úteis: `ResourceQuota` pequena por namespace, `auto_stop_in` curto e nodes baratos.

## Como escolher

| Critério | Empurra para |
|---|---|
| Orçamento apertado, time pequeno | A ou B |
| Prod com isolamento e upgrade controlado | B ou mais |
| Muitos devs e muitos review apps | C |
| Auditoria ou compliance exigindo separação | D |
| Equipe de plataforma pequena | menos clusters |

A pergunta que decide: o que acontece com prod se o ambiente de baixo cair ou for atacado?
