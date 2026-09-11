---
status: draft
spec: catalog-show-search
surface: backend
created_at: 2026-09-10
updated_at: 2026-09-10
---

# Busca e filtro de espetáculos — Backend

**Resumo:** rota pública de leitura (`GET /shows`, `GET /genres`) sobre os
aggregates `Show`/`Session` já modelados por `catalog-admin-management` — sem
aggregate novo, sem migration nova. Filtra por espetáculo publicado com ≥ 1
sessão futura à venda, agrega faixa de preço e próximas datas, pagina.
**RF:** RF01 · **RN:** — · **Módulo backend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-show-search/integration.md`
**Carregar antes:** skill `backend-architecture`, `docs.ludens/backend/overview.md`.

**Depende de:** `catalog-admin-management` mergeado — aggregates `Show`/
`Session`, `ShowRepository`/`SessionRepository`, a migration `sessions`/`shows`
e o índice `ix_sessions_starts_at_status`. Este documento só **acrescenta**
arquivos ao módulo `catalog` que ela cria — não redefine nada que já existe.
**É base de:** `catalog-session-detail` (mesmo módulo, rota irmã).

> **Nota de consistência (resolvida em 2026-09-10):** `catalog-admin-management/
> backend.md` chegou a assumir convenções que não batiam com o código real
> (camelCase via um `CamelModel` inexistente, `DomainError` com `status_code`,
> `find_by_id`/`find_by_id_for_update` como se já existissem no repositório
> base). Já foi corrigido — inclusive `SessionRepository.find_by_id_for_update`
> passou a ser criado lá (este documento só reusa). Este documento sempre usou
> as convenções reais (ver `core/` abaixo).

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | application | `src/app/modules/catalog/application/schemas/response.py` | editar — acrescentar `ShowCardResponse`, `PagedShowsResponse` ao arquivo que `catalog-admin-management` já cria |
| 2 | application | `src/app/modules/catalog/application/usecases/show_search_usecase.py` | novo |
| 3 | infrastructure | `src/app/modules/catalog/infrastructure/repositories/show_repository.py` | editar — acrescentar `search_with_upcoming` e `list_genres_in_catalog` a `ShowRepository` |
| 4 | api | `src/app/modules/catalog/api/routers/show_router.py` | novo |
| 5 | api | `src/app/modules/catalog/router.py` | editar — incluir `show_router` |

Não há migration nova: a query usa o índice `ix_sessions_starts_at_status`
(`sessions(starts_at, status)`) que `catalog-admin-management` já cria. Sem
variável de ambiente nova — a subseção DevOps não se aplica.

---

## 2. Código real que este documento assume (não recriar)

Conferido contra `api.ludens` (identity-auth mergeado) e contra
`catalog-admin-management/backend.md` (ainda não mergeado, mas é a base deste
módulo). Reproduzido aqui só como referência de import — **não colar de
novo**:

```python
# app/core/domain (já existe, real) — from app.core.domain import ...
class DomainError(Exception): ...          # sem status_code; message
class ConflictError(DomainError): ...       # -> 409 (mapeado em main.py)
class NotFoundError(DomainError): ...       # -> 404
class AggregateRoot: ...
class Model(DeclarativeBase): ...           # id, created_at, updated_at, is_active

# app.core.infrastructure.repositories.AggregateRepository — real, só tem:
#   find_by(field, value), find_all(order_by=...), find_all_by(...),
#   exists_by(field, value), save(entity)
#   NÃO tem find_by_id nem find_by_id_for_update.

