---
status: draft
spec: identity-order-history
created_at: 2026-09-01
updated_at: 2026-09-03
---

# Histórico de compras — Implementation Spec

**Resumo:** rotas de leitura `/me/orders` e `/me/orders/{id}` escopadas ao
comprador autenticado, agregando `Order` (payment) + `Ticket` (booking) + dados
da sessão (catalog).
**RF:** RF06 · **RN:** — · **Módulo backend:** `payment` (dono do `Order`) ·
**Feature frontend:** `account` · **Contrato:** `specs/identity-order-history/integration.md`

**Depende de:** `payment-pix-checkout` e `booking-ticket-issuance` mergeados. É
a última fatia de `account` a entrar.

---

## A. Backend — responsável: Igor

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | application | `src/app/modules/payment/application/schemas/response.py` | `OrderSummaryResponse`, `OrderDetailResponse` (com `tickets`, `can_cancel`, `refund_label`), `PagedOrders` |
| 2 | application | `.../application/usecases/order_history_usecase.py` | `list_for_buyer(buyer_id, page, size)`, `get_for_buyer(order_id, buyer_id)` — 404 se `buyer_id` não bate. Monta `can_cancel` (status `PAID` e sessão futura), `refund_label` (via `refund_policy.compute`) |
| 3 | infrastructure | `.../infrastructure/repositories/order_repository.py` | `list_for_buyer(buyer_id, limit, offset) -> (rows, total)` ordenado por `created_at desc` |
| 4 | fronteira | `booking/dependencies.py` | `get_tickets_for_order(session, order_id) -> list[TicketRef]` (já previsto em booking-ticket-issuance como leitura) |
| 5 | fronteira | `catalog/dependencies.py` | `get_session_summaries(session, session_ids) -> dict[UUID, SessionSummaryRef]` (batch, para não fazer N+1 na listagem) — coordenar com `catalog-session-detail` |
| 6 | api | `.../api/routers/order_history_router.py` | `GET /me/orders`, `GET /me/orders/{id}` (`Depends(get_current_buyer)`) |
| 7 | api | `.../modules/payment/router.py` | agrega |
| 8 | — | — | sem migration nova; garantir índice `orders(buyer_id, created_at)` |

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-order-history
commit 1  feat(payment): schemas e usecase de histórico de compras (escopo por comprador)
commit 2  feat(payment): listagem paginada de Order + batch de sessões (evita N+1)
commit 3  feat(payment): rotas GET /me/orders e /me/orders/{id}
/team-ludens:tbd-pr
```

---

## B. Frontend — responsável: Diego · feature `account`

Stack: **Next.js (App Router) + TypeScript estrito** — ver skill `frontend-architecture`.
Arquivos `.ts`/`.tsx`; rotas em `src/app/**/page.tsx` (Server Components; segmento
dinâmico `[orderId]`); componentes com estado, handler ou hook de React levam
`'use client'`; camadas na ordem `endpoints → schemas → server/types →
server/services → hooks/queries → components/ui → components → rota`; barrel
`index.ts` em toda subpasta. Tipos por `z.infer`. Aliases: `@account/*`, `@web/*`.

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.ts` | `orders.mine.list`, `orders.mine.byId(id)` |
| 2 | schemas | `src/features/account/schemas/order.schema.ts` | Zod: `orderSummarySchema`, `orderDetailSchema` (tickets, canCancel, refundLabel opcional), `pagedOrdersSchema` |
| 3 | server/types | `src/features/account/server/types/index.ts` | `z.infer` dos schemas — nunca `interface` manual |
| 4 | server/services | `src/features/account/server/services/order.service.ts` | `fetchMyOrders(page)`, `fetchMyOrder(id)` — request + `schema.parse` |
| 5 | queries | `src/features/account/hooks/queries/query-options.ts` + `useOrderQueries.ts` | `list(page)`, `detail(id)` |
| 6 | components | `src/features/account/components/OrderHistory.tsx` | `'use client'`; lista paginada; loading/empty ("Você ainda não fez nenhuma compra")/error |
| 7 | components | `src/features/account/components/OrderDetail.tsx` | `'use client'`; pedido + `TicketList` (de booking-ticket-issuance) + botões "Cancelar" (abre `CancelOrderDialog` de RF07 se `canCancel`) e "Reenviar por e-mail" (`useOrderMutations.resendTicket`) |
| 8 | components/ui | `src/features/account/components/ui/OrderStatusBadge.tsx` | mapeia status → rótulo pt-BR e cor |
| 9 | rotas | `src/app/minhas-compras/page.tsx` e `src/app/minhas-compras/[orderId]/page.tsx` | Server Components; cada um renderiza a tela client dentro de `<RequireAuth>` |
| 10 | barrels | `index.ts` | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-account-order-history
commit 1  feat(account): endpoints, schemas, server/types e services de histórico
commit 2  feat(account): queries de pedidos e badge de status
commit 3  feat(account): lista e detalhe de compras com ações de cancelar/reenviar
commit 4  chore(account): barrels index.ts
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. DevOps — Gabriel

Nada.

## D. Ordem entre as fatias

**Última fatia** — depende de `payment-pix-checkout` e `booking-ticket-issuance`
mergeados, e usa os exports de `catalog-session-detail`. Backend e
frontend contra o alvo.

## E. Bloqueios em aberto

- **[decidir com senior-dev]** módulo dono das rotas `/me/orders` (`payment` ou
  `identity`). Não bloqueia o frontend.
- **[coordenar]** `catalog` expõe `get_session_summaries` em batch — sub-issue de
  `catalog-session-detail`.
