---
status: draft
spec: catalog-admin-management
surface: backend
created_at: 2026-09-03
updated_at: 2026-09-04
---

# Gestão de espetáculos e sessões (admin) — Backend

**Resumo:** módulo `catalog` — aggregates `Show` e `Session`, CRUD administrativo
(criar/editar/publicar/despublicar/excluir espetáculo; criar/editar/cancelar/
excluir sessão), a regra "sessão vendida não se apaga, só se cancela" e o evento
de domínio `SessionCancelled` que, pelo Outbox, aciona o reembolso em massa
(RF07 / RN02) e o aviso aos compradores.
**RF:** RF08 · **RN:** — (aciona RN02 via RF07; RN04 no cálculo de meia) ·
**Módulo backend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-admin-management/integration.md`
**Carregar antes:** skill `backend-architecture` (todos os `references/`),
`docs.ludens/backend/overview.md`.

**Depende de:** `identity-auth` mergeado — expõe `require_admin` em
`app.modules.identity.dependencies` (403 se `role != ADMIN`).
**É base de:** `catalog-show-search`, `catalog-session-detail`,
`booking-reservation`.

---

## 1. Arquivos (ordem de dependência)

`infrastructure/repositories/` vem antes de `application/` — os usecases
dependem dos repositórios, não o contrário. `repositories/__init__.py`
reexporta as três classes do pacote (um só import em vez de três).

| #   | Camada         | Caminho                                                                         | Novo/Editar                                               |
| --- | -------------- | ------------------------------------------------------------------------------- | --------------------------------------------------------- |
| 1   | core           | `src/app/core/domain/errors.py`                                                 | novo (compartilhado — reusar se `identity-auth` já criou) |
| 2   | core           | `src/app/core/shared/schema.py`                                                 | novo (compartilhado)                                      |
| 3   | domain         | `src/app/modules/catalog/**/__init__.py` (pacotes)                              | novo                                                      |
| 4   | domain         | `src/app/modules/catalog/domain/enumerations/show_status.py`                    | novo                                                      |
| 5   | domain         | `src/app/modules/catalog/domain/enumerations/session_status.py`                 | novo                                                      |
| 6   | domain         | `src/app/modules/catalog/domain/value_objects/money.py`                         | novo                                                      |
| 7   | domain         | `src/app/modules/catalog/domain/events/catalog_events.py`                       | novo                                                      |
| 8   | domain         | `src/app/modules/catalog/domain/aggregates/show.py`                             | novo                                                      |
| 9   | domain         | `src/app/modules/catalog/domain/aggregates/session.py`                          | novo                                                      |
| 10  | infrastructure | `src/app/modules/catalog/infrastructure/repositories/show_repository.py`        | novo                                                      |
| 11  | infrastructure | `src/app/modules/catalog/infrastructure/repositories/session_repository.py`     | novo                                                      |
| 12  | infrastructure | `src/app/modules/catalog/infrastructure/repositories/seat_counts_repository.py` | novo                                                      |
| 13  | infrastructure | `src/app/modules/catalog/infrastructure/repositories/__init__.py`               | novo                                                      |
| 14  | application    | `src/app/modules/catalog/application/schemas/request.py`                        | novo                                                      |
| 15  | application    | `src/app/modules/catalog/application/schemas/response.py`                       | novo                                                      |
| 16  | application    | `src/app/modules/catalog/application/views.py`                                  | novo                                                      |
| 17  | application    | `src/app/modules/catalog/application/usecases/show_admin_usecase.py`            | novo                                                      |
| 18  | application    | `src/app/modules/catalog/application/usecases/session_admin_usecase.py`         | novo                                                      |
| 19  | api            | `src/app/modules/catalog/api/routers/admin_catalog_router.py`                   | novo                                                      |
| 20  | api            | `src/app/modules/catalog/router.py`                                             | novo                                                      |
| 21  | api            | `src/app/main.py`                                                               | editar                                                    |
| 22  | migration      | `src/migrations/env.py`                                                         | editar                                                    |
| 23  | migration      | `src/migrations/versions/0002_catalog_admin.py`                                 | novo                                                      |
| 24  | config         | `pyproject.toml`                                                                | editar (ignore `B008` em routers)                         |

Não há variável de ambiente nova — a subseção **DevOps** não se aplica.

---

## 2. Código

### `src/app/core/domain/errors.py`

Ainda não existe no repo (o `core/shared/errors.py` atual só formata erro de
validação). É compartilhado com `identity-auth`; quem mergear primeiro cria,
o outro reusa.

```python
# src/app/core/domain/errors.py  — novo
from __future__ import annotations


class DomainError(Exception):
    """Violação de invariante de domínio. A camada de API traduz para HTTP.

    `status_code` padrão 422 (regra de negócio). Subclasses ou o parâmetro
    `status_code` cobrem 404 (não encontrado) e 409 (conflito de estado).
    """

    status_code: int = 422

    def __init__(self, message: str, *, status_code: int | None = None) -> None:
        super().__init__(message)
        self.message = message
        if status_code is not None:
            self.status_code = status_code


class ConflictError(DomainError):
    """Conflito de estado do recurso — traduzido para HTTP 409."""

    status_code = 409
```

### `src/app/core/shared/schema.py`

```python
# src/app/core/shared/schema.py  — novo
from __future__ import annotations

from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel


class CamelModel(BaseModel):
    """Base de schema de I/O da API.

    Serializa em camelCase (contrato do frontend) e mantém os nomes em
    snake_case no Python. `populate_by_name=True` aceita as duas grafias na
    entrada; `from_attributes=True` permite `model_validate(orm_obj)`.
    """

    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
        from_attributes=True,
    )
