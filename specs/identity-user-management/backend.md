---
status: done
spec: identity-user-management
surface: backend
created_at: 2026-09-17
updated_at: 2026-09-21
---

# Gestão de conta de usuário — Backend

> **Nota de auditoria (2026-09-21):** este documento foi escrito depois do
> código já estar mergeado — não nasceu junto com a implementação, como é o
> padrão TBD. `user_router.py` ganhou as rotas desta feature (perfil, troca de
> e-mail, exclusão de conta, listagem admin) numa fatia de trabalho anterior
> que nunca gerou `backend.md`/`integration.md` próprios (ver a nota de estado
> real em `spec.md` e o débito registrado em `identity-auth/backend.md` §7).
> Todo código abaixo foi lido direto de `src/` nesta data — não reconstruído de
> memória. Onde o código diverge do que `logic.md` aprovou, isso está marcado
> explicitamente em vez de silenciado — ver §4.

**Resumo:** estende o aggregate `User` (já existente, de `identity-auth`) com
cinco métodos novos — `update_profile`, `request_email_change`,
`apply_email_change`, `request_account_deletion`, `deactivate` — e duas
entidades filhas de token de uso único (`EmailChangeToken`,
`AccountDeletionToken`), seguindo exatamente o mesmo padrão de
`PasswordResetToken`. Expõe 6 rotas novas/estendidas em `user_router.py`
(já existente, dono também de cadastro e leitura por id — ver
`identity-auth/backend.md`). Consome o módulo `notification`, já implementado,
via 3 handlers de outbox novos.
**RF:** RF09 (edição de perfil não tem RF numerado próprio — ver `spec.md` §5)
**RN:** — (toca RNF01 na listagem administrativa — ver §4)
**Módulo backend:** `identity` (+ `notification` para os handlers de e-mail)
**Contrato:** `docs.ludens/specs/identity-user-management/integration.md`
**Carregar antes:** skill `backend-architecture` (todos os `references/`),
`docs.ludens/backend/overview.md`, `docs.ludens/backend/conventions.md`,
`docs.ludens/backend/security/authentication.md`,
`identity-auth/backend.md` (dono do aggregate `User` e do restante de
`user_router.py`).

---

## 1. Arquivos (ordem de dependência)

Só os arquivos que esta feature cria ou edita. `domain/value_objects/{cpf,email}.py`,
`application/schemas/response.py` (`UserResponse` já existia, sem alteração de
forma), `infrastructure/services/{password_service,token_service}.py`,
`modules/identity/shared/{session,cookies}.py` e `core/domain/errors.py` são
reaproveitados sem alteração — donos em `identity-auth/backend.md`.
`core/shared/pagination.py` e `core/infrastructure/queries/pagination.py`
(`Page`, `PaginationParams`, `make_pagination_params`, `paginate`) também são
reaproveitados sem alteração — introduzidos por uma feature de `catalog`
anterior (nenhuma spec de `catalog` os lista formalmente como "novo" ainda;
sinalizado aqui só como nota, fora do escopo desta auditoria).

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | domain | `src/app/modules/identity/domain/entities/email_change_token.py` | novo |
| 2 | domain | `src/app/modules/identity/domain/entities/account_deletion_token.py` | novo |
| 3 | domain | `src/app/modules/identity/domain/entities/__init__.py` | editar |
| 4 | domain | `src/app/modules/identity/domain/events/domain_events.py` | editar |
| 5 | domain | `src/app/modules/identity/domain/events/__init__.py` | editar |
| 6 | domain | `src/app/modules/identity/domain/aggregates/user.py` | editar |
| 7 | infrastructure | `src/app/modules/identity/infrastructure/repositories/email_change_token_repository.py` | novo |
| 8 | infrastructure | `src/app/modules/identity/infrastructure/repositories/account_deletion_token_repository.py` | novo |
| 9 | infrastructure | `src/app/modules/identity/infrastructure/repositories/user_repository.py` | editar |
| 10 | infrastructure | `src/app/modules/identity/infrastructure/repositories/__init__.py` | editar |
| 11 | application | `src/app/modules/identity/application/schemas/request.py` | editar |
| 12 | application | `src/app/modules/identity/application/mappers/user_mapper.py` | novo |
| 13 | application | `src/app/modules/identity/application/mappers/__init__.py` | novo |
| 14 | application | `src/app/modules/identity/application/usecases/user_usecase.py` | editar |
| 15 | api | `src/app/modules/identity/api/routers/user_router.py` | editar |
| 16 | migration | `src/migrations/versions/0003_identity_user_lifecycle.py` | novo |
| 17 | migration | `src/migrations/env.py` | editar |
| 18 | notification (módulo diferente) | `src/app/modules/notification/handlers.py` | editar |
| 19 | notification (módulo diferente) | `src/app/modules/notification/shared/templates.py` | editar |

Os itens 18–19 pertencem a `notification`, não a `identity` — incluídos aqui
porque são os handlers de outbox que consomem os três eventos que este
recorte de `User` levanta (`EmailChangeRequested`, `EmailChanged`,
`AccountDeletionRequested`); é exatamente o cruzamento de módulo via `Event`
que `references/07-outbox-and-event-flow.md` descreve — nenhum import direto
de `domain`/`infrastructure` entre `identity` e `notification`.

---

## 2. Código

### 1. `src/app/modules/identity/domain/entities/email_change_token.py` — novo

```python
from datetime import datetime
from uuid import UUID

from sqlalchemy import DateTime, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain import Model

class EmailChangeToken(Model):
    __tablename__ = "email_change_tokens"

    user_id: Mapped[UUID] = mapped_column(nullable=False, index=True)
    new_email: Mapped[str] = mapped_column(String(254), nullable=False)
    token_hash: Mapped[str] = mapped_column(String(64), nullable=False, index=True)
    expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    used_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    def is_valid(self, now: datetime) -> bool:
        return self.used_at is None and self.expires_at > now

    def consume(self, now: datetime) -> None:
        self.used_at = now
```

Mesmo formato de `PasswordResetToken` (`identity-auth`), com um campo a mais:
`new_email` — o endereço pretendido fica gravado na própria linha do token,
não só no evento em claro. Diferente de `PasswordResetToken.consume`, aqui
`consume` **não** levanta `GoneError` sozinho — quem decide isso é o usecase
via `is_valid` (ver arquivo 14, nota abaixo).

### 2. `src/app/modules/identity/domain/entities/account_deletion_token.py` — novo

