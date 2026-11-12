# Docker: limites de recurso

Por padrão, um container não tem limite nenhum: ele usa o que o kernel do host permitir. Sem limite, um container com vazamento de memória afeta os outros e o próprio host.

## Memória

Se a memória acaba, o container recebe um `OOME` (Out Of Memory Exception) e o kernel começa a matar processos. O Docker reduz a prioridade de OOM do daemon, mas não ajusta a dos containers.

| Flag | O que faz |
|---|---|
| `-m` / `--memory` | limite máximo (mínimo 6 MB) |
| `--memory-swap` | quanto de swap pode usar (só vale junto com `--memory`) |
| `--memory-swappiness` | facilidade com que o kernel manda páginas para o swap (0 a 100) |
| `--memory-reservation` | limite **suave**, menor que `--memory`, ativado quando há disputa por memória |
| `--oom-kill-disable` | desliga o OOM killer (só junto com `-m`) |

Swap: com `--memory-swap` igual a `--memory`, não há swap. Sem `--memory-swap`, o container pode usar swap até o valor do limite de memória (se o host tiver). Com `-1`, swap ilimitado.

## CPU

Por padrão o acesso à CPU é ilimitado.

| Flag | O que faz |
|---|---|
| `--cpus` | teto de CPU: `--cpus="1.5"` = no máximo uma CPU e meia |
| `--cpu-period` / `--cpu-quota` | período do CFS (padrão 100000 µs) e o tempo permitido por período |
| `--cpuset-cpus` | restringe a núcleos específicos |
| `--cpu-shares` | peso relativo (padrão 1024), só aplicado quando falta CPU |

`--cpus=".5"` equivale a `--cpu-period=100000 --cpu-quota=50000`.

## Exemplos

Pela linha de comando, a API com meio núcleo e 512 MB:

```bash
docker run -d --cpus=".5" --memory=512m --memory-reservation=256m minha-api
docker stats      # mostra o uso de CPU e memória de cada container, ao vivo
```

No Compose (exemplo adaptado da doc):

```yaml
services:
  api:
    image: minha-api
    deploy:
      resources:
        limits:             # o teto: o container não passa disso
          cpus: '0.50'
          memory: 512M
        reservations:       # o mínimo garantido
          cpus: '0.25'
          memory: 256M
```

A doc do Compose não diz se o `deploy` é respeitado fora do Swarm. Confira com o `docker stats`.

## cgroups

O Docker modifica as configurações do cgroup do container no host. Cgroups são o mecanismo do kernel Linux que limita recursos de um grupo de processos (de memória).

Fontes: [Resource constraints](https://docs.docker.com/engine/containers/resource_constraints/) e [Compose, deploy](https://docs.docker.com/reference/compose-file/deploy/).
