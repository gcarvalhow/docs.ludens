---
status: approved
domain: catalog
created_at: 2026-09-01
approved_at: 2026-09-01
---

# Busca e filtro de espetáculos

## 1. Visão da feature

A página inicial mostra os espetáculos em cartaz — título, imagem, sinopse curta,
as próximas datas e a faixa de preço. A pessoa filtra por data e por gênero para
chegar rápido no que interessa. Espetáculos que não têm nenhuma sessão futura não
aparecem: o que está na tela é o que dá para comprar.

## 2. Problema que resolve

Sem uma vitrine própria, a pessoa depende de redes sociais e do boca a boca para
saber o que está em cartaz, e frequentemente descobre um espetáculo quando as
sessões já passaram. O teatro não tem um canal onde o catálogo esteja sempre
atualizado e comprável.

## 3. Para quem é

- **Beneficiário direto:** o visitante (não precisa estar autenticado).
- **Beneficiário indireto:** o teatro, que ganha uma vitrine sempre atual.

Entra no topo da jornada: é a porta de entrada.

## 4. Como melhora a experiência atual

**Antes:** procurar em vários lugares, achar informação desatualizada, descobrir
sessão esgotada ou já realizada.

**Depois:** uma lista única, filtrável, só com o que tem sessão futura.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** os dados de espetáculo e sessão vêm de
`catalog-admin-management` (RF08). Sem admin cadastrando, a lista fica vazia.

**O que habilita:** `catalog-session-detail` (RF02) — clicar num card leva ao
detalhe da sessão.

**Posição:** core, N1.

**RF/RN cobertos:** RF01. Meta de desempenho: busca ≤ 2 s p95 (RNF02).

## 6. O que não é (escopo negativo)

- **Não inclui** busca textual livre (por palavra na sinopse) — só filtro por
  data e por gênero no N1.
- **Não inclui** recomendação, ordenação por popularidade ou "mais vendidos".
- **Não inclui** favoritar ou salvar espetáculos.
- **Não inclui** paginação infinita sofisticada — uma paginação simples basta
  para o volume de um teatro comunitário.

## 7. Custos adicionais

Nenhum custo externo identificado.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Critério de exibição | Só espetáculos com ≥ 1 sessão futura à venda. |
| Filtros | Data (a partir de) e gênero/categoria. Combináveis. |
| Faixa de preço no card | Menor e maior preço de ingresso entre as sessões futuras. |
| Ordenação padrão | Pela data da próxima sessão, ascendente. |
| Acesso | Público, sem autenticação. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
