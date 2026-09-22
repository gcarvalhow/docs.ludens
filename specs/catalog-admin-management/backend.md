---
status: done
spec: catalog-admin-management
surface: backend
created_at: 2026-09-03
updated_at: 2026-09-21
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
`docs.ludens/backend/overview.md`, `docs.ludens/backend/conventions.md`.

**Depende de:** `identity-auth` mergeado — expõe `require_admin` em
`app.modules.identity.dependencies` (403 quando `user.is_admin` é `False`;
`User` real não tem enum `Role`, é um `bool`).
**É base de:** `catalog-show-search`, `catalog-session-detail`,
`booking-reservation`.

> **Revisão de 2026-09-10:** a versão anterior deste documento assumia
> convenções que não batiam com o código real já mergeado em `identity-auth`
> (o primeiro módulo implementado). Corrigido nesta revisão: schemas em
> camelCase via um `CamelModel` que não existe (o real é `pydantic.BaseModel`
> puro, snake_case); `DomainError` com parâmetro `status_code`, que a classe
> real não tem (o real mapeia por subclasse — `ConflictError`→409,
> `NotFoundError`→404 — numa lista central em `main.py`); `core/domain/
> errors.py` tratado como arquivo a criar, quando já existe; `ShowRepository.
> find_by_id`/`SessionRepository.find_by_id_for_update`, que o repositório
> base real não tem (o real só tem `find_by(field, value)`); edição de
> `pyproject.toml` para o Ruff, que foi removido do projeto (`chore: remove
> referências... e ao Ruff`, `docs.ludens` #5). `catalog-show-search` e
> `catalog-session-detail` (as duas fatias seguintes do módulo) já foram
> escritas contra o código real — esta revisão só alinha esta fatia à mesma
> base.
>
> **Revisão de 2026-09-21:** o código evoluiu ainda mais desde então, pelas
> fatias seguintes. `Show.genre` (string livre) virou `Show.genre_id` (UUID,
> FK pra `genres.id`) — mudança da migração `0004_catalog_genre.py`, dona de
> `catalog-genre`; a migração `0002_catalog_admin.py` abaixo continua exata
> como registro histórico do schema original. `admin_catalog_router.py` não
> existe mais: as rotas de admin de `Show`/`Session` foram divididas em
> `show_router.py` (prefixo `/catalog/shows`) e `session_router.py` (prefixo
> `/catalog/sessions`), e cada um desses arquivos passou a acumular também as
> rotas de leitura pública (`GET .../{id}` e a busca paginada), com resposta
> diferente conforme `is_admin` — o `router.py` do módulo hoje monta
> `genre_router` + `show_router` + `session_router`. `SeatCountsRepository`
> (com SQL real contra `tickets`/`reservations`) nunca foi implementado: hoje
> todo usecase usa `SeatCounts(0, 0)` fixo (ver §7). O `Session.cancel()`/
> `Session.deactivate()` reais não têm mais guarda própria de estado — a
> checagem (`já cancelada`, `tem ingresso vendido`) virou responsabilidade do
> usecase, não do aggregate. Esta revisão corrige o código desta fatia
> (write-path de admin, que continua dona deste doc); o read-path público
> (busca, detalhe) é código real também, mas descrito com detalhe em
> `catalog-show-search/backend.md` e `catalog-session-detail/backend.md`, que
> o possuem — não duplicado aqui.

---

## 1. Arquivos (ordem de dependência)

`infrastructure/repositories/` vem antes de `application/` — os usecases
dependem dos repositórios, não o contrário. `repositories/__init__.py`
reexporta as três classes do pacote (um só import em vez de três).

| #   | Camada         | Caminho                                                                     | Novo/Editar                                                                                             |
| --- | -------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| 1   | domain         | `src/app/modules/catalog/**/__init__.py` (pacotes)                          | novo                                                                                                    |
| 2   | domain         | `src/app/modules/catalog/domain/enumerations/show_status.py`                | novo                                                                                                    |
| 3   | domain         | `src/app/modules/catalog/domain/enumerations/session_status.py`             | novo                                                                                                    |
| 4   | domain         | `src/app/modules/catalog/domain/events/domain_events.py`                    | novo                                                                                                    |
| 5   | domain         | `src/app/modules/catalog/domain/aggregates/show.py`                         | novo                                                                                                    |
| 6   | domain         | `src/app/modules/catalog/domain/aggregates/session.py`                      | novo                                                                                                    |
| 7   | infrastructure | `src/app/modules/catalog/infrastructure/repositories/show_repository.py`    | novo                                                                                                    |
| 8   | infrastructure | `src/app/modules/catalog/infrastructure/repositories/session_repository.py` | novo — inclui `find_by_id_for_update` (ver §2 e nota)                                                   |
| 9   | infrastructure | ~~`seat_counts_repository.py`~~                                             | **não existe no código real** (ver §7) — não criar                                                      |
| 10  | infrastructure | `src/app/modules/catalog/infrastructure/repositories/__init__.py`           | novo                                                                                                    |
| 11  | application    | `src/app/modules/catalog/application/schemas/request.py`                    | novo                                                                                                    |
| 12  | application    | `src/app/modules/catalog/application/schemas/response.py`                   | novo                                                                                                    |
| 13  | application    | `src/app/modules/catalog/application/usecases/show_usecase.py`              | novo — inclui `_show_response`/`_session_response` (ver §2)                                             |
| 14  | application    | `src/app/modules/catalog/application/usecases/session_usecase.py`           | novo — inclui `_cents_from_reais`/`_session_response` (ver §2)                                          |
| 15  | api            | `src/app/modules/catalog/api/routers/show_router.py`                        | novo — write-path admin desta fatia; read-path público acrescentado depois por `catalog-show-search`    |
| 15b | api            | `src/app/modules/catalog/api/routers/session_router.py`                     | novo — write-path admin desta fatia; read-path público acrescentado depois por `catalog-session-detail` |
| 16  | api            | `src/app/modules/catalog/router.py`                                         | novo                                                                                                    |
| 17  | api            | `src/app/main.py`                                                           | editar (arquivo real já existe — só acrescentar)                                                        |
| 18  | migration      | `src/migrations/env.py`                                                     | editar (arquivo real já existe — só acrescentar imports)                                                |
| 19  | migration      | `src/migrations/versions/0002_catalog_admin.py`                             | novo                                                                                                    |

Não há variável de ambiente nova — a subseção **DevOps** não se aplica. Sem
edição de `pyproject.toml`: o projeto não tem `[tool.ruff]` configurado (Ruff
foi removido), não há lint automatizado a ajustar.

---

## 2. Código

### Base real que este módulo usa (não recriar)

Tudo abaixo já existe em `api.ludens` (criado por `identity-auth`) — só
importar, nunca redefinir:

```python
# app.core.domain (reexporta de model.py/events.py/aggregate.py/errors.py)
from app.core.domain import AggregateRoot, DomainEvent, Model
from app.core.domain import AuthError, ConflictError, DomainError, ForbiddenError, GoneError, NotFoundError
# DomainError NÃO tem parâmetro status_code — é Exception simples com .message.
# ConflictError -> 409, AuthError -> 401, ForbiddenError -> 403,
# NotFoundError -> 404, GoneError -> 410 (mapeados em main.py); default 422.

# app.core.infrastructure.repositories (reexporta de repository.py)
from app.core.infrastructure.repositories import AggregateRepository
# Só tem: find_by(field, value), find_all(order_by=...), find_all_by(...),
# exists_by(field, value), save(entity). Sem find_by_id nem find_by_id_for_update
# — por isso este módulo acrescenta find_by_id_for_update em SessionRepository.
```

Os schemas de request/response deste módulo são `pydantic.BaseModel` puro,
campos **snake_case** — mesma convenção do contrato real de `identity-auth`
(`RegisterRequest`, `UserResponse`). Não existe `CamelModel`/alias camelCase
no projeto.

### `src/app/modules/catalog/**/__init__.py`

Todos vazios — só marcam o pacote Python. `infrastructure/repositories/__init__.py`
**não** entra aqui — tem conteúdo real (item 11). Criar um por diretório novo:

```python
# src/app/modules/catalog/__init__.py                                   — novo
# src/app/modules/catalog/domain/__init__.py                            — novo
# src/app/modules/catalog/domain/enumerations/__init__.py               — novo
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
import enum

class ShowStatus(str, enum.Enum):
    DRAFT = "draft"
    PUBLISHED = "published"
```

### `src/app/modules/catalog/domain/enumerations/session_status.py`

```python
import enum

class SessionStatus(str, enum.Enum):
    ON_SALE = "on_sale"
    CANCELLED = "cancelled"
```

### `src/app/modules/catalog/domain/events/domain_events.py`

```python
from uuid import UUID
from datetime import datetime
from dataclasses import dataclass, field

from app.core.domain import DomainEvent

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

### `src/app/modules/catalog/domain/aggregates/show.py`

```python
from __future__ import annotations

from uuid import UUID, uuid4

from sqlalchemy import Enum as SAEnum, ForeignKey, String, Uuid
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain import AggregateRoot, DomainEvent, Model
from app.modules.catalog.domain.enumerations.show_status import ShowStatus

from app.modules.catalog.domain.events.domain_events import (
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
        SAEnum(ShowStatus, native_enum=False, length=20),
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

### `src/app/modules/catalog/domain/aggregates/session.py`

```python
from __future__ import annotations

from uuid import UUID, uuid4
from datetime import datetime, timezone

from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy import DateTime, Enum as SAEnum, ForeignKey, Integer, String, Uuid

from app.core.domain import AggregateRoot, DomainEvent, Model

from app.modules.catalog.domain.enumerations.session_status import SessionStatus
from app.modules.catalog.domain.events.domain_events import (
    SessionCancelled,
    SessionCreated,
    SessionDeactivated,
    SessionUpdated,
)

# O aggregate real não guarda mais estado por si (sem DomainError/ConflictError
# aqui) — "data futura", "capacidade > 0", "sessão já cancelada" e "tem
# ingresso vendido" viraram checagem do usecase (§2, session_usecase.py), não
# invariante do domínio. Manter assim: o aggregate só registra o evento.
class Session(AggregateRoot, Model):
    __tablename__ = "sessions"

    show_id: Mapped[UUID] = mapped_column(Uuid(), ForeignKey("shows.id"), nullable=False, index=True)
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
    def half_price_cents(self) -> int:
        # RN04 — 50% da inteira, truncado ao centavo. Derivado, nunca digitado.
        return self.full_price_cents // 2

    def is_on_sale(self, now: datetime) -> bool:
        return self.status is SessionStatus.ON_SALE and self.starts_at > now

    @classmethod
    def create(
        cls,
        *,
        show_id: UUID,
        starts_at: datetime,
        venue: str,
        capacity: int,
        full_price_cents: int,
    ) -> "Session":
        session = cls()
        session.id = uuid4()

        session.raise_event(
            lambda v: SessionCreated(
                version=v, id=session.id, show_id=show_id, starts_at=starts_at,
                venue=venue, capacity=capacity, full_price_cents=full_price_cents,
            )
        )

        return session

    def update(
        self,
        *,
        starts_at: datetime,
        venue: str,
        capacity: int,
        full_price_cents: int,
    ) -> None:
        self.raise_event(
            lambda v: SessionUpdated(
                version=v, id=self.id, starts_at=starts_at, venue=venue,
                capacity=capacity, full_price_cents=full_price_cents,
            )
        )

    def cancel(self) -> None:
        self.raise_event(
            lambda v: SessionCancelled(
                version=v, id=self.id, show_id=self.show_id,
                starts_at=self.starts_at, cancelled_at=datetime.now(timezone.utc),
            )
        )

    def deactivate(self) -> None:
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
from app.modules.catalog.domain.aggregates.show import Show
from app.core.infrastructure.repositories import AggregateRepository

class ShowRepository(AggregateRepository[Show]):
    model = Show
```

### `src/app/modules/catalog/infrastructure/repositories/session_repository.py`

Inclui `find_by_id_for_update` — o repositório base real não tem trava de
linha, e este módulo é o primeiro a precisar (concorrência ao editar/cancelar/
excluir sessão). `catalog-session-detail` e, mais tarde, `booking` **reusam**
este método via `catalog/dependencies.py` — não recriar lá.

`find_all_for_shows` não existe no código real: pra listar as sessões de um
espetáculo, o usecase usa o `find_all_by(show_id=..., order_by=[...])`
genérico que já vem de `AggregateRepository` — não precisou de um método
próprio.

```python
from uuid import UUID
from sqlalchemy import select

from app.modules.catalog.domain.aggregates import Session
from app.core.infrastructure.repositories import AggregateRepository

class SessionRepository(AggregateRepository[Session]):
    model = Session

    async def find_by_id_for_update(self, session_id: UUID) -> Session | None:
        # SELECT ... FOR UPDATE: trava a linha até o fim da transação — RN05,
        # necessário ao editar/cancelar/excluir uma sessão sob concorrência.
        result = await self._session.execute(
            select(Session)
            .where(Session.id == session_id, Session.is_active.is_(True))
            .with_for_update()
        )

        return result.scalar_one_or_none()
```

### `SeatCounts` — nunca virou um repositório real (ver §7)

Esta seção descrevia um `seat_counts_repository.py` com SQL real contra
`tickets`/`reservations`. **Esse arquivo não existe no código real.** O que
existe é só um `NamedTuple` `SeatCounts(tickets_sold, reserved_open)`,
definido direto em `domain/aggregates/session.py` (não em `infrastructure/`),
e todo usecase de `catalog` (admin, busca, detalhe) usa `SeatCounts(0, 0)`
fixo — nenhuma contagem real ainda acontece, porque `booking` (dono das
tabelas `tickets`/`reservations`) não foi mergeado. Ver §7 pra o que isso
implica em runtime.

### `src/app/modules/catalog/infrastructure/repositories/__init__.py`

```python
from app.modules.catalog.infrastructure.repositories.session_repository import SessionRepository
from app.modules.catalog.infrastructure.repositories.show_repository import ShowRepository

__all__ = ["SessionRepository", "ShowRepository"]
```

(`genre_repository.py` acrescenta `GenreRepository` a este mesmo `__init__.py`
depois — é código real, mas dono/documentado em `catalog-genre/backend.md`.)

### `src/app/modules/catalog/application/schemas/request.py`

`pydantic.BaseModel` puro — sem alias camelCase (ver nota de revisão). Sem
`image_url`: a imagem é atribuída pelo usecase, não vem do admin (spec.md §6).
**Padronização:** só `PUT`, nunca `PATCH` — o corpo é sempre completo, nunca
parcial. Por isso um único schema serve criação e edição de cada aggregate
(nada de `Create*`/`Update*` separados com campos opcionais).

`genre` (string) virou `genre_id` (UUID, FK) na `ShowRequest` desde
`catalog-genre`. `SessionRequest` ganhou `show_id` no corpo — a criação de
sessão não é mais aninhada em `/catalog/shows/{show_id}/sessions`, é
`POST /catalog/sessions/` com `show_id` no payload; em `PUT` o campo é
ignorado (a sessão não muda de espetáculo depois de criada).

```python
from uuid import UUID
from datetime import datetime
from pydantic import BaseModel, Field, field_validator

class ShowRequest(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    synopsis: str = Field(min_length=1, max_length=5000)
    genre_id: UUID

class SessionRequest(BaseModel):
    show_id: UUID = Field(description="Ignorado em PUT — sessão não muda de show após criada.")
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

### `src/app/modules/catalog/application/schemas/response.py`

```python
from uuid import UUID
from typing import Literal
from datetime import datetime

from pydantic import BaseModel

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
    genre_id: UUID
    genre: str
    status: Literal["draft", "published"]

class AdminShowResponse(AdminShowSummaryResponse):
    sessions: list[AdminSessionResponse]
```

`genre` (nome resolvido) veio junto de `genre_id` desde `catalog-genre` — o
usecase busca o `Genre` e monta os dois campos, nunca só o id.
`AdminShowResponse` virou uma extensão de `AdminShowSummaryResponse` (a
versão sem `sessions`, usada na busca paginada de admin) em vez de uma classe
solta.

> `catalog-show-search`, `catalog-session-detail` e `catalog-genre`
> acrescentam mais classes a este mesmo arquivo (`ShowCardResponse`,
> `SessionDetailResponse`, `GenreResponse`, `SessionSummaryResponse`,
> `ShowDetailResponse`, `TicketTypeResponse`, `SessionShowRef` etc.) — código
> real, mas descrito com detalhe nos docs dessas specs, não duplicado aqui.

### `src/app/modules/catalog/application/usecases/show_usecase.py`

Sem "Admin" no nome — quem restringe a rota a admin é o router
(`Depends(require_admin)`), não a identidade do usecase (mesmo raciocínio de
`identity`: existe `UserUseCase`, não `UserAdminUseCase`, mesmo tendo um
método `list_users` só de admin — ver `identity-auth/backend.md`). Monta a
resposta direto no usecase (mesmo padrão de `user_usecase.py` — sem módulo
`views.py` intermediário). A pequena derivação de `status` de sessão se
repete aqui e em `session_usecase.py` — duplicar 3 linhas puras é mais
simples do que um módulo compartilhado entre os dois agregados.

`SeatCountsRepository` não existe (ver nota acima) — todo `SeatCounts` abaixo
é `SeatCounts(0, 0)` fixo. `list_shows` (endpoint solto) virou `search`
(paginado, `is_admin`-aware) — não há mais uma rota de listagem de admin
separada da busca pública; as duas são o mesmo endpoint com resposta
diferente. `get_show_detail` (também `is_admin`-aware) é novo: não existia
nesta fatia, é o que `catalog-show-search`/`catalog-session-detail` chamam de
"detalhe". Reprodução fiel do arquivo real (inclui `search`/`get_show_detail`,
que tecnicamente pertencem ao read-path público — mantidos aqui porque vivem
no mesmo arquivo e usecase que o write-path admin desta fatia):

```python
from __future__ import annotations

import random
from uuid import UUID
from datetime import date, datetime, timezone

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.shared import Page, PaginationParams
from app.core.domain.errors import ConflictError, NotFoundError

from app.modules.catalog.application.schemas.request import ShowRequest
from app.modules.catalog.domain.aggregates import Genre, SeatCounts, Session, Show
from app.modules.catalog.application.schemas.response import (
    AdminShowResponse,
    AdminShowSummaryResponse,
    ShowCardResponse,
    SessionSummaryResponse,
    ShowDetailResponse,
)
from app.modules.catalog.application.usecases.utils import floor_from
from app.modules.catalog.application.mappers import card_response, session_response

from app.modules.catalog.infrastructure.queries import ShowSearchQuery
from app.modules.catalog.infrastructure.repositories import GenreRepository, SessionRepository, ShowRepository

# Débito técnico: upload real de imagem não existe nesta entrega. Sorteado
# a cada criação, sem repetir preferência — arquivos estáticos de web.ludens.
_DEFAULT_SHOW_IMAGES = [
    "/images/show-placeholders/1.jpg",
    "/images/show-placeholders/2.jpg",
    "/images/show-placeholders/3.jpg",
    "/images/show-placeholders/4.jpg",
    "/images/show-placeholders/5.jpg",
    "/images/show-placeholders/6.jpg",
]

def _show_summary(show: Show, genre_name: str) -> AdminShowSummaryResponse:
    return AdminShowSummaryResponse(
        id=show.id,
        title=show.title,
        synopsis=show.synopsis,
        image_url=show.image_url,
        genre_id=show.genre_id,
        genre=genre_name,
        status=show.status.value,
    )

def _show_response(
    show: Show, genre_name: str, sessions: list[Session], counts_map: dict[UUID, SeatCounts], now: datetime
) -> AdminShowResponse:
    return AdminShowResponse(
        **_show_summary(show, genre_name).model_dump(),
        sessions=[
            session_response(s, counts_map.get(s.id, SeatCounts(0, 0)), now) for s in sessions
        ],
    )

class ShowUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._show_repository = ShowRepository(session)
        self._session_repository = SessionRepository(session)
        self._genre_repository = GenreRepository(session)
        self._show_search_query = ShowSearchQuery(session)

    async def create_show(self, req: ShowRequest) -> AdminShowResponse:
        genre = await self._require_genre(req.genre_id)
        show = Show.create(
            title=req.title,
            synopsis=req.synopsis,
            image_url=random.choice(_DEFAULT_SHOW_IMAGES),
            genre_id=genre.id,
        )

        await self._show_repository.save(show)
        return _show_response(show, genre.name, [], {}, datetime.now(timezone.utc))

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
        return await self._view_for(show)

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
        sessions = await self._session_repository.find_all_by(show_id=show.id, order_by=["starts_at"])
        counts = {s.id: SeatCounts(0, 0) for s in sessions}

        if any(counts.get(s.id, SeatCounts(0, 0)).tickets_sold > 0 for s in sessions):
            raise ConflictError(
                "Cancele as sessões com ingressos vendidos antes de excluir o espetáculo."
            )

        show.deactivate()
        await self._show_repository.save(show)

    async def search(
        self, *, from_date: date | None, genre_id: UUID | None, pagination: PaginationParams, is_admin: bool
    ) -> Page[AdminShowSummaryResponse] | Page[ShowCardResponse]:
        # Documentado em detalhe por catalog-show-search/backend.md.
        if is_admin:
            shows, total = await self._show_repository.find_all_paginated(
                order_by=["-created_at"], page=pagination.page, size=pagination.size
            )
            genres = await self._genre_repository.find_all_by_ids([s.genre_id for s in shows])
            names = {g.id: g.name for g in genres}

            return Page(
                items=[_show_summary(s, names.get(s.genre_id, "")) for s in shows],
                page=pagination.page,
                size=pagination.size,
                total=total,
            )

        floor = floor_from(from_date)
        rows, total = await self._show_search_query.search_with_upcoming(
            floor=floor, genre_id=genre_id, page=pagination.page, size=pagination.size
        )

        return Page(
            items=[card_response(row) for row in rows],
            page=pagination.page,
            size=pagination.size,
            total=total,
        )

    async def get_show_detail(self, show_id: UUID, *, is_admin: bool) -> AdminShowResponse | ShowDetailResponse:
        # Documentado em detalhe por catalog-session-detail/backend.md.
        show = await self._require_show(show_id)

        if is_admin:
            return await self._view_for(show)

        if not show.is_published:
            raise NotFoundError("Espetáculo não encontrado.")
        now = datetime.now(timezone.utc)

        sessions = [
            s
            for s in await self._session_repository.find_all_by(show_id=show.id, order_by=["starts_at"])
            if s.starts_at > now
        ]
        counts = {s.id: SeatCounts(0, 0) for s in sessions}
        genre = await self._require_genre(show.genre_id)

        return ShowDetailResponse(
            id=show.id,
            title=show.title,
            synopsis=show.synopsis,
            image_url=show.image_url,
            genre_id=show.genre_id,
            genre=genre.name,
            sessions=[
                session_response(s, counts.get(s.id, SeatCounts(0, 0)), now) for s in sessions
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

    async def _view_for(self, show: Show) -> AdminShowResponse:
        sessions = await self._session_repository.find_all_by(show_id=show.id, order_by=["starts_at"])
        counts = {s.id: SeatCounts(0, 0) for s in sessions}
        genre = await self._require_genre(show.genre_id)

        return _show_response(show, genre.name, sessions, counts, datetime.now(timezone.utc))
```

### `src/app/modules/catalog/application/usecases/session_usecase.py`

Sem "Admin" no nome, mesmo raciocínio de `show_usecase.py` — `Session` é
módulo/aggregate independente de `Show`, não um apêndice dele; o que muda por
perfil é a rota, não a identidade do usecase.

`show_id` agora vem no corpo (`SessionRequest`), não na URL — `create_session`
não recebe mais `show_id` como parâmetro separado. `update`/`cancel`/`delete`
fazem a checagem de estado (sessão cancelada, capacidade abaixo do
comprometido, ingresso vendido) aqui no usecase, não no aggregate — o
aggregate (`Session.update`/`cancel`/`deactivate`) só registra o evento.
`get_session_detail` (`is_admin`-aware) é o read-path público, documentado em
detalhe por `catalog-session-detail/backend.md`; reproduzido aqui porque vive
no mesmo arquivo/usecase que o write-path admin desta fatia:

```python
from __future__ import annotations

from uuid import UUID
from datetime import datetime, timezone

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain.errors import ConflictError, DomainError, NotFoundError

from app.modules.catalog.domain.enumerations import SessionStatus
from app.modules.catalog.domain.aggregates import SeatCounts, Session
from app.modules.catalog.application.schemas.request import SessionRequest
from app.modules.catalog.application.schemas.response import (
    AdminSessionResponse,
    SessionDetailResponse,
    SessionShowRef,
    TicketTypeResponse,
)
from app.modules.catalog.application.mappers import session_response
from app.modules.catalog.application.utils import cents_from_reais, reais_from_cents
from app.modules.catalog.infrastructure.repositories import SessionRepository, ShowRepository

class SessionUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._show_repository = ShowRepository(session)
        self._session_repository = SessionRepository(session)

    async def create_session(self, req: SessionRequest) -> AdminSessionResponse:
        show = await self._show_repository.find_by("id", req.show_id)
        if show is None:
            raise NotFoundError("Espetáculo não encontrado.")

        now = datetime.now(timezone.utc)
        if req.starts_at <= now:
            raise DomainError("A data da sessão deve ser futura.")
        if req.capacity <= 0:
            raise DomainError("A capacidade deve ser maior que zero.")

        session = Session.create(
            show_id=show.id,
            starts_at=req.starts_at,
            venue=req.venue,
            capacity=req.capacity,
            full_price_cents=cents_from_reais(req.full_price),
        )

        await self._session_repository.save(session)
        return session_response(session, SeatCounts(0, 0), now)

    async def update_session(self, session_id: UUID, req: SessionRequest) -> AdminSessionResponse:
        session = await self._lock(session_id)
        if session.status is SessionStatus.CANCELLED:
            raise ConflictError("Não é possível editar uma sessão cancelada.")

        counts = SeatCounts(0, 0)
        now = datetime.now(timezone.utc)

        if req.starts_at <= now:
            raise DomainError("A data da sessão deve ser futura.")

        committed = counts.tickets_sold + counts.reserved_open
        if req.capacity < committed:
            raise ConflictError("Já há ingressos comprometidos nesta sessão.")

        session.update(
            starts_at=req.starts_at,
            venue=req.venue,
            capacity=req.capacity,
            full_price_cents=cents_from_reais(req.full_price),
        )

        await self._session_repository.save(session)
        return session_response(session, counts, now)

    async def cancel_session(self, session_id: UUID) -> None:
        session = await self._lock(session_id)
        if session.status is SessionStatus.CANCELLED:
            raise ConflictError("A sessão já está cancelada.")

        session.cancel()
        await self._session_repository.save(session)

    async def delete_session(self, session_id: UUID) -> None:
        session = await self._lock(session_id)
        counts = SeatCounts(0, 0)

        if counts.tickets_sold > 0:
            raise ConflictError("Cancele a sessão em vez de excluir.")

        session.deactivate()
        await self._session_repository.save(session)

    async def get_session_detail(self, session_id: UUID, *, is_admin: bool) -> AdminSessionResponse | SessionDetailResponse:
        # Documentado em detalhe por catalog-session-detail/backend.md.
        session = await self._session_repository.find_by("id", session_id)
        if session is None:
            raise NotFoundError("Sessão não encontrada.")

        show = await self._show_repository.find_by("id", session.show_id)
        if show is None:
            raise NotFoundError("Espetáculo não encontrado.")

        now = datetime.now(timezone.utc)
        counts = SeatCounts(0, 0)

        if is_admin:
            return session_response(session, counts, now)

        if not show.is_published:
            raise NotFoundError("Espetáculo não encontrado.")

        available = session.available_count(counts)

        return SessionDetailResponse(
            id=session.id,
            show=SessionShowRef(id=show.id, title=show.title),
            starts_at=session.starts_at,
            venue=session.venue,
            capacity=session.capacity,
            available_count=available,
            status=session.status_at(now, counts),
            ticket_types=[
                TicketTypeResponse(type="full", price=reais_from_cents(session.full_price_cents)),
                TicketTypeResponse(type="half", price=reais_from_cents(session.half_price_cents)),
            ],
        )

    async def _lock(self, session_id: UUID) -> Session:
        session = await self._session_repository.find_by_id_for_update(session_id)
        if session is None:
            raise NotFoundError("Sessão não encontrada.")

        return session
```

`session_response` (o mapper usado acima) monta `AdminSessionResponse` a
partir de `Session` + `SeatCounts` + `now` — mesma forma que a antiga
`_session_response` local descrita nas revisões anteriores deste doc, só que
hoje vive em `application/mappers/`, compartilhada entre `show_usecase.py` e
`session_usecase.py` (não duplicada nos dois arquivos como a versão anterior
deste doc descrevia).

### `src/app/modules/catalog/api/routers/show_router.py` e `session_router.py`

Não existe `admin_catalog_router.py` nem prefixo `/admin` — cada aggregate
tem seu próprio router, montado direto em `/catalog/shows` e
`/catalog/sessions`. `require_admin` vira `dependencies=[Depends(require_admin)]`
**por rota** (não no `APIRouter` inteiro), porque o mesmo router mistura
rotas de escrita (admin) com as duas rotas de leitura pública que
`catalog-show-search`/`catalog-session-detail` acrescentaram (`GET` por id e
`GET` de busca, ambas sem `require_admin`, com resposta que varia por
`is_admin` via `get_current_user_optional`). Abaixo, só as rotas de escrita
que esta fatia é dona; o `GET`/busca completo está em
`catalog-show-search/backend.md` e `catalog-session-detail/backend.md`.

```python
# src/app/modules/catalog/api/routers/show_router.py — rotas de escrita (admin)
from uuid import UUID

from fastapi import APIRouter, Depends, Response
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.modules.identity.dependencies import require_admin
from app.modules.catalog.application.schemas.request import ShowRequest
from app.modules.catalog.application.schemas.response import AdminShowResponse
from app.modules.catalog.application.usecases.show_usecase import ShowUseCase

router = APIRouter(prefix="/catalog/shows", tags=["01.Catalog - Show"])

@router.post("", response_model=AdminShowResponse, status_code=201, dependencies=[Depends(require_admin)])
async def create(body: ShowRequest, session: AsyncSession = Depends(get_db)) -> AdminShowResponse:
    return await ShowUseCase(session).create_show(body)

@router.put("/{show_id}", response_model=AdminShowResponse, dependencies=[Depends(require_admin)])
async def update(show_id: UUID, body: ShowRequest, session: AsyncSession = Depends(get_db)) -> AdminShowResponse:
    return await ShowUseCase(session).update_show(show_id, body)

@router.post("/{show_id}/publish", status_code=204, dependencies=[Depends(require_admin)])
async def publish(show_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await ShowUseCase(session).publish_show(show_id)
    return Response(status_code=204)

@router.post("/{show_id}/unpublish", status_code=204, dependencies=[Depends(require_admin)])
async def unpublish(show_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await ShowUseCase(session).unpublish_show(show_id)
    return Response(status_code=204)

@router.delete("/{show_id}", status_code=204, dependencies=[Depends(require_admin)])
async def delete(show_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await ShowUseCase(session).delete_show(show_id)
    return Response(status_code=204)

# GET /{show_id} (detalhe) e GET "" (busca) — ver catalog-session-detail/backend.md
# e catalog-show-search/backend.md.
```

```python
# src/app/modules/catalog/api/routers/session_router.py — rotas de escrita (admin)
from uuid import UUID

from fastapi import APIRouter, Depends, Response
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.modules.identity.dependencies import require_admin
from app.modules.catalog.application.schemas.request import SessionRequest
from app.modules.catalog.application.schemas.response import AdminSessionResponse
from app.modules.catalog.application.usecases.session_usecase import SessionUseCase

router = APIRouter(prefix="/catalog/sessions", tags=["02.Catalog - Session"])

@router.post("/", response_model=AdminSessionResponse, status_code=201, dependencies=[Depends(require_admin)])
async def create(body: SessionRequest, session: AsyncSession = Depends(get_db)) -> AdminSessionResponse:
    return await SessionUseCase(session).create_session(body)

@router.put("/{session_id}", response_model=AdminSessionResponse, dependencies=[Depends(require_admin)])
async def update(session_id: UUID, body: SessionRequest, session: AsyncSession = Depends(get_db)) -> AdminSessionResponse:
    return await SessionUseCase(session).update_session(session_id, body)

@router.post("/{session_id}/cancel", status_code=202, dependencies=[Depends(require_admin)])
async def cancel(session_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await SessionUseCase(session).cancel_session(session_id)
    return Response(status_code=202)

@router.delete("/{session_id}", status_code=204, dependencies=[Depends(require_admin)])
async def delete(session_id: UUID, session: AsyncSession = Depends(get_db)) -> Response:
    await SessionUseCase(session).delete_session(session_id)
    return Response(status_code=204)

# GET /{session_id} (detalhe) — ver catalog-session-detail/backend.md.
```

### `src/app/modules/catalog/router.py`

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

`genre_router` é código real, dono/documentado em `catalog-genre/backend.md`.

### `src/app/main.py`

Editar o arquivo real (já registra `identity_router` e a lista
`_DOMAIN_ERROR_STATUS`) — só acrescentar o import/registro de `catalog`.
Arquivo completo já editado, para conferência:

```python
import asyncio
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware
from fastapi.exceptions import RequestValidationError

from app.config import settings
from app.core.shared import format_validation_errors
from app.outbox.relay import run as run_outbox_relay
from app.modules.catalog.router import router as catalog_router
from app.modules.identity.router import router as identity_router
from app.core.domain import AuthError, ConflictError, DomainError, ForbiddenError, GoneError, NotFoundError

logger = logging.getLogger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI):
    tasks = [asyncio.create_task(run_outbox_relay())]

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

_DOMAIN_ERROR_STATUS = [
    (ConflictError, 409),
    (AuthError, 401),
    (ForbiddenError, 403),
    (NotFoundError, 404),
    (GoneError, 410),
]

@app.exception_handler(DomainError)
async def domain_exception_handler(request: Request, exc: DomainError):
    status_code = next((s for t, s in _DOMAIN_ERROR_STATUS if isinstance(exc, t)), 422)
    return JSONResponse(status_code=status_code, content={"detail": exc.message})

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(identity_router)
app.include_router(catalog_router)

@app.get("/health")
async def health():
    return {"status": "ok", "environment": settings.environment}
```

(A versão real de `main.py` hoje não tem o bloco `BACKGROUND_TASK_MAX_AGE_SECONDS`/
`/health` com checagem de *staleness* do outbox — se isso mudar antes desta
fatia, preservar a versão real e só acrescentar as duas linhas de `catalog`.)

### `src/migrations/env.py`

Editar: acrescentar o import dos aggregates de `catalog`, junto dos imports
que `identity-auth` já deixou.

```python
# Cada feature acrescenta o import do próprio módulo aqui.
import app.modules.identity.domain.aggregates.user  # noqa: F401,E402
import app.modules.identity.domain.entities.password_reset_token  # noqa: F401,E402
import app.modules.identity.domain.entities.refresh_token  # noqa: F401,E402

# catalog-admin-management:
import app.modules.catalog.domain.aggregates.show  # noqa: F401,E402
import app.modules.catalog.domain.aggregates.session  # noqa: F401,E402
```

### `src/migrations/versions/0002_catalog_admin.py`

```python
"""catalog admin: tabelas shows e sessions

Revision ID: 0002_catalog_admin
Revises: 0001_identity_auth
Create Date: 2026-09-03
"""
from typing import Sequence, Union

import sqlalchemy as sa
from alembic import op

revision: str = "0002_catalog_admin"
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

(`down_revision = "0001_identity_auth"` já confere com o id real da migration
de `identity-auth` em `api.ludens` — não é mais uma suposição.)

---

## 3. Onde cada regra de negócio entra

| Regra                                                       | Arquivo · função                                                                                                                             | Como                                                                                                                                                                                                                              |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RF08 — CRUD exige `user.is_admin`                           | `api/routers/show_router.py` e `session_router.py` · `dependencies=[Depends(require_admin)]` por rota                                        | As 9 rotas de escrita (5 de show, 4 de session) levam `require_admin` (403 quando `is_admin` é `False`); as rotas `GET` de leitura pública não levam                                                                              |
| RF08 — sessão vendida NÃO se apaga, só cancela              | `domain/aggregates/session.py` · `Session.deactivate(tickets_sold)`                                                                          | `tickets_sold > 0` → `ConflictError("Cancele a sessão em vez de excluir.")` (409, mapeado em `main.py`). O usecase `delete_session` passa a contagem real                                                                         |
| RF07 / RN02 — cancelar dispara reembolso em massa           | `domain/aggregates/session.py` · `Session.cancel()`                                                                                          | Levanta `SessionCancelled`; `AggregateRepository.save` grava a linha `Event` na mesma transação → o relay entrega ao handler de `payment` (aplica RN02 a partir do cancelamento; o admin não define valor) e ao de `notification` |
| RF08 — capacidade nunca abaixo do comprometido              | `domain/aggregates/session.py` · `Session.update(committed, ...)`                                                                            | `capacity < committed` → `ConflictError("Já há ingressos comprometidos nesta sessão.")` (409). `committed = tickets_sold + reserved_open`, contado no usecase após `find_by_id_for_update`                                        |
| RF08 — data de sessão sempre futura na criação              | `domain/aggregates/session.py` · `Session.create(now=...)`                                                                                   | `starts_at <= now` → `DomainError("A data da sessão deve ser futura.")` (422, default). A forma (tz-aware) é validada em `SessionRequest`                                                                                         |
| RN04 — meia = 50% da inteira, derivada                      | `domain/aggregates/session.py` · `Session.half_price_cents`                                                                                  | `full_price_cents // 2`; nunca digitado. `AdminSessionResponse.half_price` é sempre derivado                                                                                                                                      |
| RN05 (adjacente) — recontar sob concorrência                | `application/usecases/session_usecase.py` · `_lock()`                                                                                        | `find_by_id_for_update` na `Session` antes de recontar/validar capacidade em `update`/`cancel`/`delete`                                                                                                                           |
| logic.md §4 — `is_on_sale` derivado                         | `domain/aggregates/session.py` · `Session.is_on_sale(now)`; resposta admin deriva o mesmo em `_session_response` (nos dois usecases)         | `status == ON_SALE and starts_at > now`; a parte "espetáculo publicado" fica na leitura (o admin vê o `status` do `Show` no `AdminShowResponse`)                                                                                  |
| spec §8 — publicar espetáculo sem sessão futura é permitido | `application/usecases/show_usecase.py` · `publish_show`                                                                                      | Nenhuma checagem de sessão — só não aparece na vitrine (regra de `catalog-show-search`)                                                                                                                                           |
| Padronização — sem `PATCH`, só `PUT` com corpo completo     | `application/schemas/request.py` (`ShowRequest`/`SessionRequest`, sem `Create*`/`Update*`); `api/routers/show_router.py`/`session_router.py` | Mesmo schema para `POST` e `PUT`; `update_show`/`update_session` não fazem merge parcial (`exclude_unset`) — usam `req.*` direto                                                                                                  |
| Espetáculo/sessão não encontrado                            | `_require_show`, `_lock` (ambos usecases)                                                                                                    | `NotFoundError("...")` → 404, mapeado em `main.py` (nunca `DomainError(..., status_code=...)`, que não existe)                                                                                                                    |

---

## 4. DevOps

Não se aplica — a feature não introduz variável de ambiente nem segredo de CI,
e o projeto não tem lint (Ruff) configurado para ajustar.

---

## 5. Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-admin-management

# commit 1 — domínio + base compartilhada
git add src/app/modules/catalog/__init__.py src/app/modules/catalog/domain
git commit -m "feat(catalog): modelar Show, Session e eventos de domínio"

# commit 2 — infrastructure
git add src/app/modules/catalog/infrastructure
git commit -m "feat(catalog): repositorios de Show, Session e contagem de assentos"

# commit 3 — application
git add src/app/modules/catalog/application
git commit -m "feat(catalog): usecases de gestao e schemas (cancela nao exclui, capacidade)"

# commit 4 — api + migration
git add src/app/modules/catalog/api src/app/modules/catalog/router.py \
        src/app/main.py src/migrations
git commit -m "feat(catalog): expor rotas admin (require_admin), router e migration"

alembic upgrade head
```

Depois: `/team-ludens:tbd-pr` (senior-dev Modo 2 + `/code-review`) → push → PR
`Closes #<NN>` → merge (1 aprovação + CI verde).

---

## 6. Ordem entre as superfícies

`catalog-admin-management` é a **primeira fatia do módulo `catalog`** (base das
demais). Dependia só do merge de `identity-auth` (`require_admin`) —
já satisfeito, `identity-auth` está mergeado.
Backend começa a partir do
`logic.md`. Frontend começa em paralelo contra o contrato-alvo de
`integration.md`. O `integration.md` vira canônico após o merge do backend.

---

## 7. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 e logic.md fechados). Pontos de
atenção de implementação — não impedem abrir a issue:

- **`booking` ainda não mergeado — e a contagem real nem foi implementada
  ainda.** O plano original era um `SeatCountsRepository` com SQL real contra
  `tickets`/`reservations` e degradação suave pra 0 se as tabelas não
  existissem. O que existe hoje é mais simples: `SeatCounts(0, 0)` fixo,
  direto no código dos dois usecases, sem repositório e sem SQL nenhum — na
  prática a regra "não exclui sessão vendida" fica permissiva em runtime até
  `booking` entrar **e** até essa contagem virar uma consulta real. O
  invariante de domínio (`Session.deactivate`, chamado pelo usecase só depois
  de checar `tickets_sold > 0`) já está pronto para receber a contagem real
  quando ela existir.
- **`_DEFAULT_SHOW_IMAGES` aponta para arquivos que ainda não existem.**
  `web.ludens/public/images/show-placeholders/{1..6}.jpg` precisam ser
  fornecidos (design/PO) antes de implementar — não é algo que o backend gera.
  Débito técnico: upload real de imagem pelo admin, fora de escopo desta
  entrega.
