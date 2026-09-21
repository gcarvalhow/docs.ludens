---
status: done
spec: catalog-genre
surface: backend
created_at: 2026-09-11
---

# Gêneros do catálogo — Backend

**Resumo:** gênero de espetáculo deixa de ser texto livre e vira entidade própria (`Genre`) no módulo `catalog`, com CRUD completo pelo admin (criar, editar, excluir) e nome como identidade única (sem duplicata ativa). `Show.genre` (texto livre) virou `Show.genre_id` (FK); as respostas que mostram um espetáculo continuam trazendo o nome do gênero como campo plano (`genre: str`), ao lado do `genre_id`, não como objeto aninhado — só a listagem dedicada de gêneros (`GET /catalog/genres`) devolve o objeto `{id, name}`.
**RF:** fortalece RF01 · ajusta critério de aceite de RF08 · **RN:** nenhuma RN01–RN05 diretamente — regra própria da feature (nome é a identidade única do gênero) · **Módulo backend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-genre/integration.md`

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho |
| --- | --- | --- |
| 1 | domain | `src/app/modules/catalog/domain/events/domain_events.py` (`GenreCreated`, `GenreUpdated`, `GenreDeactivated`) |
| 2 | domain | `src/app/modules/catalog/domain/aggregates/genre.py` |
| 3 | application | `src/app/modules/catalog/application/schemas/request.py` (`GenreRequest`) |
| 4 | application | `src/app/modules/catalog/application/schemas/response.py` (`GenreResponse`, `genre`/`genre_id` em `AdminShowSummaryResponse`/`ShowCardResponse`/`ShowDetailResponse`) |
| 5 | application | `src/app/modules/catalog/application/mappers/genre_mapper.py` |
| 6 | application | `src/app/modules/catalog/application/usecases/genre_usecase.py` |
| 7 | infrastructure | `src/app/modules/catalog/infrastructure/repositories/genre_repository.py` |
| 8 | api | `src/app/modules/catalog/api/routers/genre_router.py` |
| 9 | migration | `src/migrations/versions/0004_catalog_genre.py` |

Não existe `Genre.icon`, ícone de gênero não foi implementado. Não existe
`admin_catalog_router.py`: todas as rotas de gênero (admin e leitura pública)
vivem em `genre_router.py`, prefixo `/catalog/genres`. Sem evento consumido
por outro módulo, sem handler de outbox, sem `dependencies.py` novo.

## 2. Código

### `src/app/modules/catalog/domain/events/domain_events.py` — trecho de gênero

```python
@dataclass(frozen=True)
class GenreCreated(DomainEvent):
    id: UUID = field(kw_only=True)
    name: str = field(kw_only=True)

@dataclass(frozen=True)
class GenreUpdated(DomainEvent):
    id: UUID = field(kw_only=True)
    name: str = field(kw_only=True)

@dataclass(frozen=True)
class GenreDeactivated(DomainEvent):
    id: UUID = field(kw_only=True)
```

### `src/app/modules/catalog/domain/aggregates/genre.py`

```python
from __future__ import annotations

from uuid import uuid4
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain.model import Model
from app.core.domain.events import DomainEvent
from app.core.domain.aggregate import AggregateRoot

from app.modules.catalog.domain.events import GenreCreated, GenreDeactivated, GenreUpdated

class Genre(AggregateRoot, Model):
    __tablename__ = "genres"

    name: Mapped[str] = mapped_column(String(80), nullable=False)

    @classmethod
    def create(cls, *, name: str) -> "Genre":
        genre = cls()
        genre.id = uuid4()

        genre.raise_event(lambda v: GenreCreated(version=v, id=genre.id, name=name))
        return genre

    def update(self, *, name: str) -> None:
        self.raise_event(lambda v: GenreUpdated(version=v, id=self.id, name=name))

    def deactivate(self) -> None:
        self.raise_event(lambda v: GenreDeactivated(version=v, id=self.id))

    def _apply(self, event: DomainEvent) -> None:
        handler = getattr(self, f"_when_{type(event).__name__}", None)
        if handler is not None:
            handler(event)

    def _when_GenreCreated(self, e: GenreCreated) -> None:
        self.name = e.name
        self.is_active = True

    def _when_GenreUpdated(self, e: GenreUpdated) -> None:
        self.name = e.name

    def _when_GenreDeactivated(self, _event: GenreDeactivated) -> None:
        self.is_active = False
