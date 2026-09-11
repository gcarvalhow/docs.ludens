---
status: draft
spec: catalog-genre
surface: backend
created_at: 2026-09-11
---

# Gêneros do catálogo — Backend

**Resumo:** gênero de espetáculo deixa de ser texto livre e vira entidade própria (`Genre`) no módulo `catalog`, criada exclusivamente pelo admin, com nome normalizado (identidade única) e ícone escolhido de uma paleta fixa. `Show.genre` vira `Show.genre_id` (FK).
**RF:** fortalece RF01 · ajusta critério de aceite de RF08 · **RN:** nenhuma RN01–RN05 diretamente — regra própria da feature ("nome normalizado é a identidade única do gênero", `logic.md` §3) · **Módulo backend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-genre/integration.md`
**Carregar antes:** skill `backend-architecture` (todos os `references/`), `docs.ludens/backend/overview.md`.

Código lido contra `master` (commit `80fc684`) — **não** contra o branch `refactor/21-show-list-detail` (PR api.ludens#23, ainda não mergeado). Quem implementar esta fatia precisa reconciliar manualmente com o resultado de #23 se ainda não tiver mergeado (ver §7).

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
|---|---|---|---|
| 1 | domain | `src/app/modules/catalog/domain/aggregates/genre.py` | novo |
| 2 | domain | `src/app/modules/catalog/domain/aggregates/__init__.py` | editar |
| 3 | domain | `src/app/modules/catalog/domain/events/domain_events.py` | editar |
| 4 | domain | `src/app/modules/catalog/domain/events/__init__.py` | editar |
| 5 | domain | `src/app/modules/catalog/domain/aggregates/show.py` | editar |
| 6 | application | `src/app/modules/catalog/application/schemas/request.py` | editar |
| 7 | application | `src/app/modules/catalog/application/schemas/response.py` | editar |
| 8 | application | `src/app/modules/catalog/application/usecases/genre_usecase.py` | novo |
| 9 | application | `src/app/modules/catalog/application/usecases/show_usecase.py` | editar |
| 10 | application | `src/app/modules/catalog/application/usecases/utils/show_card.py` | editar |
| 11 | application | `src/app/modules/catalog/application/usecases/utils/genre_slug.py` | **apagar** |
| 12 | infrastructure | `src/app/modules/catalog/infrastructure/repositories/genre_repository.py` | novo |
| 13 | infrastructure | `src/app/modules/catalog/infrastructure/repositories/show_repository.py` | editar |
| 14 | infrastructure | `src/app/modules/catalog/infrastructure/repositories/__init__.py` | editar |
| 15 | api | `src/app/modules/catalog/api/routers/admin_catalog_router.py` | editar |
| 16 | api | `src/app/modules/catalog/api/routers/show_router.py` | editar |
| 17 | migration | `src/migrations/env.py` | editar (1 linha) |
| 18 | migration | `src/migrations/versions/0003_catalog_genre.py` | novo |

Sem evento consumido por outro módulo, sem handler de outbox, sem `dependencies.py` novo (nada exportado pra outro módulo nesta fatia) — genre não sai do próprio módulo `catalog`.

## 2. Código

### `src/app/modules/catalog/domain/aggregates/genre.py` — novo

```python
from __future__ import annotations

import re
import unicodedata
from uuid import UUID, uuid4

from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain import DomainError, Model
from app.core.domain.aggregate import AggregateRoot

from app.modules.catalog.domain.events import GenreCreated

def normalize_genre_name(raw: str) -> str:
    # A forma normalizada (sem acento, sem maiúscula, espaços colapsados) é a
    # própria identidade do gênero — resolve duplicidade e migração de
    # grafias com a mesma regra (logic.md §3).
    text = unicodedata.normalize("NFKD", raw.strip()).encode("ascii", "ignore").decode("ascii")
    return re.sub(r"\s+", " ", text).lower()

class Genre(AggregateRoot, Model):
    __tablename__ = "genres"

    name: Mapped[str] = mapped_column(String(80), nullable=False)
    icon: Mapped[str] = mapped_column(String(80), nullable=False)

    @classmethod
    def create(cls, *, name: str, icon: str) -> "Genre":
        normalized = normalize_genre_name(name)
        if not normalized:
            raise DomainError("Nome do gênero inválido.")

        icon = (icon or "").strip()
        if not icon:
            raise DomainError("Ícone do gênero é obrigatório.")

        genre = cls()
        genre.id = uuid4()

        genre.raise_event(
            lambda v: GenreCreated(version=v, id=genre.id, name=normalized, icon=icon)
        )
        return genre

    def _when_GenreCreated(self, e: GenreCreated) -> None:
        self.name = e.name
        self.icon = e.icon
        self.is_active = True
```

> Ícone validado no próprio domínio (não só na forma via `GenreRequest`,
> `Field(min_length=1)`) — reforça o invariante "todo gênero precisa de
> ícone" (`logic.md` §3) na camada certa, e é o que os casos de teste
> `test_icone_*_e_recusado` de `quality.md` assumem.

### `src/app/modules/catalog/domain/aggregates/__init__.py` — editar

```python
from .genre import Genre, normalize_genre_name
from .session import Session
from .show import Show

__all__ = ["Genre", "Session", "Show", "normalize_genre_name"]
```

### `src/app/modules/catalog/domain/events/domain_events.py` — editar (arquivo completo)

```python
from __future__ import annotations

from uuid import UUID
from datetime import datetime
from dataclasses import dataclass, field

from app.core.domain.events import DomainEvent