```python
from datetime import datetime
from uuid import UUID

from sqlalchemy import DateTime, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain import Model

class AccountDeletionToken(Model):
    __tablename__ = "account_deletion_tokens"

    user_id: Mapped[UUID] = mapped_column(nullable=False, index=True)
    token_hash: Mapped[str] = mapped_column(String(64), nullable=False, index=True)
    expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    used_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    def is_valid(self, now: datetime) -> bool:
        return self.used_at is None and self.expires_at > now

    def consume(self, now: datetime) -> None:
        self.used_at = now
```

Idêntico a `EmailChangeToken` sem o campo `new_email` — não precisa carregar
nada além do vínculo com o `user_id`.

### 3. `src/app/modules/identity/domain/entities/__init__.py` — editar

```python
from .account_deletion_token import AccountDeletionToken
from .email_change_token import EmailChangeToken
from .password_reset_token import PasswordResetToken
from .refresh_token import RefreshToken

__all__ = ["AccountDeletionToken", "EmailChangeToken", "PasswordResetToken", "RefreshToken"]
```

### 4. `src/app/modules/identity/domain/events/domain_events.py` — editar

```python
from uuid import UUID
from datetime import datetime
from dataclasses import dataclass, field

from app.core.domain import DomainEvent

@dataclass(frozen=True)
class UserRegistered(DomainEvent):
    id: UUID = field(kw_only=True)
    name: str = field(kw_only=True)
    cpf: str = field(kw_only=True)
    email: str = field(kw_only=True)
    password_hash: str = field(kw_only=True)
    is_admin: bool = field(kw_only=True)
    security_stamp: UUID = field(kw_only=True)

@dataclass(frozen=True)
class UserPasswordChanged(DomainEvent):
    id: UUID = field(kw_only=True)
    password_hash: str = field(kw_only=True)

@dataclass(frozen=True)
class UserSecurityStampRotated(DomainEvent):
    id: UUID = field(kw_only=True)
    security_stamp: UUID = field(kw_only=True)

@dataclass(frozen=True)
class PasswordResetRequested(DomainEvent):
    id: UUID = field(kw_only=True)
    email: str = field(kw_only=True)
    token: str = field(kw_only=True)
    expires_at: datetime = field(kw_only=True)

@dataclass(frozen=True)
class UserProfileUpdated(DomainEvent):
    id: UUID = field(kw_only=True)
    name: str = field(kw_only=True)

@dataclass(frozen=True)
class EmailChangeRequested(DomainEvent):
    id: UUID = field(kw_only=True)
    old_email: str = field(kw_only=True)
    new_email: str = field(kw_only=True)
    token: str = field(kw_only=True)
    expires_at: datetime = field(kw_only=True)

@dataclass(frozen=True)
class EmailChanged(DomainEvent):
    id: UUID = field(kw_only=True)
    new_email: str = field(kw_only=True)

@dataclass(frozen=True)
class AccountDeletionRequested(DomainEvent):
    id: UUID = field(kw_only=True)
    email: str = field(kw_only=True)
    token: str = field(kw_only=True)
    expires_at: datetime = field(kw_only=True)

@dataclass(frozen=True)
class UserDeactivated(DomainEvent):
    id: UUID = field(kw_only=True)
```

Ponto que sustenta a regra "link vai sempre para o e-mail atual" (§3 de
`logic.md`): `EmailChangeRequested.old_email` é capturado no momento em que o
evento é levantado — antes de qualquer mutação — então carrega sempre o
e-mail **anterior** à troca, nunca o novo. `EmailChanged` carrega só
`new_email`, usado pelo aviso de cortesia.

### 5. `src/app/modules/identity/domain/events/__init__.py` — editar

```python
from .domain_events import (
    AccountDeletionRequested,
    EmailChangeRequested,
    EmailChanged,
    PasswordResetRequested,
    UserDeactivated,
    UserPasswordChanged,
    UserProfileUpdated,
    UserRegistered,
    UserSecurityStampRotated,
)

__all__ = [
    "AccountDeletionRequested",
    "EmailChangeRequested",
    "EmailChanged",
    "PasswordResetRequested",
    "UserDeactivated",
    "UserPasswordChanged",
    "UserProfileUpdated",
    "UserRegistered",
    "UserSecurityStampRotated",
]
```

### 6. `src/app/modules/identity/domain/aggregates/user.py` — editar

```python
from __future__ import annotations

from datetime import datetime
from uuid import UUID, uuid4

from sqlalchemy import Boolean, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain import AggregateRoot, DomainEvent, Model
from app.modules.identity.domain.events import (
    AccountDeletionRequested,
    EmailChangeRequested,
    EmailChanged,
    PasswordResetRequested,
    UserDeactivated,
    UserPasswordChanged,
    UserProfileUpdated,
    UserRegistered,
    UserSecurityStampRotated,
)

from app.modules.identity.domain.value_objects import CPF, Email

class User(AggregateRoot, Model):
    __tablename__ = "users"

    name: Mapped[str] = mapped_column(String(120), nullable=False)
    cpf: Mapped[str] = mapped_column(String(11), nullable=False)
    email: Mapped[str] = mapped_column(String(254), nullable=False)
    password_hash: Mapped[str] = mapped_column(String(60), nullable=False)

    is_admin: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    security_stamp: Mapped[UUID] = mapped_column(default=uuid4, nullable=False)

    @classmethod
    def register(cls, name: str, cpf: CPF, email: Email, password_hash: str, is_admin: bool = False) -> User:
        user = cls()
        user.raise_event(
            lambda v: UserRegistered(
                version=v,
                id=uuid4(),
                name=name,
                cpf=cpf.value,
                email=email.value,
                password_hash=password_hash,
                is_admin=is_admin,
                security_stamp=uuid4(),
            )
        )

        return user

    def change_password(self, new_hash: str) -> None:
        self.raise_event(
            lambda v: UserPasswordChanged(version=v, id=self.id, password_hash=new_hash)
        )
        self.rotate_security_stamp()

    def reset_password(self, new_hash: str) -> None:
        self.raise_event(
            lambda v: UserPasswordChanged(version=v, id=self.id, password_hash=new_hash)
        )
        self.rotate_security_stamp()

    def rotate_security_stamp(self) -> None:
        self.raise_event(
            lambda v: UserSecurityStampRotated(version=v, id=self.id, security_stamp=uuid4())
        )

    def request_password_reset(self, token: str, expires_at: datetime) -> None:
        self.raise_event(
            lambda v: PasswordResetRequested(
                version=v, id=self.id, email=self.email, token=token, expires_at=expires_at
            )
        )

    def update_profile(self, name: str) -> None:
        self.raise_event(lambda v: UserProfileUpdated(version=v, id=self.id, name=name))

    def request_email_change(self, new_email: str, token: str, expires_at: datetime) -> None:
        self.raise_event(
            lambda v: EmailChangeRequested(
                version=v, id=self.id, old_email=self.email, new_email=new_email,
                token=token, expires_at=expires_at,
            )
        )

    def apply_email_change(self, new_email: str) -> None:
        self.raise_event(lambda v: EmailChanged(version=v, id=self.id, new_email=new_email))
        self.rotate_security_stamp()

    def request_account_deletion(self, token: str, expires_at: datetime) -> None:
        self.raise_event(
            lambda v: AccountDeletionRequested(
                version=v, id=self.id, email=self.email, token=token, expires_at=expires_at
            )
        )

    def deactivate(self) -> None:
        self.raise_event(lambda v: UserDeactivated(version=v, id=self.id))
        self.rotate_security_stamp()

    def _apply(self, event: DomainEvent) -> None:
        handler = getattr(self, f"_when_{type(event).__name__}", None)
        if handler:
            handler(event)

    def _when_UserRegistered(self, e: UserRegistered) -> None:
        self.id = e.id
        self.name = e.name
        self.cpf = e.cpf
        self.email = e.email
        self.password_hash = e.password_hash
        self.is_admin = e.is_admin
        self.security_stamp = e.security_stamp

    def _when_UserPasswordChanged(self, e: UserPasswordChanged) -> None:
        self.password_hash = e.password_hash

    def _when_UserSecurityStampRotated(self, e: UserSecurityStampRotated) -> None:
        self.security_stamp = e.security_stamp

    def _when_UserProfileUpdated(self, e: UserProfileUpdated) -> None:
        self.name = e.name

    def _when_EmailChanged(self, e: EmailChanged) -> None:
        self.email = e.new_email

    def _when_UserDeactivated(self, _event: UserDeactivated) -> None:
        self.is_active = False
```

