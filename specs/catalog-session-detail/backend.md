---
status: done
spec: catalog-session-detail
surface: backend
created_at: 2026-09-10
updated_at: 2026-09-21
---

# Detalhe da sessão — Backend

**Resumo:** rotas públicas de leitura `GET /shows/{show_id}` (espetáculo +
próximas sessões) e `GET /sessions/{session_id}` (detalhe com
`available_count` e `status`), mais os exports de `catalog/dependencies.py`
que o módulo `booking` vai consumir (RN05): `lock_session_for_update` e
`count_confirmed_tickets_for_session`. Sem aggregate novo, sem migration nova.
**RF:** RF02 · **RN:** RN05 (leitura) · **Módulo backend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-session-detail/integration.md`
**Carregar antes:** skill `backend-architecture`, `docs.ludens/backend/overview.md`,
`docs.ludens/backend/conventions.md`.

**Depende de:** `catalog-admin-management` mergeado (`Show`/`Session`,
`ShowRepository`/`SessionRepository`/`SeatCountsRepository`) e, de
preferência, `catalog-show-search` mergeado antes (edita o mesmo
`response.py`/`router.py`; não é bloqueio técnico, só evita conflito de
merge).
**É pré-requisito de:** `booking-reservation` — os dois exports de
`dependencies.py`.

> **Nota de consistência (resolvida em 2026-09-10):** `find_by_id_for_update`
> em `SessionRepository` é criado por `catalog-admin-management/backend.md`
> (corrigida — ela é quem primeiro precisa da trava, em
> `session_usecase.py::_lock`). Este documento só **reusa** — não
> redefinir. Ver também a nota geral de convenções (snake_case, `DomainError`
> sem `status_code`) em `catalog-show-search/backend.md` §2 — vale igual aqui.
>
> **Auditado novamente em 2026-09-11** contra o código real de `identity` e
> `catalog-admin-management` em `master`/PR #17 — sem `CamelModel`, sem VO
> inventado, sem método de repositório inventado, sem `PATCH`. Corrigida uma
> referência residual: a tabela de regras de negócio ainda citava
> `Session.half_price` (nome de antes da remoção do VO `Money`) enquanto o
> código já usava `half_price_cents` corretamente — ver linha "RF02 — meia =
> 50% da inteira" abaixo.
>
> **Revisão de 2026-09-21 (código real, já mergeado):** o `SessionQueryUseCase`
> descrito abaixo nunca chegou a existir como classe própria. O read-path
> público acabou implementado direto em `ShowUseCase.get_show_detail`/`search`
> e `SessionUseCase.get_session_detail` (as mesmas classes que
> `catalog-admin-management` já cria), cada método `is_admin`-aware —
> resposta de admin ou pública no mesmo método, não um usecase separado.
> `show_router.py`/`session_router.py` também não ganharam rota nova "à
> parte": o `GET /{show_id}`/`GET /{session_id}` foram acrescentados aos
> mesmos arquivos de `catalog-admin-management`, sob `/catalog/shows` e
> `/catalog/sessions` (sem prefixo solto `/shows`/`/sessions`). O código
> completo real desses dois usecases e routers já está reproduzido em
> `catalog-admin-management/backend.md` §2 — não duplicado aqui. `SeatCounts`
> continua sem repositório real (ver `catalog-admin-management/backend.md`
> §7): `available_count`/`status` desta fatia também usam `SeatCounts(0, 0)`
> fixo, não uma contagem de fato. `catalog/dependencies.py` (§2, export pra
> `booking`) **ainda não existe** — `booking-reservation` não mergeou, então
> essa parte do documento continua sendo planejamento, não código real; o
> resto desta revisão descreve o que já está implementado.

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | application | `src/app/modules/catalog/application/schemas/response.py` | editar — acrescentar `SessionSummaryResponse`, `ShowDetailResponse`, `TicketTypeResponse`, `SessionShowRef`, `SessionDetailResponse` |
| 2 | application | `application/usecases/show_usecase.py` / `session_usecase.py` | editar — **não** `session_query_usecase.py` (não existe); `get_show_detail`/`get_session_detail` entram nas classes de `catalog-admin-management` |
| 3 | infrastructure | `src/app/modules/catalog/infrastructure/repositories/session_repository.py` | — (`find_by_id_for_update` já existe, criado por `catalog-admin-management`) |
| 4 | api | `src/app/modules/catalog/api/routers/show_router.py` | editar — acrescentar `GET /catalog/shows/{show_id}` |
| 5 | api | `src/app/modules/catalog/api/routers/session_router.py` | editar — acrescentar `GET /catalog/sessions/{session_id}` (arquivo já existe, não é novo) |
| 6 | api | `src/app/modules/catalog/router.py` | — (`session_router` já incluído por `catalog-admin-management`) |
| 7 | api | `src/app/modules/catalog/dependencies.py` | **ainda não criado** — só quando `booking-reservation` começar |

Sem migration nova. Sem variável de ambiente nova.

---

## 2. Código

### `src/app/modules/catalog/application/schemas/response.py`

Acrescentar ao arquivo que `catalog-admin-management` cria e
`catalog-show-search` já editou.

`SessionSummaryResponse` real carrega mais que `{id, starts_at, venue}`: cada
sessão na lista do espetáculo já vem com `capacity`/`available_count`/`status`
próprios — a vitrine de detalhe do espetáculo não precisa de uma segunda
chamada pra saber se uma sessão específica esgotou. `ShowDetailResponse`
ganhou `genre_id` ao lado de `genre` (de `catalog-genre`, resolvido pelo
usecase).

```python
from typing import Literal

