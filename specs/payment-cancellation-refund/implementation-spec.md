---
status: draft
spec: payment-cancellation-refund
created_at: 2026-09-01
---

# Cancelamento e reembolso — Implementation Spec

**Resumo:** cálculo da política RN02 no domínio de `Order`, bloqueio de < 24h,
invalidação de ingressos + devolução de disponibilidade na mesma transação, e
estorno no AbacatePay via handler de outbox idempotente.
**RF:** RF07 · **RN:** RN02 · **Módulo backend:** `payment` · **Feature
frontend:** `account` · **Contrato:** `specs/payment-cancellation-refund/integration.md`

**Depende de:** `payment-pix-checkout`, `booking-ticket-issuance` mergeados.
Consome `invalidate_tickets_for_order` de `booking/dependencies.py` e uma função
de "devolver disponibilidade" (é consequência da invalidação de tickets — os
tickets inválidos deixam de contar).

---

## A. Backend — responsável: Igor

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | domain | `.../payment/domain/aggregates/order.py` | adicionar `refund_amount(session_starts_at, now) -> Money` (RN02), `request_refund(amount)`, `mark_refunded()`. Eventos `OrderRefundRequested`, `OrderRefunded`. `request_refund` recusa (`DomainError`) se `status != PAID` |
| 2 | domain | `.../domain/value_objects/refund_policy.py` | função pura `compute(total, hours) -> (amount, label, allowed)` — `hours>=48 → (total,"full",True)`; `24<=hours<48 → (total*0.5,"half",True)`; `hours<24 → (0,"none",False)` |
| 3 | domain | `.../domain/events/payment_events.py` | `OrderRefundRequested` (`order_id`, `gateway_charge_id`, `amount`), `OrderRefunded` |
| 4 | application | `.../application/usecases/refund_usecase.py` | `preview(order_id, buyer)` (só cálculo); `cancel(order_id, buyer)` — valida sessão futura + `PAID`; calcula política; se `allowed=False` → `DomainError` (409); senão numa transação: `order.request_refund(amount)`, `invalidate_tickets_for_order(session, order.id)` (booking dep) |
| 5 | infrastructure | `.../infrastructure/services/payment_gateway.py` | adicionar `refund(charge_id, amount) -> RefundResult`; timeout, exceção própria |
| 6 | outbox | `.../modules/payment/handlers.py` | `@register("OrderRefundRequested")` → `PaymentGateway.refund`; on success emitir/persistir "estorno enviado"; dedupe por `order_id`. `@register("SessionCancelled")` (em `catalog` ou `payment`) → para cada pedido `PAID` da sessão, `order.request_refund(order.amount)` (100%) + invalidar tickets |
| 7 | api | `.../api/routers/refund_router.py` | `GET /orders/{id}/refund-preview`, `POST /orders/{id}/cancel` (ambas `get_current_buyer` + dono) |
| 8 | api | `.../modules/payment/router.py` | agrega |
| 9 | — | — | sem migration nova (usa colunas de `orders`); confirmar índice `orders(session_id, status)` |

### Código a colar — política RN02 (domínio puro)

```python
# src/app/modules/payment/domain/value_objects/refund_policy.py
def compute(total_cents: int, hours_to_session: float) -> tuple[int, str, bool]:
    if hours_to_session >= 48:
        return total_cents, "full", True
    if hours_to_session >= 24:
        return round(total_cents * 0.5), "half", True
    return 0, "none", False
```

```python
# Order.refund_amount usa isso; o usecase decide o bloqueio:
amount, label, allowed = compute(order.amount, hours_to_session(session_starts_at))
if not allowed:
    raise DomainError("refund window closed")   # -> 409, mensagem da RN02
```

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-payment-refund
commit 1  feat(payment): política de reembolso RN02 no domínio de Order + eventos
commit 2  feat(payment): usecase de preview/cancelamento (bloqueio <24h, invalida tickets)
commit 3  feat(payment): PaymentGateway.refund + handler de outbox idempotente
commit 4  feat(payment): rotas de preview e cancelamento; handler de SessionCancelled
/team-ludens:tbd-pr
```

---

## B. Frontend — responsável: Diego · feature `account`

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.js` | `orders.refundPreview(id)`, `orders.cancel(id)` |
| 2 | schemas | `src/features/account/schemas/refund.schema.js` | Zod: `refundPreviewSchema` ({ refundAmount, total, policyLabel enum, allowed, hoursToSession }) |
| 3 | services | `src/features/account/services/refund.service.js` | `fetchRefundPreview`, `cancelOrder` |
| 4 | queries | `.../hooks/queries/query-options.js` | `refundPreview(id)` (`enabled` quando o dialog abre) |
| 5 | mutations | `.../hooks/mutations/useRefundMutations.js` | `cancelOrder` — sucesso invalida `orderList` + toast "reembolso em processamento"; erro 409 → toast com a mensagem da RN02 |
| 6 | components | `.../components/CancelOrderDialog.jsx` | abre com o preview; mostra `policyLabel` legível; botão desabilitado se `!allowed` |
| 7 | components/ui | `.../components/ui/RefundBadge.jsx` | rótulo "Reembolso integral / de 50% / não permitido" |
| 8 | barrels | `index.js` | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-account-refund
commit 1  feat(account): endpoints, schema e service de reembolso
commit 2  feat(account): query de preview e mutation de cancelamento
commit 3  feat(account): dialog de cancelamento com a política RN02 legível
commit 4  chore(account): barrels
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_reembolso_integral_48h_ou_mais` | sessão daqui a 72h → `("full", total, allowed)` | RN02 |
| `test_reembolso_metade_entre_24_e_48` | daqui a 36h → `("half", total/2, allowed)` | RN02 |
| `test_bloqueio_menos_de_24h` | daqui a 10h → `allowed=False`; `cancel` levanta `DomainError` (409) | RN02 |
| `test_borda_48h_exatas` | exatamente 48h → `full` | RN02 |
| `test_cancel_invalida_tickets_e_devolve_disponibilidade` | após `cancel`, tickets `INVALID`, `count_valid_for_session` cai | RF07 |
| `test_refund_requested_idempotente` | handler processado 2x → um único estorno | RF07 |
| `test_sessao_cancelada_reembolsa_100` | `SessionCancelled` com pedido a 10h da sessão → reembolso de 100% mesmo assim | RF07 / RF08 |
| `test_pedido_de_sessao_passada_nao_cancela` | sessão já começou → 409 | RF07 |

Roteiro manual: comprar, pedir cancelamento com 3 dias (integral), com 30h
(50%), com 10h (bloqueado, mensagem). Admin cancela sessão de amanhã → todos
reembolsados 100%, ingressos "cancelados", lugares voltam.

```text
git checkout master && git pull && git checkout -b test/<NN>-payment-refund
git commit -m "test(payment): cobrir RN02, bloqueio <24h, invalidação e reembolso por sessão"
```

## D. DevOps — Gabriel

Nada novo (usa as credenciais do AbacatePay já configuradas em
`payment-pix-checkout`).

## E. Ordem entre as fatias

Última fatia de `payment`. Depende de `payment-pix-checkout` e
`booking-ticket-issuance` mergeados. Backend + QA juntos; frontend contra o alvo.

## F. Bloqueios em aberto

- **[confirmar na doc]** payload de estorno/consulta do AbacatePay.
