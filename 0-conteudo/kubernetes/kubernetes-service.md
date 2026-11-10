# Kubernetes: Service

Os Pods são efêmeros e cada Pod novo vem com um IP novo, então quem chama a API não pode guardar o IP de um Pod. O Service dá um ponto de acesso estável a um conjunto de Pods: quem fala com a API fala com o Service, e o Service repassa a requisição para um dos Pods, sem que quem chamou saiba qual. O Service não é um workload: ele não roda nada, só aponta.

- `selector`: escolhe os Pods por label (a mesma label do Deployment).
- O Service recebe um IP fixo (cluster IP).
- Dentro do cluster, cada Service ganha um nome interno, um DNS interno. No mesmo namespace basta o nome (`api`); entre namespaces, `api.<namespace>` (de memória, a parte do DNS da doc veio cortada).

```text
outra API ──> Service "api" (IP fixo)
                  ├─> Pod api-x1
                  ├─> Pod api-x2
                  └─> Pod api-x3     <- Pods vêm e vão, o Service continua o mesmo
```

Exemplo (meu), na frente do Deployment da API:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api              # os Pods com esta label recebem o tráfego
  ports:
  - protocol: TCP
    port: 80              # a porta do Service
    targetPort: 8080      # a porta em que o container escuta
```

## Tipos de Service

O tipo define de onde se pode chamar o Service.

1. **ClusterIP** (o padrão): somente interno ao cluster. Os Pods e os nodes podem chamar, o externo não chega.
2. **NodePort:** abre uma porta em cada node, e assim o externo consegue chamar o Service pelo node.
3. **LoadBalancer:** o Service pede à nuvem um balanceador de carga externo, uma espécie de gateway público na frente do Service, com balanceamento de carga.
4. **ExternalName:** não tem Pods por trás. É um apelido DNS dentro do cluster que aponta para um endereço de fora. Exemplo (meu): o Postgres é um serviço gerenciado fora do cluster, e a API chama o nome `banco`.
5. **Headless:** `clusterIP: None`, sem IP único. O DNS devolve os IPs dos Pods diretamente. É o que o StatefulSet exige (`serviceName`). (De memória; o usuário ainda não assimilou esse tipo.)

```yaml
# ClusterIP (padrão)
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
```

```yaml
# NodePort
spec:
  type: NodePort
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080      # porta aberta em cada node (faixa 30000-32767, de memória)
```

```yaml
# LoadBalancer
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
```

```yaml
# ExternalName
apiVersion: v1
kind: Service
metadata:
  name: banco
spec:
  type: ExternalName
  externalName: meu-banco.postgres.exemplo.com
```

```yaml
# Headless
spec:
  clusterIP: None
  selector:
    app: postgres
```

| Tipo | Quem consegue chamar | Exemplo (meu) |
|---|---|---|
| ClusterIP | só dentro do cluster | worker → API; API → Postgres |
| NodePort | quem sabe o IP e a porta de um node | teste no laboratório |
| LoadBalancer | a internet, via balanceador da nuvem | expor a API em produção |
| ExternalName | (apelido para algo de fora) | banco gerenciado fora do cluster |
| Headless | quem precisa de cada Pod individual | `postgres-0` do StatefulSet |

Fonte: [Service](https://kubernetes.io/docs/concepts/services-networking/service/). A descrição dos tipos NodePort, LoadBalancer, ExternalName e Headless é de memória: a página veio cortada.
