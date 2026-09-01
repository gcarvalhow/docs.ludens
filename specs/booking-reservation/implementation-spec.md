---
status: draft
spec: booking-reservation
created_at: 2026-09-01
---

# Reserva temporária de ingressos — Implementation Spec

**Resumo:** módulo `booking` com o aggregate `Reservation`, criação sob trava de
linha da sessão (RN05), limite por CPF (RN01) e um relay de expiração de 15 min
(RN03) resistente a restart.
**RF:** RF03 · **RN:** RN01, RN03, RN05 · **Módulo backend:** `booking` ·
**Feature frontend:** `booking` + `checkout` ·
**Contrato:** `specs/booking-reservation/integration.md`

Carregar antes: skill `backend-architecture` (com atenção a `references/04` e
`references/07`), skill `frontend-architecture`.

**Depende de:** `identity-auth` (mergeado), `catalog-session-detail` (a `Session`
e seu `is_on_sale`).

---

## A. Backend — responsável: Igor

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | domain | `src/app/modules/booking/domain/enumerations/reservation_status.py` | `ReservationStatus(str, Enum)`: `OPEN`, `CONFIRMED`, `EXPIRED`, `CANCELLED` |
| 2 | domain | `src/app/modules/booking/domain/enumerations/ticket_type.py` | `TicketType(str, Enum)`: `FULL`, `HALF` |
| 3 | domain | `.../domain/aggregates/reservation.py` | aggregate `Reservation` — campos `buyer_id`, `buyer_cpf`, `session_id`, `quantity`, `ticket_type`, `status`, `expires_at`. Métodos: `open(...)`, `confirm()`, `expire()`, `cancel()`. Cada um valida o estado atual e levanta evento |
| 4 | domain | `.../domain/events/booking_events.py` | `ReservationOpened`, `ReservationConfirmed`, `ReservationExpired`, `ReservationCancelled` |
| 5 | application | `.../application/schemas/request.py` | `OpenReservationRequest` (`session_id: UUID`, `quantity: Field(ge=1, le=6)`, `ticket_type: TicketType`) |
| 6 | application | `.../application/schemas/response.py` | `ReservationResponse` (+ `seconds_left` derivado) |
| 7 | application | `.../application/usecases/reservation_usecase.py` | `open`, `get`, `cancel`. **`open` é o método crítico — ver código abaixo.** `get` calcula `seconds_left` |
| 8 | application | `.../application/usecases/reservation_expiry.py` | `expire_due_reservations(session)` — busca reservas `OPEN` com `expires_at < now`, chama `reservation.expire()`, salva. Idempotente |
| 9 | infrastructure | `.../infrastructure/repositories/reservation_repository.py` | `ReservationRepository(AggregateRepository[Reservation])` + `count_open_quantity_for_session`, `count_open_quantity_for_buyer_in_session`, `find_open_due(limit)` |
| 10 | outbox/relay | `src/app/outbox/relay.py` (ou `asyncio.Task` irmão em `main.py` lifespan) | agenda `expire_due_reservations` a cada `OUTBOX_RELAY_INTERVAL_SECONDS` (ou intervalo próprio) numa `AsyncSession` própria |
| 11 | api | `.../api/routers/reservation_router.py` | `POST /reservations` (`get_current_buyer`), `GET /reservations/{id}` (dono), `POST /reservations/{id}/cancel` (dono) |
| 12 | api | `.../modules/booking/router.py` | agrega |
| 13 | api | `.../modules/booking/dependencies.py` | exporta `get_open_reservation(session, reservation_id, buyer_id)` para `payment` consumir na criação da cobrança |
| 14 | fronteira | `src/app/modules/catalog/dependencies.py` | precisa de `lock_session_for_update(session, session_id) -> SessionRef` e `count_confirmed_tickets_for_session(session, session_id) -> int` — **coordenar com a feature `catalog-session-detail`** (adiciona esses exports) |
| 15 | migration | `migrations/versions/xxxx_booking_reservation.py` | tabela `reservations`; índice em `(session_id, status)` e `(buyer_cpf, session_id, status)` |
| 16 | config | `src/app/config.py` | adicionar `reservation_ttl_minutes: int = 15`, `max_tickets_per_cpf: int = 6` |

