# Kubernetes: namespaces e kubectl

## Namespace

Mecanismo para isolar grupos de recursos dentro do mesmo cluster. O nome de um recurso precisa ser único dentro do namespace, não entre eles. Não se aninham, e cada recurso pertence a um só.

Para que serve (doc):

1. Dividir o cluster entre times e projetos, junto com resource quotas (limites de consumo por namespace).
2. Dar escopo aos nomes: o Service `api` pode existir em `dev` e em `prod`.
3. Separar ambientes (Development, Staging, Production).

```text
CLUSTER
 ├─ namespace dev    ── Deployment api, Service api, Postgres
 ├─ namespace hml    ── Deployment api, Service api, Postgres
 └─ namespace prod   ── Deployment api, Service api, Postgres
```

### Namespaces que já vêm criados

| Namespace | Função |
|---|---|
| `default` | para começar a usar sem criar nada |
| `kube-system` | objetos criados pelo próprio Kubernetes |
| `kube-public` | legível por todos, inclusive sem autenticação; reservado ao cluster |
| `kube-node-lease` | heartbeats dos nodes, para o control plane detectar falha |

### O que não pertence a namespace

Recursos de escopo de cluster: Nodes, PersistentVolumes e StorageClasses. O PVC, ao contrário, é de namespace.

```bash
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

### DNS entre namespaces

```text
mesmo namespace:   http://banco
outro namespace:   http://banco.producao.svc.cluster.local
```

### Exemplo (da doc, adaptado por mim)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: desenvolvimento
---
apiVersion: v1
kind: Service
metadata:
  name: banco
  namespace: desenvolvimento
spec:
  selector:
    app: banco
  ports:
  - port: 5432
```

```bash
kubectl create namespace desenvolvimento
kubectl get pods -n desenvolvimento
kubectl get pods -A
```

Boas práticas (doc): em produção, evitar o `default` e criar namespaces próprios; não usar o prefixo `kube-`, reservado ao sistema. Em cluster pequeno, a doc diz para nem se preocupar com namespaces. Para separar versões do mesmo software, usa-se labels, não namespaces.

## kubectl: comandos do dia a dia

A página consultada veio cortada antes de logs, exec, rollout e escala; esses grupos são de memória. Confira a sintaxe na sua versão.

**Contexto** (em qual cluster e namespace você está falando):

```bash
kubectl config get-contexts                               # lista os clusters configurados
kubectl config use-context NOME                           # troca de cluster
kubectl config set-context --current --namespace=dev      # namespace padrão
```

**Aplicar e criar:**

```bash
kubectl apply -f arquivo.yaml        # cria ou atualiza (declarativo; o recomendado)
kubectl apply -f ./pasta             # tudo de uma pasta
kubectl explain pods                 # a documentação dos campos, no terminal
```

**Ver o que existe:**

```bash
kubectl get pods -A                  # de todos os namespaces
kubectl get pods -o wide             # com IP e node
kubectl describe pod NOME            # detalhes e eventos: o primeiro passo no diagnóstico
kubectl get events --sort-by=.metadata.creationTimestamp
```

**Logs e entrar no container:**

```bash
kubectl logs POD                     # logs
kubectl logs -f POD -c CONTAINER     # em tempo real, de um container do Pod
kubectl logs --previous POD          # da instância anterior, depois de um reinício
kubectl exec -it POD -- /bin/sh      # abre um shell dentro do container
kubectl port-forward pod/NOME 8080:80   # porta local 8080 -> porta 80 do Pod
```

**Deploy:**

```bash
kubectl rollout status deployment/api
kubectl rollout history deployment/api
kubectl rollout undo deployment/api      # volta à revisão anterior
kubectl rollout restart deployment/api   # reinicia os Pods aos poucos
kubectl scale deployment/api --replicas=5
```

**Apagar e consumo:**

```bash
kubectl delete -f arquivo.yaml
kubectl delete pod NOME              # se tem controller, ele recria
kubectl top nodes                    # CPU e memória dos nodes
kubectl top pods --sort-by=cpu
```

O `kubectl top` precisa de um componente de métricas instalado no cluster (de memória).

### Caminho de diagnóstico típico (meu)

```text
Pod não sobe?  kubectl get pods  ->  kubectl describe pod X  (eventos)  ->  kubectl logs X
Reiniciando?   kubectl logs --previous X   (probe falhando? OOMKilled? veja o describe)
```

Fontes: [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) e [kubectl quick reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/).
