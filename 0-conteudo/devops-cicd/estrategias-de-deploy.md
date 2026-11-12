# Estratégias de deploy

## Recreate

É a que usamos em uma janela de deploy às 6 da manhã: derrubamos uma versão e subimos outra no lugar. Se der ruim, o plano de rollback é subir a que já funcionava (o rollback é lento, leva o tempo de um deploy inteiro).

## Rolling

É ir gradualmente alterando as instâncias, já que normalmente uma aplicação tem mais de uma instância publicada em produção, mais de um node talvez.

O perigo é a mudança de banco: e se a versão nova tem uma alteração no banco que a antiga não tem? A solução é o padrão **expand/contract**, dividindo a mudança em passos compatíveis com as duas versões:

1. **Expand:** adiciona a coluna nova, sem remover nada (a versão antiga ignora o que não conhece).
2. **Migrate:** a versão nova passa a usar a coluna nova.
3. **Contract:** só em um deploy posterior, quando a versão antiga já não existe, remove a coluna antiga.

Renomear ou remover coluna em um deploy só é o que quebra o rolling. Isso vale também para blue/green, porque os dois ambientes normalmente compartilham o mesmo banco.

## Blue/Green

É o que tem o rollback mais rápido, porém o mais caro, porque a ideia é manter o antigo publicado e subir também o novo em outra infra completa. Com o novo validado, o tráfego troca de uma vez.

O blue não é derrubado logo após a validação: ele fica no ar por um tempo, justamente para o rollback instantâneo valer. Só depois é desligado. E na hora da troca todos os usuários passam para a versão nova de uma vez, quem limita o estrago é o canary.

## Canary

Libera uma versão para uma pequena parte dos usuários, vai avaliando métricas e depois aumentando até chegar a 100%.

Não se controla por feature, e sim por **tráfego**: na frente das instâncias fica um balanceador (Ingress, service mesh) configurado com pesos, por exemplo 95% das requisições para a v1 e 5% para a v2. Também dá para escolher quem cai na v2 por critério, como header, cookie ou hash do id do usuário (o hash garante que o mesmo usuário sempre cai na mesma versão). No Kubernetes isso é feito por ferramentas como Argo Rollouts com Istio, Traefik ou Gateway API (o Ingress NGINX está sendo aposentado e não deve ser usado em projetos novos).

## Canary x feature flag

O flag no código indica o que vai e o que não vai subir para produção (ser liberado). O canary, em algo já publicado em produção, configura a porcentagem de usuários que acessarão a versão nova ou a antiga.

Ou seja: o canary divide **entre versões do artefato**, o flag liga ou desliga **dentro do mesmo artefato**. A fronteira é turva porque os flags também permitem liberar para uma porcentagem de usuários (*percentage rollout*).

## Exemplos visuais

Cada `[v1]` é uma instância da aplicação rodando a versão 1; `[v2]`, a versão 2.

```text
RECREATE                         ROLLING (3 instâncias)
t0  [v1] [v1] [v1]               t0  [v1] [v1] [v1]
t1  [  ] [  ] [  ]  <- fora      t1  [v2] [v1] [v1]
t2  [v2] [v2] [v2]               t2  [v2] [v2] [v1]
                                 t3  [v2] [v2] [v2]

BLUE/GREEN                       CANARY
antes da troca:                  usuários --95%--> [v1] [v1] [v1]
 usuários --> [BLUE  v1 x3]                 --5%--> [v2]
              [GREEN v2 x3] (validando)
depois da troca:
 usuários --> [GREEN v2 x3]
              [BLUE  v1 x3] (fica no ar p/ rollback)
```

Expand/contract com um banco real: renomear a coluna `Nome` para `NomeCompleto` sem quebrar o rolling (SQL ilustrativo, estilo Postgres):

```sql
-- 1. Expand (deploy 1): cria a coluna nova; a v1 ignora o que não conhece
ALTER TABLE Cliente ADD COLUMN NomeCompleto text NULL;

-- 2. Migrate (deploy 2): a v2 passa a gravar e ler NomeCompleto; copia o que já existia
UPDATE Cliente SET NomeCompleto = Nome WHERE NomeCompleto IS NULL;

-- 3. Contract (deploy 3, depois que a v1 não existe mais): remove a antiga
ALTER TABLE Cliente DROP COLUMN Nome;
```

Fazer os três passos num deploy só (`RENAME COLUMN`) é o que derruba a v1 enquanto ela ainda está no ar.

## Resumo

| Estratégia | Indisponibilidade | Custo extra | Rollback | Risco exposto |
|---|---|---|---|---|
| Recreate | sim | nenhum | lento | todos |
| Rolling | não | baixo | médio | gradual |
| Blue/Green | não | alto | instantâneo | todos, na troca |
| Canary | não | baixo | rápido | poucos primeiro |