### Código a colar — `ReservationUseCase.open` (o coração da RN05)

```python
async def open(self, request: OpenReservationRequest, buyer: Buyer) -> ReservationResponse:
    # 1. TRAVA a linha da sessão — nada de leitura simples
    session_ref = await lock_session_for_update(self._session, request.session_id)
    if session_ref is None or not session_ref.is_on_sale:
        raise DomainError("session not on sale")  # -> 409

    # 2. RN01 — limite por CPF (confirmados + reservas abertas do CPF nessa sessão)
    confirmed_for_cpf = await self._reservation_repo.count_confirmed_quantity_for_buyer_in_session(
        buyer.cpf, session_ref.id)
    open_for_cpf = await self._reservation_repo.count_open_quantity_for_buyer_in_session(
        buyer.cpf, session_ref.id)
    if confirmed_for_cpf + open_for_cpf + request.quantity > settings.max_tickets_per_cpf:
        raise DomainError("ticket limit per CPF exceeded")  # -> 409

    # 3. RN05 — disponibilidade, com a linha da sessão travada
    confirmed = await count_confirmed_tickets_for_session(self._session, session_ref.id)
    held = await self._reservation_repo.count_open_quantity_for_session(session_ref.id)  # só não vencidas
    if confirmed + held + request.quantity > session_ref.capacity:
        raise DomainError("not enough seats available")  # -> 409

    reservation = Reservation.open(
        buyer_id=buyer.id, buyer_cpf=buyer.cpf, session_id=session_ref.id,
        quantity=request.quantity, ticket_type=request.ticket_type,
        ttl_minutes=settings.reservation_ttl_minutes,
    )
    await self._reservation_repo.save(reservation)   # grava Reservation + ReservationOpened na mesma tx
    return self._to_response(reservation)
```

> `count_open_quantity_for_session` **filtra `expires_at > now()`** no SQL — uma
> reserva aberta vencida não segura disponibilidade, mesmo antes do relay a
> transicionar para `EXPIRED`.

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-booking-reservation
commit 1  feat(booking): modelar Reservation, estados e eventos
commit 2  feat(booking): usecase de reserva (open sob trava, RN01/RN05) + schemas
commit 3  feat(booking): repositório com contagens + relay de expiração (RN03)
commit 4  feat(booking): rotas de reserva, dependencies e migration
/team-ludens:tbd-pr
```

---

## B. Frontend — responsável: Diego · features `booking` + `checkout`

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.js` | grupo `booking`: `reservations.create`, `reservations.byId(id)`, `reservations.cancel(id)` |
| 2 | schemas | `src/features/booking/schemas/reservation.schema.js` | Zod: `openReservationSchema` (quantity 1..6, ticketType enum), `reservationSchema` (com `expiresAt` coerce.date, `secondsLeft`, `status` enum) |
| 3 | services | `src/features/booking/services/reservation.service.js` | `openReservation`, `fetchReservation`, `cancelReservation` |
| 4 | queries | `src/features/booking/hooks/queries/query-options.js` + `useReservationQueries.js` | `detail(id)` com `refetchInterval` curto enquanto `status === 'open'` |
| 5 | mutations | `src/features/booking/hooks/mutations/useReservationMutations.js` | `open` (sucesso → navega a `/checkout/:id` + toast; erro 409 → toast por caso e volta à sessão), `cancel` (invalida a sessão + navega) |
| 6 | hooks | `src/features/checkout/hooks/useReservationCountdown.js` | deriva `secondsLeft` de `expiresAt`; ao zerar, dispara refetch da reserva |
| 7 | components | `src/features/booking/components/TicketPicker.jsx` | quantidade + tipo, na página da sessão; chama `open` |
| 8 | components | `src/features/checkout/components/CheckoutFrame.jsx` | mostra contador (`useReservationCountdown`), estado da reserva, botão cancelar; ao `status` virar `expired`/`cancelled`, mostra o estado final e CTA de reservar de novo |
| 9 | components/ui | `src/features/checkout/components/ui/Countdown.jsx` | apresentacional puro (recebe `secondsLeft`) |
| 10 | rotas | `src/App.jsx` | `/checkout/:reservationId` protegida por `RequireAuth` |
| 11 | barrels | `index.js` em toda subpasta + raiz das features | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-booking-reservation
commit 1  feat(booking): endpoints, schemas e services de reserva
commit 2  feat(booking): queries, mutations e hook de contador de reserva
commit 3  feat(booking): seletor de ingresso e frame de checkout com contador
commit 4  chore(booking): barrels index.js
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

