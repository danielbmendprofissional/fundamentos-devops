# Kubernetes: Pod

É a menor parte do Kubernetes. É nele que fica o container. Pode armazenar 1 ou N containers, sendo por padrão 1 container, e os Pods são descartáveis.

## Exemplo (da doc)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

## Exemplo (meu): a API .NET em um Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api
spec:
  containers:
  - name: api
    image: minha-api:abc123     # imagem gerada pelo pipeline (tag = SHA do commit)
    ports:
    - containerPort: 8080
```

Fonte: [Pods](https://kubernetes.io/docs/concepts/workloads/pods/).
