---
status: done
spec: catalog-show-search
surface: backend
created_at: 2026-09-10
updated_at: 2026-09-21
---

# Busca e filtro de espetáculos — Backend

**Resumo:** leitura pública (`GET /catalog/shows`, `GET /catalog/genres`) sobre
os aggregates `Show`/`Session`/`Genre` de `catalog-admin-management`. A mesma
rota `GET /catalog/shows` serve busca pública (card, filtro por `genre_id` e
`from_date`) e listagem administrativa (resumo completo), decidido por
`is_admin` na própria função da rota. Filtra por espetáculo publicado com ≥ 1
sessão futura à venda, agrega faixa de preço e próximas datas, pagina via um
`Page[T]` genérico compartilhado.
**RF:** RF01 · **RN:** — · **Módulo backend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-show-search/integration.md`
**Carregar antes:** skill `backend-architecture`, `docs.ludens/backend/overview.md`,
`docs.ludens/backend/conventions.md`.

**Depende de:** `catalog-admin-management` (aggregates `Show`/`Session`/`Genre`,
repositórios, migrations) e `catalog-genre` (entidade `Genre` própria, com
`genre_id` em `Show`) — ambas já mergeadas. **É base de:**
`catalog-session-detail` (mesmo módulo, rota irmã).

> **Nota histórica.** Este documento previa originalmente um router e um
> usecase próprios (`show_search_usecase.py`, métodos novos em
> `ShowRepository`) e um filtro por `genre: str` (texto livre). O código real
> juntou busca e CRUD administrativo num único `show_router.py`/`ShowUseCase`
> (a busca é só mais um método, `search`), moveu a query agregada para uma
> classe de consulta dedicada (`ShowSearchQuery`) e passou a filtrar por
> `genre_id: UUID` depois que `catalog-genre` transformou gênero de texto
> livre em entidade própria. Reescrito em 2026-09-21 para refletir o código
> real — ver "Ajustes feitos no `integration.md`" ao final.

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | application | `src/app/modules/catalog/application/schemas/response.py` | editar — `ShowCardResponse` (com `genre_id` e `genre`) |
| 2 | application | `src/app/modules/catalog/application/usecases/utils/search_floor.py` | novo — `floor_from(from_date)`, piso em horário de Brasília |
| 3 | infrastructure | `src/app/modules/catalog/infrastructure/queries/show_search_query.py` | novo — `ShowSearchQuery.search_with_upcoming` |
| 4 | application | `src/app/modules/catalog/application/mappers/show_mapper.py` | editar — `card_response(row)`, `synopsis_short` |
| 5 | infrastructure | `src/app/modules/catalog/infrastructure/repositories/genre_repository.py` | editar — `find_all_by_ids` (resolve nome do gênero em lote) |
| 6 | application | `src/app/modules/catalog/application/usecases/show_usecase.py` | editar — `ShowUseCase.search` |
| 7 | api | `src/app/modules/catalog/api/routers/show_router.py` | editar — `GET /catalog/shows` (sem sub-rota própria de busca) |

Não há migration nova: a query usa o mesmo índice que `catalog-admin-management`
já cria sobre `sessions(starts_at, status)`. Sem variável de ambiente nova.

---

## 2. Código real que este documento assume (não recriar)

```python
# app.core.shared — real, compartilhado por todo o backend
class Page(BaseModel, Generic[T]):
    items: list[T]
    page: int
    size: int
    total: int

@dataclass(frozen=True)
class PaginationParams:
    page: int
    size: int

def make_pagination_params(*, default_size: int = 20, max_size: int = 50) -> Callable[..., PaginationParams]: ...
# GET /catalog/shows usa make_pagination_params(default_size=12, max_size=48)

# app.core.infrastructure.queries.paginate(session, stmt, page, size) -> (rows, total)
```

Schemas neste módulo são `pydantic.BaseModel` puro, campos **snake_case** —
não existe `CamelModel`/alias camelCase no repo real (única exceção histórica,
`fromDate`, foi removida: ver nota acima e `integration.md`).

---

## 3. Código

### `src/app/modules/catalog/application/usecases/utils/search_floor.py`

```python
from datetime import date, datetime, time, timedelta, timezone

# Horário de Brasília. Offset fixo, não ZoneInfo: o Brasil não observa horário
# de verão desde 2019, e ZoneInfo exigiria o pacote tzdata no Windows.
CATALOG_TZ = timezone(timedelta(hours=-3))

def floor_from(from_date: date | None) -> datetime:
    now = datetime.now(timezone.utc)
    if from_date is None:
        return now

    # A data vem do calendário de quem visita, não de UTC: "a partir de
    # 13/09" precisa começar à meia-noite de Brasília, senão pega a noite do
    # dia 12. Uma data passada não faz sentido para uma sessão futura — o
    # piso nunca anda pra trás.
    return max(datetime.combine(from_date, time.min, tzinfo=CATALOG_TZ), now)
```

### `src/app/modules/catalog/infrastructure/queries/show_search_query.py`

```python
from __future__ import annotations

