---
status: alvo
spec: payment-cancellation-refund
updated_at: 2026-09-01
responsavel: Igor (Backend)
---

# Integration Contract — Cancelamento e reembolso

**Status:** alvo. **Módulo backend:** `payment`.

## Rotas

| Método | Caminho | Auth | Sucesso |
| --- | --- | --- | --- |
| GET | `/orders/{id}/refund-preview` | Bearer (dono) | 200 |
| POST | `/orders/{id}/cancel` | Bearer (dono) | 202 |

## Request / Response

- `GET /orders/{id}/refund-preview` → 200
  `{ refundAmount, total, policyLabel: "full" | "half" | "none", allowed: bool,
  hoursToSession }`.
- `POST /orders/{id}/cancel` → 202
  `{ orderId, status: "refund_processing", refundAmount }`.

## Erros esperados

| Status | Quando | Mensagem |
| --- | --- | --- |
| 409 | `hoursToSession < 24` | "Cancelamentos com menos de 24h da sessão não têm reembolso e não podem ser feitos pelo site." |
| 409 | sessão já começou | "A sessão já começou." |
| 409 | pedido não está "pago" | "Este pedido não pode ser cancelado." |
| 404 | pedido de outro comprador | "Pedido não encontrado." |

## Estados

`paid → refund_processing` (cancel) → `refunded` (estorno confirmado). Ingressos:
`valid → invalid` já no `cancel`.

## Eventos

`OrderRefundRequested` (outbox → `PaymentGateway.refund`) · `OrderRefunded`
(estorno confirmado → e-mail). `SessionCancelled` (de `catalog`) → um
`OrderRefundRequested` de 100% por pedido confirmado da sessão.

## Impacto de UX

Mostrar o preview antes de confirmar ("Reembolso integral" / "Reembolso de 50%").
`allowed=false` → botão desabilitado + explicação. Após confirmar, pedido em
"reembolso em processamento" (polling `GET /orders/{id}`), depois "reembolsado".

## Lacunas / decisões em aberto

- Payload do estorno/consulta do AbacatePay — confirmar na doc do gateway.
- Prefixo/base path, versionamento, envelope de erro — `<a definir globalmente>`.