Conformidade com `references/03-domain-layer.md`: nenhum método seta
`self.email`/`self.name`/`self.is_active` fora de um `_when_*`. `deactivate()`
é o "encerramento", modelado exatamente como o soft delete padrão do core
(`is_active = False`), mas via evento de domínio — não um
`user.is_active = False` solto. `apply_email_change` e `deactivate` chamam
`rotate_security_stamp()` internamente, reaproveitando o mesmo mecanismo que
`identity-auth` já usa em `logout`/`change_password`/`reset_password` — é
esse `security_stamp` novo, comparado em `get_current_user`
(`identity/dependencies.py`, inalterado por esta feature), que derruba as
sessões (ver §4).

Nem `request_email_change`, `request_account_deletion` nem `deactivate`
verificam pré-condição de negócio (e-mail duplicado, último administrador,
reserva aberta) dentro do aggregate — consistente com a regra de
`references/03`: aggregate não levanta erro com mensagem de negócio, só quem
lê o estado (o usecase) decide o `raise`.

### 7. `src/app/modules/identity/infrastructure/repositories/email_change_token_repository.py` — novo

```python
from datetime import datetime
from uuid import UUID

from app.core.infrastructure.repositories import BaseRepository
from app.modules.identity.domain.entities.email_change_token import EmailChangeToken

class EmailChangeTokenRepository(BaseRepository[EmailChangeToken]):
    model = EmailChangeToken

    async def invalidate_all_for_user(self, user_id: UUID, now: datetime) -> None:
        for token in await self.find_all_by(user_id=user_id):
            if token.used_at is None:
                token.used_at = now
```

### 8. `src/app/modules/identity/infrastructure/repositories/account_deletion_token_repository.py` — novo

```python
from datetime import datetime
from uuid import UUID

from app.core.infrastructure.repositories import BaseRepository
from app.modules.identity.domain.entities.account_deletion_token import AccountDeletionToken

class AccountDeletionTokenRepository(BaseRepository[AccountDeletionToken]):
    model = AccountDeletionToken

    async def invalidate_all_for_user(self, user_id: UUID, now: datetime) -> None:
        for token in await self.find_all_by(user_id=user_id):
            if token.used_at is None:
                token.used_at = now
```

Mesmo padrão de `PasswordResetTokenRepository.invalidate_all_for_user`
(`identity-auth`) e `RefreshTokenRepository.deactivate_all_for_user` — método
próprio porque é uma consulta multi-linha, não um lookup de campo único.

### 9. `src/app/modules/identity/infrastructure/repositories/user_repository.py` — editar

```python
from sqlalchemy import func, select

from app.modules.identity.domain.aggregates import User
from app.core.infrastructure.repositories import AggregateRepository
from app.core.infrastructure.queries import paginate

class UserRepository(AggregateRepository[User]):
    model = User

    async def list_paginated(self, *, page: int, size: int) -> tuple[list[User], int]:
        stmt = select(User).where(User.is_active.is_(True)).order_by(User.created_at.desc())
        rows, total = await paginate(self._session, stmt, page=page, size=size)

        return [row[0] for row in rows], total

    async def count_active_admins(self) -> int:
        result = await self._session.execute(
            select(func.count(User.id)).where(User.is_admin.is_(True), User.is_active.is_(True))
        )
        return int(result.scalar_one())
```

`list_paginated` filtra `is_active` explicitamente mesmo sabendo que toda
leitura genérica de `BaseRepository` já filtra por padrão — aqui é
`select()` cru (não passa por `find_all`), então o filtro precisa ser
reafirmado à mão. `count_active_admins` é a contagem que sustenta a trava do
"último administrador" (§4).

### 10. `src/app/modules/identity/infrastructure/repositories/__init__.py` — editar

```python
from .account_deletion_token_repository import AccountDeletionTokenRepository
from .email_change_token_repository import EmailChangeTokenRepository
from .password_reset_token_repository import PasswordResetTokenRepository
from .refresh_token_repository import RefreshTokenRepository
from .user_repository import UserRepository

__all__ = [
    "AccountDeletionTokenRepository",
    "EmailChangeTokenRepository",
    "PasswordResetTokenRepository",
    "RefreshTokenRepository",
    "UserRepository",
]
```

### 11. `src/app/modules/identity/application/schemas/request.py` — editar