# app.modules.catalog.domain.aggregates.show.Show          (catalog-admin-management)
# app.modules.catalog.domain.aggregates.session.Session    (catalog-admin-management)
# app.modules.catalog.domain.enumerations.show_status.ShowStatus       (DRAFT, PUBLISHED)
# app.modules.catalog.domain.enumerations.session_status.SessionStatus (ON_SALE, CANCELLED)
# app.modules.catalog.infrastructure.repositories.ShowRepository(AggregateRepository[Show])
# app.modules.catalog.infrastructure.repositories.SessionRepository(AggregateRepository[Session])
```

Schemas neste módulo são `pydantic.BaseModel` puro, campos **snake_case**
(mesma convenção do contrato de `identity-auth`: `UserResponse`,
`RegisterRequest`) — não existe `CamelModel`/alias camelCase no repo real.

---

## 3. Código

### `src/app/modules/catalog/application/schemas/response.py`

Acrescentar ao arquivo que `catalog-admin-management` já cria (que tem
`AdminSessionResponse`/`AdminShowResponse`). Não remover o que já existe.

```python
from datetime import datetime
from uuid import UUID

from pydantic import BaseModel

class ShowCardResponse(BaseModel):
    id: UUID
    title: str
    synopsis_short: str
    image_url: str
    genre: str
    upcoming_dates: list[datetime]
    price_min: float
    price_max: float

class PagedShowsResponse(BaseModel):
    items: list[ShowCardResponse]
    page: int
    size: int
    total: int
```

### `src/app/modules/catalog/infrastructure/repositories/show_repository.py`

Acrescentar os dois métodos de leitura à classe que `catalog-admin-management`
já cria (`class ShowRepository(AggregateRepository[Show]): model = Show`).
Arquivo completo já com os métodos novos:

```python
from datetime import date as Date, datetime, timezone
from typing import NamedTuple
from uuid import UUID

from sqlalchemy import func, select

from app.core.infrastructure.repositories import AggregateRepository
from app.modules.catalog.domain.aggregates.session import Session
from app.modules.catalog.domain.aggregates.show import Show
from app.modules.catalog.domain.enumerations.session_status import SessionStatus
from app.modules.catalog.domain.enumerations.show_status import ShowStatus

class ShowSearchRow(NamedTuple):
    id: UUID
    title: str
    synopsis: str
    image_url: str
    genre: str
    price_min_cents: int
    price_max_cents: int
    upcoming_dates: list[datetime]

class ShowRepository(AggregateRepository[Show]):
    model = Show

    # RF01 — só espetáculo publicado com >= 1 sessão futura à venda.
    def _visible_conditions(self, threshold: datetime, genre: str | None) -> list:
        conditions = [
            Show.is_active.is_(True),
            Show.status == ShowStatus.PUBLISHED,
            Session.is_active.is_(True),
            Session.status == SessionStatus.ON_SALE,
            Session.starts_at > threshold,
        ]
        if genre:
            conditions.append(Show.genre == genre)
        return conditions

    async def search_with_upcoming(
        self, *, from_date: Date, genre: str | None, page: int, size: int
    ) -> tuple[list[ShowSearchRow], int]:
        now = datetime.now(timezone.utc)
        # fromDate no passado vira "hoje" (logic.md §5).
        from_as_dt = datetime.combine(from_date, datetime.min.time(), tzinfo=timezone.utc)
        threshold = max(from_as_dt, now)
        conditions = self._visible_conditions(threshold, genre)

        count_stmt = (
            select(Show.id)
            .join(Session, Session.show_id == Show.id)
            .where(*conditions)
            .group_by(Show.id)
        )
        total = await self._session.scalar(select(func.count()).select_from(count_stmt.subquery()))
        total = total or 0
        if total == 0:
            return [], 0

        page_stmt = (
            select(Show.id)
            .join(Session, Session.show_id == Show.id)
            .where(*conditions)
            .group_by(Show.id)
            .order_by(func.min(Session.starts_at).asc())
            .offset((page - 1) * size)
            .limit(size)
        )
        page_ids = [row[0] for row in (await self._session.execute(page_stmt)).all()]
        if not page_ids:
            return [], total

        rows_stmt = (
            select(
                Show.id,
                Show.title,
                Show.synopsis,
                Show.image_url,
                Show.genre,
                func.min(Session.full_price_cents).label("price_min_cents"),
                func.max(Session.full_price_cents).label("price_max_cents"),
                func.array_agg(Session.starts_at).label("upcoming_dates"),
            )
            .join(Session, Session.show_id == Show.id)
            .where(*conditions, Show.id.in_(page_ids))
            .group_by(Show.id)
        )
        by_id = {row.id: row for row in (await self._session.execute(rows_stmt)).all()}
        ordered = [by_id[sid] for sid in page_ids if sid in by_id]
        return [
            ShowSearchRow(
                id=r.id,
                title=r.title,
                synopsis=r.synopsis,
                image_url=r.image_url,
                genre=r.genre,
                price_min_cents=r.price_min_cents,
                price_max_cents=r.price_max_cents,
                upcoming_dates=sorted(r.upcoming_dates),
            )
            for r in ordered
        ], total

    async def list_genres_in_catalog(self) -> list[str]:
        now = datetime.now(timezone.utc)
        conditions = self._visible_conditions(now, genre=None)
        result = await self._session.execute(
            select(Show.genre)
            .join(Session, Session.show_id == Show.id)
            .where(*conditions)
            .distinct()
            .order_by(Show.genre.asc())
        )
        return [row[0] for row in result.all()]
