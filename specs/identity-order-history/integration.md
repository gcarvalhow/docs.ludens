---
status: alvo
spec: identity-order-history
updated_at: 2026-09-01
responsavel: Igor (Backend)
---

# Integration Contract — Histórico de compras

**Status:** alvo. **Módulo backend:** `identity` (leitura) consumindo dados de
`payment` e `booking` via dependências exportadas.

## Rotas

| Método | Caminho | Auth | Sucesso |
| --- | --- | --- | --- |
| GET | `/me/orders` | Bearer (comprador) | 200 |
| GET | `/me/orders/{id}` | Bearer (dono) | 200 |

(As ações "cancelar" e "reenviar" usam rotas de `payment` — ver
`payment-cancellation-refund` e `notification-transactional-email`.)

## Response

- `GET /me/orders?page=1&size=10` → `{ items: [OrderSummary], page, size, total }`.
- `OrderSummary`: `{ id, showTitle, sessionStartsAt, venue, quantity, ticketType,
  amount, status, createdAt }`.
- `GET /me/orders/{id}` → `OrderDetail` = `OrderSummary` +
  `{ tickets: [Ticket], canCancel: bool, refundLabel?: "full"|"half"|"none" }`.
- `status` ∈ `pending | paid | refund_processing | refunded | failed`. O
  frontend mapeia para os rótulos pt-BR.

## Erros

- 404 — pedido inexistente ou de outro comprador ("Pedido não encontrado").

## Regras aplicadas no servidor

Filtro por `buyer_id` autenticado em toda rota; `tickets` só quando `paid` ou
`refunded`; `canCancel = paid AND session futura`; ordenação `createdAt desc`.

## Impacto de UX

Estados loading / vazio ("Você ainda não fez nenhuma compra") / erro. Paginação
simples. Botões "Cancelar" (se `canCancel`) e "Reenviar por e-mail" (se `paid`).

## Lacunas / decisões em aberto

- Se `/me/orders` mora em `identity` (agrega) ou em `payment` (dono do `Order`) —
  recomendação: `payment` expõe `list_orders_for_buyer` em `dependencies.py`, e a
  rota fica em `payment` com prefixo `/me/orders` OU em `identity`. Decidir com o
  senior-dev; não bloqueia o frontend (o contrato é o mesmo).
- Prefixo/base path, versionamento — `<a definir globalmente>`.
