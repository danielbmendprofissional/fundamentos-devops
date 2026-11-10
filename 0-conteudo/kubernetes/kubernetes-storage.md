# Kubernetes: storage

## Por que o Kubernetes tem o seu próprio volume

O Docker cuida do volume de uma máquina só: o volume nomeado do Compose (`pgdata`) é uma pasta no disco daquela máquina. Se o container morre, ele sobe de novo na mesma máquina e encontra o volume.

No Kubernetes o cluster tem vários nodes, e o Pod pode ser recriado em outro node. Um volume que mora no disco do node 1 não acompanha o Pod.

```text
ANTES                       DEPOIS (node 1 caiu)
node 1: Pod postgres        node 1: (morto, com o disco dele)
        volume local        node 2: Pod postgres novo
                                    volume local? NÃO EXISTE aqui. Dados perdidos.
```

O PV é normalmente um disco que não pertence a um node (disco de nuvem, NFS) e pode ser desconectado de um node e plugado em outro. Por isso o Kubernetes tem o seu próprio sistema de volumes (explicação de memória, não está na doc consultada).

```text
Pod postgres no node 1 ──> PVC ──> PV (disco de nuvem)
node 1 cai
Pod postgres no node 2 ──> PVC ──> o mesmo PV, plugado agora no node 2
```

Num cluster de um node só (um minikube local, por exemplo), esse problema quase não aparece.

## Kubernetes x Docker Compose

O Compose não tem relação com o Kubernetes: são esquemas diferentes. O Kubernetes não roda um Compose file. Você configura volume, variáveis e segredos do jeito do Kubernetes. O que os dois têm em comum é a imagem: o Dockerfile continua valendo, e o Kubernetes roda a imagem num Pod.

| | Docker Compose | Kubernetes |
|---|---|---|
| Escopo | uma máquina | vários nodes |
| Onde o volume mora | disco da máquina | disco que pode seguir o Pod entre nodes |
| Se o container cai | sobe de novo na mesma máquina | pode subir em outra |
| Volume nomeado (`pgdata`) | | PVC |
| `environment:` / `.env` | | ConfigMap |
| `configs:` | | ConfigMap (arquivo em volume) |
| `secrets:` | | Secret |
| Bind mount | | `hostPath` |
| Pasta temporária | | `emptyDir` |

O Kubernetes precisa de um container runtime em cada node, e o Docker é só uma das opções (o containerd é outra).

## Volume

Diretório que os containers do Pod enxergam. Resolve dois problemas (doc):

1. **Persistência:** se o container cai, o kubelet o reinicia com o disco limpo, e o volume guarda os arquivos.
2. **Compartilhamento:** containers do mesmo Pod trocam arquivos por ele.

### Volumes que vivem e morrem com o Pod

| Tipo | O que é |
|---|---|
| `emptyDir` | pasta temporária, existe enquanto o Pod existir; sobrevive a reinício do container, mas some com o Pod |
| `hostPath` | monta uma pasta do node no Pod; prende o Pod a um node e pode expor arquivos do host |
| `configMap` / `secret` | entregues como arquivos (ver [kubernetes-configmap-e-secret.md](kubernetes-configmap-e-secret.md)) |

Exemplo (meu): a API grava arquivos temporários.

```yaml
containers:
- name: api
  image: minha-api:abc123
  volumeMounts:
  - name: temporario
    mountPath: /tmp/uploads
volumes:
- name: temporario
  emptyDir: {}
```

Deixado de fora: o restante da lista de tipos da doc (nfs, iscsi, fc, csi, local, projected, downwardAPI, image e os depreciados gcePersistentDisk, portworxVolume, flexVolume, além do gitRepo, desabilitado).

## PV, PVC e StorageClass

```text
Pod ──usa──> PVC ──se liga a──> PV ──representa──> disco real (nuvem, NFS, ...)
                                  ^
                          StorageClass (a receita que cria o PV sob demanda)
```

| Objeto | Papel |
|---|---|
| **PersistentVolume (PV)** | um pedaço de armazenamento do cluster, com vida independente dos Pods (o disco) |
| **PersistentVolumeClaim (PVC)** | o pedido de armazenamento de quem vai usar: tamanho e modo de acesso |
| **StorageClass** | define como criar o armazenamento: tipo, desempenho, provisionador (a receita) |

O Pod nunca pede o PV: ele pede um PVC, e o Kubernetes liga o PVC a um PV compatível. Isso separa quem administra o disco de quem o usa.

- **Provisionamento estático:** o administrador cria os PVs antes.
- **Provisionamento dinâmico:** o PVC aponta para uma StorageClass e o cluster cria o disco na hora. `storageClassName: ""` desliga isso.

### accessModes

| Modo | Significado |
|---|---|
| `ReadWriteOnce` (RWO) | leitura e escrita por um node |
| `ReadOnlyMany` (ROX) | só leitura por vários nodes |
| `ReadWriteMany` (RWX) | leitura e escrita por vários nodes |
| `ReadWriteOncePod` (RWOP) | leitura e escrita por um Pod no cluster inteiro |

Nem todo disco suporta todos os modos; disco de bloco de nuvem costuma ser só RWO.

### reclaimPolicy

O que acontece com o PV quando o PVC é apagado:

- **Retain:** o PV e os dados ficam, e o administrador limpa à mão.
- **Delete:** apaga o PV e o disco junto; é o comum nos dinâmicos.

### Ciclo de vida do PV

`Available` (livre) → `Bound` (ligado a um PVC) → `Released` (o PVC foi apagado, aguardando recuperação) ou `Failed` (a recuperação falhou).

### Exemplo (meu): PVC e Pod do Postgres

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-dados
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: rapido        # a StorageClass do seu cluster
  resources:
    requests:
      storage: 5Gi
```

```yaml
# dentro do Pod / template do Postgres
volumeMounts:
- name: dados
  mountPath: /var/lib/postgresql/data
volumes:
- name: dados
  persistentVolumeClaim:
    claimName: postgres-dados
```

O StatefulSet, com o `volumeClaimTemplates`, cria um PVC por Pod automaticamente: é a mesma coisa, gerada pelo Kubernetes (ver [kubernetes-workloads.md](kubernetes-workloads.md)).

Fontes: [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) e [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). Ambas vieram cortadas depois dos primeiros 100 mil caracteres.