```

### `src/app/modules/catalog/application/usecases/show_search_usecase.py`

```python
from datetime import date as Date

from sqlalchemy.ext.asyncio import AsyncSession

from app.modules.catalog.application.schemas.response import (
    PagedShowsResponse,
    ShowCardResponse,
)
from app.modules.catalog.infrastructure.repositories import ShowRepository

_SYNOPSIS_SHORT_MAX = 160

def _synopsis_short(synopsis: str) -> str:
    if len(synopsis) <= _SYNOPSIS_SHORT_MAX:
        return synopsis
    return synopsis[:_SYNOPSIS_SHORT_MAX].rstrip() + "…"

class ShowSearchUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._show_repository = ShowRepository(session)

    async def search(
        self, *, from_date: Date, genre: str | None, page: int, size: int
    ) -> PagedShowsResponse:
        rows, total = await self._show_repository.search_with_upcoming(
            from_date=from_date, genre=genre, page=page, size=size
        )
        items = [
            ShowCardResponse(
                id=row.id,
                title=row.title,
                synopsis_short=_synopsis_short(row.synopsis),
                image_url=row.image_url,
                genre=row.genre,
                upcoming_dates=row.upcoming_dates,
                price_min=row.price_min_cents / 100,
                price_max=row.price_max_cents / 100,
            )
            for row in rows
        ]
        return PagedShowsResponse(items=items, page=page, size=size, total=total)

    async def list_genres(self) -> list[str]:
        return await self._show_repository.list_genres_in_catalog()
```

### `src/app/modules/catalog/api/routers/show_router.py`

```python
from datetime import date as Date

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.modules.catalog.application.schemas.response import PagedShowsResponse
from app.modules.catalog.application.usecases.show_search_usecase import ShowSearchUseCase

router = APIRouter(tags=["Catalog"])

@router.get("/shows", response_model=PagedShowsResponse)
async def list_shows(
    from_date: Date = Query(default_factory=Date.today, alias="fromDate"),
    genre: str | None = Query(default=None),
    page: int = Query(default=1, ge=1),
    size: int = Query(default=12, ge=1, le=50),
    session: AsyncSession = Depends(get_db),
) -> PagedShowsResponse:
    return await ShowSearchUseCase(session).search(
        from_date=from_date, genre=genre, page=page, size=size
    )

@router.get("/genres", response_model=list[str])
async def list_genres(session: AsyncSession = Depends(get_db)) -> list[str]:
    return await ShowSearchUseCase(session).list_genres()
```

`fromDate` é o único parâmetro em camelCase de todo o contrato — decisão
deliberada: é query string de URL pública, não corpo JSON; mantém a mesma
grafia que aparece em `integration.md` para não obrigar o frontend a
transformar nome de parâmetro de busca. Os demais (`genre`, `page`, `size`)
já são iguais nas duas grafias.

### `src/app/modules/catalog/router.py`

Editar: incluir `show_router` ao lado do `admin_catalog_router` que
`catalog-admin-management` já registra.

```python
from fastapi import APIRouter