### Casos de teste de domínio (pytest)

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_abre_reserva_dentro_da_capacidade` | capacidade 10, 3 confirmados, pede 4 → reserva `OPEN`, evento `ReservationOpened` | RF03 |
| `test_recusa_quando_excede_capacidade` | capacidade 10, 7 confirmados + 2 abertos, pede 2 → `DomainError` | RN05 |
| `test_recusa_limite_cpf` | CPF já com 5 (confirmados+abertos) na sessão, pede 2 → `DomainError` | RN01 |
| `test_reserva_vencida_nao_conta_disponibilidade` | reserva aberta com `expires_at` no passado não entra na contagem `count_open_quantity_for_session` | RN03 |
| `test_expire_transiciona_e_emite_evento` | `expire_due_reservations` sobre reserva vencida → `status=EXPIRED`, evento `ReservationExpired` | RN03 |
| `test_confirmar_reserva_nao_aberta_recusa` | `confirm()` numa reserva já `EXPIRED` → `DomainError` | RF03 |
| `test_cancelar_por_dono` | `cancel()` numa reserva `OPEN` → `CANCELLED`, evento | RF03 |

### Teste de concorrência (usecase, Postgres real)

`test_duas_reservas_concorrentes_na_ultima_poltrona`: capacidade 1, 0
confirmados; duas corrotinas chamam `open` simultaneamente → exatamente uma
retorna 201, a outra `DomainError` (409). Roda ≥ 20 vezes.

### Roteiro manual

Reservar 2 na sessão com 2 lugares restantes em duas abas ao mesmo tempo → só
uma sucede. Deixar a reserva vencer → contador zera → mensagem de expiração →
lugares voltam. Cancelar manualmente → lugares voltam na hora. Matar a API com
reserva aberta e subir de novo → reserva vence normalmente.

### Passo a passo TBD (QA)

```text
git checkout master && git pull && git checkout -b test/<NN>-booking-reservation
git commit -m "test(booking): cobrir RN01, RN03, RN05 e concorrência de reserva"
```

---

## D. DevOps — Gabriel

- `.env.example`: adicionar `RESERVATION_TTL_MINUTES=15`, `MAX_TICKETS_PER_CPF=6`
  (CONFIG). Nenhum segredo novo.

## E. Ordem entre as fatias

Backend depende do merge de `identity-auth` e de `catalog-session-detail` (exports
de trava/contagem da `Session`). QA de domínio pode escrever os casos junto com
o backend. Frontend contra o contrato-alvo; a integração real após o merge do
backend.

## F. Bloqueios em aberto

- **[coordenar, não bloqueio]** `catalog-session-detail` precisa expor
  `lock_session_for_update` e `count_confirmed_tickets_for_session` em
  `catalog/dependencies.py`. Fatiar essa dependência como sub-issue de
  `catalog-session-detail`, não de `booking-reservation`.