```python
from pydantic import BaseModel, Field

class RegisterRequest(BaseModel):
    name: str = Field(min_length=1, max_length=120)
    cpf: str = Field(min_length=11, max_length=11)
    email: str = Field(max_length=254)
    password: str = Field(min_length=8)

class LoginRequest(BaseModel):
    email: str
    password: str

class ChangePasswordRequest(BaseModel):
    current_password: str
    new_password: str = Field(min_length=8)

class ForgotPasswordRequest(BaseModel):
    email: str

class ResetPasswordRequest(BaseModel):
    token: str
    password: str = Field(min_length=8)

class UpdateProfileRequest(BaseModel):
    name: str = Field(min_length=1, max_length=120)

class RequestEmailChangeRequest(BaseModel):
    new_email: str = Field(max_length=254)
```

`UpdateProfileRequest` e `RequestEmailChangeRequest` são os dois schemas
novos desta feature — ambos sem campo `cpf`, o que é a forma como "CPF
imutável, nunca aceito em payload de edição" (§8 de `spec.md`) é garantido
estruturalmente: nem chega a existir um campo pra validar. **Ver débito
técnico em §4** — `RequestEmailChangeRequest` tem só `new_email`, não a
dupla digitação que `logic.md` §3/§8 exige.

### 12. `src/app/modules/identity/application/mappers/user_mapper.py` — novo

```python
from app.modules.identity.domain.aggregates import User
from app.modules.identity.application.schemas.response import UserResponse

def user_response(user: User) -> UserResponse:
    return UserResponse(
        id=user.id, name=user.name, email=user.email, cpf=user.cpf, is_admin=user.is_admin
    )
```

Único ponto de conversão `User` → `UserResponse`, usado tanto por
`get_by_id` (já existia) quanto por `update_profile` e `list_users` (novos).
**Ver débito técnico em §4** — é este mapeamento único e compartilhado que
faz a listagem administrativa devolver `cpf` (proibido por `logic.md`) e não
devolver `created_at` (exigido por `logic.md`).

### 13. `src/app/modules/identity/application/mappers/__init__.py` — novo

```python
from .user_mapper import user_response

__all__ = ["user_response"]
```

### 14. `src/app/modules/identity/application/usecases/user_usecase.py` — editar

```python
from uuid import UUID
from datetime import datetime, timedelta, timezone

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.shared import Page, PaginationParams
from app.core.domain import ConflictError, GoneError, NotFoundError

from app.modules.identity.domain.aggregates import User
from app.modules.identity.domain.value_objects import CPF, Email
from app.modules.identity.domain.entities.email_change_token import EmailChangeToken
from app.modules.identity.domain.entities.account_deletion_token import AccountDeletionToken

from app.modules.identity.application.schemas.request import (
    RegisterRequest,
    RequestEmailChangeRequest,
    UpdateProfileRequest,
)

from app.modules.identity.shared import issue_session
from app.modules.identity.application.schemas.response import TokenResponse, UserResponse

from app.modules.identity.infrastructure.services import PasswordService, TokenService

from app.modules.identity.application.mappers import user_response

from app.modules.identity.infrastructure.repositories import (
    AccountDeletionTokenRepository,
    EmailChangeTokenRepository,
    RefreshTokenRepository,
    UserRepository
)

class UserUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

        self._user_repository = UserRepository(session)
        self._refresh_repository = RefreshTokenRepository(session)
        self._email_change_repository = EmailChangeTokenRepository(session)
        self._account_deletion_repository = AccountDeletionTokenRepository(session)

        self._token_service = TokenService()
        self._password_service = PasswordService()

    async def register(self, req: RegisterRequest) -> tuple[TokenResponse, str]:
        cpf = CPF(req.cpf)
        email = Email(req.email.strip().lower())

        if await self._user_repository.exists_by("cpf", cpf.value):
            raise ConflictError("Este CPF já possui cadastro.")

        if await self._user_repository.exists_by("email", email.value):
            raise ConflictError("Este e-mail já está em uso.")

        user = User.register(req.name, cpf, email, self._password_service.hash(req.password))

        await self._user_repository.save(user)
        return await issue_session(user, self._token_service, self._refresh_repository)

    async def get_by_id(self, user_id: UUID) -> UserResponse:
        user = await self._user_repository.find_by("id", user_id)

        if user is None:
            raise NotFoundError("Usuário não encontrado.")

        return user_response(user)

    async def list_users(self, pagination: PaginationParams) -> Page[UserResponse]:
        users, total = await self._user_repository.list_paginated(
            page=pagination.page, size=pagination.size
        )

        return Page(
            items=[user_response(user) for user in users],
            page=pagination.page,
            size=pagination.size,
            total=total,
        )

    async def update_profile(self, user: User, req: UpdateProfileRequest) -> UserResponse:
        user.update_profile(req.name)
        await self._user_repository.save(user)

        return user_response(user)

    async def request_email_change(self, user: User, req: RequestEmailChangeRequest) -> dict[str, str]:
        new_email = Email(req.new_email.strip().lower())

        if await self._user_repository.exists_by("email", new_email.value):
            raise ConflictError("Este e-mail já está em uso.")

        now = datetime.now(timezone.utc)
        await self._email_change_repository.invalidate_all_for_user(user.id, now)

        raw = self._token_service.new_opaque_token()
        expires_at = now + timedelta(hours=1)

        await self._email_change_repository.save(
            EmailChangeToken(
                user_id=user.id,
                new_email=new_email.value,
                token_hash=self._token_service.hash_opaque(raw),
                expires_at=expires_at,
            )
        )

        user.request_email_change(new_email.value, raw, expires_at)
        await self._user_repository.save(user)

        return {"message": f"Enviamos um link de confirmação para {user.email}."}

    async def confirm_email_change(self, token: str) -> None:
        token_hash = self._token_service.hash_opaque(token)
        record = await self._email_change_repository.find_by("token_hash", token_hash)

        now = datetime.now(timezone.utc)
        if record is None or not record.is_valid(now):
            raise GoneError("Este link não é mais válido, solicite um novo.")

        if await self._user_repository.exists_by("email", record.new_email):
            raise ConflictError("Este e-mail já está em uso.")

        record.consume(now)
        user = await self._user_repository.find_by("id", record.user_id)

        if user is None:
            raise GoneError("Este link não é mais válido, solicite um novo.")

        user.apply_email_change(record.new_email)

        await self._user_repository.save(user)
        await self._refresh_repository.deactivate_all_for_user(user.id)

    async def request_account_deletion(self, user: User) -> dict[str, str]:
        await self._ensure_not_last_admin(user)

        now = datetime.now(timezone.utc)
        await self._account_deletion_repository.invalidate_all_for_user(user.id, now)

        raw = self._token_service.new_opaque_token()
        expires_at = now + timedelta(hours=1)

        await self._account_deletion_repository.save(
            AccountDeletionToken(
                user_id=user.id,
                token_hash=self._token_service.hash_opaque(raw),
                expires_at=expires_at,
            )
        )

        user.request_account_deletion(raw, expires_at)
        await self._user_repository.save(user)

        return {"message": f"Enviamos um link de confirmação para {user.email}."}

    async def confirm_account_deletion(self, token: str) -> None:
        token_hash = self._token_service.hash_opaque(token)
        record = await self._account_deletion_repository.find_by("token_hash", token_hash)

        now = datetime.now(timezone.utc)
        if record is None or not record.is_valid(now):
            raise GoneError("Este link não é mais válido, solicite um novo.")

        record.consume(now)
        user = await self._user_repository.find_by("id", record.user_id)

        if user is None:
            raise GoneError("Este link não é mais válido, solicite um novo.")

        await self._ensure_not_last_admin(user)
        user.deactivate()

        await self._user_repository.save(user)
        await self._refresh_repository.deactivate_all_for_user(user.id)

    async def _ensure_not_last_admin(self, user: User) -> None:
        if user.is_admin and await self._user_repository.count_active_admins() <= 1:
            raise ConflictError(
                "Você é a única pessoa administradora da plataforma. "
                "Convide outra pessoa administradora antes de encerrar sua conta."
            )
```

