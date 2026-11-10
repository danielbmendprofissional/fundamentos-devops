# Kubernetes: workloads

Workloads são as formas de gerenciar Pods por você. Lista da doc, em ordem:

1. Pod
2. Deployment e ReplicaSet
3. ReplicationController (legado)
4. StatefulSet
5. DaemonSet
6. Job
7. CronJob

## ReplicaSet

Mantém N Pods idênticos sempre rodando. É um controller aplicado a Pods.

- `replicas`: quantos Pods manter.
- `selector`: como achar os Pods dele (por label).
- `template`: o molde do Pod, usado quando precisa criar mais.

Exemplo (meu):

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: minha-api:abc123
        ports:
        - containerPort: 8080
```

Se um Pod morre, o ReplicaSet percebe 2 de 3 e cria outro.

```text
replicas: 3   ->  api-x1   api-x2   api-x3
um Pod morre  ->  cria outro para voltar a 3
```

Fonte: [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/).

## Deployment

Gerencia um conjunto de Pods para rodar uma aplicação, normalmente stateless. Fica por cima do ReplicaSet e acrescenta a atualização de versão sem derrubar tudo.

```text
Deployment  ──cria e gerencia──>  ReplicaSet  ──cria e mantém──>  Pods
```

Exemplo (meu): o YAML é igual ao do ReplicaSet, só muda o `kind`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: minha-api:abc123
        ports:
        - containerPort: 8080
```

```bash
kubectl apply -f deployment.yaml
kubectl get rs                                       # o ReplicaSet criado pelo Deployment
kubectl set image deployment/api api=minha-api:def456   # troca a versão
kubectl rollout status deployment/api
```

Ao trocar a imagem, o Deployment cria um ReplicaSet novo e reduz o antigo até 0 (rolling update). Por padrão, durante a troca ficam no mínimo 75% e no máximo 125% dos Pods.

```text
antes:   RS v1 [Pod][Pod][Pod]
durante: RS v1 [Pod][Pod]      RS v2 [Pod]
depois:  RS v1 (0 Pods)        RS v2 [Pod][Pod][Pod]
```

De memória (não confirmado na doc): `kubectl rollout undo deployment/api` volta à revisão anterior, `kubectl scale deployment/api --replicas=5` muda a escala e a estratégia `Recreate` derruba tudo antes de subir os novos.

