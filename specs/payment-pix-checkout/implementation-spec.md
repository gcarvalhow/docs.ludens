---
status: draft
spec: payment-pix-checkout
created_at: 2026-09-01
---

# Pagamento Pix e criação do pedido — Implementation Spec

**Resumo:** módulo `payment` — aggregate `Order`, cobrança Pix síncrona no
AbacatePay (exceção do outbox), e webhook idempotente que confirma o pedido,
confirma a reserva e emite os ingressos numa única transação.
**RF:** RF04 · **RN:** — · **Módulo backend:** `payment` · **Feature frontend:**
`checkout` · **Contrato:** `specs/payment-pix-checkout/integration.md`

**Depende de:** `booking-reservation` mergeado (consome `get_open_reservation` de
`booking/dependencies.py`), `catalog-session-detail` (preços via
`get_ticket_prices`).

Carregar antes: skill `backend-architecture` (`references/05` e `references/07`).

---

## A. Backend — responsável: Igor

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | domain | `.../payment/domain/enumerations/order_status.py` | `OrderStatus`: `PENDING`, `PAID`, `FAILED`, `REFUNDED` |
| 2 | domain | `.../domain/aggregates/order.py` | aggregate `Order` — `buyer_id`, `reservation_id`, `session_id`, `quantity`, `ticket_type`, `amount`, `status`, `gateway_charge_id`. Métodos `create`, `attach_charge(charge_id)`, `mark_paid()`, `mark_failed()`, `mark_refunded()`. Eventos `OrderCreated`, `OrderPaid`, `OrderFailed`, `OrderRefunded` |
| 3 | domain | `.../domain/events/payment_events.py` | os eventos acima |
| 4 | application | `.../application/schemas/request.py` | `CheckoutRequest` (`reservation_id: UUID`); webhook parseado num schema tolerante |
| 5 | application | `.../application/schemas/response.py` | `CheckoutResponse` (`order_id`, `status`, `amount`, `pix_qr_code`, `pix_copy_paste`, `expires_at`), `OrderResponse` |
| 6 | application | `.../application/usecases/checkout_usecase.py` | `start(reservation_id, buyer)` — ver código abaixo |
| 7 | application | `.../application/usecases/payment_webhook_usecase.py` | `handle(payload)` — valida assinatura; dedupe por `gateway_charge_id`; se aprovado e pedido `PENDING`: numa transação → `order.mark_paid()`, `reservation.confirm()` (via dependency de `booking`), emitir tickets (via dependency de `booking`); se recusado: `order.mark_failed()` + `reservation` liberada. Idempotente |
| 8 | infrastructure | `.../infrastructure/services/payment_gateway.py` | `PaymentGateway` — `httpx.AsyncClient`, `create_pix_charge(order_id, amount)`, `verify_webhook_signature(headers, body)`. Exceção própria `PaymentGatewayError`, timeout explícito |
| 9 | infrastructure | `.../infrastructure/repositories/order_repository.py` | `OrderRepository(AggregateRepository[Order])` + `find_by_gateway_charge_id`, `find_by_reservation` |
| 10 | outbox | `.../modules/payment/handlers.py` | `@register("OrderFailed")` → garante `ReservationCancelled` (se ainda não); (o e-mail de confirmação é handler de `notification` em `OrderPaid`) |
| 11 | api | `.../api/routers/checkout_router.py`, `webhook_router.py` | `POST /checkout` (`get_current_buyer`), `GET /orders/{id}` (dono), `POST /payments/webhook` (sem auth de usuário; valida assinatura) |
| 12 | api | `.../modules/payment/router.py` | agrega |
| 13 | api | `.../modules/payment/dependencies.py` | exporta `get_order(session, order_id, buyer_id)` para `identity-order-history` e `payment-cancellation-refund` |
| 14 | fronteira | `booking/dependencies.py` | precisa expor `confirm_reservation(session, reservation_id)` e `issue_tickets_for_reservation(session, reservation_id)` — **coordenar com `booking-ticket-issuance`** |
| 15 | migration | `migrations/versions/xxxx_payment.py` | tabela `orders`; índice único em `gateway_charge_id`, índice em `reservation_id` |
| 16 | config | `src/app/config.py` | `abacatepay_api_key` (SECRET), `abacatepay_base_url` (CONFIG), `abacatepay_webhook_secret` (SECRET) |

### Código a colar — `CheckoutUseCase.start`