Pontos de conformidade que valem nota:

- `_ensure_not_last_admin` é chamado **duas vezes** — no pedido e na
  confirmação — mesmo racional da checagem dupla de e-mail duplicado em
  `confirm_email_change`: o estado pode mudar entre pedir e confirmar (outro
  administrador pode ter sido rebaixado/apagado nesse meio-tempo — ainda que,
  hoje, não exista rota nenhuma que rebaixe administrador). Nenhum dos dois
  pontos usa `find_by_id_for_update` — não é operação de disponibilidade de
  assento (RN05 não se aplica aqui), é uma contagem de `is_admin` simples;
  correto não travar a linha.
- `confirm_email_change`/`confirm_account_deletion` chamam
  `self._refresh_repository.deactivate_all_for_user(user.id)` depois de
  `save(user)` — mesma ordem e mesmo mecanismo de "derruba tudo" que
  `identity-auth` usa em `logout`/`change_password`/`reset_password`; nenhuma
  reimplementação.
- Nenhum efeito colateral externo (envio de e-mail) é chamado direto — os
  três `request_*` levantam evento via o aggregate e persistem via
  `AggregateRepository.save` (implícito em `UserRepository`); quem envia é o
  handler de outbox (arquivo 18). Conforme com `references/04` e `references/07`.
- **Débito técnico:** nenhum destes métodos verifica reserva aberta/pagamento
  em processamento antes de aceitar o pedido de exclusão — ver §4.

### 15. `src/app/modules/identity/api/routers/user_router.py` — editar

```python
from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import APIRouter, Depends, Query, Response, status

from app.core.domain import ForbiddenError
from app.core.shared import Page, PaginationParams, make_pagination_params

from app.dependencies import get_db
from app.modules.identity.dependencies import get_current_user, require_admin

from app.modules.identity.domain.aggregates import User
from app.modules.identity.application.schemas.request import (
    RegisterRequest,
    RequestEmailChangeRequest,
    UpdateProfileRequest,
)
from app.modules.identity.application.schemas.response import TokenResponse, UserResponse

from app.modules.identity.shared import set_refresh_cookie
from app.modules.identity.application.usecases.user_usecase import UserUseCase

router = APIRouter(prefix="/identity/users", tags=["02.Identity - User"])

@router.post("", response_model=TokenResponse, status_code=status.HTTP_201_CREATED)
async def register(
    body: RegisterRequest, 
    response: Response, 
    session: AsyncSession = Depends(get_db)
) -> TokenResponse:
    token, raw_refresh = await UserUseCase(session).register(body)
    set_refresh_cookie(response, raw_refresh)

    return token

@router.patch("", response_model=UserResponse)
async def update(
    body: UpdateProfileRequest, 
    session: AsyncSession = Depends(get_db), 
    current_user: User = Depends(get_current_user)
) -> UserResponse:
    return await UserUseCase(session).update_profile(current_user, body)

@router.post("/email/change", status_code=status.HTTP_202_ACCEPTED)
async def request_email_change(
    body: RequestEmailChangeRequest,
    session: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
) -> dict[str, str]:
    return await UserUseCase(session).request_email_change(current_user, body)

@router.patch("/email/change", status_code=status.HTTP_204_NO_CONTENT)
async def confirm_email_change(
    token: str = Query(...), 
    session: AsyncSession = Depends(get_db)
) -> None:
    await UserUseCase(session).confirm_email_change(token)

@router.post("/deletion", status_code=status.HTTP_202_ACCEPTED)
async def request_deletion(
    session: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
) -> dict[str, str]:
    return await UserUseCase(session).request_account_deletion(current_user)

@router.delete("/deletion", status_code=status.HTTP_204_NO_CONTENT)
async def confirm_deletion(
    token: str = Query(...),
    session: AsyncSession = Depends(get_db)
) -> None:
    await UserUseCase(session).confirm_account_deletion(token)

@router.get("", response_model=Page[UserResponse])
async def list_users(
    pagination: PaginationParams = Depends(make_pagination_params()),
    session: AsyncSession = Depends(get_db),
    _admin: User = Depends(require_admin)
) -> Page[UserResponse]:
    return await UserUseCase(session).list_users(pagination)

@router.get("/{user_id}", response_model=UserResponse)
async def get(user_id: UUID, session: AsyncSession = Depends(get_db), current_user: User = Depends(get_current_user)) -> UserResponse:
    if not current_user.is_admin and current_user.id != user_id:
        raise ForbiddenError("Acesso restrito ao próprio usuário.")

    return await UserUseCase(session).get_by_id(user_id)
```