Fonte: [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

## ReplicationController (legado)

Antecessor do ReplicaSet: também garante que N réplicas de Pods estejam sempre rodando. A doc o chama de API legada, substituída por Deployment e ReplicaSet.

| | ReplicationController | ReplicaSet |
|---|---|---|
| `apiVersion` | `v1` | `apps/v1` |
| `selector` | mapa simples, só igualdade | `matchLabels` e também `matchExpressions` |

Exemplo (meu):

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: api
spec:
  replicas: 3
  selector:
    app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: minha-api:abc123
```

Fonte: [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/).

## StatefulSet

Workload para aplicações com estado. Como o Deployment, cria Pods a partir de um template, mas cada Pod tem identidade fixa e os Pods não são intercambiáveis.

Garantias (doc):

1. **Nome estável:** `web-0`, `web-1`, `web-2`. Se um Pod morre, volta com o mesmo nome.
2. **Armazenamento próprio por Pod:** o `volumeClaimTemplates` cria um volume para cada Pod, e o Pod recriado volta a usá-lo.
3. **Ordem:** por padrão sobe um de cada vez, do 0 para cima, e remove na ordem inversa.
4. **Dados protegidos:** apagar ou reduzir o StatefulSet não apaga os volumes.

O campo `serviceName` aponta para um Service headless (`clusterIP: None`) que precisa ser criado. Ele dá a cada Pod um nome DNS estável. O Service é visto adiante.

| | Deployment | StatefulSet |
|---|---|---|
| Pods | intercambiáveis | cada um é único |
| Nome | aleatório | `nome-0`, `nome-1`... |
| Disco | compartilhado ou nenhum | um por Pod |
| Exemplo (meu) | a API | Postgres, RabbitMQ |

Exemplo (meu): Postgres. A estrutura é a da doc.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None             # headless
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres       # aponta para o Service headless acima
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:17
        volumeMounts:
        - name: dados
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:       # um disco por Pod
  - metadata:
      name: dados
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

Com `replicas: 3`, apareceriam `postgres-0`, `postgres-1` e `postgres-2`, cada um com o seu disco.

Fonte: [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/). A página veio cortada na parte de identidade do Pod.

## DaemonSet

Garante que todos os nodes (ou alguns deles) rodem uma cópia do Pod: 1 Pod em cada node. Node novo no cluster ganha o Pod automaticamente; node removido perde o dele; apagar o DaemonSet apaga os Pods que ele criou.

```text
DaemonSet "coletor-de-logs"
 ├─ node 1 ── Pod coletor
 ├─ node 2 ── Pod coletor
 └─ node 3 ── Pod coletor    <- node novo: o Pod aparece sozinho
```

Usos citados na doc:

1. Daemon de armazenamento do cluster em cada node.
2. Daemon de coleta de logs em cada node.
3. Daemon de monitoramento do node em cada node.

Sem restrição, vale para todos os nodes. Com `nodeSelector`, só para os nodes que combinam com a label (exemplo meu):

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
```

Exemplo (da doc, enxugado por mim): coletor de logs que lê o `/var/log` de cada node. Não tem `replicas`: a quantidade é uma por node.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

Fora do arquivo por decisão de enxugar: prioridade e preempção, `tolerations` para rodar nos nodes do control plane e como o scheduler vincula o Pod ao node.

Fonte: [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/).

## Job

Tarefa pontual que roda até terminar e depois para. Cria um ou mais Pods e continua tentando até que um número definido deles termine com sucesso. Se o Pod falha ou o node cai, cria outro. Apagar o Job apaga os Pods dele.

Diferente dos outros workloads, que mantêm algo rodando para sempre.

| Campo | Função |
|---|---|
| `completions` | quantos Pods precisam terminar com sucesso |
| `parallelism` | quantos Pods rodam ao mesmo tempo |
| `backoffLimit` | quantas tentativas antes de marcar como falho |
| `restartPolicy` | `Never` ou `OnFailure`; o Job não aceita `Always`, porque precisa terminar |
| `ttlSecondsAfterFinished` | quanto tempo depois de terminar o Job se apaga sozinho |

De memória (a parte final da página veio cortada), os 3 jeitos de usar: execução única (`completions` e `parallelism` iguais a 1), contagem fixa (`completions` maior que 1) e fila de trabalho (`completions` sem definir e `parallelism` maior que 1).

Exemplo (da doc): calcula o π com 2000 casas.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

```bash
kubectl apply -f job.yaml
kubectl describe job pi
kubectl logs jobs/pi
```

Exemplo (meu): migrations da API, rodando antes de subir a versão nova.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrar-banco
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 600
  template:
    spec:
      containers:
      - name: migrar
        image: minha-api-migracoes:abc123
      restartPolicy: Never
```

Fonte: [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/).

## CronJob

Cria Jobs em horários recorrentes, como uma linha do `crontab` do Linux. Serve para tarefas periódicas, como backup e relatório. Ele não executa nada sozinho.

```text
CronJob  ──no horário──>  Job  ──cria──>  Pod
```

O `schedule` tem 5 campos:

```text
┌───────── minuto (0-59)
│ ┌─────── hora (0-23)
│ │ ┌───── dia do mês (1-31)
│ │ │ ┌─── mês (1-12)
│ │ │ │ ┌─ dia da semana (0-6, domingo a sábado)
* * * * *
```

| Expressão | Significado |
|---|---|
| `* * * * *` | a cada minuto |
| `*/15 * * * *` | a cada 15 minutos |
| `0 0 * * *` | todo dia à meia-noite |
| `0 3 * * 1` | toda segunda às 3h |

Atalhos aceitos: `@hourly`, `@daily`, `@weekly`, `@monthly` e `@yearly`.

| Campo | Função |
|---|---|
| `schedule` | o horário (obrigatório) |
| `jobTemplate` | o molde do Job que será criado (obrigatório) |
| `concurrencyPolicy` | se o Job anterior ainda roda: `Allow` (padrão, roda junto), `Forbid` (pula a nova), `Replace` (troca pelo novo) |
| `startingDeadlineSeconds` | prazo para iniciar um Job que perdeu o horário; passou, pula |
| `suspend` | `true` suspende as próximas execuções |
| `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` | quantos Jobs terminados guardar (padrões 3 e 1, de memória: essa parte da doc veio cortada) |
| `timeZone` | o fuso horário (essa parte da doc veio cortada) |

Exemplo (da doc):

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            command: ["/bin/sh", "-c", "date; echo Hello from the Kubernetes cluster"]
          restartPolicy: OnFailure
```

Exemplo (meu): limpeza da API toda madrugada.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: limpar-pedidos-antigos
spec:
  schedule: "0 3 * * *"            # todo dia às 3h
  concurrencyPolicy: Forbid        # se o de ontem ainda roda, pula
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: limpar
            image: minha-api-limpeza:abc123
          restartPolicy: OnFailure
```

Detalhes da doc: o nome do CronJob pode ter no máximo 52 caracteres, e um Job que perde o prazo é tratado como falha.

Fonte: [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).