```

Sem normalização de acento/maiúscula: o nome é comparado como string exata
(mais simples do que o desenho original, que previa uma forma normalizada
sem acento como identidade). A checagem de duplicata roda no usecase.

### `src/app/modules/catalog/application/schemas/request.py` — `GenreRequest`

```python
class GenreRequest(BaseModel):
    name: str = Field(min_length=1, max_length=80)

    @field_validator("name")
    @classmethod
    def _strip(cls, value: str) -> str:
        stripped = value.strip()
        if not stripped:
            raise ValueError("nome do gênero não pode ser vazio")
        return stripped
```

Sem campo `icon`.

### `src/app/modules/catalog/application/schemas/response.py` — trechos de gênero

```python
class GenreResponse(BaseModel):
    id: UUID
    name: str

class AdminShowSummaryResponse(BaseModel):
    id: UUID
    title: str
    synopsis: str
    image_url: str
    genre_id: UUID
    genre: str
    status: Literal["draft", "published"]

class ShowCardResponse(BaseModel):
    id: UUID
    title: str
    synopsis_short: str
    image_url: str
    genre_id: UUID
    genre: str
    upcoming_dates: list[datetime]
    price_min: float
    price_max: float

class ShowDetailResponse(BaseModel):
    id: UUID
    title: str
    synopsis: str
    image_url: str
    genre_id: UUID
    genre: str
    sessions: list[SessionSummaryResponse]
```

`AdminShowResponse(AdminShowSummaryResponse)` herda os mesmos dois campos.
Nenhuma dessas quatro respostas embute `GenreResponse` como objeto; só
`GET /catalog/genres` devolve `GenreResponse{id, name}`.

### `src/app/modules/catalog/application/mappers/genre_mapper.py`

```python
from app.modules.catalog.domain.aggregates import Genre
from app.modules.catalog.application.schemas.response import GenreResponse

def genre_response(genre: Genre) -> GenreResponse:
    return GenreResponse(id=genre.id, name=genre.name)
```

### `src/app/modules/catalog/application/usecases/genre_usecase.py`

```python
from __future__ import annotations

from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain.errors import ConflictError, NotFoundError

from app.modules.catalog.domain.aggregates import Genre
from app.modules.catalog.application.schemas.request import GenreRequest
from app.modules.catalog.application.schemas.response import GenreResponse
from app.modules.catalog.application.mappers import genre_response
from app.modules.catalog.infrastructure.repositories import GenreRepository, ShowRepository

class GenreUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._genre_repository = GenreRepository(session)
        self._show_repository = ShowRepository(session)

    async def create_genre(self, req: GenreRequest) -> GenreResponse:
        if await self._genre_repository.exists_by("name", req.name):
            raise ConflictError("Já existe um gênero ativo com este nome.")

        genre = Genre.create(name=req.name)
        await self._genre_repository.save(genre)
        return genre_response(genre)

    async def update_genre(self, genre_id: UUID, req: GenreRequest) -> GenreResponse:
        genre = await self._require_genre(genre_id)

        if req.name != genre.name and await self._genre_repository.exists_by("name", req.name):
            raise ConflictError("Já existe um gênero ativo com este nome.")

        genre.update(name=req.name)
        await self._genre_repository.save(genre)
        return genre_response(genre)

    async def deactivate_genre(self, genre_id: UUID) -> None:
        genre = await self._require_genre(genre_id)

        if await self._show_repository.exists_by("genre_id", genre.id):
            raise ConflictError(
                "Existem espetáculos usando este gênero; altere-os para outro gênero antes de excluir."
            )

        genre.deactivate()
        await self._genre_repository.save(genre)

    async def list_genres(self) -> list[GenreResponse]:
        genres = await self._genre_repository.find_all(order_by=["name"])
        return [genre_response(g) for g in genres]

    async def _require_genre(self, genre_id: UUID) -> Genre:
        genre = await self._genre_repository.find_by("id", genre_id)
        if genre is None:
            raise NotFoundError("Gênero não encontrado.")

        return genre
```

### `src/app/modules/catalog/infrastructure/repositories/genre_repository.py`

```python
from __future__ import annotations

from typing import Sequence
from uuid import UUID

from sqlalchemy import select

from app.modules.catalog.domain.aggregates import Genre
from app.core.infrastructure.repositories import AggregateRepository

class GenreRepository(AggregateRepository[Genre]):
    model = Genre

    async def find_all_by_ids(self, ids: Sequence[UUID]) -> list[Genre]:
        # Sem filtro is_active: resolve o nome de exibição de genre_ids que um
        # show já referencia, mesmo que o gênero tenha sido desativado depois
        # — é um estado legítimo, não uma falha de busca.
        if not ids:
            return []

        result = await self._session.execute(select(Genre).where(Genre.id.in_(ids)))
        return list(result.scalars().all())