`list_users` usa `require_admin` (RF08) — `update`, `request_email_change`,
`request_deletion` usam `get_current_user` puro (qualquer pessoa autenticada
edita só a própria conta, porque o usecase só recebe o `current_user` do
`Depends`, nunca um id vindo do corpo). `confirm_email_change` e
`confirm_deletion` **não têm nenhuma dependency de auth** — são rotas
públicas por natureza: quem prova identidade é o token na query string, não
uma sessão. **Ver débito técnico em §4** sobre o token ir na query string em
vez do corpo, e sobre a ordem das rotas `""` vs `"/{user_id}"` (sem colisão
real, mas vale registrar por que).

### 16. `src/migrations/versions/0003_identity_user_lifecycle.py` — novo

```python
"""identity user lifecycle: account_deletion_tokens, email_change_tokens

Revision ID: 0003_identity_user_lifecycle
Revises: 0002_catalog_admin
Create Date: 2026-09-17
"""

from typing import Sequence, Union

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

revision: str = "0003_identity_user_lifecycle"
down_revision: Union[str, None] = "0002_catalog_admin"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

def _model_columns() -> list[sa.Column]:
    # Columns inherited from core Model, the same in every table.
    return [
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("is_active", sa.Boolean(), nullable=False),
    ]

def upgrade() -> None:
    op.create_table(
        "account_deletion_tokens",
        *_model_columns(),
        sa.Column("user_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("token_hash", sa.String(length=64), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("used_at", sa.DateTime(timezone=True), nullable=True),
    )
    op.create_index("ix_account_deletion_tokens_user_id", "account_deletion_tokens", ["user_id"])
    op.create_index("ix_account_deletion_tokens_token_hash", "account_deletion_tokens", ["token_hash"])

    op.create_table(
        "email_change_tokens",
        *_model_columns(),
        sa.Column("user_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("new_email", sa.String(length=254), nullable=False),
        sa.Column("token_hash", sa.String(length=64), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("used_at", sa.DateTime(timezone=True), nullable=True),
    )
    op.create_index("ix_email_change_tokens_user_id", "email_change_tokens", ["user_id"])
    op.create_index("ix_email_change_tokens_token_hash", "email_change_tokens", ["token_hash"])

def downgrade() -> None:
    op.drop_table("email_change_tokens")
    op.drop_table("account_deletion_tokens")
```

Não altera a tabela `users` — nenhuma coluna nova precisou entrar nela;
"conta encerrada" continua sendo só `is_active = False` (a coluna já existia,
herdada de `Model`).

### 17. `src/migrations/env.py` — editar

```python
# Each feature adds its own module's import here.
import app.modules.identity.domain.aggregates.user  # noqa: F401,E402
import app.modules.identity.domain.entities.password_reset_token  # noqa: F401,E402
import app.modules.identity.domain.entities.refresh_token  # noqa: F401,E402
import app.modules.identity.domain.entities.account_deletion_token  # noqa: F401,E402
import app.modules.identity.domain.entities.email_change_token  # noqa: F401,E402
```

### 18. `src/app/modules/notification/handlers.py` — editar (módulo `notification`)

```python
from app.config import settings
from app.outbox.registry import register

from app.modules.notification.infrastructure.services import get_email_service
from app.modules.notification.shared import (
    account_deletion_requested_email,
    email_change_requested_email,
    email_changed_courtesy_email,
    password_reset_email,
)

@register("PasswordResetRequested")
async def handle_password_reset_requested(payload: dict) -> None:
    reset_url = f"{settings.frontend_base_url}/redefinir-senha?token={payload['token']}"
    subject, html_body = password_reset_email(reset_url)

    await get_email_service().send(payload["email"], subject, html_body)

@register("EmailChangeRequested")
async def handle_email_change_requested(payload: dict) -> None:
    confirm_url = f"{settings.frontend_base_url}/confirmar-troca-de-email?token={payload['token']}"
    subject, html_body = email_change_requested_email(confirm_url, payload["new_email"])

    await get_email_service().send(payload["old_email"], subject, html_body)

@register("EmailChanged")
async def handle_email_changed(payload: dict) -> None:
    subject, html_body = email_changed_courtesy_email()

    await get_email_service().send(payload["new_email"], subject, html_body)

@register("AccountDeletionRequested")
async def handle_account_deletion_requested(payload: dict) -> None:
    confirm_url = f"{settings.frontend_base_url}/confirmar-exclusao-de-conta?token={payload['token']}"
    subject, html_body = account_deletion_requested_email(confirm_url)

    await get_email_service().send(payload["email"], subject, html_body)
```

