---
status: draft
spec: catalog-session-detail
surface: backend
created_at: 2026-09-10
updated_at: 2026-09-11
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

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | application | `src/app/modules/catalog/application/schemas/response.py` | editar — acrescentar `SessionSummaryResponse`, `ShowDetailResponse`, `TicketTypeResponse`, `SessionShowRef`, `SessionDetailResponse` |
| 2 | application | `src/app/modules/catalog/application/usecases/session_query_usecase.py` | novo |
| 3 | infrastructure | `src/app/modules/catalog/infrastructure/repositories/session_repository.py` | — (`find_by_id_for_update` já existe, criado por `catalog-admin-management`) |
| 4 | api | `src/app/modules/catalog/api/routers/show_router.py` | editar — acrescentar `GET /shows/{show_id}` |
| 5 | api | `src/app/modules/catalog/api/routers/session_router.py` | novo — `GET /sessions/{session_id}` |
| 6 | api | `src/app/modules/catalog/router.py` | editar — incluir `session_router` |
| 7 | api | `src/app/modules/catalog/dependencies.py` | novo — exports para `booking` |

Sem migration nova. Sem variável de ambiente nova.

---

## 2. Código

### `src/app/modules/catalog/application/schemas/response.py`

Acrescentar ao arquivo que `catalog-admin-management` cria e
`catalog-show-search` já editou.

```python
from typing import Literal

class SessionSummaryResponse(BaseModel):
    id: UUID
    starts_at: datetime
    venue: str

class ShowDetailResponse(BaseModel):
    id: UUID
    title: str
    synopsis: str
    image_url: str
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

### `src/app/modules/catalog/application/usecases/session_query_usecase.py`

Reusa `SeatCountsRepository` de `catalog-admin-management` (já resolve
`tickets_sold` + `reserved_open`, com fallback a 0 enquanto `booking` não
existe) em vez de recalcular disponibilidade com uma query própria.

```python
from datetime import datetime, timezone
from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain import NotFoundError
from app.modules.catalog.application.schemas.response import (
    SessionDetailResponse,
    SessionShowRef,
    SessionSummaryResponse,
    ShowDetailResponse,
    TicketTypeResponse,
)
from app.modules.catalog.domain.aggregates.session import Session
from app.modules.catalog.domain.enumerations.session_status import SessionStatus
from app.modules.catalog.domain.enumerations.show_status import ShowStatus
from app.modules.catalog.infrastructure.repositories import (
    SeatCountsRepository,
    SessionRepository,
    ShowRepository,
)

def _public_status(session: Session, available: int, now: datetime) -> str:
    if session.status is SessionStatus.CANCELLED:
        return "cancelled"
    if session.starts_at <= now:
        return "closed"
    if available <= 0:
        return "sold_out"
    return "on_sale"

class SessionQueryUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._show_repository = ShowRepository(session)
        self._session_repository = SessionRepository(session)
        self._seat_counts_repository = SeatCountsRepository(session)

    async def get_show_detail(self, show_id: UUID) -> ShowDetailResponse:
        show = await self._show_repository.find_by("id", show_id)
        # Espetáculo despublicado não é navegável por link direto.
        if show is None or show.status is not ShowStatus.PUBLISHED:
            raise NotFoundError("Espetáculo não encontrado.")

        now = datetime.now(timezone.utc)
        sessions = await self._session_repository.find_all_for_shows([show.id])
        upcoming = [
            s for s in sessions if s.status is SessionStatus.ON_SALE and s.starts_at > now
        ]
        return ShowDetailResponse(
            id=show.id,
            title=show.title,
            synopsis=show.synopsis,
            image_url=show.image_url,
            genre=show.genre,
            sessions=[
                SessionSummaryResponse(id=s.id, starts_at=s.starts_at, venue=s.venue)
                for s in upcoming
            ],
        )

    async def get_session_detail(self, session_id: UUID) -> SessionDetailResponse:
        session = await self._session_repository.find_by("id", session_id)
        if session is None:
            raise NotFoundError("Sessão não encontrada.")

        show = await self._show_repository.find_by("id", session.show_id)
        if show is None or show.status is not ShowStatus.PUBLISHED:
            raise NotFoundError("Sessão não encontrada.")

        counts = (await self._seat_counts_repository.for_sessions([session.id]))[session.id]
        available = max(session.capacity - counts.tickets_sold - counts.reserved_open, 0)
        now = datetime.now(timezone.utc)

        return SessionDetailResponse(
            id=session.id,
            show=SessionShowRef(id=show.id, title=show.title),
            starts_at=session.starts_at,
            venue=session.venue,
            capacity=session.capacity,
            available_count=available,
            status=_public_status(session, available, now),
            ticket_types=[
                TicketTypeResponse(type="full", price=session.full_price_cents / 100),
                TicketTypeResponse(type="half", price=session.half_price_cents / 100),
            ],
        )
```

### `src/app/modules/catalog/api/routers/show_router.py`

Editar o arquivo que `catalog-show-search` cria: acrescentar
`GET /shows/{show_id}`.

```python
from uuid import UUID

from app.modules.catalog.application.schemas.response import ShowDetailResponse
from app.modules.catalog.application.usecases.session_query_usecase import SessionQueryUseCase

@router.get("/shows/{show_id}", response_model=ShowDetailResponse)
async def get_show(show_id: UUID, session: AsyncSession = Depends(get_db)) -> ShowDetailResponse:
    return await SessionQueryUseCase(session).get_show_detail(show_id)
```

(`UUID`, `AsyncSession`, `Depends`, `get_db` já importados no topo do arquivo
real por `list_shows`/`list_genres`; não duplicar import.)

### `src/app/modules/catalog/api/routers/session_router.py`

```python
from uuid import UUID

from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.modules.catalog.application.schemas.response import SessionDetailResponse
from app.modules.catalog.application.usecases.session_query_usecase import SessionQueryUseCase

router = APIRouter(tags=["Catalog"])

@router.get("/sessions/{session_id}", response_model=SessionDetailResponse)
async def get_session(
    session_id: UUID, session: AsyncSession = Depends(get_db)
) -> SessionDetailResponse:
    return await SessionQueryUseCase(session).get_session_detail(session_id)
```

### `src/app/modules/catalog/router.py`

```python
from fastapi import APIRouter

from app.modules.catalog.api.routers.admin_catalog_router import router as admin_catalog_router
from app.modules.catalog.api.routers.session_router import router as session_router
from app.modules.catalog.api.routers.show_router import router as show_router

router = APIRouter()
router.include_router(admin_catalog_router)
router.include_router(show_router)
router.include_router(session_router)
```

### `src/app/modules/catalog/dependencies.py`

Contrato interno para `booking-reservation`. `SessionRef` nunca expõe o
aggregate `Session` — só os campos que `booking` precisa.

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
| RN05 (leitura) — disponível = capacidade − confirmados − reservas abertas | `session_query_usecase.py` · `get_session_detail` | `available = capacity - counts.tickets_sold - counts.reserved_open`, via `SeatCountsRepository` (reuso, não recálculo) |
| RF02 — esgotado quando disponível ≤ 0 | `session_query_usecase.py` · `_public_status` | `available <= 0` → `"sold_out"`, antes de checar `on_sale` |
| RF02 — sessão encerrada/cancelada bloqueia | `session_query_usecase.py` · `_public_status` | `status is CANCELLED` → `"cancelled"`; `starts_at <= now` → `"closed"` (checados antes de `sold_out`) |
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

pytest -q
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
