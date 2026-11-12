# Kubernetes: ConfigMap e Secret

O critério é o mesmo do Docker: configuração não sensível vai para o ConfigMap, dado sensível vai para o Secret.

## ConfigMap

Guarda pares chave-valor não sensíveis, separados da imagem, para que a mesma imagem rode em dev e em prod mudando só a configuração. Limite de 1 MiB. Não oferece sigilo nenhum.

Exemplo, para a API:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Logging__LogLevel__Default: "Information"
  appsettings.Production.json: |        # um arquivo inteiro como valor
    {
      "Features": { "NovoCheckout": true }
    }
```

4 formas de usar (doc):

1. nos `command` e `args` do container;
2. como variável de ambiente;
3. como arquivo num volume, somente leitura;
4. código dentro do Pod que consulta a API do Kubernetes.

Exemplo, no Deployment da API, com variável de ambiente e arquivo:

```yaml
containers:
- name: api
  image: minha-api:abc123
  envFrom:
  - configMapRef:
      name: api-config
  volumeMounts:
  - name: config
    mountPath: /app/config
    readOnly: true
volumes:
- name: config
  configMap:
    name: api-config
```

De memória (a página veio cortada): arquivos em volume se atualizam sozinhos quando o ConfigMap muda; variáveis de ambiente não, exigem reiniciar o Pod. `immutable: true` (v1.19) trava o ConfigMap contra alterações.

## Secret

Guarda senha, token e chave, para que não fiquem na spec do Pod nem na imagem.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credenciais
type: Opaque
stringData:
  senha: "exemplo"
```

Exemplo, usando na API:

```yaml
env:
- name: ConnectionStrings__Senha
  valueFrom:
    secretKeyRef:
      name: db-credenciais
      key: senha
```

Tipos de Secret (lista visível na doc, que veio cortada): Opaque (padrão), ServiceAccount token, Docker config, Basic authentication, SSH authentication, TLS e Bootstrap token. Os mais usados: Opaque, TLS e Docker config.

### Secret não é cofre

Por padrão fica sem criptografia no etcd, e os valores em `data` são só base64, que não é criptografia. Quem acessa a API ou o etcd lê o Secret, e quem pode criar um Pod num namespace consegue ler todos os Secrets dele.

Nenhum segredo é 100% secreto: a API precisa ler a senha, então quem tem acesso ao processo também lê. O que se faz é reduzir quem chega até ele (camadas, de memória):

```text
1. Disco / backup do etcd vazou       -> criptografia em repouso protege
2. Alguém com acesso à API do cluster -> RBAC decide quem lê Secrets
3. Alguém dentro do Pod (exec)        -> lê tudo que a aplicação lê
```

```bash
kubectl get secret db-credenciais -o jsonpath='{.data.senha}' | base64 -d
kubectl exec api-7d9f -- env
```

O que a doc manda fazer:

1. Ativar criptografia em repouso no etcd.
2. RBAC com privilégio mínimo.
3. Restringir o acesso aos containers que precisam.
4. Considerar um provedor externo de segredos.

### Cofres externos (só os nomes)

O segredo fica fora do cluster e é buscado quando precisa, com auditoria de quem acessou.

- **Secrets Store CSI Driver** (citado na doc): faz a ponte entre o cluster e um cofre.
- **Azure Key Vault**
- **HashiCorp Vault**
- **AWS Secrets Manager** (de memória)
- **Google Secret Manager** (de memória)

## Docker Compose x Kubernetes

| Docker Compose | Kubernetes |
|---|---|
| `environment:` / `.env` | ConfigMap (variável de ambiente) |
| `configs:` | ConfigMap (arquivo em volume) |
| `secrets:` | Secret |
| `/run/secrets/db_password` | Secret montado em volume |

O `AddKeyPerFile` do ASP.NET Core continua valendo: lê o Secret montado como arquivos.

Fontes: [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) e [Secret](https://kubernetes.io/docs/concepts/configuration/secret/).
