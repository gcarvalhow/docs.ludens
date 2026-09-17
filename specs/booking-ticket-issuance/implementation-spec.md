---
status: draft
spec: booking-ticket-issuance
created_at: 2026-09-01
updated_at: 2026-09-03
---

# Emissão do ingresso — Implementation Spec

**Resumo:** aggregate `Ticket` no módulo `booking`, emitido dentro da transação
do pagamento aprovado; código único opaco + QR; `type` sem documento (RN04); e os
exports de `booking/dependencies.py` que `payment` e `catalog` consomem.
**RF:** RF05 · **RN:** RN04 · **Módulo backend:** `booking` · **Feature
frontend:** `checkout` + `account` · **Contrato:** `specs/booking-ticket-issuance/integration.md`

**Depende de:** `booking-reservation` mergeado. **Acoplada a:**
`payment-pix-checkout` (o webhook chama os exports daqui).

---

## A. Backend — responsável: Igor

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | domain | `.../booking/domain/enumerations/ticket_status.py` | `TicketStatus`: `VALID`, `INVALID` |
| 2 | domain | `.../domain/aggregates/ticket.py` | aggregate `Ticket` — `reservation_id`, `order_id`, `session_id`, `buyer_id`, `type` (`TicketType`), `code` (str única), `unit_price`, `status`. Métodos `issue(...)` (classmethod), `invalidate()`. Eventos `TicketIssued`, `TicketInvalidated`. **Sem campo de documento de estudante.** |
| 3 | domain | `.../domain/services/ticket_code.py` | `generate_ticket_code()` — string opaca não sequencial (ex.: `secrets.token_urlsafe(12)` normalizado) |
| 4 | domain | `.../domain/events/booking_events.py` | adicionar `TicketIssued`, `TicketInvalidated` |
| 5 | application | `.../application/usecases/ticket_issuance.py` | `issue_for_reservation(reservation_id)` — cria N tickets (um por `quantity`), com retry curto em colisão de `code`; `invalidate_for_order(order_id)` |
| 6 | infrastructure | `.../infrastructure/repositories/ticket_repository.py` | `TicketRepository(AggregateRepository[Ticket])` + `find_by_order`, `find_by_code`, `count_valid_for_session` |
| 7 | **dependencies** | `.../modules/booking/dependencies.py` | **exportar** `confirm_reservation`, `issue_tickets_for_reservation` (→ `list[TicketRef]`), `invalidate_tickets_for_order` — todas usam a `session` recebida, dentro da transação do chamador |
| 8 | outbox | `.../modules/booking/handlers.py` | `@register("SessionCancelled")` → `invalidate_for_order` para cada pedido confirmado da sessão (idempotente) |
| 9 | api | `.../api/routers/ticket_router.py` (opcional) | `GET /tickets/{code}` só para debug/admin no N1 — decidir; a validação de porta é N2 |
| 10 | migration | `migrations/versions/xxxx_booking_tickets.py` | tabela `tickets`; **constraint única em `code`**; índices `(order_id)`, `(session_id, status)`, `(buyer_id)` |

### Código a colar — `Ticket` (recorte, foco RN04)

```python
class Ticket(AggregateRoot, Model):
    __tablename__ = "tickets"

    reservation_id: Mapped[UUID] = mapped_column(nullable=False)
    order_id: Mapped[UUID] = mapped_column(nullable=False)
    session_id: Mapped[UUID] = mapped_column(nullable=False, index=True)
    buyer_id: Mapped[UUID] = mapped_column(nullable=False)
    type: Mapped[TicketType] = mapped_column(nullable=False)      # FULL | HALF
    code: Mapped[str] = mapped_column(String(32), nullable=False, unique=True)
    unit_price: Mapped[int] = mapped_column(nullable=False)       # centavos
    status: Mapped[TicketStatus] = mapped_column(default=TicketStatus.VALID, nullable=False)
    # NÃO existe campo student_document — RN04: meia não exige documento

    @classmethod
    def issue(cls, *, reservation_id, order_id, session_id, buyer_id, type_, unit_price, code) -> "Ticket":
        t = cls()
        t.raise_event(lambda v: TicketIssued(
            version=v, id=t.id, reservation_id=reservation_id, order_id=order_id,
            session_id=session_id, buyer_id=buyer_id, type=type_, unit_price=unit_price, code=code,
        ))
        return t

    def invalidate(self) -> None:
        if self.status is TicketStatus.INVALID:
            return
        self.raise_event(lambda v: TicketInvalidated(version=v, id=self.id))
```

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-booking-ticket-issuance
commit 1  feat(booking): modelar Ticket, código opaco e eventos (meia sem documento, RN04)
commit 2  feat(booking): usecase de emissão/invalidação de ingressos
commit 3  feat(booking): repositório de Ticket + exports em dependencies
commit 4  feat(booking): handler de SessionCancelled + migration (code único)
/team-ludens:tbd-pr
```

---

## B. Frontend — responsável: Diego · features `checkout` + `account`

Stack: **Next.js (App Router) + TypeScript estrito** — ver skill `frontend-architecture`.
Arquivos `.ts`/`.tsx`; rotas em `src/app/**/page.tsx` (Server Components; segmento
dinâmico `[id]`); componentes com estado, handler ou hook de React levam
`'use client'`; barrel `index.ts` em toda subpasta. Tipos por `z.infer` do schema.
Aliases: `@checkout/*`, `@account/*`, `@web/*`.

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | schemas | `src/features/checkout/schemas/ticket.schema.ts` | Zod: `ticketSchema` ({ id, type, code, qrData, status, session }) — reusado por `account` |
| 2 | server/types | `src/features/checkout/server/types/index.ts` | `Ticket = z.infer<typeof ticketSchema>` |
| 3 | server/services | (usa `fetchOrder` de checkout / `fetchMyOrders` de account) | tickets vêm embutidos no pedido pago |
| 4 | components | `src/features/checkout/components/ConfirmationView.tsx` | `'use client'`; tela pós-pagamento: dados da sessão + lista de ingressos |
| 5 | components/ui | `src/features/checkout/components/ui/TicketCard.tsx` | `'use client'` (gera o QR client-side a partir de `qrData`): código + QR + tipo; badge "cancelado" se `status === 'invalid'` |
| 6 | components | `src/features/account/components/TicketList.tsx` | mesma `TicketCard`, dentro de "Minhas compras" |
| 7 | rotas | `src/app/pedido/[orderId]/page.tsx` | Server Component; `await params`, renderiza `<ConfirmationView>` |
| 8 | barrels | `index.ts` | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-ticket-view
commit 1  feat(checkout): schema e tipo de ingresso e card com QR
commit 2  feat(checkout): tela de confirmação com lista de ingressos + rota
commit 3  feat(account): reusar card de ingresso em Minhas compras
commit 4  chore: barrels index.ts
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. DevOps — Gabriel

Nada. (Geração de QR é client-side; código é gerado no backend sem dependência
externa.)

## D. Ordem entre as fatias

Sai **junto ou logo após** `booking-reservation` e **antes** do merge de
`payment-pix-checkout` (que depende dos exports). Frontend contra o alvo.

## E. Bloqueios em aberto

- **[decidir com identity-order-history]** onde mora a rota `resend-ticket`.