from uuid import UUID
from typing import NamedTuple
from datetime import datetime

from sqlalchemy import distinct, func, select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.dialects.postgresql import aggregate_order_by

from app.core.infrastructure.queries import paginate
from app.modules.catalog.domain.aggregates import Genre, Session, Show
from app.modules.catalog.domain.enumerations import SessionStatus, ShowStatus

class ShowCardRow(NamedTuple):
    id: UUID
    title: str
    synopsis: str
    image_url: str
    genre_id: UUID
    genre_name: str
    upcoming_dates: list[datetime]
    price_min_cents: int
    price_max_cents: int

class ShowSearchQuery:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def search_with_upcoming(
        self, *, floor: datetime, genre_id: UUID | None, page: int, size: int
    ) -> tuple[list[ShowCardRow], int]:
        next_at = func.min(Session.starts_at).label("next_at")
        stmt = (
            select(
                Show.id, Show.title, Show.synopsis, Show.image_url, Show.genre_id,
                Genre.name.label("genre_name"),
                func.array_agg(
                    aggregate_order_by(distinct(Session.starts_at), Session.starts_at.asc())
                ).label("upcoming_dates"),
                func.min(Session.full_price_cents).label("price_min_cents"),
                func.max(Session.full_price_cents).label("price_max_cents"),
                next_at,
            )
            .join(Session, Session.show_id == Show.id)
            .join(Genre, Genre.id == Show.genre_id)
            .where(*self._filters(floor, genre_id))
            .group_by(Show.id, Genre.name)
            .order_by(next_at.asc(), Show.id.asc())
        )

        rows, total = await paginate(self._session, stmt, page=page, size=size)

        return [
            ShowCardRow(
                id=row.id, title=row.title, synopsis=row.synopsis, image_url=row.image_url,
                genre_id=row.genre_id, genre_name=row.genre_name,
                upcoming_dates=list(row.upcoming_dates),
                price_min_cents=row.price_min_cents, price_max_cents=row.price_max_cents,
            )
            for row in rows
        ], total

    @staticmethod
    def _filters(floor: datetime, genre_id: UUID | None) -> list:
        filters = [
            Show.is_active.is_(True),
            Show.status == ShowStatus.PUBLISHED,
            Session.is_active.is_(True),
            Session.status == SessionStatus.ON_SALE,
            Session.starts_at >= floor,
        ]
        if genre_id is not None:
            filters.append(Show.genre_id == genre_id)
        return filters
```

### `src/app/modules/catalog/application/mappers/show_mapper.py`

```python
from app.modules.catalog.infrastructure.queries import ShowCardRow
from app.modules.catalog.application.schemas.response import ShowCardResponse
from app.modules.catalog.application.utils import reais_from_cents

_SYNOPSIS_MAX = 160
_UPCOMING_DATES_MAX = 5

def synopsis_short(synopsis: str) -> str:
    if len(synopsis) <= _SYNOPSIS_MAX:
        return synopsis
    return synopsis[: _SYNOPSIS_MAX - 1].rstrip() + "…"

def card_response(row: ShowCardRow) -> ShowCardResponse:
    return ShowCardResponse(
        id=row.id,
        title=row.title,
        synopsis_short=synopsis_short(row.synopsis),
        image_url=row.image_url,
        genre_id=row.genre_id,
        genre=row.genre_name,
        upcoming_dates=row.upcoming_dates[:_UPCOMING_DATES_MAX],
        price_min=reais_from_cents(row.price_min_cents),
        price_max=reais_from_cents(row.price_max_cents),
    )
```

`upcoming_dates` corta em 5 datas (não documentado na versão original) — a
tela de busca mostra "próximas datas", não o calendário inteiro.

### `src/app/modules/catalog/application/usecases/show_usecase.py` (recorte, `search`)

```python
async def search(
    self, *, from_date: date | None, genre_id: UUID | None, pagination: PaginationParams, is_admin: bool
) -> Page[AdminShowSummaryResponse] | Page[ShowCardResponse]:
    if is_admin:
        shows, total = await self._show_repository.find_all_paginated(
            order_by=["-created_at"], page=pagination.page, size=pagination.size
        )
        genres = await self._genre_repository.find_all_by_ids([s.genre_id for s in shows])
        names = {g.id: g.name for g in genres}
        return Page(
            items=[_show_summary(s, names.get(s.genre_id, "")) for s in shows],
            page=pagination.page, size=pagination.size, total=total,
        )

    floor = floor_from(from_date)
    rows, total = await self._show_search_query.search_with_upcoming(
        floor=floor, genre_id=genre_id, page=pagination.page, size=pagination.size
    )
    return Page(
        items=[card_response(row) for row in rows],
        page=pagination.page, size=pagination.size, total=total,
    )
