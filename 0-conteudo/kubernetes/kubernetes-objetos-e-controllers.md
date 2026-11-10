# Kubernetes: objetos, controllers e imperativo x declarativo

## Objeto

É o que se registra no Kubernetes, como o Pod. É um "registro de intenção": você diz o que quer e o Kubernetes trabalha para manter isso.

- `spec`: o estado **desejado**, escrito por você.
- `status`: o estado **atual**, escrito pelo Kubernetes.

Campos obrigatórios do manifesto: `apiVersion`, `kind`, `metadata` (nome, UID, namespace opcional) e `spec`.

## Controller

É um serviço em loop que fica analisando o estado **desejado** e o estado **real** do cluster e vai alinhando o real para o esperado.

Ele não sobe o container sozinho: pede ao kube-apiserver (por exemplo, para criar um Pod). O scheduler escolhe o node e o kubelet sobe o container.

Exemplo (meu), a mesma ideia em C#:

```csharp
while (true)
{
    var desejado = 3;                      // o que foi pedido (spec)
    var atual = ContarPodsDaApi();         // o que existe de verdade (status)

    if (atual < desejado) CriarPod();
    if (atual > desejado) RemoverPod();

    await Task.Delay(1000);
}
```

## Imperativo x declarativo

Esta parte é de memória, não está na doc consultada.

- **Imperativo:** você manda a ação. Rodar de novo pode dar erro.
- **Declarativo:** você escreve o estado final num arquivo e aplica. Rodar de novo dá no mesmo. É o que combina com o pipeline: o YAML fica no Git.

```bash
kubectl run api --image=minha-api:abc123   # imperativo (comando)
kubectl create -f pod.yaml                 # imperativo (a partir de arquivo)
kubectl apply -f pod.yaml                  # declarativo
```

Fontes: [Objetos](https://kubernetes.io/docs/concepts/overview/working-with-objects/) e [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/).
