---
status: draft
spec: booking-ticket-issuance
created_at: 2026-09-01
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

> **Stack:** Next.js (App Router) + TypeScript. Convenções completas na skill `frontend-architecture` do `team.ludens`: `services/` fica sob `server/`, tipos em `server/types/` (`z.infer`), rotas em `src/app/<rota>/page.tsx` (Server Component), `'use client'` só onde há hook/estado/handler, barrels `index.ts`. Os caminhos abaixo são o mapa da feature — ajuste a extensão/pasta ao padrão da skill.

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | schemas | `src/features/checkout/schemas/ticket.schema.ts` | Zod: `ticketSchema` ({ id, type, code, qrData, status, session }) — reusado por `account` |
| 2 | services | (usa `fetchOrder` de checkout / `fetchOrders` de account) | tickets vêm embutidos no pedido pago |
| 3 | components | `src/features/checkout/components/ConfirmationView.tsx` | tela pós-pagamento: dados da sessão + lista de ingressos |
| 4 | components/ui | `src/features/checkout/components/ui/TicketCard.tsx` | apresentacional: código + QR (lib de QR client-side a partir de `qrData`) + tipo; badge "cancelado" se `status === 'invalid'` |
| 5 | components | `src/features/account/components/TicketList.tsx` | mesma `TicketCard`, dentro de "Minhas compras" |
| 6 | rotas | `src/app/` (App Router: uma `page.tsx` por rota) | `/pedido/[orderId]` (confirmação) |
| 7 | barrels | `index.ts` | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-ticket-view
commit 1  feat(checkout): schema de ingresso e card com QR
commit 2  feat(checkout): tela de confirmação com lista de ingressos
commit 3  feat(account): reusar card de ingresso em Minhas compras
commit 4  chore: barrels
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_emite_um_ticket_por_unidade` | reserva confirmada de 3 → 3 `Ticket` `VALID`, eventos `TicketIssued` | RF05 |
| `test_meia_entrada_nao_exige_documento` | `issue` com `type=HALF` → ticket criado; **não existe atributo de documento** | RN04 |
| `test_codigo_unico_e_opaco` | 1000 emissões → nenhum código repetido; código não é sequencial | RF05 |
| `test_preco_congelado_no_ticket` | admin muda o preço da sessão depois → `ticket.unit_price` não muda | RF05 |
| `test_invalidar_por_reembolso` | `invalidate_for_order` → tickets `INVALID`, evento; `count_valid_for_session` cai | RF07 |
| `test_emissao_falha_faz_rollback` | forçar erro na emissão dentro da transação → pedido não fica `PAID`, nenhum ticket persiste | RF05 |

Roteiro manual: comprar 2 (uma inteira, uma meia) → confirmação mostra 2 cards
com QR → e-mail chega com os mesmos códigos → derrubar o serviço de e-mail e
comprar de novo: a compra vale, ingressos em "Minhas compras", botão reenviar.
Reembolsar → cards ficam "cancelado", QR não vale.

```text
git checkout master && git pull && git checkout -b test/<NN>-booking-ticket-issuance
git commit -m "test(booking): cobrir emissão, RN04, unicidade de código e invalidação"
```

## D. DevOps — Gabriel

Nada. (Geração de QR é client-side; código é gerado no backend sem dependência
externa.)

## E. Ordem entre as fatias

Sai **junto ou logo após** `booking-reservation` e **antes** do merge de
`payment-pix-checkout` (que depende dos exports). QA de domínio com o backend;
frontend contra o alvo.

## F. Bloqueios em aberto

- **[decidir com identity-order-history]** onde mora a rota `resend-ticket`.
