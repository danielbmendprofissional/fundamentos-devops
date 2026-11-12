# Métricas DORA

1. **Deployment Frequency:** com que frequência as mudanças vão para produção?
2. **Lead time for changes:** quanto tempo do commit até a etapa de produção?
3. **Change failure rate:** qual é a porcentagem dos deploys que falha em produção?
4. **Time to Restore:** quanto tempo leva para o time restaurar o serviço após falha? (hoje chamada de *Failed deployment recovery time*)
5. **Deployment rework rate:** qual a proporção de deploys não planejados, feitos como consequência de um incidente em produção?

## Exemplo visual: onde cada métrica é medida

```text
commit ---- merge ---- deploy em prod ---------- falha ---------- serviço restaurado
  |<-- lead time for changes -->|                  |<-- failed deployment recovery time -->|

deployment frequency  = quantos deploys em produção por período
change failure rate   = deploys que falharam / total de deploys
deployment rework rate = deploys não planejados (por incidente) / total de deploys
```

Números ilustrativos de um mês: 20 deploys, 3 falharam e 2 foram correções não planejadas por incidente. Frequência: 20 por mês. Change failure rate: 3/20 = 15%. Rework rate: 2/20 = 10%. Se o primeiro desses deploys levou 2 dias do commit à produção, o lead time dele é de 2 dias.

## Como se agrupam

- **Throughput (vazão):** lead time, deployment frequency e failed deployment recovery time.
- **Instability (instabilidade):** change failure rate e deployment rework rate.

A conclusão da pesquisa é que velocidade e estabilidade não são opostos: os melhores times vão bem em todas as métricas ao mesmo tempo.

Fonte: [dora.dev/guides/dora-metrics](https://dora.dev/guides/dora-metrics/). O modelo original tinha 4 métricas ("four keys"); o atual tem 5.