```

### `src/app/modules/catalog/**/__init__.py`

Todos vazios — só marcam o pacote Python. `infrastructure/repositories/__init__.py`
**não** entra aqui — tem conteúdo real (item 13). Criar um por diretório novo:

```python
# src/app/modules/catalog/__init__.py                                   — novo
# src/app/modules/catalog/domain/__init__.py                            — novo
# src/app/modules/catalog/domain/enumerations/__init__.py               — novo
# src/app/modules/catalog/domain/value_objects/__init__.py              — novo
# src/app/modules/catalog/domain/events/__init__.py                     — novo
# src/app/modules/catalog/domain/aggregates/__init__.py                 — novo
# src/app/modules/catalog/application/__init__.py                       — novo
# src/app/modules/catalog/application/schemas/__init__.py               — novo
# src/app/modules/catalog/application/usecases/__init__.py              — novo
# src/app/modules/catalog/infrastructure/__init__.py                    — novo
# src/app/modules/catalog/api/__init__.py                               — novo
# src/app/modules/catalog/api/routers/__init__.py                       — novo
# (arquivo vazio)
```

### `src/app/modules/catalog/domain/enumerations/show_status.py`

```python
# src/app/modules/catalog/domain/enumerations/show_status.py  — novo
from __future__ import annotations

import enum


class ShowStatus(str, enum.Enum):
    # "inativo" (excluído) não é um valor aqui — é is_active=False no Model.
    DRAFT = "draft"
    PUBLISHED = "published"
```

### `src/app/modules/catalog/domain/enumerations/session_status.py`

```python
# src/app/modules/catalog/domain/enumerations/session_status.py  — novo
from __future__ import annotations

import enum


class SessionStatus(str, enum.Enum):
    # "encerrada" é derivada de starts_at (não é um valor persistido);
    # "inativa" (excluída) é is_active=False no Model.
    ON_SALE = "on_sale"
    CANCELLED = "cancelled"
```

### `src/app/modules/catalog/domain/value_objects/money.py`

```python
# src/app/modules/catalog/domain/value_objects/money.py  — novo
from __future__ import annotations

from dataclasses import dataclass
from decimal import ROUND_HALF_UP, Decimal

from app.core.domain.errors import DomainError


