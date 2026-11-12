# Kubernetes: probes e resources

## Probes

Medem a saúde da aplicação dentro do container. São definidas por container.

| Probe | A pergunta | Se falhar |
|---|---|---|
| **Liveness** | travou sem conseguir se recuperar? | o kubelet mata o container e reinicia |
| **Readiness** | está pronto para receber tráfego? | o Pod sai do Service, sem reiniciar |
| **Startup** | já terminou de subir? | enquanto não passar, liveness e readiness não começam; se falhar demais, reinicia |

A diferença que importa: liveness reinicia, readiness só tira do tráfego. Exemplo: a API perdeu a conexão com o banco por um minuto. A readiness falha e ela sai do Service, sem usuário ver erro, e volta quando o banco voltar. Se fosse liveness, seria reiniciada à toa.

A readiness é medida por container, mas o efeito é no Pod: o Pod só recebe tráfego do Service quando todos os containers dele estão prontos (de memória).

A startup serve para apps lentas ao subir: sem ela, o liveness poderia matar a app antes de ela terminar de iniciar.

### Mecanismos

- `httpGet`: um GET HTTP; 200 a 399 é sucesso.
- `tcpSocket`: a porta TCP aceita conexão?
- `exec`: roda um comando no container; saída `0` é sucesso.
- `grpc`: o protocolo de health check do gRPC, que a app precisa implementar.

### Parâmetros e padrões

Os padrões vieram do resumo de uma página que veio cortada; confira antes de depender deles.

| Parâmetro | Função | Padrão |
|---|---|---|
| `initialDelaySeconds` | espera antes da primeira checagem | 0 |
| `periodSeconds` | intervalo entre checagens | 10 |
| `timeoutSeconds` | tempo limite de cada checagem | 1 |
| `successThreshold` | sucessos seguidos para considerar OK | 1 |
| `failureThreshold` | falhas seguidas para considerar falha | 3 |

### Exemplo: a API .NET

Pressupõe os endpoints `/health/live` e `/health/ready` (os health checks do ASP.NET Core).

```yaml
containers:
- name: api
  image: minha-api:abc123
  ports:
  - containerPort: 8080
  startupProbe:                  # dá até 30 x 5 = 150 s para subir
    httpGet:
      path: /health/live
      port: 8080
    periodSeconds: 5
    failureThreshold: 30
  livenessProbe:                 # travou? reinicia
    httpGet:
      path: /health/live
      port: 8080
    periodSeconds: 10
  readinessProbe:                # sem banco? sai do Service
    httpGet:
      path: /health/ready
      port: 8080
    periodSeconds: 5
    failureThreshold: 3
```

## Resources: requests e limits

| | O que é | Quem usa |
|---|---|---|
| **request** | o mínimo que o container precisa | o scheduler escolhe um node com essa quantidade livre, e o kubelet reserva |
| **limit** | o máximo que ele pode usar | o kernel impõe, via cgroups |

Se só o limit for definido, o Kubernetes copia o valor para o request. O scheduler decide pelos valores declarados, não pelo consumo real.

Unidades: CPU em núcleos (`1` é um núcleo, `500m` é meio núcleo); memória com sufixo (`128Mi`, `1Gi`; `Mi` é binário, `M` é decimal).

Ao passar do limite:

| Recurso | Consequência |
|---|---|
| CPU | throttling: o container é freado, fica lento, mas não morre |
| Memória | OOMKilled: o kernel mata o container, e a política de reinício decide se ele volta |

### Classes de QoS

Quem sai primeiro quando o node aperta:

| Classe | Critério | Remoção sob pressão |
|---|---|---|
| **Guaranteed** | todos os containers com request igual ao limit, em CPU e memória | por último |
| **Burstable** | pelo menos um request ou limit, mas não é Guaranteed | no meio |
| **BestEffort** | nenhum request nem limit | primeiro |

Exemplo: a API, com request menor que o limit (Burstable).

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"        # acima: throttling
    memory: "256Mi"    # acima: OOMKilled
```

Fontes: [Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) e [Resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/). As duas páginas vieram cortadas depois de 100 mil caracteres.
