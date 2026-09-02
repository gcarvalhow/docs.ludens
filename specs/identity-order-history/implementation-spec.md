---
status: draft
spec: identity-order-history
created_at: 2026-09-01
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

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.js` | `orders.mine.list`, `orders.mine.byId(id)` |
| 2 | schemas | `src/features/account/schemas/order.schema.js` | Zod: `orderSummarySchema`, `orderDetailSchema` (tickets, canCancel, refundLabel opcional), `pagedOrdersSchema` |
| 3 | services | `src/features/account/services/order.service.js` | `fetchMyOrders(page)`, `fetchMyOrder(id)` |
| 4 | queries | `.../hooks/queries/query-options.js` + `useOrderQueries.js` | `list(page)`, `detail(id)` |
| 5 | components | `.../components/OrderHistory.jsx` | lista paginada; loading/empty ("Você ainda não fez nenhuma compra")/error |
| 6 | components | `.../components/OrderDetail.jsx` | pedido + `TicketList` (de booking-ticket-issuance) + botões "Cancelar" (abre `CancelOrderDialog` de RF07 se `canCancel`) e "Reenviar por e-mail" (`useOrderMutations.resendTicket`) |
| 7 | components/ui | `.../components/ui/OrderStatusBadge.jsx` | mapeia status → rótulo pt-BR e cor |
| 8 | rotas | `src/App.jsx` | `/minhas-compras` e `/minhas-compras/:orderId` protegidas por `RequireAuth` |
| 9 | barrels | `index.js` | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-account-order-history
commit 1  feat(account): endpoints, schemas e services de histórico
commit 2  feat(account): queries de pedidos e badge de status
commit 3  feat(account): lista e detalhe de compras com ações de cancelar/reenviar
commit 4  chore(account): barrels
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_lista_so_pedidos_do_comprador` | comprador A não vê pedido de B; `GET /me/orders/{id_de_B}` → 404 | RF06 / RNF01 |
| `test_ordenacao_mais_recente_primeiro` | 3 pedidos em datas diferentes → ordem desc | RF06 |
| `test_detalhe_traz_ingressos_so_se_pago` | pedido `pending` → sem tickets; `paid` → N tickets | RF06 |
| `test_can_cancel` | `paid` + sessão futura → `canCancel=True`; sessão passada → `False` | RF06 / RF07 |
| `test_status_labels` | cada `Order.status` mapeia para o rótulo certo | RF06 |
| `test_paginacao` | `size=10` com 25 pedidos → 3 páginas, `total=25` | RF06 |

Roteiro manual: fazer 3 compras (uma pendente, uma paga, uma reembolsada) → a
lista mostra as 3 com status certos; abrir a paga → ingressos + botões; abrir a
pendente → sem ingressos; tentar `GET /me/orders/{id}` de outro usuário → 404.

```text
git checkout master && git pull && git checkout -b test/<NN>-order-history
git commit -m "test(payment): cobrir escopo por comprador, ordenação, ingressos e canCancel"
```

## D. DevOps — Gabriel

Nada.

## E. Ordem entre as fatias

**Última fatia** — depende de `payment-pix-checkout` e `booking-ticket-issuance`
mergeados, e usa os exports de `catalog-session-detail`. Backend + QA juntos;
frontend contra o alvo.

## F. Bloqueios em aberto

- **[decidir com senior-dev]** módulo dono das rotas `/me/orders` (`payment` ou
  `identity`). Não bloqueia o frontend.
- **[coordenar]** `catalog` expõe `get_session_summaries` em batch — sub-issue de
  `catalog-session-detail`.