```python
async def start(self, reservation_id: UUID, buyer: Buyer) -> CheckoutResponse:
    reservation = await get_open_reservation(self._session, reservation_id, buyer.id)  # booking dep
    if reservation is None:
        raise DomainError("reservation not active")  # -> 409

    prices = await get_ticket_prices(self._session, reservation.session_id)  # catalog dep
    amount = prices.unit_price(reservation.ticket_type) * reservation.quantity

    order = Order.create(
        buyer_id=buyer.id, reservation_id=reservation.id, session_id=reservation.session_id,
        quantity=reservation.quantity, ticket_type=reservation.ticket_type, amount=amount,
    )
    await self._order_repo.save(order)          # Order + OrderCreated na mesma tx

    # EXCEÇÃO documentada do outbox: cobrança síncrona, DEPOIS do save
    try:
        charge = await self._gateway.create_pix_charge(order.id, amount)
    except PaymentGatewayError:
        return CheckoutResponse(order_id=order.id, status=order.status, amount=amount,
                                pix_qr_code=None, pix_copy_paste=None, expires_at=None)  # -> frontend mostra "tente de novo"

    order.attach_charge(charge.id)
    await self._order_repo.save(order)
    return CheckoutResponse(order_id=order.id, status=order.status, amount=amount,
                            pix_qr_code=charge.qr_code, pix_copy_paste=charge.copy_paste,
                            expires_at=charge.expires_at)
```

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-payment-pix-checkout
commit 1  feat(payment): modelar Order, estados e eventos
commit 2  feat(payment): usecase de checkout (cobrança síncrona) + schemas
commit 3  feat(payment): PaymentGateway (httpx) + repositório de Order
commit 4  feat(payment): webhook idempotente (pago -> confirma reserva + emite tickets)
commit 5  feat(payment): rotas, dependencies, migration e config do AbacatePay
/team-ludens:tbd-pr
```

---

## B. Frontend — responsável: Diego · feature `checkout`

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.js` | `checkout.start`, `orders.byId(id)` |
| 2 | schemas | `src/features/checkout/schemas/checkout.schema.js` | Zod: `checkoutResponseSchema`, `orderSchema` (status enum, tickets opcional) |
| 3 | services | `src/features/checkout/services/checkout.service.js` | `startCheckout(reservationId)`, `fetchOrder(id)` |
| 4 | queries | `.../hooks/queries/query-options.js` | `order(id)` com `refetchInterval` enquanto `status === 'pending'` |
| 5 | mutations | `.../hooks/mutations/useCheckoutMutations.js` | `start` — sucesso guarda a resposta (QR); erro 409 → volta à sessão; 502 → estado "tente de novo" |
| 6 | hooks | `.../hooks/usePaymentStatus.js` | observa a query de order; ao virar `paid` navega para a confirmação; `failed` para a sessão |
| 7 | components | `.../components/PaymentPanel.jsx` | integra `CheckoutFrame` (contador da reserva) + QR + estados |
| 8 | components/ui | `.../components/ui/{PixQr.jsx,AwaitingPayment.jsx,PaymentFailed.jsx}` | apresentacionais puros |
| 9 | rotas | `src/App.jsx` | `/checkout/:reservationId` já existe (booking); adiciona `/pedido/:orderId` de confirmação |
| 10 | barrels | `index.js` | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-checkout-pix
commit 1  feat(checkout): endpoints, schemas e services de checkout
commit 2  feat(checkout): query de pedido com polling + mutations
commit 3  feat(checkout): painel de pagamento (QR, aguardando, falha)
commit 4  chore(checkout): barrels
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_checkout_cria_order_pendente` | reserva aberta → `Order` `PENDING`, evento `OrderCreated`, total = qtd × preço | RF04 |
| `test_checkout_reserva_expirada_recusa` | reserva não "open" → `DomainError` (409) | RF04 |
| `test_webhook_aprovado_confirma_tudo` | webhook aprovado, order `PENDING` → order `PAID`, reserva `CONFIRMED`, N tickets emitidos, evento `OrderPaid` | RF04/RF05 |
| `test_webhook_duplicado_idempotente` | mesmo webhook 2x → nenhuma mudança na segunda; nenhum ticket extra | RF04 |
| `test_webhook_recusado_libera_reserva` | webhook recusa → order `FAILED`, reserva liberada | RF04 |
| `test_meia_entrada_valor` | tipo `half` → total = qtd × (inteira × 0.5) | RN04 |
| `test_nenhum_dado_de_pagador_persistido` | após webhook, `Order` não tem campo com dado do pagador; nada logado | RNF01 |

Roteiro manual (sandbox AbacatePay): reservar → checkout → pagar no sandbox →
ver "aguardando" → confirmação com ingressos. Repetir com pagamento recusado →
reserva volta. Derrubar o gateway na criação → "tente de novo". Reenviar o
webhook manualmente → sem efeito duplicado.

```text
git checkout master && git pull && git checkout -b test/<NN>-payment-pix-checkout
git commit -m "test(payment): cobrir checkout, webhook idempotente e liberação de reserva"
```

## D. DevOps — Gabriel

- `.env.example`: `ABACATEPAY_API_KEY` (SECRET), `ABACATEPAY_BASE_URL` (CONFIG),
  `ABACATEPAY_WEBHOOK_SECRET` (SECRET). Configurar sandbox no ambiente de dev e
  os secrets no CI. Registrar a URL do webhook no painel do AbacatePay.

## E. Ordem entre as fatias

Depende de `booking-reservation` e `catalog-session-detail` mergeados. A fatia
`booking/dependencies.py` (`confirm_reservation`, `issue_tickets_for_reservation`)
sai com `booking-ticket-issuance`. Backend + QA juntos; frontend contra o alvo.

## F. Bloqueios em aberto

- **[coordenar]** `booking` precisa expor `confirm_reservation` e
  `issue_tickets_for_reservation` — fatiar como sub-issue de
  `booking-ticket-issuance`.
- **[confirmar na doc]** formato do payload/assinatura do webhook do AbacatePay.