class SessionSummaryResponse(BaseModel):
    id: UUID
    starts_at: datetime
    venue: str
    capacity: int
    available_count: int
    status: Literal["on_sale", "sold_out", "closed", "cancelled"]

class ShowDetailResponse(BaseModel):
    id: UUID
    title: str
    synopsis: str
    image_url: str
    genre_id: UUID
    genre: str
    sessions: list[SessionSummaryResponse]

class TicketTypeResponse(BaseModel):
    type: Literal["full", "half"]
    price: float

class SessionShowRef(BaseModel):
    id: UUID
    title: str

class SessionDetailResponse(BaseModel):
    id: UUID
    show: SessionShowRef
    starts_at: datetime
    venue: str
    capacity: int
    available_count: int
    status: Literal["on_sale", "sold_out", "closed", "cancelled"]
    ticket_types: list[TicketTypeResponse]
```

(`BaseModel`, `UUID`, `datetime` já importados no topo do arquivo real pelas
classes anteriores — só `Literal` costuma faltar se `catalog-show-search` não
tiver usado; conferir antes de duplicar import.)

### `src/app/modules/catalog/infrastructure/repositories/session_repository.py`

Nenhuma edição aqui — `find_by_id_for_update` já é criado por
`catalog-admin-management/backend.md` (que também depende dele em
`session_usecase.py::_lock`). Este documento só reusa via
`catalog/dependencies.py`, abaixo.

### Não existe `session_query_usecase.py` no código real

Não há uma classe `SessionQueryUseCase` separada. `get_show_detail` (com
`is_admin`) vive em `ShowUseCase`, e `get_session_detail` (com `is_admin`)
vive em `SessionUseCase` — as mesmas duas classes que
`catalog-admin-management` já cria, no mesmo arquivo. A disponibilidade
(`available_count`) e o status derivado (`status_at`) viraram métodos do
próprio aggregate `Session` (`domain/aggregates/session.py`), não uma função
solta `_public_status` no usecase:

```python
# domain/aggregates/session.py — já reproduzido em catalog-admin-management/backend.md §2
def available_count(self, counts: SeatCounts) -> int:
    return max(0, self.capacity - counts.tickets_sold - counts.reserved_open)

def status_at(self, now: datetime, counts: SeatCounts) -> str:
    if self.status is SessionStatus.CANCELLED:
        return "cancelled"
    if self.starts_at <= now:
        return "closed"
    if self.available_count(counts) <= 0:
        return "sold_out"
    return "on_sale"
```

O código completo real de `ShowUseCase.get_show_detail` e
`SessionUseCase.get_session_detail` (incluindo a resposta pública vs. admin)
já está em `catalog-admin-management/backend.md` §2 — não duplicado aqui.
Diferença de comportamento real que vale registrar: a checagem "espetáculo
despublicado não é navegável" (`NotFoundError`) é igual à descrita abaixo,
mas `get_show_detail` real filtra sessões futuras (`s.starts_at > now`) sem
exigir `status is ON_SALE` — uma sessão `on_sale` mas já esgotada (`sold_out`)
ainda aparece na lista, só marcada como esgotada.

### `src/app/modules/catalog/api/routers/show_router.py` e `session_router.py`

`GET /catalog/shows/{show_id}` e `GET /catalog/sessions/{session_id}` reais
são as rotas `get` que já aparecem em `catalog-admin-management/backend.md`
§2 — no mesmo arquivo, mesmo `APIRouter`, sem `require_admin` (autenticação
opcional via `get_current_user_optional`, resposta muda por `is_admin`).
Não existe rota solta `/shows/{id}`/`/sessions/{id}` fora do prefixo
`/catalog/shows`/`/catalog/sessions`, e `session_router.py` não é um arquivo
"novo": já existe desde `catalog-admin-management` (dono do write-path).

### `src/app/modules/catalog/router.py`

Sem `admin_catalog_router` (não existe — ver
`catalog-admin-management/backend.md`). Já reproduzido lá:

```python
from fastapi import APIRouter

from app.modules.catalog.api.routers.genre_router import router as genre_router
from app.modules.catalog.api.routers.session_router import router as session_router
from app.modules.catalog.api.routers.show_router import router as show_router

