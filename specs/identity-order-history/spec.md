---
status: approved
domain: identity
created_at: 2026-09-01
approved_at: 2026-09-01
---

# Histórico de compras

## 1. Visão da feature

Em "Minhas compras", a pessoa vê todos os pedidos que já fez, do mais recente ao
mais antigo, cada um com o espetáculo, a sessão, a quantidade, o valor e o
status (confirmado, cancelado, reembolsado). Abrindo um pedido, ela vê os
ingressos com código/QR e pode reenviar o e-mail de confirmação. É também de onde
ela solicita o cancelamento e reembolso de um pedido (RF07).

## 2. Problema que resolve

Depois de comprar, a pessoa não tem onde acompanhar o que comprou, checar o
status de um reembolso, ou recuperar um ingresso que não achou no e-mail. Hoje a
"consulta" é perguntar ao teatro.

## 3. Para quem é

- **Beneficiário direto:** o `Comprador` cadastrado.
- **Beneficiário indireto:** o teatro, que recebe menos perguntas de "cadê meu
  ingresso?" e "meu reembolso saiu?".

## 4. Como melhora a experiência atual

**Antes:** sem visão das próprias compras; recuperar ingresso = falar com o
teatro.

**Depois:** lista completa, status claro, ingressos sempre à mão, reenvio e
cancelamento autônomos.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** `identity-auth` (comprador autenticado);
`payment-pix-checkout` (os pedidos); `booking-ticket-issuance` (os ingressos).

**O que habilita:** é a superfície de `payment-cancellation-refund` (RF07) e do
reenvio de e-mail (RF05/`notification`).

**Posição:** core, N1.

**RF/RN cobertos:** RF06.

## 6. O que não é (escopo negativo)

- **Não inclui** filtro/busca avançada no histórico — lista simples ordenada por
  data, com paginação básica.
- **Não inclui** exportar o histórico (CSV/PDF).
- **Não inclui** ver compras de outra pessoa — nem para o admin, por esta tela
  (o admin usa a gestão de sessões, RF08).
- **Não inclui** a lógica de reembolso em si (feature separada) — aqui só o ponto
  de entrada.
- **Não inclui** re-compra rápida ("comprar de novo").

## 7. Custos adicionais

Nenhum custo externo identificado. É leitura sobre dados que já existem.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Conteúdo da lista | Um item por pedido: espetáculo, sessão (data/hora), quantidade, tipo, valor, status. |
| Status exibidos | Confirmado · Cancelado · Reembolsado · Reembolso em processamento · Pagamento pendente. |
| Detalhe do pedido | Ingressos com código/QR e `status` de cada um; botões "Reenviar por e-mail" e "Cancelar" (quando aplicável). |
| Ordenação | Por data da compra, mais recente primeiro. |
| Escopo de acesso | Só os pedidos do comprador autenticado — nenhuma rota devolve pedido de terceiro. |
| Pedidos pendentes/falhos | Aparecem (com status), mas sem ingressos. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