@dataclass(frozen=True)
class Money:
    """Valor monetário em centavos — evita float em preço."""

    cents: int

    def __post_init__(self) -> None:
        if not isinstance(self.cents, int):
            raise DomainError("valor monetário deve ser inteiro de centavos")
        if self.cents < 0:
            raise DomainError("valor monetário não pode ser negativo")

    @classmethod
    def from_reais(cls, value: float | str | Decimal) -> "Money":
        cents = (Decimal(str(value)) * 100).quantize(Decimal("1"), rounding=ROUND_HALF_UP)
        return cls(cents=int(cents))

    @classmethod
    def zero(cls) -> "Money":
        return cls(cents=0)

    @property
    def reais(self) -> float:
        return self.cents / 100

    def half(self) -> "Money":
        # Meia-entrada = 50% do inteira, truncado ao centavo (RN04). Derivado,
        # nunca digitado pelo admin.
        return Money(cents=self.cents // 2)
```

### `src/app/modules/catalog/domain/events/catalog_events.py`

```python
# src/app/modules/catalog/domain/events/catalog_events.py  — novo
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID

from app.core.domain.events import DomainEvent


@dataclass(frozen=True)
class ShowCreated(DomainEvent):
    id: UUID = field(kw_only=True)
    title: str = field(kw_only=True)
    synopsis: str = field(kw_only=True)
    image_url: str = field(kw_only=True)
    genre: str = field(kw_only=True)


@dataclass(frozen=True)
class ShowUpdated(DomainEvent):
    id: UUID = field(kw_only=True)
    title: str = field(kw_only=True)
    synopsis: str = field(kw_only=True)
    image_url: str = field(kw_only=True)
    genre: str = field(kw_only=True)


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
    # Consumido por `payment` (reembolso em massa — RF07 / RN02 a partir do
    # cancelamento) e por `notification` (aviso aos compradores).
    id: UUID = field(kw_only=True)
    show_id: UUID = field(kw_only=True)
    starts_at: datetime = field(kw_only=True)
    cancelled_at: datetime = field(kw_only=True)


@dataclass(frozen=True)
class SessionDeactivated(DomainEvent):
    id: UUID = field(kw_only=True)
```

### `src/app/modules/catalog/domain/aggregates/show.py`

```python
# src/app/modules/catalog/domain/aggregates/show.py  — novo
from __future__ import annotations

from uuid import uuid4

from sqlalchemy import Enum as SAEnum, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain.aggregate import AggregateRoot
from app.core.domain.events import DomainEvent
from app.core.domain.model import Model
from app.modules.catalog.domain.enumerations.show_status import ShowStatus
from app.modules.catalog.domain.events.catalog_events import (
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
    genre: Mapped[str] = mapped_column(String(80), nullable=False)
    status: Mapped[ShowStatus] = mapped_column(
        SAEnum(ShowStatus, native_enum=False, length=20),
        nullable=False,
        default=ShowStatus.DRAFT,
    )

    @classmethod
    def create(cls, *, title: str, synopsis: str, image_url: str, genre: str) -> "Show":
        show = cls()
        show.id = uuid4()
        show.raise_event(
            lambda v: ShowCreated(
                version=v, id=show.id, title=title, synopsis=synopsis,
                image_url=image_url, genre=genre,
            )
        )
        return show

    def update(self, *, title: str, synopsis: str, image_url: str, genre: str) -> None:
        self.raise_event(
            lambda v: ShowUpdated(
                version=v, id=self.id, title=title, synopsis=synopsis,
                image_url=image_url, genre=genre,
            )
        )

    def publish(self) -> None:
        # Idempotente: publicar um espetáculo já publicado não muda nada.
        if self.status is ShowStatus.PUBLISHED:
            return
        self.raise_event(lambda v: ShowPublished(version=v, id=self.id))

    def unpublish(self) -> None:
        if self.status is ShowStatus.DRAFT:
            return
        self.raise_event(lambda v: ShowUnpublished(version=v, id=self.id))

    def deactivate(self) -> None:
        # Soft delete. O usecase garante que não há sessão com venda antes.
        self.raise_event(lambda v: ShowDeactivated(version=v, id=self.id))

    def _apply(self, event: DomainEvent) -> None:
        handler = getattr(self, f"_when_{type(event).__name__}", None)
        if handler is not None:
            handler(event)

    def _when_ShowCreated(self, e: ShowCreated) -> None:
        self.title = e.title
        self.synopsis = e.synopsis
        self.image_url = e.image_url
        self.genre = e.genre
        self.status = ShowStatus.DRAFT
        self.is_active = True

    def _when_ShowUpdated(self, e: ShowUpdated) -> None:
        self.title = e.title
        self.synopsis = e.synopsis
        self.image_url = e.image_url
        self.genre = e.genre

    def _when_ShowPublished(self, _event: ShowPublished) -> None:
        self.status = ShowStatus.PUBLISHED

    def _when_ShowUnpublished(self, _event: ShowUnpublished) -> None:
        self.status = ShowStatus.DRAFT

    def _when_ShowDeactivated(self, _event: ShowDeactivated) -> None:
        self.is_active = False
```

### `src/app/modules/catalog/domain/aggregates/session.py`

```python
# src/app/modules/catalog/domain/aggregates/session.py  — novo
from __future__ import annotations

from datetime import datetime, timezone
from uuid import UUID, uuid4

from sqlalchemy import DateTime, Enum as SAEnum, ForeignKey, Integer, String, Uuid
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain.aggregate import AggregateRoot
from app.core.domain.errors import ConflictError, DomainError
from app.core.domain.events import DomainEvent
from app.core.domain.model import Model
from app.modules.catalog.domain.enumerations.session_status import SessionStatus
from app.modules.catalog.domain.events.catalog_events import (
    SessionCancelled,
    SessionCreated,
    SessionDeactivated,
    SessionUpdated,
)
from app.modules.catalog.domain.value_objects.money import Money


class Session(AggregateRoot, Model):
    __tablename__ = "sessions"

    show_id: Mapped[UUID] = mapped_column(
        Uuid(), ForeignKey("shows.id"), nullable=False, index=True
    )
    starts_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    venue: Mapped[str] = mapped_column(String(200), nullable=False)
    capacity: Mapped[int] = mapped_column(Integer, nullable=False)
    full_price_cents: Mapped[int] = mapped_column(Integer, nullable=False)
    status: Mapped[SessionStatus] = mapped_column(
        SAEnum(SessionStatus, native_enum=False, length=20),
        nullable=False,
        default=SessionStatus.ON_SALE,
    )

    @property
    def full_price(self) -> Money:
        return Money(cents=self.full_price_cents)

    @property
    def half_price(self) -> Money:
        return self.full_price.half()

    def is_on_sale(self, now: datetime) -> bool:
        # A parte "espetáculo publicado" é resolvida na leitura/usecase
        # (logic.md §4) — o aggregate só conhece o próprio estado.
        return self.status is SessionStatus.ON_SALE and self.starts_at > now

    @classmethod
    def create(
        cls,
        *,
        show_id: UUID,
        starts_at: datetime,
        venue: str,
        capacity: int,
        full_price: Money,
        now: datetime,
    ) -> "Session":
        if starts_at <= now:
            raise DomainError("A data da sessão deve ser futura.")
        if capacity <= 0:
            raise DomainError("A capacidade deve ser maior que zero.")
        session = cls()
        session.id = uuid4()
        session.raise_event(
            lambda v: SessionCreated(
                version=v, id=session.id, show_id=show_id, starts_at=starts_at,
                venue=venue, capacity=capacity, full_price_cents=full_price.cents,
            )
        )
        return session

    def update(
        self,
        *,
        starts_at: datetime,
        venue: str,
        capacity: int,
        full_price: Money,
        committed: int,
        now: datetime,
    ) -> None:
        # `committed` = confirmados + reservas abertas não vencidas (contado
        # pelo usecase). Reduzir capacidade abaixo disso é bloqueado.
        if self.status is SessionStatus.CANCELLED:
            raise ConflictError("Não é possível editar uma sessão cancelada.")
        if starts_at <= now:
            raise DomainError("A data da sessão deve ser futura.")
        if capacity < committed:
            raise ConflictError("Já há ingressos comprometidos nesta sessão.")
        self.raise_event(
            lambda v: SessionUpdated(
                version=v, id=self.id, starts_at=starts_at, venue=venue,
                capacity=capacity, full_price_cents=full_price.cents,
            )
        )

    def cancel(self) -> None:
        # Sessão à venda → cancelada. O SessionCancelled gravado na mesma
        # transação dispara o reembolso em massa (RF07) e o aviso.
        if self.status is SessionStatus.CANCELLED:
            raise ConflictError("A sessão já está cancelada.")
        self.raise_event(
            lambda v: SessionCancelled(
                version=v, id=self.id, show_id=self.show_id,
                starts_at=self.starts_at, cancelled_at=datetime.now(timezone.utc),
            )
        )

    def deactivate(self, *, tickets_sold: int) -> None:
        # Regra central RF08: sessão com ingresso vendido NÃO se apaga.
        if tickets_sold > 0:
            raise ConflictError("Cancele a sessão em vez de excluir.")
        self.raise_event(lambda v: SessionDeactivated(version=v, id=self.id))

    def _apply(self, event: DomainEvent) -> None:
        handler = getattr(self, f"_when_{type(event).__name__}", None)
        if handler is not None:
            handler(event)

    def _when_SessionCreated(self, e: SessionCreated) -> None:
        self.show_id = e.show_id
        self.starts_at = e.starts_at
        self.venue = e.venue
        self.capacity = e.capacity
        self.full_price_cents = e.full_price_cents
        self.status = SessionStatus.ON_SALE
        self.is_active = True

    def _when_SessionUpdated(self, e: SessionUpdated) -> None:
        self.starts_at = e.starts_at
        self.venue = e.venue
        self.capacity = e.capacity
        self.full_price_cents = e.full_price_cents

    def _when_SessionCancelled(self, _event: SessionCancelled) -> None:
        self.status = SessionStatus.CANCELLED

    def _when_SessionDeactivated(self, _event: SessionDeactivated) -> None:
        self.is_active = False
```

### `src/app/modules/catalog/infrastructure/repositories/show_repository.py`

```python
# src/app/modules/catalog/infrastructure/repositories/show_repository.py  — novo
from __future__ import annotations

from app.core.infrastructure.repositories.repository import AggregateRepository
from app.modules.catalog.domain.aggregates.show import Show


class ShowRepository(AggregateRepository[Show]):
    model = Show
```

### `src/app/modules/catalog/infrastructure/repositories/session_repository.py`

```python
# src/app/modules/catalog/infrastructure/repositories/session_repository.py  — novo
from __future__ import annotations

from uuid import UUID

from sqlalchemy import select

from app.core.infrastructure.repositories.repository import AggregateRepository
from app.modules.catalog.domain.aggregates.session import Session


class SessionRepository(AggregateRepository[Session]):
    model = Session

    async def find_all_for_shows(self, show_ids: list[UUID]) -> list[Session]:
        if not show_ids:
            return []
        result = await self._session.execute(
            select(Session)
            .where(Session.show_id.in_(show_ids), Session.is_active.is_(True))
            .order_by(Session.starts_at.asc())
        )
        return list(result.scalars().all())
```

### `src/app/modules/catalog/infrastructure/repositories/seat_counts_repository.py`

```python
# src/app/modules/catalog/infrastructure/repositories/seat_counts_repository.py  — novo
from __future__ import annotations

import logging
from typing import NamedTuple
from uuid import UUID

from sqlalchemy import bindparam, text
from sqlalchemy.exc import OperationalError, ProgrammingError
from sqlalchemy.ext.asyncio import AsyncSession

logger = logging.getLogger(__name__)


class SeatCounts(NamedTuple):
    tickets_sold: int
    reserved_open: int


# As tabelas `tickets` e `reservations` pertencem ao módulo `booking`. Enquanto
# `booking` não é mergeado elas não existem — nesse caso a contagem é 0 (aviso
# no log, sem silenciar). Ver "Bloqueios em aberto". Os nomes de coluna/status
# seguem a spec de `booking-reservation` / `booking-ticket-issuance`; revisar ao
# integrar.
_SOLD_SQL = text(
    """
    SELECT session_id, COUNT(*) AS total
    FROM tickets
    WHERE session_id IN :ids AND is_active = true AND status = 'valid'
    GROUP BY session_id
    """
).bindparams(bindparam("ids", expanding=True))

_OPEN_SQL = text(
    """
    SELECT session_id, COALESCE(SUM(quantity), 0) AS total
    FROM reservations
    WHERE session_id IN :ids AND is_active = true
      AND status = 'open' AND expires_at > now()
    GROUP BY session_id
    """
).bindparams(bindparam("ids", expanding=True))


class SeatCountsRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def for_sessions(self, session_ids: list[UUID]) -> dict[UUID, SeatCounts]:
        base: dict[UUID, SeatCounts] = {sid: SeatCounts(0, 0) for sid in session_ids}
        if not session_ids:
            return base

        ids = [str(sid) for sid in session_ids]
        try:
            # SAVEPOINT: se as tabelas de `booking` ainda não existem, o erro
            # rola de volta só este bloco e a transação da request segue.
            async with self._session.begin_nested():
                sold_rows = (await self._session.execute(_SOLD_SQL, {"ids": ids})).all()
                open_rows = (await self._session.execute(_OPEN_SQL, {"ids": ids})).all()
        except (ProgrammingError, OperationalError) as exc:
            logger.warning(
                "catalog: contagem de assentos indisponível (%s) — assumindo 0",
                exc.__class__.__name__,
            )
            return base

        sold = {UUID(str(row[0])): int(row[1]) for row in sold_rows}
        held = {UUID(str(row[0])): int(row[1]) for row in open_rows}
        return {sid: SeatCounts(sold.get(sid, 0), held.get(sid, 0)) for sid in session_ids}
```

### `src/app/modules/catalog/infrastructure/repositories/__init__.py`

```python
# src/app/modules/catalog/infrastructure/repositories/__init__.py  — novo
from __future__ import annotations

from app.modules.catalog.infrastructure.repositories.seat_counts_repository import (
    SeatCounts,
    SeatCountsRepository,
)
from app.modules.catalog.infrastructure.repositories.session_repository import SessionRepository
from app.modules.catalog.infrastructure.repositories.show_repository import ShowRepository

__all__ = ["SeatCounts", "SeatCountsRepository", "SessionRepository", "ShowRepository"]
```

### `src/app/modules/catalog/application/schemas/request.py`

```python
# src/app/modules/catalog/application/schemas/request.py  — novo
from __future__ import annotations

from datetime import datetime

from pydantic import Field, field_validator

from app.core.shared.schema import CamelModel


class CreateShowRequest(CamelModel):
    title: str = Field(min_length=1, max_length=200)
    synopsis: str = Field(min_length=1, max_length=5000)
    image_url: str = Field(min_length=1, max_length=2048)
    genre: str = Field(min_length=1, max_length=80)


class UpdateShowRequest(CamelModel):
    title: str | None = Field(default=None, min_length=1, max_length=200)
    synopsis: str | None = Field(default=None, min_length=1, max_length=5000)
    image_url: str | None = Field(default=None, min_length=1, max_length=2048)
    genre: str | None = Field(default=None, min_length=1, max_length=80)


class CreateSessionRequest(CamelModel):
    starts_at: datetime
    venue: str = Field(min_length=1, max_length=200)
    capacity: int = Field(gt=0, le=100_000)
    full_price: float = Field(gt=0)

    @field_validator("starts_at")
    @classmethod
    def _tz_aware(cls, value: datetime) -> datetime:
        # A regra "futura" é do domínio (precisa de `now`); aqui só a forma.
        if value.tzinfo is None:
            raise ValueError("informe a data com fuso horário (ISO 8601 com offset)")
        return value


class UpdateSessionRequest(CamelModel):
    starts_at: datetime | None = None
    venue: str | None = Field(default=None, min_length=1, max_length=200)
    capacity: int | None = Field(default=None, gt=0, le=100_000)
    full_price: float | None = Field(default=None, gt=0)

    @field_validator("starts_at")
    @classmethod
    def _tz_aware(cls, value: datetime | None) -> datetime | None:
        if value is not None and value.tzinfo is None:
            raise ValueError("informe a data com fuso horário (ISO 8601 com offset)")
        return value
```

### `src/app/modules/catalog/application/schemas/response.py`

```python
# src/app/modules/catalog/application/schemas/response.py  — novo
from __future__ import annotations

from datetime import datetime
from typing import Literal
from uuid import UUID

from app.core.shared.schema import CamelModel


class AdminSessionResponse(CamelModel):
    id: UUID
    show_id: UUID
    starts_at: datetime
    venue: str
    capacity: int
    full_price: float
    half_price: float
    # Derivado: "closed" quando starts_at já passou; "cancelled" quando
    # cancelada; senão "on_sale".
    status: Literal["on_sale", "closed", "cancelled"]
    tickets_sold: int
    reserved_open: int
    can_delete: bool


class AdminShowResponse(CamelModel):
    id: UUID
    title: str
    synopsis: str
    image_url: str
    genre: str
    status: Literal["draft", "published"]
    sessions: list[AdminSessionResponse]
```

### `src/app/modules/catalog/application/views.py`

```python
# src/app/modules/catalog/application/views.py  — novo
from __future__ import annotations

from datetime import datetime
from uuid import UUID

from app.modules.catalog.application.schemas.response import (
    AdminSessionResponse,
    AdminShowResponse,
)
from app.modules.catalog.domain.aggregates.session import Session
from app.modules.catalog.domain.aggregates.show import Show
from app.modules.catalog.domain.enumerations.session_status import SessionStatus
from app.modules.catalog.infrastructure.repositories import SeatCounts


def _session_status(session: Session, now: datetime) -> str:
    if session.status is SessionStatus.CANCELLED:
        return "cancelled"
    if session.starts_at <= now:
        return "closed"
    return "on_sale"


def session_view(session: Session, counts: SeatCounts, now: datetime) -> AdminSessionResponse:
    return AdminSessionResponse(
        id=session.id,
        show_id=session.show_id,
        starts_at=session.starts_at,
        venue=session.venue,
        capacity=session.capacity,
        full_price=session.full_price.reais,
        half_price=session.half_price.reais,
        status=_session_status(session, now),
        tickets_sold=counts.tickets_sold,
        reserved_open=counts.reserved_open,
        can_delete=counts.tickets_sold == 0,
    )


def show_view(
    show: Show,
    sessions: list[Session],
    counts_map: dict[UUID, SeatCounts],
    now: datetime,
) -> AdminShowResponse:
    return AdminShowResponse(
        id=show.id,
        title=show.title,
        synopsis=show.synopsis,
        image_url=show.image_url,
        genre=show.genre,
        status=show.status.value,
        sessions=[
            session_view(s, counts_map.get(s.id, SeatCounts(0, 0)), now) for s in sessions
        ],
    )
```

### `src/app/modules/catalog/application/usecases/show_admin_usecase.py`

```python
# src/app/modules/catalog/application/usecases/show_admin_usecase.py  — novo
from __future__ import annotations

from datetime import datetime, timezone
from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain.errors import ConflictError, DomainError
from app.modules.catalog.application.schemas.request import (
    CreateShowRequest,
    UpdateShowRequest,
)
from app.modules.catalog.application.schemas.response import AdminShowResponse
from app.modules.catalog.application.views import show_view
from app.modules.catalog.domain.aggregates.show import Show
from app.modules.catalog.infrastructure.repositories import (
    SeatCounts,
    SeatCountsRepository,
    SessionRepository,
    ShowRepository,
)


class ShowAdminUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._show_repo = ShowRepository(session)
        self._session_repo = SessionRepository(session)
        self._seat_counts_repo = SeatCountsRepository(session)

    async def list_shows(self) -> list[AdminShowResponse]:
        # Inclui rascunhos (find_all filtra só is_active, não status).
        shows = await self._show_repo.find_all(order_by=["-created_at"])
        sessions = await self._session_repo.find_all_for_shows([s.id for s in shows])
        counts = await self._seat_counts_repo.for_sessions([s.id for s in sessions])
        now = datetime.now(timezone.utc)
        grouped: dict[UUID, list[object]] = {}
        for sess in sessions:
            grouped.setdefault(sess.show_id, []).append(sess)
        return [show_view(show, grouped.get(show.id, []), counts, now) for show in shows]

    async def create_show(self, req: CreateShowRequest) -> AdminShowResponse:
        show = Show.create(
            title=req.title,
            synopsis=req.synopsis,
            image_url=req.image_url,
            genre=req.genre,
        )
        await self._show_repo.save(show)
        return show_view(show, [], {}, datetime.now(timezone.utc))

    async def update_show(self, show_id: UUID, req: UpdateShowRequest) -> AdminShowResponse:
        show = await self._require_show(show_id)
        data = req.model_dump(exclude_unset=True)
        show.update(
            title=data.get("title", show.title),
            synopsis=data.get("synopsis", show.synopsis),
            image_url=data.get("image_url", show.image_url),
            genre=data.get("genre", show.genre),
        )
        await self._show_repo.save(show)
        return await self._view_for(show)

    async def publish_show(self, show_id: UUID) -> None:
        show = await self._require_show(show_id)
        show.publish()
        await self._show_repo.save(show)

    async def unpublish_show(self, show_id: UUID) -> None:
        show = await self._require_show(show_id)
        show.unpublish()
        await self._show_repo.save(show)

    async def delete_show(self, show_id: UUID) -> None:
        show = await self._require_show(show_id)
        sessions = await self._session_repo.find_all_for_shows([show.id])
        counts = await self._seat_counts_repo.for_sessions([s.id for s in sessions])
        if any(counts.get(s.id, SeatCounts(0, 0)).tickets_sold > 0 for s in sessions):
            raise ConflictError(
                "Cancele as sessões com ingressos vendidos antes de excluir o espetáculo."
            )
        show.deactivate()
        await self._show_repo.save(show)

    async def _require_show(self, show_id: UUID) -> Show:
        show = await self._show_repo.find_by_id(show_id)
        if show is None:
            raise DomainError("Espetáculo não encontrado.", status_code=404)
        return show

    async def _view_for(self, show: Show) -> AdminShowResponse:
        sessions = await self._session_repo.find_all_for_shows([show.id])
        counts = await self._seat_counts_repo.for_sessions([s.id for s in sessions])
        return show_view(show, sessions, counts, datetime.now(timezone.utc))
```

### `src/app/modules/catalog/application/usecases/session_admin_usecase.py`

```python
# src/app/modules/catalog/application/usecases/session_admin_usecase.py  — novo
from __future__ import annotations

from datetime import datetime, timezone
from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain.errors import DomainError
from app.modules.catalog.application.schemas.request import (
    CreateSessionRequest,
    UpdateSessionRequest,
)
from app.modules.catalog.application.schemas.response import AdminSessionResponse
from app.modules.catalog.application.views import session_view
from app.modules.catalog.domain.aggregates.session import Session
from app.modules.catalog.domain.value_objects.money import Money
from app.modules.catalog.infrastructure.repositories import (
    SeatCounts,
    SeatCountsRepository,
    SessionRepository,
    ShowRepository,
)


class SessionAdminUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._show_repo = ShowRepository(session)
        self._session_repo = SessionRepository(session)
        self._seat_counts_repo = SeatCountsRepository(session)

    async def create_session(
        self, show_id: UUID, req: CreateSessionRequest
    ) -> AdminSessionResponse:
        show = await self._show_repo.find_by_id(show_id)
        if show is None:
            raise DomainError("Espetáculo não encontrado.", status_code=404)
        now = datetime.now(timezone.utc)
        session = Session.create(
            show_id=show.id,
            starts_at=req.starts_at,
            venue=req.venue,
            capacity=req.capacity,
            full_price=Money.from_reais(req.full_price),
            now=now,
        )
        await self._session_repo.save(session)
        return session_view(session, SeatCounts(0, 0), now)

    async def update_session(
        self, session_id: UUID, req: UpdateSessionRequest
    ) -> AdminSessionResponse:
        # Trava a linha da sessão antes de recontar/validar capacidade — o
        # mesmo mecanismo de RN05 que `booking` usa para reservar.
        session = await self._lock(session_id)
        counts = (await self._seat_counts_repo.for_sessions([session.id]))[session.id]
        now = datetime.now(timezone.utc)
        data = req.model_dump(exclude_unset=True)
        session.update(
            starts_at=data.get("starts_at", session.starts_at),
            venue=data.get("venue", session.venue),
            capacity=data.get("capacity", session.capacity),
            full_price=(
                Money.from_reais(data["full_price"])
                if "full_price" in data
                else session.full_price
            ),
            committed=counts.tickets_sold + counts.reserved_open,
            now=now,
        )
        await self._session_repo.save(session)
        return session_view(session, counts, now)

    async def cancel_session(self, session_id: UUID) -> None:
        session = await self._lock(session_id)
        session.cancel()
        await self._session_repo.save(session)

    async def delete_session(self, session_id: UUID) -> None:
        session = await self._lock(session_id)
        counts = (await self._seat_counts_repo.for_sessions([session.id]))[session.id]
        session.deactivate(tickets_sold=counts.tickets_sold)
        await self._session_repo.save(session)

    async def _lock(self, session_id: UUID) -> Session:
        session = await self._session_repo.find_by_id_for_update(session_id)
        if session is None:
            raise DomainError("Sessão não encontrada.", status_code=404)
        return session
```

### `src/app/modules/catalog/api/routers/admin_catalog_router.py`

```python
# src/app/modules/catalog/api/routers/admin_catalog_router.py  — novo
from __future__ import annotations

from uuid import UUID

from fastapi import APIRouter, Depends, Response
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.modules.catalog.application.schemas.request import (
    CreateSessionRequest,
    CreateShowRequest,
    UpdateSessionRequest,
    UpdateShowRequest,
)
from app.modules.catalog.application.schemas.response import (
    AdminSessionResponse,
    AdminShowResponse,
)
from app.modules.catalog.application.usecases.session_admin_usecase import SessionAdminUseCase
from app.modules.catalog.application.usecases.show_admin_usecase import ShowAdminUseCase
from app.modules.identity.dependencies import require_admin

# require_admin em TODAS as rotas do router (403 se role != ADMIN).
router = APIRouter(
    prefix="/admin",
    tags=["Catalog (admin)"],
    dependencies=[Depends(require_admin)],
)


@router.get("/shows", response_model=list[AdminShowResponse])
async def list_shows(session: AsyncSession = Depends(get_db)) -> list[AdminShowResponse]:
    return await ShowAdminUseCase(session).list_shows()


@router.post("/shows", response_model=AdminShowResponse, status_code=201)
async def create_show(
    body: CreateShowRequest, session: AsyncSession = Depends(get_db)
) -> AdminShowResponse:
    return await ShowAdminUseCase(session).create_show(body)


@router.patch("/shows/{show_id}", response_model=AdminShowResponse)
async def update_show(
    show_id: UUID, body: UpdateShowRequest, session: AsyncSession = Depends(get_db)
) -> AdminShowResponse:
    return await ShowAdminUseCase(session).update_show(show_id, body)


@router.post("/shows/{show_id}/publish", status_code=204)
async def publish_show(show_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await ShowAdminUseCase(session).publish_show(show_id)
    return Response(status_code=204)


@router.post("/shows/{show_id}/unpublish", status_code=204)
async def unpublish_show(show_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await ShowAdminUseCase(session).unpublish_show(show_id)
    return Response(status_code=204)


@router.delete("/shows/{show_id}", status_code=204)
async def delete_show(show_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await ShowAdminUseCase(session).delete_show(show_id)
    return Response(status_code=204)


@router.post("/shows/{show_id}/sessions", response_model=AdminSessionResponse, status_code=201)
async def create_session(
    show_id: UUID, body: CreateSessionRequest, session: AsyncSession = Depends(get_db)
) -> AdminSessionResponse:
    return await SessionAdminUseCase(session).create_session(show_id, body)


@router.patch("/sessions/{session_id}", response_model=AdminSessionResponse)
async def update_session(
    session_id: UUID, body: UpdateSessionRequest, session: AsyncSession = Depends(get_db)
) -> AdminSessionResponse:
    return await SessionAdminUseCase(session).update_session(session_id, body)


@router.post("/sessions/{session_id}/cancel", status_code=202)
async def cancel_session(session_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await SessionAdminUseCase(session).cancel_session(session_id)
    return Response(status_code=202)


@router.delete("/sessions/{session_id}", status_code=204)
async def delete_session(session_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await SessionAdminUseCase(session).delete_session(session_id)
    return Response(status_code=204)
```

### `src/app/modules/catalog/router.py`

```python
# src/app/modules/catalog/router.py  — novo
from fastapi import APIRouter

from app.modules.catalog.api.routers.admin_catalog_router import router as admin_catalog_router

router = APIRouter()
router.include_router(admin_catalog_router)
```

### `src/app/main.py`

Editar: importar o router de `catalog`, registrá-lo e adicionar o handler de
`DomainError` (traduz erro de domínio para HTTP). Arquivo completo já editado:

```python
# src/app/main.py  — editar
import asyncio
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse

from app.config import settings
from app.core.domain.errors import DomainError
from app.core.shared.errors import format_validation_errors
from app.core.shared.health import seconds_since_beat
from app.modules.catalog.router import router as catalog_router
from app.outbox.relay import run as run_outbox_relay

logger = logging.getLogger(__name__)

BACKGROUND_TASK_MAX_AGE_SECONDS = {"outbox_relay": 20}

@asynccontextmanager
async def lifespan(app: FastAPI):
    tasks = [
        asyncio.create_task(run_outbox_relay()),
    ]

    yield

    for task in tasks:
        task.cancel()

app = FastAPI(title="Ludens API", version="0.1.0", lifespan=lifespan)

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content={"detail": format_validation_errors(exc.errors())},
    )

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError):
    # Violação de invariante de domínio → status HTTP declarado no erro
    # (422 regra de negócio, 409 conflito, 404 não encontrado).
    return JSONResponse(status_code=exc.status_code, content={"detail": exc.message})

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(catalog_router)

@app.get("/health")
async def health():
    stale = [
        name
        for name, max_age in BACKGROUND_TASK_MAX_AGE_SECONDS.items()
        if (age := seconds_since_beat(name)) is not None and age > max_age
    ]

    if stale:
        return JSONResponse(status_code=503, content={"status": "unhealthy", "stale": stale})

    return {"status": "ok", "environment": settings.environment}
```

### `src/migrations/env.py`

Editar: acrescentar o import dos aggregates de `catalog` para o SQLAlchemy
registrar as tabelas no metadata (autogenerate e checagem). Trecho novo, logo
abaixo do import de `app.outbox.models`:

```python
# src/migrations/env.py  — editar (adicionar após "import app.outbox.models")
# Importar todos os models para o SQLAlchemy registrar as tabelas no metadata.
import app.outbox.models  # noqa: F401,E402

# catalog-admin-management:
import app.modules.catalog.domain.aggregates.show  # noqa: F401,E402
import app.modules.catalog.domain.aggregates.session  # noqa: F401,E402
```

### `src/migrations/versions/0002_catalog_admin.py`

```python
# src/migrations/versions/0002_catalog_admin.py  — novo
"""catalog admin: tabelas shows e sessions

Revision ID: 0002_catalog_admin
Revises: 0001_identity_auth
Create Date: 2026-09-03
"""
from typing import Sequence, Union

import sqlalchemy as sa
from alembic import op

revision: str = "0002_catalog_admin"
# Ajuste conforme a ordem de merge: se `identity-auth` ainda não fixou esse id,
# use o id real da migration dela aqui; se `catalog` entrar primeiro, use None.
down_revision: Union[str, None] = "0001_identity_auth"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

_SHOW_STATUS = sa.Enum(
    "draft", "published", name="show_status", native_enum=False, length=20
)
_SESSION_STATUS = sa.Enum(
    "on_sale", "cancelled", name="session_status", native_enum=False, length=20
)


def upgrade() -> None:
    op.create_table(
        "shows",
        sa.Column("id", sa.Uuid(), primary_key=True),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            server_default=sa.func.now(),
            nullable=False,
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            server_default=sa.func.now(),
            nullable=False,
        ),
        sa.Column("is_active", sa.Boolean(), server_default=sa.text("true"), nullable=False),
        sa.Column("title", sa.String(length=200), nullable=False),
        sa.Column("synopsis", sa.String(length=5000), nullable=False),
        sa.Column("image_url", sa.String(length=2048), nullable=False),
        sa.Column("genre", sa.String(length=80), nullable=False),
        sa.Column("status", _SHOW_STATUS, nullable=False),
    )
    op.create_table(
        "sessions",
        sa.Column("id", sa.Uuid(), primary_key=True),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            server_default=sa.func.now(),
            nullable=False,
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            server_default=sa.func.now(),
            nullable=False,
        ),
        sa.Column("is_active", sa.Boolean(), server_default=sa.text("true"), nullable=False),
        sa.Column(
            "show_id",
            sa.Uuid(),
            sa.ForeignKey("shows.id"),
            nullable=False,
        ),
        sa.Column("starts_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("venue", sa.String(length=200), nullable=False),
        sa.Column("capacity", sa.Integer(), nullable=False),
        sa.Column("full_price_cents", sa.Integer(), nullable=False),
        sa.Column("status", _SESSION_STATUS, nullable=False),
    )
    op.create_index("ix_sessions_show_id", "sessions", ["show_id"])
    op.create_index("ix_sessions_starts_at_status", "sessions", ["starts_at", "status"])


def downgrade() -> None:
    op.drop_index("ix_sessions_starts_at_status", table_name="sessions")
    op.drop_index("ix_sessions_show_id", table_name="sessions")
    op.drop_table("sessions")
    op.drop_table("shows")
```

### `pyproject.toml`

Editar: os routers FastAPI usam `Depends(...)` como valor default de parâmetro,
o que o `flake8-bugbear` (regra `B008`) acusa. É o padrão do framework — ignorar
`B008` só nos routers. Acrescentar uma linha em
`[tool.ruff.lint.per-file-ignores]`:

```toml
# pyproject.toml  — editar (dentro de [tool.ruff.lint.per-file-ignores])
"src/app/core/**" = ["I001", "E501"]
"src/app/outbox/models.py" = ["I001"]
"src/app/core/**/__init__.py" = ["F401"]
"src/migrations/env.py" = ["I001", "E402"]
"src/app/modules/*/api/routers/*.py" = ["B008"]
```

---

## 3. Onde cada regra de negócio entra

| Regra                                                       | Arquivo · função                                                                                        | Como                                                                                                                                                                                                                              |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RF08 — CRUD exige papel `ADMIN`                             | `api/routers/admin_catalog_router.py` · `APIRouter(dependencies=[Depends(require_admin)])`              | Todas as 10 rotas herdam `require_admin` (403 se `role != ADMIN`)                                                                                                                                                                 |
| RF08 — sessão vendida NÃO se apaga, só cancela              | `domain/aggregates/session.py` · `Session.deactivate(tickets_sold)`                                     | `tickets_sold > 0` → `ConflictError("Cancele a sessão em vez de excluir.")` (409). O usecase `delete_session` passa a contagem real                                                                                               |
| RF07 / RN02 — cancelar dispara reembolso em massa           | `domain/aggregates/session.py` · `Session.cancel()`                                                     | Levanta `SessionCancelled`; `AggregateRepository.save` grava a linha `Event` na mesma transação → o relay entrega ao handler de `payment` (aplica RN02 a partir do cancelamento; o admin não define valor) e ao de `notification` |
| RF08 — capacidade nunca abaixo do comprometido              | `domain/aggregates/session.py` · `Session.update(committed, ...)`                                       | `capacity < committed` → `ConflictError("Já há ingressos comprometidos nesta sessão.")` (409). `committed = tickets_sold + reserved_open`, contado no usecase após `find_by_id_for_update`                                        |
| RF08 — data de sessão sempre futura na criação              | `domain/aggregates/session.py` · `Session.create(now=...)`                                              | `starts_at <= now` → `DomainError("A data da sessão deve ser futura.")` (422). A forma (tz-aware) é validada em `CreateSessionRequest`                                                                                            |
| RN04 — meia = 50% da inteira, derivada                      | `domain/value_objects/money.py` · `Money.half()` + `Session.half_price`                                 | `cents // 2`; nunca digitado. `AdminSessionResponse.half_price` é sempre derivado                                                                                                                                                 |
| RN05 (adjacente) — recontar sob concorrência                | `application/usecases/session_admin_usecase.py` · `_lock()`                                             | `find_by_id_for_update` na `Session` antes de recontar/validar capacidade em `update`/`cancel`/`delete`                                                                                                                           |
| logic.md §4 — `is_on_sale` derivado                         | `domain/aggregates/session.py` · `Session.is_on_sale(now)` + `application/views.py` · `_session_status` | `status == ON_SALE and starts_at > now`; a parte "espetáculo publicado" fica na leitura (o admin vê o `status` do `Show` no `AdminShowResponse`)                                                                                  |
| spec §8 — publicar espetáculo sem sessão futura é permitido | `application/usecases/show_admin_usecase.py` · `publish_show`                                           | Nenhuma checagem de sessão — só não aparece na vitrine (regra de `catalog-show-search`)                                                                                                                                           |

---

## 4. DevOps

Não se aplica — a feature não introduz variável de ambiente nem segredo de CI.
A única mudança de config é o ignore de `B008` em `pyproject.toml` (item 24).

---

## 5. Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-admin-management

# commit 1 — domínio + base compartilhada
git add src/app/core/domain/errors.py src/app/core/shared/schema.py \
        src/app/modules/catalog/__init__.py src/app/modules/catalog/domain
git commit -m "feat(catalog): modelar Show, Session, Money e eventos de domínio"

# commit 2 — infrastructure
git add src/app/modules/catalog/infrastructure
git commit -m "feat(catalog): repositorios de Show, Session e contagem de assentos"

# commit 3 — application
git add src/app/modules/catalog/application
git commit -m "feat(catalog): usecases de gestão e schemas (cancela nao exclui, capacidade)"

# commit 4 — api + migration + config
git add src/app/modules/catalog/api src/app/modules/catalog/router.py \
        src/app/main.py src/migrations pyproject.toml
git commit -m "feat(catalog): expor rotas admin (require_admin), router e migration"

alembic upgrade head
ruff check . && pytest -q
```

Depois: `/team-ludens:tbd-pr` (senior-dev Modo 2 + `/code-review`) → push → PR
`Closes #<NN>` → merge (1 aprovação + CI verde).

---

## 6. Ordem entre as superfícies

`catalog-admin-management` é a **primeira fatia do módulo `catalog`** (base das
demais). Depende só do merge de `identity-auth` (`require_admin`, `Role`).
Backend e QA (casos de domínio de `quality.md`) começam juntos a partir do
`logic.md`. Frontend começa em paralelo contra o contrato-alvo de
`integration.md`. O `integration.md` vira canônico após o merge do backend.

---

## 7. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 e logic.md fechados). Pontos de
atenção de implementação — não impedem abrir a issue:

- **`booking` ainda não mergeado.** `tickets_sold` / `reserved_open` vêm de
  `SeatCountsRepository`, que lê as tabelas `tickets` / `reservations` do módulo
  `booking`. Enquanto `booking` não entra, essas tabelas não existem e a
  contagem é **0** (aviso no log, sem silenciar) — na prática a regra
  "não exclui sessão vendida" fica permissiva em runtime até `booking` entrar.
  O invariante de domínio (`Session.deactivate` / `Session.update`) já está
  pronto e coberto por teste em `quality.md`. Ao mergear `booking`, revisar os
  nomes de coluna/status no SQL de `seat_counts_repository.py`
  (`tickets.status = 'valid'`, `reservations.status = 'open'`), que seguem a
  spec de `booking` mas não foram validados contra código.
- **`core/domain/errors.py` e `core/shared/schema.py`** são compartilhados com
  `identity-auth`. Quem mergear primeiro cria; o outro reusa (não duplicar).
- **`down_revision` da migration** (`0001_identity_auth`) assume o id da
  migration de `identity-auth`. Ajustar ao id real na hora do merge, ou `None`
  se `catalog` entrar primeiro.
