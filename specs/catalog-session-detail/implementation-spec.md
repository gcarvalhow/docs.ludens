---
status: draft
spec: catalog-session-detail
created_at: 2026-09-01
---

# Detalhe da sessão — Implementation Spec

**Resumo:** rota pública de detalhe da sessão com `availableCount` calculado, e
os exports de `catalog/dependencies.py` que o `booking` usa para RN05.
**RF:** RF02 · **RN:** RN05 (leitura) · **Módulo backend:** `catalog` ·
**Feature frontend:** `catalog` ·
**Contrato:** `specs/catalog-session-detail/integration.md`

**Depende de:** `catalog-admin-management` mergeado (aggregates `Show`/`Session`).
**É pré-requisito de:** `booking-reservation` (por causa dos exports).

---

## A. Backend — responsável: Igor

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | application | `src/app/modules/catalog/application/schemas/response.py` | `SessionDetailResponse`, `SessionSummaryResponse`, `ShowDetailResponse` |
| 2 | application | `.../application/usecases/session_query_usecase.py` | `get_session_detail(session_id)` — monta `availableCount` e `status` |
| 3 | infrastructure | `.../infrastructure/repositories/session_repository.py` | `find_by_id_for_update` (herdado); `count_confirmed_tickets(session_id)` (JOIN com `tickets` de `booking` — **exceção controlada:** leitura por SQL de tabela de outro módulo é permitida aqui só para contagem, sem importar o aggregate; alternativa: `booking` expõe a contagem — decidir com o senior-dev) |
| 4 | api | `.../api/routers/session_router.py` | `GET /sessions/{id}`, `GET /shows/{id}` (sem auth) |
| 5 | api | `.../modules/catalog/router.py` | incluir `session_router` |
| 6 | **dependencies** | `.../modules/catalog/dependencies.py` | **exportar** `SessionRef` (frozen: id, capacity, starts_at, is_on_sale), `lock_session_for_update(session, session_id)`, `count_confirmed_tickets_for_session(session, session_id)` |
| 7 | — | — | garantir na migration de `catalog-admin-management` os índices `sessions(starts_at, is_on_sale)` e (na de `booking`) `reservations(session_id, status, expires_at)` |

### Código a colar — `catalog/dependencies.py`

```python
@dataclass(frozen=True)
class SessionRef:
    id: UUID
    capacity: int
    starts_at: datetime
    is_on_sale: bool


async def lock_session_for_update(session: AsyncSession, session_id: UUID) -> SessionRef | None:
    s = await SessionRepository(session).find_by_id_for_update(session_id)
    if s is None:
        return None
    return SessionRef(id=s.id, capacity=s.capacity, starts_at=s.starts_at, is_on_sale=s.is_on_sale)


async def count_confirmed_tickets_for_session(session: AsyncSession, session_id: UUID) -> int:
    result = await session.execute(
        select(func.count()).select_from(Ticket).where(
            Ticket.session_id == session_id, Ticket.is_active.is_(True)
        )
    )
    return result.scalar_one()
```

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-session-detail
commit 1  feat(catalog): schemas e usecase de detalhe da sessão
commit 2  feat(catalog): contagem de disponibilidade + rotas GET de sessão/espetáculo
commit 3  feat(catalog): exportar SessionRef, lock e contagem em dependencies
/team-ludens:tbd-pr
```

---

## B. Frontend — responsável: Diego · feature `catalog`

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.js` | `catalog.shows.byId(id)`, `catalog.sessions.byId(id)` |
| 2 | schemas | `src/features/catalog/schemas/session.schema.js` | Zod: `sessionDetailSchema` (status enum, ticketTypes), `showDetailSchema` |
| 3 | services | `src/features/catalog/services/session.service.js` | `fetchSessionById`, `fetchShowById` |
| 4 | queries | `.../hooks/queries/query-options.js` | `sessionDetail(id)` com `refetchInterval: 15000` e `staleTime: 5000` |
| 5 | components | `.../components/SessionDetail.jsx` | conecta a query; loading/error/empty; passa dados ao view |
| 6 | components/ui | `.../components/ui/SessionDetailView.jsx` | apresentacional: cabeçalho, preços, `availableCount`, rótulo de status; slot para o `TicketPicker` de `booking` |
| 7 | rotas | `src/App.jsx` | `/espetaculos/:showId`, `/sessoes/:sessionId` |
| 8 | barrels | `index.js` | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-session-detail
commit 1  feat(catalog): endpoints, schemas e services de detalhe da sessão
commit 2  feat(catalog): query com polling de disponibilidade
commit 3  feat(catalog): tela e view de detalhe da sessão
commit 4  chore(catalog): barrels
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_available_count_desconta_confirmados_e_reservas` | capacidade 10, 3 confirmados, 2 reservas abertas → `availableCount == 5` | RF02 / RN05 |
| `test_reserva_vencida_nao_desconta` | reserva aberta vencida não entra na conta | RN03 |
| `test_status_sold_out` | `availableCount == 0` → `status == "sold_out"` | RF02 |
| `test_status_closed_apos_horario` | `starts_at` no passado → `status == "closed"` | RF02 |
| `test_preco_meia_e_metade` | `half.price == full.price * 0.5` | RN04 |

Roteiro manual: abrir o detalhe, reservar de outra aba, ver `availableCount` cair
no *polling*; deixar a reserva vencer e ver voltar; medir tempo de resposta de
`GET /sessions/{id}` (meta ≤ 1 s p95).

```text
git checkout master && git pull && git checkout -b test/<NN>-catalog-session-detail
git commit -m "test(catalog): cobrir cálculo de disponibilidade e status da sessão"
```

## D. DevOps — Gabriel

Nada.

## E. Ordem entre as fatias

Merge de `catalog-admin-management` primeiro. Esta feature **antes** de
`booking-reservation` (exports). Backend + QA juntos; frontend contra o alvo.

## F. Bloqueios em aberto

- **[decidir com senior-dev]** contagem de `tickets` (tabela de `booking`) lida
  de dentro de `catalog`: por SQL direto sem importar o aggregate, ou `booking`
  expõe a contagem em `dependencies.py`. Não bloqueia o restante da fatia.