@dataclass(frozen=True)
class GenreCreated(DomainEvent):
    id: UUID = field(kw_only=True)
    name: str = field(kw_only=True)
    icon: str = field(kw_only=True)

@dataclass(frozen=True)
class ShowCreated(DomainEvent):
    id: UUID = field(kw_only=True)
    title: str = field(kw_only=True)
    synopsis: str = field(kw_only=True)
    image_url: str = field(kw_only=True)
    genre_id: UUID = field(kw_only=True)

@dataclass(frozen=True)
class ShowUpdated(DomainEvent):
    id: UUID = field(kw_only=True)
    title: str = field(kw_only=True)
    synopsis: str = field(kw_only=True)
    image_url: str = field(kw_only=True)
    genre_id: UUID = field(kw_only=True)

@dataclass(frozen=True)
class ShowPublished(DomainEvent):
    id: UUID = field(kw_only=True)

@dataclass(frozen=True)
class ShowUnpublished(DomainEvent):
    id: UUID = field(kw_only=True)

@dataclass(frozen=True)
class ShowDeactivated(DomainEvent):
    id: UUID = field(kw_only=True)

@dataclass(frozen=True)
class SessionCreated(DomainEvent):
    id: UUID = field(kw_only=True)
    show_id: UUID = field(kw_only=True)
    starts_at: datetime = field(kw_only=True)
    venue: str = field(kw_only=True)
    capacity: int = field(kw_only=True)
    full_price_cents: int = field(kw_only=True)

@dataclass(frozen=True)
class SessionUpdated(DomainEvent):
    id: UUID = field(kw_only=True)
    starts_at: datetime = field(kw_only=True)
    venue: str = field(kw_only=True)
    capacity: int = field(kw_only=True)
    full_price_cents: int = field(kw_only=True)

@dataclass(frozen=True)
class SessionCancelled(DomainEvent):
    id: UUID = field(kw_only=True)
    show_id: UUID = field(kw_only=True)
    starts_at: datetime = field(kw_only=True)
    cancelled_at: datetime = field(kw_only=True)

@dataclass(frozen=True)
class SessionDeactivated(DomainEvent):
    id: UUID = field(kw_only=True)
```

### `src/app/modules/catalog/domain/events/__init__.py` — editar

```python
from .domain_events import (
    GenreCreated,
    SessionCancelled,
    SessionCreated,
    SessionDeactivated,
    SessionUpdated,
    ShowCreated,
    ShowDeactivated,
    ShowPublished,
    ShowUnpublished,
    ShowUpdated,
)

__all__ = [
    "GenreCreated",
    "SessionCancelled",
    "SessionCreated",
    "SessionDeactivated",
    "SessionUpdated",
    "ShowCreated",
    "ShowDeactivated",
    "ShowPublished",
    "ShowUnpublished",
    "ShowUpdated",
]
```

### `src/app/modules/catalog/domain/aggregates/show.py` — editar (arquivo completo)

```python
from __future__ import annotations

from uuid import UUID, uuid4
from sqlalchemy import ForeignKey, String, Uuid
from sqlalchemy import Enum as SAEnum
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain.model import Model
from app.core.domain.events import DomainEvent
from app.core.domain.aggregate import AggregateRoot

from app.modules.catalog.domain.enumerations import ShowStatus
from app.modules.catalog.domain.events import (
    ShowCreated,
    ShowDeactivated,
    ShowPublished,
    ShowUnpublished,
    ShowUpdated,
)

class Show(AggregateRoot, Model):
    __tablename__ = "shows"

    title: Mapped[str] = mapped_column(String(200), nullable=False)
    synopsis: Mapped[str] = mapped_column(String(5000), nullable=False)
    image_url: Mapped[str] = mapped_column(String(2048), nullable=False)
    genre_id: Mapped[UUID] = mapped_column(Uuid(), ForeignKey("genres.id"), nullable=False, index=True)

    status: Mapped[ShowStatus] = mapped_column(
        SAEnum(
            ShowStatus,
            native_enum=False,
            length=20,
            values_callable=lambda enum: [m.value for m in enum],
        ),
        nullable=False,
        default=ShowStatus.DRAFT,
    )

    @classmethod
    def create(cls, *, title: str, synopsis: str, image_url: str, genre_id: UUID) -> "Show":
        show = cls()
        show.id = uuid4()

        show.raise_event(
            lambda v: ShowCreated(
                version=v, id=show.id, title=title, synopsis=synopsis,
                image_url=image_url, genre_id=genre_id,
            )
        )
        return show

    def update(self, *, title: str, synopsis: str, image_url: str, genre_id: UUID) -> None:
        self.raise_event(
            lambda v: ShowUpdated(
                version=v, id=self.id, title=title, synopsis=synopsis,
                image_url=image_url, genre_id=genre_id,
            )
        )

    def publish(self) -> None:
        if self.status is ShowStatus.PUBLISHED:
            return

        self.raise_event(lambda v: ShowPublished(version=v, id=self.id))

    def unpublish(self) -> None:
        if self.status is ShowStatus.DRAFT:
            return

        self.raise_event(lambda v: ShowUnpublished(version=v, id=self.id))

    def deactivate(self) -> None:
        self.raise_event(lambda v: ShowDeactivated(version=v, id=self.id))

    def _apply(self, event: DomainEvent) -> None:
        handler = getattr(self, f"_when_{type(event).__name__}", None)
        if handler is not None:
            handler(event)

    def _when_ShowCreated(self, e: ShowCreated) -> None:
        self.title = e.title
        self.synopsis = e.synopsis
        self.image_url = e.image_url
        self.genre_id = e.genre_id
        self.status = ShowStatus.DRAFT
        self.is_active = True

    def _when_ShowUpdated(self, e: ShowUpdated) -> None:
        self.title = e.title
        self.synopsis = e.synopsis
        self.image_url = e.image_url
        self.genre_id = e.genre_id

    def _when_ShowPublished(self, _event: ShowPublished) -> None:
        self.status = ShowStatus.PUBLISHED

    def _when_ShowUnpublished(self, _event: ShowUnpublished) -> None:
        self.status = ShowStatus.DRAFT

    def _when_ShowDeactivated(self, _event: ShowDeactivated) -> None:
        self.is_active = False
