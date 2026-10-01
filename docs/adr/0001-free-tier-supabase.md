# ADR 0001 — Plano gratuito no Supabase, com risco aceito

**Status:** aceito · **Data:** 2026-09-30 · **Decisão:** D3

## Contexto

O projeto depende de dois ambientes Supabase (dev e produção), o que consome exatamente o
limite do plano gratuito: **2 projetos ativos**, folga zero. O plano gratuito **pausa projetos**
**após 7 dias de baixa atividade de banco** e **não faz backup automático**.

O piloto do produto é, por definição, de baixo volume — um bairro. Esse é exatamente o padrão
que dispara a pausa. O plano gratuito também tem egress a 5 GB, storage a 1 GB e arquivo a
50 MB.

## Decisão

**Ambos os projetos no plano gratuito. Sem job anti-pausa, sem Pro.**

Opções considered e recusadas:

| Opção | Por que não |
| --- | --- |
| Pro em produção (~$5/mês) | Resolve pausa e dá backup, mas é custo recorrente para um piloto que talvez nunca tenha usuário |
| Free + job anti-pausa diário | Resolve o modo de falha provável por ~15 linhas, mas é mecanismo de infraestrutura que existe só para contornar um limite do plano — ruído no repositório |
| Free aceitando o risco | **Escolhida.** É a mais simples, e a simplicidade é o que sustenta o custo de manutenção de um projeto solo |

## Risco aceito

**O projeto de produção pode pausar sozinho** se o bairro ficar uma semana sem atividade. O
piloto morre sem aviso: o usuário vê erro, não "volta logo".

**Não há rollback de migração** em produção, porque não há backup. Uma migration com erro vai
precisar de correção manual, na frente.

Esses dois riscos são o preço de não pagar, e ficam **escritos aqui e no README** — não
escondidos. Um leitor do portfólio precisa saber que a escolha foi consciente.

## Consequências

- **T-7.1** (workflow de migrações) revisa a migration duas vezes antes de produção, já que não
há rede de segurança.
- **T-8.4** (piloto) trata a pausa como modo de falha esperado no relatório.
- O README declara **R$ 0/mês de infraestrutura** durante o piloto, mais o domínio (~US$ 5/ano,
único custo, e opcional — o Cloudflare Pages dá `*.pages.dev` de graça).

## Gatilho para reavaliar

**Uso real, conforme o §15 do INTENT** — anúncios criados e contatos gerados consultáveis na view
do T-1.8. Antes disso, o plano gratuito continua. O gatilho não é calendário nem discomfort;
é métrica.

Upgrade para Pro quando: houver mais que um punhado de usuários reais, **ou** uma migration em
produção ficar pendente de rollback, **ou** a pausa atrapalhar o piloto.

## Nota sobre o limite que estourará primeiro

Não é a pausa — é o **egress de 5 GB**. As fotos das listagens são públicas e contam como
egress. A ~300 KB por foto (o que o T-5.1 já produz), 5 GB dão cerca de 16 mil visualizações.
Para um piloto de bairro isso basta, mas é o número que quebra primeiro se der certo, e o projeto
morre de sucesso. O T-5.1 comprimir as imagens é, por isso, requisito de custo de infra, não só
de UX.

## Relacionado

[ARCHITECTURE.md](../ARCHITECTURE.md) e a decisão D3 do plano de fases. A decisão D2
(pgTAP no CI) cobre a área que a pausa e a ausência de backup não cobrem: a suíte roda em
stack local descartável, em todo PR, e é ela que impede uma migration com erro de chegar a
produção.