**Este é o ponto que confirma a regra mais sensível da feature**: o handler
de `EmailChangeRequested` envia para `payload["old_email"]` — não
`new_email` — batendo com a decisão de 2026-09-17 registrada em `spec.md`
§8. O handler de `EmailChanged` (aviso de cortesia) envia para
`payload["new_email"]`, depois da troca já confirmada — também correto.
Nenhum dos quatro handlers checa idempotência por marcador próprio (tipo
"e-mail já enviado para este `user_id`+tipo de evento?") — se o relay
reprocessar o mesmo evento (at-least-once, `references/07`), o e-mail sai de
novo. Para e-mail transacional isso é um incômodo, não uma corrupção de
dado — mas diverge do que `references/07` pede ("todo handler precisa ser
idempotente"). Mesmo padrão (mesmo "problema") já existe em
`handle_password_reset_requested`, então não é regressão introduzida por
esta feature — é um débito pré-existente do módulo `notification` que esta
feature herdou sem piorar nem corrigir.

### 19. `src/app/modules/notification/shared/templates.py` — editar (módulo `notification`)

```python
def password_reset_email(reset_url: str) -> tuple[str, str]:
    subject = "Redefinição de senha — Ludens"
    html_body = (
        "<p>Você pediu para redefinir sua senha no Ludens.</p>"
        f'<p><a href="{reset_url}">Clique aqui para escolher uma nova senha</a>. '
        "O link vale por 1 hora e só pode ser usado uma vez.</p>"
        "<p>Se você não pediu essa redefinição, ignore este e-mail — sua senha "
        "continua a mesma.</p>"
    )

    return subject, html_body

def email_change_requested_email(confirm_url: str, new_email: str) -> tuple[str, str]:
    subject = "Confirme a troca de e-mail — Ludens"
    html_body = (
        "<p>Você pediu para trocar o e-mail da sua conta Ludens para "
        f"<strong>{new_email}</strong>.</p>"
        f'<p><a href="{confirm_url}">Clique aqui para confirmar a troca</a>. '
        "O link vale por 1 hora e só pode ser usado uma vez. Ao confirmar, "
        "todas as sessões ativas são encerradas.</p>"
        "<p>Se você não pediu essa troca, ignore este e-mail — nada muda até "
        "que o link seja aberto.</p>"
    )

    return subject, html_body

def email_changed_courtesy_email() -> tuple[str, str]:
    subject = "Seu e-mail foi alterado — Ludens"
    html_body = (
        "<p>O e-mail da sua conta Ludens foi alterado para este endereço.</p>"
        "<p>Se você não reconhece essa mudança, entre em contato com o suporte "
        "o quanto antes.</p>"
    )

    return subject, html_body

def account_deletion_requested_email(confirm_url: str) -> tuple[str, str]:
    subject = "Confirme a exclusão da sua conta — Ludens"
    html_body = (
        "<p>Você pediu para excluir sua conta Ludens.</p>"
        f'<p><a href="{confirm_url}">Clique aqui para confirmar a exclusão</a>. '
        "O link vale por 1 hora e só pode ser usado uma vez. Essa ação não pode "
        "ser desfeita.</p>"
        "<p>Se você não pediu essa exclusão, ignore este e-mail — sua conta "
        "continua ativa.</p>"
    )

    return subject, html_body
```

As URLs de confirmação (`/confirmar-troca-de-email?token=...`,
`/confirmar-exclusao-de-conta?token=...`) apontam para páginas de
**frontend** (`settings.frontend_base_url`), não para a rota de API
diretamente — o link do e-mail é um `GET` de navegador, mas as rotas reais
de confirmação são `PATCH`/`DELETE` (arquivo 15). Isso é intencional e
correto (o frontend media a chamada), mas é um contrato que `web.ludens`
**precisa** respeitar — ver `integration.md` "Impacto de UX".

---

## 3. Onde cada regra de negócio entra

| Regra (`logic.md`) | Arquivo · função | Como | Status |
| --- | --- | --- | --- |
| Nome editável sem confirmação por e-mail | `application/schemas/request.py::UpdateProfileRequest` + `user.py::update_profile` + `user_router.py::update` | `PATCH /identity/users` muda só `name`, sem token/e-mail envolvido | conforme |
| CPF imutável, nunca em payload de edição | `UpdateProfileRequest` (sem campo `cpf`) | omissão estrutural do campo — nada a validar porque não existe | conforme |
| Ninguém edita/vê dado de terceiro nas ações de perfil/e-mail/exclusão | `user_router.py::update/request_email_change/request_deletion` | usecase só recebe `current_user` do `Depends(get_current_user)`, nunca um id do corpo | conforme |
| Troca de e-mail não exige senha, só sessão | `RequestEmailChangeRequest` (sem campo senha) + `Depends(get_current_user)` | omissão estrutural + dependency de auth | conforme |
| Link de troca vai para o **e-mail atual** | `user.py::request_email_change` (`old_email=self.email`, capturado antes da troca) + `notification/handlers.py::handle_email_change_requested` (`payload["old_email"]`) | evento carrega o e-mail antigo; handler envia pra ele | conforme |
| Novo e-mail já em uso → recusado no pedido **e** na confirmação | `user_usecase.py::request_email_change` + `confirm_email_change` (dois `exists_by("email", ...)`) | checagem dupla, cobre corrida entre pedido e confirmação | conforme |
| Confirmar troca → e-mail muda e **todas as sessões caem** | `user.py::apply_email_change` (`rotate_security_stamp`) + `user_usecase.py::confirm_email_change` (`deactivate_all_for_user`) | mesmo mecanismo de `identity-auth` (stamp + refresh tokens) | conforme |
| Aviso de cortesia ao novo endereço após confirmar | `user.py::apply_email_change`→`EmailChanged` + `notification/handlers.py::handle_email_changed` | handler envia pra `payload["new_email"]` | conforme |
| Token de troca de e-mail: uso único, 1h, novo pedido invalida anterior | `EmailChangeToken.is_valid/consume` + `EmailChangeTokenRepository.invalidate_all_for_user` | mesmo padrão de `PasswordResetToken` | conforme |
| Novo endereço informado **duas vezes** no pedido | — | **não implementado** — `RequestEmailChangeRequest` só tem `new_email` | **débito — ver §4** |
| Só o próprio dono encerra a própria conta | `user_router.py::request_deletion/confirm_deletion` | `Depends(get_current_user)`, nenhum id de terceiro aceito | conforme |
| Encerramento só vale após abrir o link | `user_usecase.py::request_account_deletion` (não muta `is_active`) vs. `confirm_account_deletion` (`user.deactivate()`) | dois métodos separados, só o segundo muda estado | conforme |
| Único administrador restante não encerra a própria conta | `user_usecase.py::_ensure_not_last_admin` (chamado no pedido e na confirmação) + `UserRepository.count_active_admins` | `ConflictError` 409 nos dois pontos | conforme |
| Reserva aberta/pagamento em processamento bloqueia o pedido | — | **não implementado** — módulos `booking`/`payment` ainda não existem no backend | **débito — ver §4** |
| Token de exclusão: uso único, 1h, novo pedido invalida anterior | `AccountDeletionToken.is_valid/consume` + `AccountDeletionTokenRepository.invalidate_all_for_user` | mesmo padrão | conforme |
| Confirmar exclusão → conta encerrada, todas as sessões caem | `user.py::deactivate` (`is_active=False` + `rotate_security_stamp`) + `user_usecase.py::confirm_account_deletion` (`deactivate_all_for_user`) | mesmo mecanismo de invalidação de sessão (stamp + refresh tokens) | conforme |
| Conta encerrada não autentica mais, mensagem genérica | `AuthUseCase.login` (`identity-auth`, inalterado) — `find_by("email", ...)` já filtra `is_active=True` por padrão de `BaseRepository` | conta encerrada não é encontrada → mesmo `AuthError("E-mail ou senha inválidos.")` | conforme |
| Histórico (pedidos/ingressos/convites) permanece íntegro | `deactivate()` via soft delete, sem `DELETE FROM` | conforme (vazio hoje — `booking`/`payment`/`identity-admin-invite` ainda não existem para ter histórico a preservar) | conforme, mas sem carga real ainda |
| Só administrador lista contas | `user_router.py::list_users` (`Depends(require_admin)`) | 403 via `ForbiddenError` pra quem não é admin | conforme |
| Listagem: nome, e-mail, papel, data de criação — **nunca CPF** | `application/mappers/user_mapper.py::user_response` + `UserResponse` | `UserResponse` inclui `cpf`, **não** inclui `created_at` | **violação — ver §4** |
| Contas encerradas não aparecem na listagem | `UserRepository.list_paginated` | filtro `where(User.is_active.is_(True))` na query | conforme |
| Listagem/detalhe não habilitam nenhuma escrita sobre terceiro | `user_router.py` (todas as rotas) | nenhuma rota aceita id de terceiro em corpo de escrita | conforme |

---

## 4. Débitos técnicos registrados

1. **Listagem administrativa expõe CPF e não expõe data de criação —
   contradiz `logic.md` §3 diretamente (RNF01).** `GET /identity/users`
   (admin) e `GET /identity/users/{id}` (quando quem pede é admin olhando a
   conta de outra pessoa) devolvem o mesmo `UserResponse` de sempre — que
   inclui `cpf` e não inclui `created_at`. `logic.md` §3 é explícito:
   "A listagem expõe nome, e-mail, papel e data de criação. **Nunca CPF**,
   nunca senha (RNF01)." Hoje o backend faz o oposto do que a regra exige
   num dos dois campos citados. Correção sugerida: um `AdminUserResponse`
   (ou `UserSummaryResponse`) próprio, sem `cpf`, com `created_at`, usado só
   em `list_users` e em `get_by_id` quando `current_user.is_admin and
   current_user.id != user_id`; `UserResponse` completo (com `cpf`)
   continuaria só para "ver o próprio perfil". **Bloqueante para o
   frontend usar a listagem como documentado no produto.**
2. **Troca de e-mail não pede o novo endereço duas vezes.**
   `RequestEmailChangeRequest` tem um campo só (`new_email`). `logic.md` §3
   e §8 são explícitos sobre a dupla digitação ser a mitigação combinada
   para erro de digitação (já que, com a confirmação indo ao e-mail atual,
   nada mais comprova que o endereço novo existe). O aviso de cortesia
   (`EmailChanged` → `notification`) está implementado e cobre parte do
   risco, mas não substitui a dupla digitação que `logic.md` fechou como
   parte da mesma decisão — as duas mitigações foram aprovadas juntas, só
   uma foi implementada.
3. **Nenhuma checagem de reserva aberta/pagamento em processamento antes de
   aceitar o pedido de exclusão de conta.** `logic.md` §3 ("Reserva aberta
   ou pagamento em processamento → o pedido de encerramento é recusado")
   depende de dados que **não existem ainda** — os módulos `booking` e
   `payment` não foram criados neste repositório (confirmado por
   `ls src/app/modules/`: só `identity`, `catalog`, `notification`). Não é
   um bug de implementação — é uma regra que não tem como ser
   implementada até `booking`/`payment` existirem. Fica registrado aqui
   para não ser esquecido quando esses módulos nascerem: `request_account_deletion`
   vai precisar consumir uma dependência exportada de `booking`/`payment`
   (`dependencies.py`, nunca import direto de `domain`/`infrastructure`
   deles — `references/06`) que responda "este usuário tem reserva aberta
   ou pagamento pendente?".
4. **Token de confirmação de troca de e-mail e de exclusão de conta
   trafega em query string (`?token=...`) em rotas `PATCH`/`DELETE`, não no
   corpo.** Diferente do padrão já estabelecido por `reset_password`
   (`identity-auth`), que recebe o token no corpo de um `POST`. Duas
   consequências: (a) o link do e-mail não pode ser um simples `<a href>`
   clicável direto pra API — precisa necessariamente passar por uma página
   de frontend que faça a chamada `PATCH`/`DELETE` com o token lido da
   query string (o que `notification/shared/templates.py` já assume,
   apontando pra `frontend_base_url`, não pra API); (b) `logic.md` §3 exige
   "nunca exposto em tela nem em log (RNF01)" para o token — uma query
   string tem mais chance de acabar em log de acesso/proxy/CDN do que um
   campo de corpo de `POST`. Não é um erro fatal (o valor não aparece em
   log de aplicação, só potencialmente em log de infraestrutura fora do
   controle deste código), mas é uma escolha que diverge do padrão mais
   seguro já usado no mesmo módulo para o mesmo tipo de dado.
5. **Handlers de `notification` para os três eventos novos
   (`EmailChangeRequested`, `EmailChanged`, `AccountDeletionRequested`) não
   têm checagem de idempotência própria** (nenhum marcador tipo "já enviei
   pra este `user_id`+evento?"). Sob at-least-once (`references/07`), um
   reprocessamento do relay reenvia o e-mail. Debito pré-existente do
   módulo `notification` (o mesmo já vale para `PasswordResetRequested`),
   não uma regressão desta feature — registrado aqui só porque esta feature
   herdou o padrão sem corrigi-lo.
6. **Boa notícia, não débito:** a nota de `spec.md` (2026-09-17) que
   bloqueava as duas ações confirmadas por e-mail até
   `notification-transactional-email` existir ("essa spec está aprovada mas
   ainda não implementada no backend") **está desatualizada.** O módulo
   `notification` já existe, com `EmailService`/`SMTPEmailService`/
   `ACSEmailService` reais e os quatro handlers registrados e funcionando —
   confirmado lendo `src/app/modules/notification/` por inteiro. A
   dependência que a spec citava como bloqueante já foi resolvida.

---

## 5. Nota de reconciliação

Este documento fecha o gap que `identity-auth/backend.md` §7 (revisão de
2026-09-21) e a nota de estado real de `spec.md` (2026-09-21) já sinalizavam:
o backend de `identity-user-management` está **implementado e mergeado**, não
é mais "spec sem código". Das regras aprovadas em `logic.md`, a esmagadora
maioria está corretamente implementada — inclusive a mais sensível de todas
(o link de troca de e-mail ir para o e-mail **atual**, não o novo, foi
verificado linha a linha em `user.py`, `user_usecase.py` e
`notification/handlers.py`, e está correto). Os pontos que não batem (§4,
itens 1–4) não são hipóteses — foram lidos direto do código. `logic.md`
segue `status: draft` porque pede "nova revisão conjunta de FE/BE" desde o
reescopo de 2026-09-17; esta auditoria é evidência concreta de dois motivos
reais para essa revisão não ter acontecido ainda de fato (a listagem
expondo CPF é o mais grave dos quatro, porque é uma violação direta de RNF01
em produção, não uma lacuna de escopo futuro). Recomenda-se que a próxima
sessão que tocar `identity-user-management` resolva o item 1 antes de
qualquer coisa — é o único dos quatro que expõe dado sensível de terceiro
hoje, em vez de só deixar uma mitigação pendente.