```

### `src/app/modules/catalog/application/schemas/request.py` — editar (arquivo completo)

```python
from __future__ import annotations

from uuid import UUID
from datetime import datetime

from pydantic import BaseModel, Field, field_validator

class GenreRequest(BaseModel):
    name: str = Field(min_length=1, max_length=80)
    icon: str = Field(min_length=1, max_length=80, pattern=r"^[a-z][a-z0-9-]*$")

class ShowRequest(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    synopsis: str = Field(min_length=1, max_length=5000)
    genre_id: UUID

class SessionRequest(BaseModel):
    starts_at: datetime
    venue: str = Field(min_length=1, max_length=200)
    capacity: int = Field(gt=0, le=100_000)
    full_price: float = Field(gt=0)

    @field_validator("starts_at")
    @classmethod
    def _tz_aware(cls, value: datetime) -> datetime:
        if value.tzinfo is None:
            raise ValueError("informe a data com fuso horário (ISO 8601 com offset)")
        return value
```

### `src/app/modules/catalog/application/schemas/response.py` — editar (arquivo completo)

```python
from __future__ import annotations

from uuid import UUID
from typing import Literal
from datetime import datetime

from pydantic import BaseModel

class GenreResponse(BaseModel):
    id: UUID
    name: str
    icon: str

class AdminSessionResponse(BaseModel):
    id: UUID
    show_id: UUID
    starts_at: datetime
    venue: str
    capacity: int
    full_price: float
    half_price: float
    status: Literal["on_sale", "closed", "cancelled"]
    tickets_sold: int
    reserved_open: int
    can_delete: bool

class AdminShowSummaryResponse(BaseModel):
    id: UUID
    title: str
    synopsis: str
    image_url: str
    genre: GenreResponse
    status: Literal["draft", "published"]

class AdminShowResponse(AdminShowSummaryResponse):
    sessions: list[AdminSessionResponse]

class ShowCardResponse(BaseModel):
    id: UUID
    title: str
    synopsis_short: str
    image_url: str
    genre: GenreResponse
    upcoming_dates: list[datetime]
    price_min: float
    price_max: float

class PagedShowsResponse(BaseModel):
    items: list[ShowCardResponse]
    page: int
    size: int
    total: int

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
    genre: GenreResponse
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

> Nota de reconciliação com `api.ludens` PR #23 (`refactor/21-show-list-detail`, ainda não mergeado): esse PR introduziu `AdminShowSummaryResponse`/`AdminShowResponse(AdminShowSummaryResponse)` — já incorporado acima. Se #23 mergear antes desta feature, o `genre: str` que existia nesses schemas já terá virado `genre: GenreResponse` nesta versão; aplique só o diff de `str`→`GenreResponse` sobre o código real, não reescreva o arquivo inteiro por cima.

### `src/app/modules/catalog/application/usecases/genre_usecase.py` — novo

```python
from __future__ import annotations

from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain import ConflictError

from app.modules.catalog.domain.aggregates import Genre, normalize_genre_name
from app.modules.catalog.application.schemas.request import GenreRequest
from app.modules.catalog.application.schemas.response import GenreResponse

from app.modules.catalog.infrastructure.repositories import GenreRepository

def _genre_response(genre: Genre) -> GenreResponse:
    return GenreResponse(id=genre.id, name=genre.name, icon=genre.icon)

class GenreUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session
        self._genre_repository = GenreRepository(session)

    async def create_genre(self, req: GenreRequest) -> GenreResponse:
        normalized = normalize_genre_name(req.name)

        # Duplicidade sem diferenciar maiúscula/acento (logic.md §3). Checagem
        # otimista aqui + backstop atômico no índice único parcial
        # uq_genres_name_active (migration 0003_catalog_genre) — cobre criação
        # concorrente com o mesmo nome (ver quality.md §5, risco de concorrência).
        if await self._genre_repository.exists_by("name", normalized):
            raise ConflictError("Já existe um gênero com esse nome.")

        genre = Genre.create(name=req.name, icon=req.icon)
        await self._genre_repository.save(genre)

        try:
            await self._session.flush()
        except IntegrityError as exc:
            raise ConflictError("Já existe um gênero com esse nome.") from exc

        return _genre_response(genre)

    async def list_all(self) -> list[GenreResponse]:
        genres = await self._genre_repository.find_all(order_by=["name"])
        return [_genre_response(g) for g in genres]
```

### `src/app/modules/catalog/application/usecases/show_usecase.py` — editar (arquivo completo)

```python
from __future__ import annotations

import random
from uuid import UUID
from datetime import date, datetime, timezone

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain.errors import ConflictError, NotFoundError

from app.modules.catalog.domain.aggregates import Genre, Session, Show
from app.modules.catalog.application.schemas.request import ShowRequest
from app.modules.catalog.application.schemas.response import (
    AdminShowResponse,
    AdminShowSummaryResponse,
    GenreResponse,
    PagedShowsResponse,
    SessionSummaryResponse,
    ShowDetailResponse,
)
from app.modules.catalog.application.usecases.utils.show_card import card_response
from app.modules.catalog.application.usecases.utils.search_floor import floor_from
from app.modules.catalog.application.usecases.utils.published_show import require_published_show
from app.modules.catalog.application.usecases.utils.session_response import session_response
from app.modules.catalog.application.usecases.utils.session_availability import (
    available_count,
    session_status,
)

from app.modules.catalog.infrastructure.repositories import (
    GenreRepository,
    SeatCounts,
    SeatCountsRepository,
    SessionRepository,
    ShowRepository,
)

_DEFAULT_SHOW_IMAGES = [
    "/images/show-placeholders/1.jpg",
    "/images/show-placeholders/2.jpg",
    "/images/show-placeholders/3.jpg",
    "/images/show-placeholders/4.jpg",
    "/images/show-placeholders/5.jpg",
    "/images/show-placeholders/6.jpg",
]

def _genre_response(genre: Genre) -> GenreResponse:
    return GenreResponse(id=genre.id, name=genre.name, icon=genre.icon)

def _show_summary(show: Show, genre: Genre) -> AdminShowSummaryResponse:
    return AdminShowSummaryResponse(
        id=show.id,
        title=show.title,
        synopsis=show.synopsis,
        image_url=show.image_url,
        genre=_genre_response(genre),
        status=show.status.value,
    )

def _show_response(show: Show, genre: Genre, sessions: list[Session], counts_map: dict[UUID, SeatCounts], now: datetime) -> AdminShowResponse:
    return AdminShowResponse(
        **_show_summary(show, genre).model_dump(),
        sessions=[
            session_response(s, counts_map.get(s.id, SeatCounts(0, 0)), now) for s in sessions
        ],
    )

def _session_summary(session: Session, counts: SeatCounts, now: datetime) -> SessionSummaryResponse:
    available = available_count(session, counts)

    return SessionSummaryResponse(
        id=session.id,
        starts_at=session.starts_at,
        venue=session.venue,
        capacity=session.capacity,
        available_count=available,
        status=session_status(session, available, now),
    )

class ShowUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._show_repository = ShowRepository(session)
        self._session_repository = SessionRepository(session)
        self._seat_counts_repository = SeatCountsRepository(session)
        self._genre_repository = GenreRepository(session)

    async def list_shows(self) -> list[AdminShowSummaryResponse]:
        shows = await self._show_repository.find_all(order_by=["-created_at"])
        genres_by_id = {
            g.id: g for g in await self._genre_repository.find_all_by_ids([s.genre_id for s in shows])
        }
        return [_show_summary(show, genres_by_id[show.genre_id]) for show in shows]

    async def get_show(self, show_id: UUID) -> AdminShowResponse:
        show = await self._require_show(show_id)
        genre = await self._require_genre(show.genre_id)
        return await self._view_for(show, genre)

    async def create_show(self, req: ShowRequest) -> AdminShowResponse:
        genre = await self._require_genre(req.genre_id)
        show = Show.create(
            title=req.title,
            synopsis=req.synopsis,
            image_url=random.choice(_DEFAULT_SHOW_IMAGES),
            genre_id=genre.id,
        )

        await self._show_repository.save(show)
        return _show_response(show, genre, [], {}, datetime.now(timezone.utc))

    async def update_show(self, show_id: UUID, req: ShowRequest) -> AdminShowResponse:
        show = await self._require_show(show_id)
        genre = await self._require_genre(req.genre_id)
        show.update(
            title=req.title,
            synopsis=req.synopsis,
            image_url=show.image_url,
            genre_id=genre.id,
        )

        await self._show_repository.save(show)
        return await self._view_for(show, genre)

    async def publish_show(self, show_id: UUID) -> None:
        show = await self._require_show(show_id)
        show.publish()

        await self._show_repository.save(show)

    async def unpublish_show(self, show_id: UUID) -> None:
        show = await self._require_show(show_id)
        show.unpublish()

        await self._show_repository.save(show)

    async def delete_show(self, show_id: UUID) -> None:
        show = await self._require_show(show_id)
        sessions = await self._session_repository.find_all_for_shows([show.id])
        counts = await self._seat_counts_repository.for_sessions([s.id for s in sessions])

        if any(counts.get(s.id, SeatCounts(0, 0)).tickets_sold > 0 for s in sessions):
            raise ConflictError(
                "Cancele as sessões com ingressos vendidos antes de excluir o espetáculo."
            )

        show.deactivate()
        await self._show_repository.save(show)

    async def search(self, *, from_date: date | None, genre_id: UUID | None, page: int, size: int) -> PagedShowsResponse:
        floor = floor_from(from_date)

        result = await self._show_repository.search_with_upcoming(
            floor=floor, genre_id=genre_id, page=page, size=size
        )

        return PagedShowsResponse(
            items=[card_response(row) for row in result.rows],
            page=page,
            size=size,
            total=result.total,
        )

    async def list_genres(self) -> list[GenreResponse]:
        genres = await self._show_repository.list_genres_in_catalog(
            floor=datetime.now(timezone.utc)
        )
        return [_genre_response(g) for g in genres]

    async def get_show_detail(self, show_id: UUID) -> ShowDetailResponse:
        show = await require_published_show(self._show_repository, show_id)
        genre = await self._require_genre(show.genre_id)
        now = datetime.now(timezone.utc)

        sessions = [
            s
            for s in await self._session_repository.find_all_for_shows([show.id])
            if s.starts_at > now
        ]
        counts = await self._seat_counts_repository.for_sessions([s.id for s in sessions])

        return ShowDetailResponse(
            id=show.id,
            title=show.title,
            synopsis=show.synopsis,
            image_url=show.image_url,
            genre=_genre_response(genre),
            sessions=[
                _session_summary(s, counts.get(s.id, SeatCounts(0, 0)), now) for s in sessions
            ],
        )

    async def _require_show(self, show_id: UUID) -> Show:
        show = await self._show_repository.find_by("id", show_id)
        if show is None:
            raise NotFoundError("Espetáculo não encontrado.")

        return show

    async def _require_genre(self, genre_id: UUID) -> Genre:
        genre = await self._genre_repository.find_by("id", genre_id)
        if genre is None:
            raise NotFoundError("Gênero não encontrado.")

        return genre

    async def _view_for(self, show: Show, genre: Genre) -> AdminShowResponse:
        sessions = await self._session_repository.find_all_for_shows([show.id])
        counts = await self._seat_counts_repository.for_sessions([s.id for s in sessions])

        return _show_response(show, genre, sessions, counts, datetime.now(timezone.utc))
```

> Reconciliação com PR #23: `list_shows`/`get_show`/`_show_summary`/`_view_for` acima já incorporam o split de resumo/detalhe daquele PR — não reaplique #23 por cima disto, é a mesma mudança já composta com a de gênero.

### `src/app/modules/catalog/application/usecases/utils/show_card.py` — editar (arquivo completo)

```python
from app.modules.catalog.application.schemas.response import GenreResponse, ShowCardResponse
from app.modules.catalog.application.usecases.utils.money import reais_from_cents
from app.modules.catalog.infrastructure.repositories import ShowCardRow

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
        genre=GenreResponse(id=row.genre_id, name=row.genre_name, icon=row.genre_icon),
        upcoming_dates=row.upcoming_dates[:_UPCOMING_DATES_MAX],
        price_min=reais_from_cents(row.price_min_cents),
        price_max=reais_from_cents(row.price_max_cents),
    )
```

### `src/app/modules/catalog/application/usecases/utils/genre_slug.py` — **apagar**

Fica morto assim que `search()`/`list_genres()` deixam de resolver slug contra texto livre. Sem outro uso no repo além de `tests/test_catalog_show_search.py` (ver `quality.md` §5, testes existentes a reescrever).

### `src/app/modules/catalog/infrastructure/repositories/genre_repository.py` — novo

```python
from __future__ import annotations

from uuid import UUID
from sqlalchemy import select

from app.core.infrastructure.repositories import AggregateRepository
from app.modules.catalog.domain.aggregates import Genre

class GenreRepository(AggregateRepository[Genre]):
    model = Genre

    async def find_all_by_ids(self, genre_ids: list[UUID]) -> list[Genre]:
        if not genre_ids:
            return []

        result = await self._session.execute(
            select(Genre).where(Genre.id.in_(genre_ids), Genre.is_active.is_(True))
        )

        return list(result.scalars().all())
```

### `src/app/modules/catalog/infrastructure/repositories/show_repository.py` — editar (arquivo completo)

```python
from __future__ import annotations

from uuid import UUID
from typing import NamedTuple
from datetime import datetime

from sqlalchemy import distinct, func, select
from sqlalchemy.dialects.postgresql import aggregate_order_by

from app.modules.catalog.domain.aggregates import Genre, Session, Show
from app.core.infrastructure.repositories import AggregateRepository
from app.modules.catalog.domain.enumerations import SessionStatus, ShowStatus

class ShowCardRow(NamedTuple):
    id: UUID
    title: str
    synopsis: str
    image_url: str
    genre_id: UUID
    genre_name: str
    genre_icon: str
    upcoming_dates: list[datetime]
    price_min_cents: int
    price_max_cents: int

class ShowSearchPage(NamedTuple):
    rows: list[ShowCardRow]
    total: int

class ShowRepository(AggregateRepository[Show]):
    model = Show

    async def search_with_upcoming(self, *, floor: datetime, genre_id: UUID | None, page: int, size: int) -> ShowSearchPage:
        next_at = func.min(Session.starts_at).label("next_at")
        stmt = (
            select(
                Show.id,
                Show.title,
                Show.synopsis,
                Show.image_url,
                Genre.id.label("genre_id"),
                Genre.name.label("genre_name"),
                Genre.icon.label("genre_icon"),
                func.array_agg(
                    aggregate_order_by(distinct(Session.starts_at), Session.starts_at.asc())
                ).label("upcoming_dates"),
                func.min(Session.full_price_cents).label("price_min_cents"),
                func.max(Session.full_price_cents).label("price_max_cents"),
                next_at,
                func.count().over().label("total"),
            )
            .join(Session, Session.show_id == Show.id)
            .join(Genre, Genre.id == Show.genre_id)
            .where(*self._filters(floor, genre_id))
            .group_by(Show.id, Genre.id, Genre.name, Genre.icon)
            .order_by(next_at.asc(), Show.id.asc())
            .limit(size)
            .offset((page - 1) * size)
        )

        result = (await self._session.execute(stmt)).all()
        if not result:
            return ShowSearchPage(rows=[], total=await self._count_in_catalog(floor, genre_id))

        rows = [
            ShowCardRow(
                id=row.id,
                title=row.title,
                synopsis=row.synopsis,
                image_url=row.image_url,
                genre_id=row.genre_id,
                genre_name=row.genre_name,
                genre_icon=row.genre_icon,
                upcoming_dates=list(row.upcoming_dates),
                price_min_cents=row.price_min_cents,
                price_max_cents=row.price_max_cents,
            )
            for row in result
        ]

        return ShowSearchPage(rows=rows, total=int(result[0].total))

    async def list_genres_in_catalog(self, *, floor: datetime) -> list[Genre]:
        result = await self._session.execute(
            select(Genre)
            .join(Show, Show.genre_id == Genre.id)
            .join(Session, Session.show_id == Show.id)
            .where(*self._filters(floor, None), Genre.is_active.is_(True))
            .distinct()
            .order_by(Genre.name.asc())
        )

        return list(result.scalars().all())

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

    async def _count_in_catalog(self, floor: datetime, genre_id: UUID | None) -> int:
        grouped = (
            select(Show.id)
            .join(Session, Session.show_id == Show.id)
            .where(*self._filters(floor, genre_id))
            .group_by(Show.id)
            .subquery()
        )

        result = await self._session.execute(select(func.count()).select_from(grouped))
        return int(result.scalar_one())
```

### `src/app/modules/catalog/infrastructure/repositories/__init__.py` — editar

```python
from .genre_repository import GenreRepository
from .seat_counts_repository import SeatCounts, SeatCountsRepository
from .session_repository import SessionRepository
from .show_repository import ShowCardRow, ShowRepository, ShowSearchPage

__all__ = [
    "GenreRepository",
    "SeatCounts",
    "SeatCountsRepository",
    "SessionRepository",
    "ShowCardRow",
    "ShowRepository",
    "ShowSearchPage",
]
```

### `src/app/modules/catalog/api/routers/admin_catalog_router.py` — editar (arquivo completo)

```python
from __future__ import annotations

from uuid import UUID

from fastapi import APIRouter, Depends, Response
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.modules.catalog.application.schemas.request import GenreRequest, ShowRequest, SessionRequest
from app.modules.catalog.application.schemas.response import (
    AdminSessionResponse,
    AdminShowResponse,
    AdminShowSummaryResponse,
    GenreResponse,
)
from app.modules.catalog.application.usecases.genre_usecase import GenreUseCase
from app.modules.catalog.application.usecases.session_usecase import SessionUseCase
from app.modules.catalog.application.usecases.show_usecase import ShowUseCase
from app.modules.identity.dependencies import require_admin

router = APIRouter(
    prefix="/admin",
    tags=["Catalog (admin)"],
    dependencies=[Depends(require_admin)],
)

@router.get("/genres", response_model=list[GenreResponse])
async def list_all_genres(session: AsyncSession = Depends(get_db)) -> list[GenreResponse]:
    return await GenreUseCase(session).list_all()

@router.post("/genres", response_model=GenreResponse, status_code=201)
async def create_genre(body: GenreRequest, session: AsyncSession = Depends(get_db)) -> GenreResponse:
    return await GenreUseCase(session).create_genre(body)

@router.get("/shows", response_model=list[AdminShowSummaryResponse])
async def list_shows(session: AsyncSession = Depends(get_db)) -> list[AdminShowSummaryResponse]:
    return await ShowUseCase(session).list_shows()

@router.get("/shows/{show_id}", response_model=AdminShowResponse)
async def get_show(show_id: UUID, session: AsyncSession = Depends(get_db)) -> AdminShowResponse:
    return await ShowUseCase(session).get_show(show_id)

@router.post("/shows", response_model=AdminShowResponse, status_code=201)
async def create_show(
    body: ShowRequest, session: AsyncSession = Depends(get_db)
) -> AdminShowResponse:
    return await ShowUseCase(session).create_show(body)

@router.put("/shows/{show_id}", response_model=AdminShowResponse)
async def update_show(
    show_id: UUID, body: ShowRequest, session: AsyncSession = Depends(get_db)
) -> AdminShowResponse:
    return await ShowUseCase(session).update_show(show_id, body)

@router.post("/shows/{show_id}/publish", status_code=204)
async def publish_show(show_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await ShowUseCase(session).publish_show(show_id)
    return Response(status_code=204)

@router.post("/shows/{show_id}/unpublish", status_code=204)
async def unpublish_show(show_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await ShowUseCase(session).unpublish_show(show_id)
    return Response(status_code=204)

@router.delete("/shows/{show_id}", status_code=204)
async def delete_show(show_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await ShowUseCase(session).delete_show(show_id)
    return Response(status_code=204)

@router.post("/shows/{show_id}/sessions", response_model=AdminSessionResponse, status_code=201)
async def create_session(
    show_id: UUID, body: SessionRequest, session: AsyncSession = Depends(get_db)
) -> AdminSessionResponse:
    return await SessionUseCase(session).create_session(show_id, body)

@router.put("/sessions/{session_id}", response_model=AdminSessionResponse)
async def update_session(
    session_id: UUID, body: SessionRequest, session: AsyncSession = Depends(get_db)
) -> AdminSessionResponse:
    return await SessionUseCase(session).update_session(session_id, body)

@router.post("/sessions/{session_id}/cancel", status_code=202)
async def cancel_session(session_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await SessionUseCase(session).cancel_session(session_id)
    return Response(status_code=202)

@router.delete("/sessions/{session_id}", status_code=204)
async def delete_session(session_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await SessionUseCase(session).delete_session(session_id)
    return Response(status_code=204)
```

### `src/app/modules/catalog/api/routers/show_router.py` — editar (arquivo completo)

```python
from __future__ import annotations

from uuid import UUID
from datetime import date

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.modules.catalog.application.schemas.response import (
    GenreResponse,
    PagedShowsResponse,
    ShowDetailResponse,
)
from app.modules.catalog.application.usecases.show_usecase import ShowUseCase

router = APIRouter(tags=["Catalog"])

@router.get("/shows", response_model=PagedShowsResponse)
async def search_shows(
    from_date: date | None = Query(default=None),
    genre_id: UUID | None = Query(default=None),
    page: int = Query(default=1, ge=1),
    size: int = Query(default=12, ge=1, le=48),
    session: AsyncSession = Depends(get_db),
) -> PagedShowsResponse:
    return await ShowUseCase(session).search(
        from_date=from_date, genre_id=genre_id, page=page, size=size
    )

@router.get("/shows/{show_id}", response_model=ShowDetailResponse)
async def get_show(show_id: UUID, session: AsyncSession = Depends(get_db)) -> ShowDetailResponse:
    return await ShowUseCase(session).get_show_detail(show_id)

@router.get("/genres", response_model=list[GenreResponse])
async def list_genres(session: AsyncSession = Depends(get_db)) -> list[GenreResponse]:
    return await ShowUseCase(session).list_genres()
```

### `src/migrations/env.py` — editar (1 linha)

Adicionar junto do bloco de imports de modelo já existente (logo abaixo de `import app.modules.catalog.domain.aggregates.session`):

```python
import app.modules.catalog.domain.aggregates.genre  # noqa: F401,E402
```

### `src/migrations/versions/0003_catalog_genre.py` — novo

```python
"""catalog genre: tabela genres + migra shows.genre (texto livre) para shows.genre_id (FK)

Revision ID: 0003_catalog_genre
Revises: 0002_catalog_admin
Create Date: 2026-09-11

Cria a tabela `genres` e migra os valores hoje em `shows.genre` (texto livre)
para registros de `Genre`, agrupando variações de grafia (acento/maiúscula)
sob a mesma forma normalizada — a normalizada É o `Genre.name` final.

[bloqueio: decidir antes de rodar contra dado real] Não há decisão de produto
sobre qual ícone um gênero migrado automaticamente recebe (spec/logic só
cobrem criação manual pelo admin, onde o ícone é sempre escolhido). Usa um
placeholder fixo (`_MIGRATED_GENRE_ICON`) só para a coluna NOT NULL não
travar a migration. Ver quality.md / bloqueios.
"""

import re
import unicodedata
import uuid
from datetime import datetime, timezone
from typing import Sequence, Union

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

revision: str = "0003_catalog_genre"
down_revision: Union[str, None] = "0002_catalog_admin"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

_MIGRATED_GENRE_ICON = "drama-masks"  # [bloqueio: confirmar com o PO]


def _model_columns() -> list[sa.Column]:
    return [
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("is_active", sa.Boolean(), nullable=False),
    ]


def _normalize_genre_name(raw: str) -> str:
    # Duplicada de propósito: a migration não importa app.domain (precisa
    # ficar estável mesmo se Genre.normalize_genre_name mudar no futuro).
    text = unicodedata.normalize("NFKD", (raw or "").strip()).encode("ascii", "ignore").decode("ascii")
    return re.sub(r"\s+", " ", text).lower()


genres_table = sa.table(
    "genres",
    sa.column("id", postgresql.UUID(as_uuid=True)),
    sa.column("name", sa.String),
    sa.column("icon", sa.String),
    sa.column("created_at", sa.DateTime(timezone=True)),
    sa.column("updated_at", sa.DateTime(timezone=True)),
    sa.column("is_active", sa.Boolean),
)

shows_table = sa.table(
    "shows",
    sa.column("id", postgresql.UUID(as_uuid=True)),
    sa.column("genre", sa.String),
    sa.column("genre_id", postgresql.UUID(as_uuid=True)),
)


def _migrate_existing_genres() -> None:
    bind = op.get_bind()
    now = datetime.now(timezone.utc)

    rows = bind.execute(sa.select(shows_table.c.id, shows_table.c.genre)).fetchall()

    normalized_to_id: dict[str, uuid.UUID] = {}

    for show_id, raw_genre in rows:
        normalized = _normalize_genre_name(raw_genre)
        genre_id = normalized_to_id.get(normalized)

        if genre_id is None:
            genre_id = uuid.uuid4()
            normalized_to_id[normalized] = genre_id
            bind.execute(
                genres_table.insert().values(
                    id=genre_id,
                    name=normalized,
                    icon=_MIGRATED_GENRE_ICON,
                    created_at=now,
                    updated_at=now,
                    is_active=True,
                )
            )

        bind.execute(
            shows_table.update()
            .where(shows_table.c.id == show_id)
            .values(genre_id=genre_id)
        )


def upgrade() -> None:
    op.create_table(
        "genres",
        *_model_columns(),
        sa.Column("name", sa.String(length=80), nullable=False),
        sa.Column("icon", sa.String(length=80), nullable=False),
    )
    op.create_index(
        "uq_genres_name_active",
        "genres",
        ["name"],
        unique=True,
        postgresql_where=sa.text("is_active"),
    )

    op.add_column("shows", sa.Column("genre_id", postgresql.UUID(as_uuid=True), nullable=True))

    _migrate_existing_genres()

    op.alter_column("shows", "genre_id", nullable=False)
    op.create_foreign_key("fk_shows_genre_id_genres", "shows", "genres", ["genre_id"], ["id"])
    op.create_index("ix_shows_genre_id", "shows", ["genre_id"])
    op.drop_column("shows", "genre")


def downgrade() -> None:
    op.add_column("shows", sa.Column("genre", sa.String(length=80), nullable=True))

    bind = op.get_bind()
    genre_names = dict(bind.execute(sa.select(genres_table.c.id, genres_table.c.name)).fetchall())
    rows = bind.execute(sa.select(shows_table.c.id, shows_table.c.genre_id)).fetchall()

    for show_id, genre_id in rows:
        bind.execute(
            shows_table.update()
            .where(shows_table.c.id == show_id)
            .values(genre=genre_names.get(genre_id, ""))
        )

    op.alter_column("shows", "genre", nullable=False)

    op.drop_index("ix_shows_genre_id", table_name="shows")
    op.drop_constraint("fk_shows_genre_id_genres", "shows", type_="foreignkey")
    op.drop_column("shows", "genre_id")

    op.drop_index("uq_genres_name_active", table_name="genres")
    op.drop_table("genres")
```

## 3. Onde cada regra de negócio entra

| Regra (`logic.md`) | Arquivo · função | Como |
|---|---|---|
| Nome duplicado recusado, atômico sob concorrência | `genre_usecase.py` · `create_genre` | pré-check `exists_by` + `flush()` + catch `IntegrityError` → `ConflictError`, backstop no índice único parcial `uq_genres_name_active` (migration) |
| Ícone obrigatório na criação | `schemas/request.py` · `GenreRequest.icon` | `Field(min_length=1, ...)` — forma; coluna `NOT NULL` garante em domínio |
| Só admin cria gênero | `admin_catalog_router.py` | rota dentro do `APIRouter(dependencies=[Depends(require_admin)])` |
| Espetáculo sempre referencia gênero já cadastrado | `show.py` (`genre_id` FK `NOT NULL`) + `show_usecase.py` · `_require_genre` | garantia estrutural + garantia de existência antes de `Show.create`/`update` |
| Filtro público só com gênero de espetáculo publicado + sessão futura à venda | `show_repository.py` · `list_genres_in_catalog` | join `Genre`+`Show`+`Session`, mesmos filtros de sempre |
| Lista completa (form do espetáculo) ≠ lista do filtro público | `genre_usecase.py` · `list_all` vs. `show_usecase.py` · `list_genres` | duas queries diferentes, ver §6 |
| Migração automática agrupando variações de grafia | `0003_catalog_genre.py` · `_migrate_existing_genres` | normalização determinística, sem "grafia vencedora" |
| Sem edição/exclusão nesta entrega | Ausência deliberada | `Genre` só tem `create()`; nenhuma rota `PUT`/`DELETE` de gênero |

## 4. DevOps

Não aplicável — nenhuma variável de ambiente nova, nenhum segredo de CI.

## 5. Passo a passo TBD (Backend)

```
git checkout master && git pull && git checkout -b feat/<NN>-catalog-genre
# commit 1 — domínio
git add src/app/modules/catalog/domain && git commit -m "feat(catalog): modelar Genre e trocar Show.genre por genre_id"
# commit 2 — application
git add src/app/modules/catalog/application && git commit -m "feat(catalog): usecase e schemas de gênero"
# commit 3 — infrastructure
git add src/app/modules/catalog/infrastructure && git commit -m "feat(catalog): repositório de gênero e ajuste de show_repository"
# commit 4 — api + migration
git add src/app/modules/catalog/api src/migrations && git commit -m "feat(catalog): expor rotas de gênero e migration de dados"
```
Depois: `/team-ludens:tbd-pr` (senior-dev Modo 2 + `/code-review`) → push → PR
`Closes #<NN>` → merge (1 aprovação + CI verde).

## 6. Ordem entre as superfícies

Backend e QA (casos de domínio) começam juntos a partir do `logic.md`. Frontend
começa em paralelo contra o contrato-alvo de `integration.md` — mas o shape
de `genre` embutido em `AdminShow`/`ShowCard`/`ShowDetail` só fica 100%
estável depois que este documento for revisado (não é mudança pequena: troca
`genre: str` por `genre: GenreResponse` em quatro schemas de resposta).

## 7. Bloqueios em aberto

- **Ícone padrão de gênero migrado** — não há decisão de produto sobre qual
  ícone os gêneros migrados automaticamente (`0003_catalog_genre.py`) recebem.
  Não rodar `alembic upgrade head` contra dado real antes de confirmar com o
  PO.
- **Reconciliação com `api.ludens` PR #23** (`refactor/21-show-list-detail`,
  separa `GET /admin/shows` resumo de `GET /admin/shows/{id}` detalhe) — este
  documento já assume o resultado de #23 composto com a mudança de gênero.
  Se #23 ainda não tiver mergeado quando esta fatia for implementada, aplicar
  as duas mudanças juntas (não como dois merges sequenciais que se pisam).
- **Contrato de `catalog-show-search` fica desatualizado por esta feature**
  (fora do escopo deste documento, que só cobre `catalog-genre`): `GET /shows`
  troca `?genre=<slug>` por `?genre_id=<uuid>`; `ShowCardResponse.genre` deixa
  de ser `str` e vira objeto `GenreResponse`. Isso quebra o contrato hoje
  documentado em `docs.ludens/specs/catalog-show-search/integration.md` — meu
  parecer intencionalmente não editou aquele arquivo (isolamento pedido nesta
  sessão). **Alguém precisa atualizar `catalog-show-search/integration.md`
  antes ou junto desta feature ir a produção**, e o time de frontend
  (`web.ludens`) precisa saber que a vitrine pública também muda de contrato,
  não só a área admin.