from app.modules.catalog.api.routers.admin_catalog_router import router as admin_catalog_router
from app.modules.catalog.api.routers.show_router import router as show_router

router = APIRouter()
router.include_router(admin_catalog_router)
router.include_router(show_router)
```

---

## 4. Onde cada regra de negócio entra

| Regra | Arquivo · função | Como |
| --- | --- | --- |
| RF01 — só espetáculo publicado com sessão futura à venda | `infrastructure/repositories/show_repository.py` · `_visible_conditions` | `Show.status == PUBLISHED` + `Session.status == ON_SALE` + `Session.starts_at > threshold`, aplicado em toda leitura (busca e gêneros) |
| RF01 — faixa de preço e datas só de sessões futuras | `show_repository.py` · `search_with_upcoming` | `func.min/max(full_price_cents)` e `array_agg(starts_at)` calculados sobre o mesmo `JOIN` filtrado — nunca sobre todas as sessões |
| RF01 — ordenação pela próxima sessão, ascendente | `show_repository.py` · `search_with_upcoming` (`page_stmt`) | `ORDER BY MIN(Session.starts_at) ASC` antes de paginar |
| RF01 — filtro de data no passado vira hoje | `show_repository.py` · `search_with_upcoming` | `threshold = max(from_date, now)` |
| RF01 — filtro por gênero | `show_repository.py` · `_visible_conditions` | `Show.genre == genre` quando informado |
| RNF02 — busca ≤ 2 s p95 | índice `ix_sessions_starts_at_status` (criado por `catalog-admin-management`) | a query filtra e ordena por `starts_at`/`status`, cobertos pelo índice composto |

**Decisão de implementação (não é decisão de produto nova):** "sinopse curta"
não existe como campo no aggregate `Show` (só `synopsis`, até 5000 chars) — o
card trunca em 160 caracteres com reticências (`_synopsis_short`). Se o PO
quiser outro tamanho ou um campo dedicado, é ajuste neste usecase, não no
domínio.

---

## 5. DevOps

Não se aplica — nenhuma variável de ambiente, segredo ou CI novo.

---

## 6. Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-show-search

# commit 1 — leitura: schemas e usecase
git add src/app/modules/catalog/application/schemas/response.py \
        src/app/modules/catalog/application/usecases/show_search_usecase.py
git commit -m "feat(catalog): schemas e usecase de busca de espetaculos"

# commit 2 — repositório
git add src/app/modules/catalog/infrastructure/repositories/show_repository.py
git commit -m "feat(catalog): query de espetaculos com sessao futura e generos"

# commit 3 — rotas
git add src/app/modules/catalog/api/routers/show_router.py src/app/modules/catalog/router.py
git commit -m "feat(catalog): expor GET /shows e GET /genres"

ruff check . 2>/dev/null; pytest -q
```

Depois: `/team-ludens:tbd-pr` (senior-dev Modo 2 + `/code-review`) → push → PR
`Closes #<NN>` → merge (1 aprovação + CI verde). (`ruff` foi removido do
projeto — `pyproject.toml` não tem `[tool.ruff]`; rodar só se o lint local
existir, sem bloquear no CI.)

---

## 7. Ordem entre as superfícies

Depende do merge de `catalog-admin-management` (aggregates + índice). Backend
e QA começam juntos a partir do `logic.md`; frontend contra o contrato-alvo de
`integration.md`. `catalog-session-detail` é a fatia irmã seguinte, no mesmo
módulo.

---

## 8. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 fechada). Pontos de atenção:

- **`catalog-admin-management` ainda não mergeado** (spec já corrigida, ver
  nota no topo do documento) — os aggregates/repositórios que este documento
  importa só existem depois do merge.
- **`array_agg` é específico do Postgres** — consistente com o driver real do
  projeto (`asyncpg`), sem portabilidade a outro banco. Se isso mudar, revisar
  `search_with_upcoming`.