```

A mesma rota devolve `Page[AdminShowSummaryResponse]` (todo espetáculo, com
`status`) para admin e `Page[ShowCardResponse]` (só publicado, com sessão
futura) para o público — não existe rota de busca separada.

### `src/app/modules/catalog/api/routers/show_router.py` (recorte, rota de listagem)

O router real (`prefix="/catalog/shows"`) também expõe `POST`, `PUT /{id}`,
`POST /{id}/publish`, `POST /{id}/unpublish`, `DELETE /{id}` (CRUD
administrativo, escopo de `catalog-admin-management`) e `GET /{id}` (detalhe,
escopo de `catalog-session-detail`); aqui só a rota de listagem/busca:

```python
router = APIRouter(prefix="/catalog/shows", tags=["01.Catalog - Show"])

@router.get("", response_model=None)
async def search(
    from_date: date | None = Query(default=None),
    genre_id: UUID | None = Query(default=None),
    pagination: PaginationParams = Depends(make_pagination_params(default_size=12, max_size=48)),
    current_user: User | None = Depends(get_current_user_optional),
    session: AsyncSession = Depends(get_db),
) -> Page[AdminShowSummaryResponse] | Page[ShowCardResponse]:
    is_admin = current_user is not None and current_user.is_admin
    return await ShowUseCase(session).search(
        from_date=from_date, genre_id=genre_id, pagination=pagination, is_admin=is_admin
    )
```

A lista de gêneros para o filtro vem de uma rota separada, de `catalog-genre`:
`GET /catalog/genres` → `list[GenreResponse]` (`{id, name}`), não um `list[str]`
próprio deste documento.

---

## 4. Onde cada regra de negócio entra

| Regra | Arquivo · função | Como |
| --- | --- | --- |
| RF01 — só espetáculo publicado com sessão futura à venda | `infrastructure/queries/show_search_query.py` · `_filters` | `Show.status == PUBLISHED` + `Session.status == ON_SALE` + `Session.starts_at >= floor`, aplicado em toda busca pública |
| RF01 — faixa de preço e datas só de sessões futuras | `show_search_query.py` · `search_with_upcoming` | `func.min/max(full_price_cents)` e `array_agg` calculados sobre o mesmo `JOIN` filtrado |
| RF01 — ordenação pela próxima sessão, ascendente | `show_search_query.py` · `search_with_upcoming` | `ORDER BY next_at ASC, Show.id ASC` antes de paginar |
| RF01 — filtro de data no passado vira "agora" (fuso de Brasília) | `usecases/utils/search_floor.py` · `floor_from` | `max(meia-noite de Brasília do from_date, now UTC)` |
| RF01 — filtro por gênero | `show_search_query.py` · `_filters` | `Show.genre_id == genre_id` quando informado |
| RNF02 — busca ≤ 2 s p95 | índice de `sessions(starts_at, status)` (criado por `catalog-admin-management`) | a query filtra e ordena por `starts_at`/`status`, cobertos pelo índice |

**Decisão de implementação (não é decisão de produto nova):** "sinopse curta"
não existe como campo no aggregate `Show` (só `synopsis`) — o card trunca em
160 caracteres com reticências (`synopsis_short`). `upcoming_dates` corta em 5
itens.

---

## 5. DevOps

Não se aplica — nenhuma variável de ambiente, segredo ou CI novo.

---

## 6. Passo a passo TBD (Backend)

Já mergeado. Sequência real, pra referência:

```text
feat(catalog): piso de busca em horário de Brasília (floor_from)
feat(catalog): ShowSearchQuery com filtro por genre_id e paginação genérica
feat(catalog): card_response e ShowUseCase.search (público x admin)
feat(catalog): GET /catalog/shows aceita from_date/genre_id/pagination
```

Sem lint automatizado — o projeto não usa Ruff nem outro formatter (ver
[`backend/code-style.md`](../../backend/code-style.md)).

---

## 7. Ordem entre as superfícies

Depende do merge de `catalog-admin-management` e `catalog-genre` (ambos já
mergeados). `catalog-session-detail` é a fatia irmã seguinte, no mesmo router.

---

## 8. Débitos técnicos registrados

- `array_agg`/`aggregate_order_by` são específicos do Postgres (driver real:
  `asyncpg`), sem portabilidade a outro banco.
- A listagem administrativa (`is_admin=True`) resolve o nome do gênero em lote
  via `GenreRepository.find_all_by_ids` — um segundo round-trip ao banco, não
  um `JOIN` único como no caminho público (`ShowSearchQuery`).

## 9. Ajustes feitos no `integration.md`

- Filtro passou de `genre: str` (texto livre) para `genre_id: UUID`, depois
  que `catalog-genre` criou a entidade `Genre`.
- `fromDate` (alias camelCase) foi removido; o parâmetro real é `from_date`,
  sem alias.
- `GET /genres` próprio deste documento não existe; a lista de gêneros é
  `GET /catalog/genres`, de `catalog-genre`.
- A resposta paginada não é um `PagedShowsResponse` bespoke; é o `Page[T]`
  genérico usado em todo o backend.
- A rota de busca não é `GET /shows`, é `GET /catalog/shows` — a mesma rota
  também serve a listagem administrativa completa.
