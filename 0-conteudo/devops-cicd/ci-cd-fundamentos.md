# CI/CD: fundamentos

## CI

É o processo de integração do código, passando por build, tests e artefato (dependendo do contexto de continuidade ao CD). Exemplos:

1. Finalizei um código na feature/kkkkk e abro um MR para main. Nesse momento roda um pipeline de CI: build e tests para validar. O artefato gerado aqui, se houver, é descartável, só para validar a feature em um ambiente temporário, porque ainda não sei se será aprovado ou não.

2. Aprovado para main roda novamente o pipeline, porém agora um pipeline de CI-CD: build, test e agora o artefato oficial, que será reaproveitado no CD, que seria o deploy seguindo a ideia da separação de ambientes, começando nesse exemplo em DEV.

A ideia é que o artefato gerado no CI seja o mesmo publicado em todos os ambientes até chegar em produção.

## Entrega Contínua

O artefato é publicado automaticamente nos ambientes anteriores e fica sempre pronto para produção, mas o deploy em produção depende de aprovação de uma pessoa.

## Implantação Contínua

É quando o deploy em produção também é automático, sem aprovação humana.

## Branch por ambiente

Existe uma ideia errada de manter branchs (dev, qa, hml e main) por uma falsa sensação de processo de devops, porém isso fere o princípio de que todo fluxo precisa ter o MESMO artefato desde o início. Normalmente nesses cenários cada branch inicia um pipeline e gera um artefato dela mesma. Claro que em determinados conceitos isso pode ser mitigado e considerado um processo aceitável de ci-cd, porém não é o correto e recomendado.

## O recomendado

O recomendado seria existir somente a branch main e as branchs de feature, e lidar com os ambientes pelo processo de ci-cd. Exemplos:

1. Eu termino uma feature e abro o MR para main. Na abertura desse MR eu publico a imagem descartável em um ambiente (dev|qa ou um ambiente temporário) para que seja validada a implementação da feature.

2. Assim que validada eu aprovo o MR e jogo o código na main, utilizando o conceito de feature flags: todas as N features de N processos ficam dentro do código da MAIN enquanto são validadas em ambientes de dev, qa e talvez até mesmo homolog. O flag fica ligado nesses ambientes e desligado em produção até a aprovação, e quem liga é a configuração do ambiente.

Também existe o git revert para desfazer uma mudança na main sem reescrever o histórico.

Confesso que é um pouco complicado enxergar esse cenário em uma aplicação com demandas constantes com áreas de negócio confusas que podem pedir, negar e corrigir de um dia para noite com prazos complicados, porém esses são os conceitos corretos...

---

## Possível cenário: develop, release e main

Só a main não é a única forma aceita. O modelo clássico com várias branchs é o **Gitflow**, muito comum em empresas com janela de release, auditoria ou várias versões em paralelo. A recomendação por uma main única (trunk-based development) vem da pesquisa do DORA, que associa os melhores resultados de entrega a esse modelo, mas é uma recomendação e não uma lei.

### As branchs

- **main:** só o que já foi para produção. Cada release vira uma tag (`v1.4.0`).
- **develop:** onde as features são integradas. É o equivalente da "main" do modelo recomendado.
- **feature/\*:** nascem da develop e voltam para ela por MR.
- **release/\*:** cortada da develop quando se decide fechar uma versão. Ali só entram correções da estabilização, nenhuma feature nova.
- **hotfix/\*:** correção urgente de produção, nasce da main.

### O fluxo

```text
feature/x --MR--> develop --corta--> release/1.4 --valida--> main (tag v1.4.0)
                     ^                    |                      |
                     +---- correções -----+                      |
                     +----------- volta o que mudou -------------+
```

1. As features entram na develop por MR, com CI.
2. Quando é hora de fechar a versão, corta-se a `release/1.4`. A develop segue recebendo features novas sem afetar a release.
3. A release é validada em qa e hml. Correções feitas ali também voltam para a develop.
4. Aprovada, a release é mergeada na main e ganha a tag.
5. Um hotfix nasce da main, é corrigido, volta para a main e também para a develop.

### Como fica a regra do mesmo artefato

O problema da branch por ambiente é cada branch gerar um artefato diferente. No Gitflow ele pode ser mitigado assim:

- O artefato é gerado **uma vez**, na `release/1.4`.
- Esse mesmo artefato é promovido por qa, hml e produção.
- O merge na main serve para registrar a versão (tag), e a pipeline da main **não faz novo build**: reaproveita a imagem da release.

Se a main rebuildar, o que vai para produção deixa de ser o que foi testado.

### Prós e contras

| | |
|---|---|
| **A favor** | Controle explícito de quais mudanças entram em cada versão. Bom para auditoria e para manter versões antigas. |
| **Contra** | Mais merges, mais conflitos, risco de a release divergir da develop, e as features ficam mais tempo sem chegar a produção. |

Quando o objetivo é entregar com frequência, o modelo de main única com feature flags costuma ser mais simples. Quando o negócio exige escolher exatamente o que vai em cada versão, o Gitflow faz sentido.