router = APIRouter()
router.include_router(genre_router)
router.include_router(show_router)
router.include_router(session_router)
```

### `src/app/modules/catalog/dependencies.py`

**Ainda não existe no código real** — `booking-reservation` não mergeou, e é
o consumidor deste arquivo. O que segue é planejamento (contrato interno
proposto), não descrição de código já implementado; criar quando
`booking-reservation` começar. `SessionRef` nunca deve expor o aggregate
`Session` — só os campos que `booking` precisa.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from uuid import UUID

from sqlalchemy import text
from sqlalchemy.exc import OperationalError, ProgrammingError
from sqlalchemy.ext.asyncio import AsyncSession

from app.modules.catalog.infrastructure.repositories import SessionRepository

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
    now = datetime.now(timezone.utc)
    return SessionRef(id=s.id, capacity=s.capacity, starts_at=s.starts_at, is_on_sale=s.is_on_sale(now))

_CONFIRMED_SQL = text(
    "SELECT COUNT(*) FROM tickets WHERE session_id = :session_id "
    "AND is_active = true AND status = 'valid'"
)

async def count_confirmed_tickets_for_session(session: AsyncSession, session_id: UUID) -> int:
    # `tickets` é de `booking` — mesmo fallback de SeatCountsRepository.
    try:
        async with session.begin_nested():
            result = await session.execute(_CONFIRMED_SQL, {"session_id": str(session_id)})
    except (ProgrammingError, OperationalError):
        return 0
    return int(result.scalar_one())
```

---

## 3. Onde cada regra de negócio entra

| Regra | Arquivo · função | Como |
| --- | --- | --- |
| RN05 (leitura) — disponível = capacidade − confirmados − reservas abertas | `domain/aggregates/session.py` · `Session.available_count` | `max(0, capacity - counts.tickets_sold - counts.reserved_open)` — hoje sempre `SeatCounts(0, 0)`, ver nota de revisão acima |
| RF02 — esgotado quando disponível ≤ 0 | `domain/aggregates/session.py` · `Session.status_at` | `available_count(counts) <= 0` → `"sold_out"`, antes de checar `on_sale` |
| RF02 — sessão encerrada/cancelada bloqueia | `domain/aggregates/session.py` · `Session.status_at` | `status is CANCELLED` → `"cancelled"`; `starts_at <= now` → `"closed"` (checados antes de `sold_out`) |
| RF02 — meia = 50% da inteira | `domain/aggregates/session.py` · `Session.half_price_cents` (já existe) | `ticket_types` usa `session.full_price_cents`/`session.half_price_cents` direto, nunca recalcula |
| RN05 — trava de linha para `booking` | `session_repository.py` · `find_by_id_for_update` + `dependencies.py` · `lock_session_for_update` | `SELECT ... FOR UPDATE`; exportado, nunca chamado pelo próprio `catalog` fora de `catalog-admin-management` |
| logic.md §5 — espetáculo despublicado não é navegável | `session_query_usecase.py` · `get_show_detail`/`get_session_detail` | `show.status is not PUBLISHED` → `NotFoundError` (decisão de implementação, mesma inferência de `catalog-show-search`) |

---

## 4. DevOps

Não se aplica.

---

## 5. Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-session-detail

# commit 1 — leitura: schemas e usecase
git add src/app/modules/catalog/application/schemas/response.py \
        src/app/modules/catalog/application/usecases/session_query_usecase.py
git commit -m "feat(catalog): schemas e usecase de detalhe da sessao"

# commit 2 — repositório
git add src/app/modules/catalog/infrastructure/repositories/session_repository.py
git commit -m "feat(catalog): find_by_id_for_update em SessionRepository"

# commit 3 — rotas
git add src/app/modules/catalog/api/routers/show_router.py \
        src/app/modules/catalog/api/routers/session_router.py \
        src/app/modules/catalog/router.py
git commit -m "feat(catalog): expor GET /shows/{id} e GET /sessions/{id}"

# commit 4 — export para booking
git add src/app/modules/catalog/dependencies.py
git commit -m "feat(catalog): exportar SessionRef, lock e contagem confirmada para booking"
```

Depois: `/team-ludens:tbd-pr`.

---

## 6. Ordem entre as superfícies

Depende de `catalog-admin-management` mergeado; idealmente depois de
`catalog-show-search` (mesmos arquivos `response.py`/`router.py` editados —
evita conflito, não é bloqueio real). É pré-requisito de
`booking-reservation` por causa de `dependencies.py`.

---

## 7. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 fechada). Pontos de atenção:

- **`count_confirmed_tickets_for_session` lê `tickets` (tabela de
  `booking`) por SQL direto.** Mesma ressalva já registrada no
  `implementation-spec.md` legado: alternativa seria `booking` expor a
  contagem em vez de `catalog` ler a tabela alheia — decisão de arquitetura
  que não bloqueia esta fatia (o fallback a 0 cobre o meio-tempo).