```

### `src/app/modules/catalog/api/routers/genre_router.py`

```python
from __future__ import annotations

from uuid import UUID

from fastapi import APIRouter, Depends, Response
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.modules.identity.dependencies import require_admin

from app.modules.catalog.application.schemas.request import GenreRequest
from app.modules.catalog.application.schemas.response import GenreResponse
from app.modules.catalog.application.usecases.genre_usecase import GenreUseCase

router = APIRouter(prefix="/catalog/genres", tags=["03.Catalog - Genre"])

@router.post("", response_model=GenreResponse, status_code=201, dependencies=[Depends(require_admin)])
async def create(
    body: GenreRequest,
    session: AsyncSession = Depends(get_db)
) -> GenreResponse:
    return await GenreUseCase(session).create_genre(body)

@router.put("/{genre_id}", response_model=GenreResponse, dependencies=[Depends(require_admin)])
async def update(
    genre_id: UUID,
    body: GenreRequest,
    session: AsyncSession = Depends(get_db)
) -> GenreResponse:
    return await GenreUseCase(session).update_genre(genre_id, body)

@router.delete("/{genre_id}", status_code=204, dependencies=[Depends(require_admin)])
async def delete(
    genre_id: UUID,
    session: AsyncSession = Depends(get_db)
) -> Response:
    await GenreUseCase(session).deactivate_genre(genre_id)
    return Response(status_code=204)

@router.get("", response_model=list[GenreResponse])
async def list_genres(session: AsyncSession = Depends(get_db)) -> list[GenreResponse]:
    return await GenreUseCase(session).list_genres()
```

Leitura (`GET`) pública; criação, edição e exclusão atrás de `require_admin`.

### `src/migrations/versions/0004_catalog_genre.py`

Cria a tabela `genres` (índice único parcial `uq_genres_name_active` só entre
ativos), adiciona `shows.genre_id`, faz o backfill de um `Genre` por valor
distinto hoje em `shows.genre` (dedupe por string exata, sem normalização),
torna `genre_id` obrigatório, cria a FK e o índice, e remove a coluna antiga
`shows.genre`. Revisa `0003_identity_user_lifecycle`.

## 3. Onde cada regra de negócio entra

| Regra | Arquivo · função | Como |
| --- | --- | --- |
| Nome duplicado (ativo) recusado | `genre_usecase.py` · `create_genre`/`update_genre` | `exists_by("name", ...)` antes de salvar; backstop no índice único parcial `uq_genres_name_active` |
| Só admin cria/edita/exclui gênero | `genre_router.py` | `Depends(require_admin)` nas três rotas de escrita |
| Não exclui gênero em uso | `genre_usecase.py` · `deactivate_genre` | `ShowRepository.exists_by("genre_id", ...)` antes de desativar → `ConflictError` |
| Espetáculo sempre referencia gênero existente | `show.py` (`genre_id` FK `NOT NULL`) | garantia estrutural |
| Exclusão é lógica (`is_active`), não `DELETE` real | `genre.py` · `deactivate()` | igual ao resto do domínio |

## 4. DevOps

Não aplicável — nenhuma variável de ambiente nova, nenhum segredo de CI.

## 5. Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-genre
commit 1  feat(catalog): modelar Genre (CRUD) e trocar Show.genre por genre_id
commit 2  feat(catalog): usecase e schemas de gênero
commit 3  feat(catalog): repositório de gênero
commit 4  feat(catalog): rotas de gênero e migration de dados
```

## 6. Ordem entre as superfícies

Mergeado. Frontend consumiu o contrato já com CRUD completo.

## 7. Débitos técnicos registrados

Nenhum específico desta fatia.

## 8. Ajustes feitos no `integration.md`

O `integration.md` original (`status: alvo`) previa `Genre{name, icon}`,
criação apenas (sem `PUT`/`DELETE`) e rotas de gênero dentro de um
`admin_catalog_router.py` separado do `show_router.py` público. O que
mergeou: **sem `icon`** em nenhum lugar; CRUD completo; todas as rotas de
gênero (leitura pública e escrita admin) em `genre_router.py`,
`/catalog/genres`. A resposta de espetáculo (`AdminShowSummaryResponse`,
`ShowCardResponse`, `ShowDetailResponse`) manteve `genre` como campo de texto
plano (mais `genre_id`) em vez de virar um objeto `GenreResponse` aninhado —
o `integration.md` original previa a troca para objeto, isso não aconteceu.
`integration.md` foi atualizado pra `status: canônico` com essas correções.
