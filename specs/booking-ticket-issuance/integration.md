---
status: alvo
spec: booking-ticket-issuance
updated_at: 2026-09-01
responsavel: Igor (Backend)
---

# Integration Contract — Emissão do ingresso

**Status:** alvo. **Módulo backend:** `booking`. Sem rota HTTP própria de
emissão (a emissão acontece dentro do webhook de `payment`). O comprador vê os
ingressos por `GET /orders/{id}` (payment) e por `identity-order-history` (RF06).

## Shape do Ticket

```json
{ "id": "uuid", "type": "full" | "half", "code": "string opaca",
  "qrData": "string", "status": "valid" | "invalid",
  "session": { "id": "uuid", "showTitle": "string", "startsAt": "ISO", "venue": "string" } }
```

`qrData` pode ser igual a `code`; o frontend renderiza como QR.

## Contrato interno (booking/dependencies.py)

- `confirm_reservation(session, reservation_id) -> None` — chama
  `reservation.confirm()`; recusa (`DomainError`) se a reserva não está "open".
- `issue_tickets_for_reservation(session, reservation_id) -> list[TicketRef]` —
  emite um `Ticket` por unidade; `TicketRef` = frozen `{ id, code, type }`.
- `invalidate_tickets_for_order(session, order_id) -> int` — marca inválidos e
  devolve a contagem (usado no reembolso, RF07).

Todas devem ser chamadas **dentro da transação** do usecase que as invoca.

## Regras aplicadas no servidor

Um ticket por unidade; `code` único (constraint) e opaco; `type` registrado sem
documento (RN04); preço unitário congelado no ticket; emissão na mesma transação
do pagamento aprovado.

## Reenvio de e-mail

`POST /orders/{id}/resend-ticket` (payment ou identity, a definir na fatia de
RF06) → emite evento `TicketEmailResendRequested` → handler de `notification`.

## Lacunas / decisões em aberto

- Em qual módulo mora a rota `resend-ticket` — decidir junto com
  `identity-order-history`.
- Formato do `qrData` (código puro vs. payload assinado para a validação N2) —
  N1 usa o código puro; assinatura fica para o N2.
