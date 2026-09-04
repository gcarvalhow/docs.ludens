---
status: done
spec: identity-auth
surface: backend
created_at: 2026-09-03
updated_at: 2026-09-04
---

# Cadastro e autenticação do comprador — Backend

**Resumo:** módulo `identity` com o aggregate `User` (CPF validado, e-mail, hash
bcrypt, `security_stamp`), dual-token JWT (access HS256 + refresh opaco SHA-256
rotacionado a cada uso), recuperação de senha por token de uso único com validade
de 1 hora e envio do link via handler de outbox. Duas usecases — `AuthUseCase`
(sessão: login/refresh/logout/senha) e `UserUseCase` (cadastro, leitura e
listagem paginada de usuário) — expostas em dois routers: 6 rotas em `/auth` e
3 em `/users`. Dependências `get_current_user` / `require_admin` exportadas
para os demais módulos.
**RF:** RF09 · **RN:** — (reforça RNF01) · **Módulo backend:** `identity`
**Contrato:** `docs.ludens/specs/identity-auth/integration.md`
**Carregar antes:** skill `backend-architecture` (todos os `references/`),
`docs.ludens/backend/overview.md`,
`docs.ludens/backend/security/authentication.md`,
`docs.ludens/backend/security/configuration.md`.

Este é o primeiro módulo de negócio do repositório. Ele traz também dois arquivos
de infraestrutura compartilhada que ainda não existiam e que qualquer feature
seguinte reaproveita: `core/domain/errors.py` (`DomainError` + tradução para
HTTP) e `core/shared/schemas.py` (`CamelModel`), além da primeira migration do
Alembic (tabela `events` do outbox).

---

## 1. Arquivos (ordem de dependência)

`infrastructure/` (repositórios e services) vem antes de `application/` — o
usecase depende dos dois, não o contrário. `repositories/__init__.py` e
`services/__init__.py` reexportam as classes do pacote (um só import para as
três repositórios, ou os dois services, em vez de um import por arquivo).

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | core | `src/app/core/domain/errors.py` | novo |
| 2 | core | `src/app/core/shared/schemas.py` | novo |
| 3 | pacotes | `src/app/modules/identity/**/__init__.py` e `src/app/modules/notification/**/__init__.py` | novo |
| 4 | domain | `src/app/modules/identity/domain/value_objects/cpf.py` | novo |
| 5 | domain | `src/app/modules/identity/domain/value_objects/email.py` | novo |
| 6 | domain | `src/app/modules/identity/domain/enumerations/role.py` | novo |
| 7 | domain | `src/app/modules/identity/domain/events/identity_events.py` | novo |
| 8 | domain | `src/app/modules/identity/domain/aggregates/user.py` | novo |
| 9 | domain | `src/app/modules/identity/domain/entities/refresh_token.py` | novo |
| 10 | domain | `src/app/modules/identity/domain/entities/password_reset_token.py` | novo |
| 11 | infrastructure | `src/app/modules/identity/infrastructure/services/password_hasher_service.py` | novo |
| 12 | infrastructure | `src/app/modules/identity/infrastructure/services/token_service.py` | novo |
| 13 | infrastructure | `src/app/modules/identity/infrastructure/services/__init__.py` | novo |
| 14 | infrastructure | `src/app/modules/identity/infrastructure/repositories/user_repository.py` | novo |
| 15 | infrastructure | `src/app/modules/identity/infrastructure/repositories/refresh_token_repository.py` | novo |
| 16 | infrastructure | `src/app/modules/identity/infrastructure/repositories/password_reset_token_repository.py` | novo |
| 17 | infrastructure | `src/app/modules/identity/infrastructure/repositories/__init__.py` | novo |
| 18 | application | `src/app/modules/identity/application/schemas/request.py` | novo |
| 19 | application | `src/app/modules/identity/application/schemas/response.py` | novo |
| 20 | application | `src/app/modules/identity/application/usecases/auth_usecase.py` | novo |
| 21 | application | `src/app/modules/identity/application/usecases/user_usecase.py` | novo |
| 22 | infrastructure | `src/app/modules/notification/infrastructure/services/email_service.py` | novo |
| 23 | dependências | `src/app/modules/notification/dependencies.py` | novo |
| 24 | outbox | `src/app/modules/identity/handlers.py` | novo |
| 25 | api | `src/app/modules/identity/api/routers/auth_router.py` | novo |
| 26 | api | `src/app/modules/identity/api/routers/user_router.py` | novo |
| 27 | api | `src/app/modules/identity/router.py` | novo |
| 28 | api | `src/app/modules/identity/dependencies.py` | novo |
| 29 | migration | `src/migrations/versions/0001_outbox_events.py` | novo |
| 30 | migration | `src/migrations/versions/0002_identity_auth.py` | novo |
| 31 | migration | `src/migrations/env.py` | editar |
| 32 | api | `src/app/main.py` | editar |
| 33 | config | `src/app/config.py` | editar |
| 34 | config | `.env.example` | editar |
| 35 | script | `scripts/seed_admin.py` | novo |
| 36 | config | `pyproject.toml` | editar |
| 37 | devops | `.github/workflows/ci.yml` | editar |

---

## 2. Código

### 1. `src/app/core/domain/errors.py` — novo

```python
from __future__ import annotations

class DomainError(Exception):
    status_code: int = 422
    field: str = "body"

    def __init__(self, message: str, *, status_code: int | None = None, field: str | None = None) -> None:
        super().__init__(message)
        self.message = message

        if status_code is not None:
            self.status_code = status_code
        if field is not None:
            self.field = field

class ConflictError(DomainError):
    status_code = 409

class UnauthorizedError(DomainError):
    status_code = 401

class ForbiddenError(DomainError):
    status_code = 403

class GoneError(DomainError):
    status_code = 410
```

### 2. `src/app/core/shared/schemas.py` — novo

```python
from __future__ import annotations

from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel

class CamelModel(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)
```

### 3. Pacotes — `__init__.py` vazios

`infrastructure/repositories/__init__.py` e `infrastructure/services/__init__.py`
**não** entram aqui — têm conteúdo real (item 13 e 17).

```text
# todos novos, conteúdo vazio
src/app/modules/identity/__init__.py
src/app/modules/identity/domain/__init__.py
src/app/modules/identity/domain/aggregates/__init__.py
src/app/modules/identity/domain/entities/__init__.py
src/app/modules/identity/domain/value_objects/__init__.py
src/app/modules/identity/domain/enumerations/__init__.py
src/app/modules/identity/domain/events/__init__.py
src/app/modules/identity/application/__init__.py
src/app/modules/identity/application/schemas/__init__.py
src/app/modules/identity/application/usecases/__init__.py
src/app/modules/identity/infrastructure/__init__.py
src/app/modules/identity/api/__init__.py
src/app/modules/identity/api/routers/__init__.py
src/app/modules/notification/__init__.py
src/app/modules/notification/infrastructure/__init__.py
src/app/modules/notification/infrastructure/services/__init__.py
```

### 4. `src/app/modules/identity/domain/value_objects/cpf.py` — novo

```python
from __future__ import annotations

import re
from dataclasses import dataclass

from app.core.domain.errors import DomainError

_NON_DIGITS = re.compile(r"\D")

def _is_valid_cpf(digits: str) -> bool:
    if len(digits) != 11 or digits == digits[0] * 11:
        return False

    for length in (9, 10):
        total = sum(int(digits[i]) * (length + 1 - i) for i in range(length))
        check = (total * 10) % 11
        check = 0 if check == 10 else check

        if check != int(digits[length]):
            return False

    return True

@dataclass(frozen=True)
class CPF:
    value: str

    def __post_init__(self) -> None:
        digits = _NON_DIGITS.sub("", self.value)
        if not _is_valid_cpf(digits):
            raise DomainError("CPF inválido.", field="cpf")

        object.__setattr__(self, "value", digits)
```

### 5. `src/app/modules/identity/domain/value_objects/email.py` — novo

```python
from __future__ import annotations

import re
from dataclasses import dataclass

from app.core.domain.errors import DomainError

_EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self) -> None:
        normalized = self.value.strip().lower()
        if len(normalized) > 254 or _EMAIL_RE.match(normalized) is None:
            raise DomainError("E-mail inválido.", field="email")

        object.__setattr__(self, "value", normalized)
```

### 6. `src/app/modules/identity/domain/enumerations/role.py` — novo

```python
from __future__ import annotations

import enum

class Role(str, enum.Enum):
    BUYER = "BUYER"
    ADMIN = "ADMIN"
```

### 7. `src/app/modules/identity/domain/events/domain_event.py` — novo

```python
from __future__ import annotations

from uuid import UUID
from datetime import datetime
from dataclasses import dataclass, field

from app.core.domain.events import DomainEvent

@dataclass(frozen=True)
class UserRegistered(DomainEvent):
    id: UUID = field(kw_only=True)
    name: str = field(kw_only=True)
    cpf: str = field(kw_only=True)
    email: str = field(kw_only=True)
    password_hash: str = field(kw_only=True)
    role: str = field(kw_only=True)
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
    token_hash: str = field(kw_only=True)
    expires_at: datetime = field(kw_only=True)

    # Campos abaixo só alimentam o handler de outbox (envio do e-mail); não são
    # persistidos em coluna do aggregate. `reset_token` é o token opaco em claro
    # — vive apenas nesta linha de `events`, nunca numa tabela de domínio.

    reset_token: str = field(kw_only=True)
    email: str = field(kw_only=True)
    name: str = field(kw_only=True)
```

### 8. `src/app/modules/identity/domain/aggregates/user.py` — novo

```python
from __future__ import annotations

from datetime import datetime
from uuid import UUID, uuid4

from sqlalchemy import Enum as SAEnum, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain.model import Model
from app.core.domain.events import DomainEvent
from app.core.domain.aggregate import AggregateRoot
from app.modules.identity.domain.enumerations.role import Role

from app.modules.identity.domain.value_objects.cpf import CPF
from app.modules.identity.domain.value_objects.email import Email

from app.modules.identity.domain.events.identity_events import (
    PasswordResetRequested,
    UserPasswordChanged,
    UserRegistered,
    UserSecurityStampRotated,
)

class User(AggregateRoot, Model):
    __tablename__ = "users"

    name: Mapped[str] = mapped_column(String(120), nullable=False)
    cpf: Mapped[str] = mapped_column(String(11), nullable=False)
    email: Mapped[str] = mapped_column(String(254), nullable=False)
    password_hash: Mapped[str] = mapped_column(String(60), nullable=False)
    role: Mapped[Role] = mapped_column(
        SAEnum(Role, native_enum=False, length=16),
        default=Role.BUYER,
        nullable=False,
    )
    security_stamp: Mapped[UUID] = mapped_column(default=uuid4, nullable=False)

    @classmethod
    def register(cls, name: str, cpf: CPF, email: Email, password_hash: str, role: Role = Role.BUYER) -> "User":
        user = cls()

        user.id = uuid4()
        user.raise_event(
            lambda v: UserRegistered(
                version=v,
                id=user.id,
                name=name.strip(),
                cpf=cpf.value,
                email=email.value,
                password_hash=password_hash,
                role=role.value,
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

    def request_password_reset(self, *, token_hash: str, raw_token: str, expires_at: datetime) -> None:
        self.raise_event(
            lambda v: PasswordResetRequested(
                version=v,
                id=self.id,
                token_hash=token_hash,
                expires_at=expires_at,
                reset_token=raw_token,
                email=self.email,
                name=self.name,
            )
        )

    def _apply(self, event: DomainEvent) -> None:
        handler = getattr(self, f"_when_{type(event).__name__}", None)
        if handler is not None:
            handler(event)

    def _when_UserRegistered(self, e: UserRegistered) -> None:
        self.name = e.name
        self.cpf = e.cpf
        self.email = e.email
        self.password_hash = e.password_hash
        self.role = Role(e.role)
        self.security_stamp = e.security_stamp

    def _when_UserPasswordChanged(self, e: UserPasswordChanged) -> None:
        self.password_hash = e.password_hash

    def _when_UserSecurityStampRotated(self, e: UserSecurityStampRotated) -> None:
        self.security_stamp = e.security_stamp
```

### 9. `src/app/modules/identity/domain/entities/refresh_token.py` — novo

```python
from __future__ import annotations

from uuid import UUID, uuid4
from datetime import datetime

from sqlalchemy import Boolean, DateTime, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain.model import Model
from app.core.domain.errors import UnauthorizedError

class RefreshToken(Model):
    __tablename__ = "refresh_tokens"

    user_id: Mapped[UUID] = mapped_column(nullable=False, index=True)
    token_hash: Mapped[str] = mapped_column(String(64), nullable=False, unique=True)
    expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    rotated_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    used: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)

    @classmethod
    def issue(cls, user_id: UUID, token_hash: str, expires_at: datetime) -> "RefreshToken":
        token = cls()
        token.id = uuid4()
        token.user_id = user_id
        token.token_hash = token_hash
        token.expires_at = expires_at
        token.used = False

        return token

    def is_expired(self, now: datetime) -> bool:
        return now >= self.expires_at

    def rotate(self, now: datetime) -> None:
        if self.used:
            raise UnauthorizedError("Sessão expirada.")

        self.used = True
        self.rotated_at = now
```

### 10. `src/app/modules/identity/domain/entities/password_reset_token.py` — novo

```python
from __future__ import annotations

from uuid import UUID, uuid4
from datetime import datetime

from sqlalchemy import DateTime, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain.model import Model
from app.core.domain.errors import GoneError

_INVALID_LINK = "Este link não é mais válido, solicite um novo."

class PasswordResetToken(Model):
    __tablename__ = "password_reset_tokens"

    user_id: Mapped[UUID] = mapped_column(nullable=False, index=True)
    token_hash: Mapped[str] = mapped_column(String(64), nullable=False, unique=True)
    expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    used_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    @classmethod
    def issue(cls, user_id: UUID, token_hash: str, expires_at: datetime) -> "PasswordResetToken":
        token = cls()
        token.id = uuid4()
        token.user_id = user_id
        token.token_hash = token_hash
        token.expires_at = expires_at

        return token

    def consume(self, now: datetime) -> None:
        if self.used_at is not None:
            raise GoneError(_INVALID_LINK)
        if now >= self.expires_at:
            raise GoneError(_INVALID_LINK)

        self.used_at = now

    def invalidate(self, now: datetime) -> None:
        if self.used_at is None:
            self.used_at = now
```

### 11. `src/app/modules/identity/infrastructure/services/password_hasher_service.py` — novo

```python
from __future__ import annotations

import bcrypt

_MAX_BYTES = 72

class PasswordHasherService:
    def hash(self, plain: str) -> str:
        digest = bcrypt.hashpw(self._encode(plain), bcrypt.gensalt())
        return digest.decode("utf-8")

    def verify(self, plain: str, hashed: str) -> bool:
        try:
            return bcrypt.checkpw(self._encode(plain), hashed.encode("utf-8"))
        except ValueError:
            return False

    @staticmethod
    def _encode(plain: str) -> bytes:
        return plain.encode("utf-8")[:_MAX_BYTES]
```

### 12. `src/app/modules/identity/infrastructure/services/token_service.py` — novo

```python
from __future__ import annotations

import jwt
import hashlib
import secrets

from typing import Any
from datetime import datetime, timedelta, timezone

from app.config import settings

class TokenError(Exception):
    """Access token ausente, malformado, expirado ou de tipo errado."""

class TokenService:
    _ALGORITHM = "HS256"

    def issue_access(self, user: Any) -> str:
        now = datetime.now(timezone.utc)
        payload = {
            "sub": str(user.id),
            "role": user.role.value,
            "security_stamp": str(user.security_stamp),
            "type": "access",
            "iat": int(now.timestamp()),
            "exp": int(
                (now + timedelta(minutes=settings.access_token_expire_minutes)).timestamp()
            ),
        }

        return jwt.encode(payload, settings.jwt_secret_key, algorithm=self._ALGORITHM)

    def decode_access(self, token: str) -> dict[str, Any]:
        try:
            payload = jwt.decode(
                token, settings.jwt_secret_key, algorithms=[self._ALGORITHM]
            )
        except jwt.PyJWTError as exc:
            raise TokenError("token inválido") from exc

        if payload.get("type") != "access":
            raise TokenError("tipo de token inválido")

        return payload

    def new_opaque_token(self) -> str:
        return secrets.token_urlsafe(48)

    def hash_opaque(self, token: str) -> str:
        return hashlib.sha256(token.encode("utf-8")).hexdigest()
```

### 13. `src/app/modules/identity/infrastructure/services/__init__.py` — novo

```python
from __future__ import annotations

from app.modules.identity.infrastructure.services.password_hasher_service import (
    PasswordHasherService,
)
from app.modules.identity.infrastructure.services.token_service import (
    TokenError,
    TokenService,
)

__all__ = ["PasswordHasherService", "TokenError", "TokenService"]
```

### 14. `src/app/modules/identity/infrastructure/repositories/user_repository.py` — novo

```python
from __future__ import annotations

from sqlalchemy import func, select

from app.modules.identity.domain.aggregates.user import User
from app.core.infrastructure.repositories.repository import AggregateRepository

class UserRepository(AggregateRepository[User]):
    model = User

    async def find_by_email(self, email: str) -> User | None:
        return await self.find_by("email", email)

    async def find_by_cpf(self, cpf: str) -> User | None:
        return await self.find_by("cpf", cpf)

    async def list_paginated(self, *, limit: int, offset: int) -> tuple[list[User], int]:
        rows = await self._session.execute(
            select(User)
            .where(User.is_active == True)  # noqa: E712
            .order_by(User.created_at.desc())
            .limit(limit)
            .offset(offset)
        )

        total = await self._session.execute(
            select(func.count()).select_from(User).where(User.is_active == True)  # noqa: E712
        )

        return list(rows.scalars().all()), total.scalar_one()
```

### 15. `src/app/modules/identity/infrastructure/repositories/refresh_token_repository.py` — novo

```python
from __future__ import annotations

from app.core.infrastructure.repositories.repository import BaseRepository
from app.modules.identity.domain.entities.refresh_token import RefreshToken

class RefreshTokenRepository(BaseRepository[RefreshToken]):
    model = RefreshToken

    async def find_by_token_hash(self, token_hash: str) -> RefreshToken | None:
        return await self.find_by("token_hash", token_hash)
```

### 16. `src/app/modules/identity/infrastructure/repositories/password_reset_token_repository.py` — novo

```python
from __future__ import annotations

from uuid import UUID
from datetime import datetime

from sqlalchemy import select

from app.core.infrastructure.repositories.repository import BaseRepository
from app.modules.identity.domain.entities.password_reset_token import PasswordResetToken

class PasswordResetTokenRepository(BaseRepository[PasswordResetToken]):
    model = PasswordResetToken

    async def find_by_token_hash(self, token_hash: str) -> PasswordResetToken | None:
        return await self.find_by("token_hash", token_hash)

    async def find_active_for_user(
        self, user_id: UUID, now: datetime
    ) -> list[PasswordResetToken]:
        result = await self._session.execute(
            select(PasswordResetToken).where(
                PasswordResetToken.user_id == user_id,
                PasswordResetToken.used_at.is_(None),
                PasswordResetToken.expires_at > now,
                PasswordResetToken.is_active == True,  # noqa: E712
            )
        )

        return list(result.scalars().all())
```

### 17. `src/app/modules/identity/infrastructure/repositories/__init__.py` — novo

```python
from __future__ import annotations

from app.modules.identity.infrastructure.repositories.password_reset_token_repository import (
    PasswordResetTokenRepository,
)
from app.modules.identity.infrastructure.repositories.refresh_token_repository import (
    RefreshTokenRepository,
)
from app.modules.identity.infrastructure.repositories.user_repository import UserRepository

__all__ = ["PasswordResetTokenRepository", "RefreshTokenRepository", "UserRepository"]
```

### 18. `src/app/modules/identity/application/schemas/request.py` — novo

```python
from __future__ import annotations

from pydantic import Field

from app.core.shared.schemas import CamelModel

_EMAIL_PATTERN = r"^[^@\s]+@[^@\s]+\.[^@\s]+$"

class RegisterRequest(CamelModel):
    name: str = Field(min_length=1, max_length=120)
    cpf: str = Field(pattern=r"^\d{11}$", description="Só dígitos, sem máscara.")
    email: str = Field(pattern=_EMAIL_PATTERN, max_length=254)
    password: str = Field(min_length=8, max_length=128)

class LoginRequest(CamelModel):
    email: str = Field(min_length=3, max_length=254)
    password: str = Field(min_length=1, max_length=128)

class ChangePasswordRequest(CamelModel):
    current_password: str = Field(min_length=1, max_length=128)
    new_password: str = Field(min_length=8, max_length=128)

class ForgotPasswordRequest(CamelModel):
    email: str = Field(min_length=3, max_length=254)

class ResetPasswordRequest(CamelModel):
    token: str = Field(min_length=1, max_length=512)
    password: str = Field(min_length=8, max_length=128)
```

### 19. `src/app/modules/identity/application/schemas/response.py` — novo

```python
from __future__ import annotations

from uuid import UUID
from pydantic import Field

from app.core.shared.schemas import CamelModel
from app.modules.identity.domain.enumerations.role import Role

class TokenResponse(CamelModel):
    access_token: str
    expires_in: int = Field(description="Segundos até o access token expirar.")

class UserDetailResponse(CamelModel):
    id: UUID
    name: str
    email: str
    cpf: str
    role: Role

class UserSummaryResponse(CamelModel):
    id: UUID
    name: str
    email: str
    role: Role

class PagedUsersResponse(CamelModel):
    items: list[UserSummaryResponse]
    page: int
    size: int
    total: int

class MessageResponse(CamelModel):
    message: str
```

### 20. `src/app/modules/identity/application/usecases/auth_usecase.py` — novo

```python
from __future__ import annotations

from uuid import UUID
from datetime import datetime, timedelta, timezone

from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.core.domain.errors import DomainError, GoneError, UnauthorizedError

from app.modules.identity.application.schemas.request import (
    ChangePasswordRequest,
    ForgotPasswordRequest,
    LoginRequest,
    ResetPasswordRequest,
)

from app.modules.identity.domain.aggregates.user import User
from app.modules.identity.domain.entities.refresh_token import RefreshToken
from app.modules.identity.domain.entities.password_reset_token import PasswordResetToken

from app.modules.identity.application.schemas.response import TokenResponse

from app.modules.identity.infrastructure.repositories import (
    PasswordResetTokenRepository,
    RefreshTokenRepository,
    UserRepository,
)
from app.modules.identity.infrastructure.services import PasswordHasherService, TokenService

PASSWORD_RESET_TOKEN_TTL_HOURS = 1
_GENERIC_LOGIN_ERROR = "E-mail ou senha inválidos."
_GENERIC_SESSION_ERROR = "Sessão expirada."

def _now() -> datetime:
    return datetime.now(timezone.utc)

class AuthUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._user_repo = UserRepository(session)
        self._refresh_token_repo = RefreshTokenRepository(session)
        self._reset_password_repo = PasswordResetTokenRepository(session)
        self._password_hasher_service = PasswordHasherService()
        self._token_service = TokenService()

    async def login(self, request: LoginRequest) -> tuple[TokenResponse, str]:
        user = await self._user_repo.find_by_email(request.email.strip().lower())
        if user is None or not self._password_hasher_service.verify(
            request.password, user.password_hash
        ):
            raise UnauthorizedError(_GENERIC_LOGIN_ERROR)

        return await self.issue_session(user)

    async def refresh(self, raw_refresh: str | None) -> tuple[TokenResponse, str]:
        if not raw_refresh:
            raise UnauthorizedError(_GENERIC_SESSION_ERROR)

        token = await self._refresh_token_repo.find_by_token_hash(
            self._token_service.hash_opaque(raw_refresh)
        )

        if token is None:
            raise UnauthorizedError(_GENERIC_SESSION_ERROR)

        user = await self._user_repo.find_by_id(token.user_id)
        if token.used:
            if user is not None:
                user.rotate_security_stamp()
                await self._user_repo.save(user)

            raise UnauthorizedError(_GENERIC_SESSION_ERROR)

        if user is None or token.is_expired(_now()):
            raise UnauthorizedError(_GENERIC_SESSION_ERROR)

        token.rotate(_now())

        await self._refresh_token_repo.save(token)
        return await self.issue_session(user)

    async def logout(self, user_id: UUID) -> None:
        user = await self._user_repo.find_by_id(user_id)
        if user is None:
            return

        user.rotate_security_stamp()
        await self._user_repo.save(user)

    async def change_password(self, user_id: UUID, request: ChangePasswordRequest) -> None:
        user = await self._user_repo.find_by_id(user_id)
        if user is None:
            raise UnauthorizedError(_GENERIC_SESSION_ERROR)

        if not self._password_hasher_service.verify(request.current_password, user.password_hash):
            raise DomainError("A senha atual não confere.", field="currentPassword")

        user.change_password(self._password_hasher_service.hash(request.new_password))
        await self._user_repo.save(user)

    async def forgot_password(self, request: ForgotPasswordRequest) -> None:
        user = await self._user_repo.find_by_email(request.email.strip().lower())
        if user is None:
            return

        now = _now()
        for previous in await self._reset_password_repo.find_active_for_user(user.id, now):
            previous.invalidate(now)
            await self._reset_password_repo.save(previous)

        raw_token = self._token_service.new_opaque_token()
        token_hash = self._token_service.hash_opaque(raw_token)
        expires_at = now + timedelta(hours=PASSWORD_RESET_TOKEN_TTL_HOURS)

        reset_token = PasswordResetToken.issue(
            user_id=user.id, token_hash=token_hash, expires_at=expires_at
        )

        await self._reset_password_repo.save(reset_token)

        user.request_password_reset(
            token_hash=token_hash, raw_token=raw_token, expires_at=expires_at
        )

        await self._user_repo.save(user)

    async def reset_password(self, request: ResetPasswordRequest) -> None:
        token = await self._reset_password_repo.find_by_token_hash(
            self._token_service.hash_opaque(request.token)
        )

        if token is None:
            raise GoneError("Este link não é mais válido, solicite um novo.")

        token.consume(_now())
        await self._reset_password_repo.save(token)

        user = await self._user_repo.find_by_id(token.user_id)
        if user is None:
            raise GoneError("Este link não é mais válido, solicite um novo.")

        user.reset_password(self._password_hasher_service.hash(request.password))
        await self._user_repo.save(user)

    async def issue_session(self, user: User) -> tuple[TokenResponse, str]:
        # Público — o `UserUseCase.register` chama isto pra logar o usuário
        # recém-criado; o resto das chamadas é interno a este usecase.
        access = self._token_service.issue_access(user)
        raw_refresh = self._token_service.new_opaque_token()
        refresh = RefreshToken.issue(
            user_id=user.id,
            token_hash=self._token_service.hash_opaque(raw_refresh),
            expires_at=_now() + timedelta(days=settings.refresh_token_expire_days),
        )

        await self._refresh_token_repo.save(refresh)
        response = TokenResponse(
            access_token=access,
            expires_in=settings.access_token_expire_minutes * 60,
        )

        return response, raw_refresh
```

### 21. `src/app/modules/identity/application/usecases/user_usecase.py` — novo

```python
from __future__ import annotations

from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain.errors import ConflictError, DomainError, ForbiddenError

from app.modules.identity.application.schemas.request import RegisterRequest
from app.modules.identity.domain.aggregates.user import User
from app.modules.identity.domain.enumerations.role import Role
from app.modules.identity.domain.value_objects.cpf import CPF
from app.modules.identity.domain.value_objects.email import Email
from app.modules.identity.infrastructure.repositories import UserRepository
from app.modules.identity.infrastructure.services import PasswordHasherService

class UserUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._user_repo = UserRepository(session)
        self._password_hasher_service = PasswordHasherService()

    async def register(self, request: RegisterRequest) -> User:
        cpf = CPF(request.cpf)
        email = Email(request.email)

        if await self._user_repo.find_by_cpf(cpf.value) is not None:
            raise ConflictError("Este CPF já possui cadastro.", field="cpf")

        if await self._user_repo.find_by_email(email.value) is not None:
            raise ConflictError("Este e-mail já está em uso.", field="email")

        user = User.register(
            request.name, cpf, email, self._password_hasher_service.hash(request.password)
        )

        await self._user_repo.save(user)
        return user

    async def get_user(self, *, requester_id: UUID, requester_role: Role, target_id: UUID) -> User:
        if requester_role is not Role.ADMIN and requester_id != target_id:
            raise ForbiddenError("Você só pode ver os próprios dados.")

        user = await self._user_repo.find_by_id(target_id)
        if user is None:
            raise DomainError("Usuário não encontrado.", status_code=404)

        return user

    async def list_users(self, *, page: int, size: int) -> tuple[list[User], int]:
        offset = (page - 1) * size
        return await self._user_repo.list_paginated(limit=size, offset=offset)
```

### 22. `src/app/modules/notification/infrastructure/services/email_service.py` — novo

```python
from __future__ import annotations

import aiosmtplib
from email.message import EmailMessage

from app.config import settings

class EmailServiceError(Exception):
    """Falha de transporte no envio de e-mail. Nunca sobe crua ao handler."""

class EmailService:
    async def send(self, *, to: str, subject: str, body: str) -> None:
        message = EmailMessage()
        message["From"] = settings.email_sender
        message["To"] = to
        message["Subject"] = subject
        message.set_content(body)

        try:
            await aiosmtplib.send(
                message,
                hostname=settings.smtp_host,
                port=settings.smtp_port,
                username=settings.smtp_username or None,
                password=settings.smtp_password or None,
                start_tls=settings.smtp_port == 587,
                timeout=10,
            )
        except (aiosmtplib.SMTPException, OSError) as exc:
            raise EmailServiceError("falha ao enviar e-mail") from exc
```

### 23. `src/app/modules/notification/dependencies.py` — novo

```python
from __future__ import annotations

from app.modules.notification.infrastructure.services.email_service import EmailService

async def send_transactional_email(*, to: str, subject: str, body: str) -> None:
    await EmailService().send(to=to, subject=subject, body=body)
```

### 24. `src/app/modules/identity/handlers.py` — novo

```python
from __future__ import annotations

import logging

from app.config import settings
from app.outbox.registry import register
from app.modules.notification.dependencies import send_transactional_email

logger = logging.getLogger(__name__)

_SUBJECT = "Redefinição de senha — Ludens"

@register("PasswordResetRequested")
async def send_password_reset_email(payload: dict) -> None:

    email = payload.get("email")
    reset_token = payload.get("reset_token")
    name = payload.get("name") or ""

    if not email or not reset_token:
        logger.error("PasswordResetRequested sem email/token no payload; ignorado")
        return

    link = f"{settings.web_app_url}/redefinir-senha?token={reset_token}"
    body = (
        f"Olá, {name}.\n\n"
        "Recebemos um pedido para redefinir a sua senha na Ludens. Abra o link "
        "abaixo para escolher uma nova senha (o link expira em 1 hora e só pode "
        f"ser usado uma vez):\n\n{link}\n\n"
        "Se não foi você, ignore este e-mail — nada muda na sua conta."
    )

    await send_transactional_email(to=email, subject=_SUBJECT, body=body)
```

### 25. `src/app/modules/identity/api/routers/auth_router.py` — novo

```python
from __future__ import annotations

from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import APIRouter, Depends, Request, Response

from app.config import settings
from app.dependencies import get_db

from app.modules.identity.application.schemas.request import (
    ChangePasswordRequest,
    ForgotPasswordRequest,
    LoginRequest,
    ResetPasswordRequest,
)
from app.modules.identity.application.schemas.response import MessageResponse, TokenResponse
from app.modules.identity.application.usecases.auth_usecase import AuthUseCase
from app.modules.identity.dependencies import CurrentUser, get_current_user

router = APIRouter(prefix="/auth", tags=["Identity"])

REFRESH_COOKIE_NAME = "refresh_token"
REFRESH_COOKIE_PATH = "/auth"
_NEUTRAL_FORGOT_MESSAGE = "Se houver uma conta com esse e-mail, enviamos um link."

def set_refresh_cookie(response: Response, raw_refresh: str) -> None:
    response.set_cookie(
        key=REFRESH_COOKIE_NAME,
        value=raw_refresh,
        max_age=settings.refresh_token_expire_days * 24 * 3600,
        httponly=True,
        secure=settings.environment != "development",
        samesite="strict",
        path=REFRESH_COOKIE_PATH,
    )

@router.post("/login", response_model=TokenResponse)
async def login(
    body: LoginRequest,
    response: Response,
    session: AsyncSession = Depends(get_db),
) -> TokenResponse:
    token, raw_refresh = await AuthUseCase(session).login(body)
    set_refresh_cookie(response, raw_refresh)
    return token

@router.post("/logout", status_code=204)
async def logout(
    response: Response,
    session: AsyncSession = Depends(get_db),
    user: CurrentUser = Depends(get_current_user),
) -> None:
    await AuthUseCase(session).logout(user.id)
    response.delete_cookie(REFRESH_COOKIE_NAME, path=REFRESH_COOKIE_PATH)

@router.post("/refresh", response_model=TokenResponse)
async def refresh(
    request: Request,
    response: Response,
    session: AsyncSession = Depends(get_db),
) -> TokenResponse:
    token, raw_refresh = await AuthUseCase(session).refresh(
        request.cookies.get(REFRESH_COOKIE_NAME)
    )
    set_refresh_cookie(response, raw_refresh)
    return token

@router.post("/password/change", status_code=204)
async def change_password(
    body: ChangePasswordRequest,
    session: AsyncSession = Depends(get_db),
    user: CurrentUser = Depends(get_current_user),
) -> None:
    await AuthUseCase(session).change_password(user.id, body)

@router.post("/password/forgot", response_model=MessageResponse, status_code=202)
async def forgot_password(
    body: ForgotPasswordRequest,
    session: AsyncSession = Depends(get_db),
) -> MessageResponse:
    await AuthUseCase(session).forgot_password(body)
    return MessageResponse(message=_NEUTRAL_FORGOT_MESSAGE)

@router.post("/password/reset", status_code=204)
async def reset_password(
    body: ResetPasswordRequest,
    session: AsyncSession = Depends(get_db),
) -> None:
    await AuthUseCase(session).reset_password(body)
```

### 26. `src/app/modules/identity/api/routers/user_router.py` — novo

```python
from __future__ import annotations

from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import APIRouter, Depends, Query, Response

from app.dependencies import get_db

from app.modules.identity.api.routers.auth_router import set_refresh_cookie
from app.modules.identity.application.schemas.request import RegisterRequest
from app.modules.identity.application.schemas.response import (
    PagedUsersResponse,
    TokenResponse,
    UserDetailResponse,
    UserSummaryResponse,
)
from app.modules.identity.application.usecases.auth_usecase import AuthUseCase
from app.modules.identity.application.usecases.user_usecase import UserUseCase
from app.modules.identity.dependencies import CurrentUser, get_current_user, require_admin

router = APIRouter(prefix="/users", tags=["Identity"])

@router.post("", response_model=TokenResponse, status_code=201)
async def register(
    body: RegisterRequest,
    response: Response,
    session: AsyncSession = Depends(get_db),
) -> TokenResponse:
    user = await UserUseCase(session).register(body)
    token, raw_refresh = await AuthUseCase(session).issue_session(user)
    set_refresh_cookie(response, raw_refresh)
    return token

@router.get("", response_model=PagedUsersResponse)
async def list_users(
    page: int = Query(default=1, ge=1),
    size: int = Query(default=20, ge=1, le=100),
    session: AsyncSession = Depends(get_db),
    _admin: CurrentUser = Depends(require_admin),
) -> PagedUsersResponse:
    users, total = await UserUseCase(session).list_users(page=page, size=size)
    return PagedUsersResponse(
        items=[
            UserSummaryResponse(id=u.id, name=u.name, email=u.email, role=u.role)
            for u in users
        ],
        page=page,
        size=size,
        total=total,
    )

@router.get("/{user_id}", response_model=UserDetailResponse)
async def get_user(
    user_id: UUID,
    session: AsyncSession = Depends(get_db),
    current_user: CurrentUser = Depends(get_current_user),
) -> UserDetailResponse:
    user = await UserUseCase(session).get_user(
        requester_id=current_user.id,
        requester_role=current_user.role,
        target_id=user_id,
    )
    return UserDetailResponse(
        id=user.id,
        name=user.name,
        email=user.email,
        cpf=user.cpf,
        role=user.role,
    )
```

### 27. `src/app/modules/identity/router.py` — novo

```python
from __future__ import annotations

from fastapi import APIRouter

from app.modules.identity.api.routers.auth_router import router as auth_router
from app.modules.identity.api.routers.user_router import router as user_router

router = APIRouter()
router.include_router(auth_router)
router.include_router(user_router)
```

### 28. `src/app/modules/identity/dependencies.py` — novo

```python
from __future__ import annotations

from dataclasses import dataclass
from uuid import UUID

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.modules.identity.domain.enumerations.role import Role
from app.modules.identity.infrastructure.repositories import UserRepository
from app.modules.identity.infrastructure.services import TokenError, TokenService

_bearer = HTTPBearer(auto_error=False)

@dataclass(frozen=True)
class CurrentUser:
    id: UUID
    role: Role
    name: str
    email: str
    cpf: str

def _unauthorized(message: str) -> HTTPException:
    return HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail=[{"field": "authorization", "message": message}],
        headers={"WWW-Authenticate": "Bearer"},
    )

async def get_current_user(
    credentials: HTTPAuthorizationCredentials | None = Depends(_bearer),
    session: AsyncSession = Depends(get_db),
) -> CurrentUser:
    if credentials is None or not credentials.credentials:
        raise _unauthorized("Sessão expirada.")

    try:
        payload = TokenService().decode_access(credentials.credentials)
    except TokenError as exc:
        raise _unauthorized("Sessão expirada.") from exc

    try:
        user_id = UUID(str(payload.get("sub")))
    except (ValueError, TypeError) as exc:
        raise _unauthorized("Sessão expirada.") from exc

    user = await UserRepository(session).find_by_id(user_id)
    if user is None or str(user.security_stamp) != payload.get("security_stamp"):
        raise _unauthorized("Sessão expirada.")

    return CurrentUser(
        id=user.id,
        role=user.role,
        name=user.name,
        email=user.email,
        cpf=user.cpf,
    )

async def require_admin(
    user: CurrentUser = Depends(get_current_user),
) -> CurrentUser:
    if user.role is not Role.ADMIN:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=[
                {"field": "authorization", "message": "Acesso restrito a administradores."}
            ],
        )
    return user
```

### 29. `src/migrations/versions/0001_outbox_events.py` — novo

```python
"""outbox: tabela events

Revision ID: 0001_outbox_events
Revises:
Create Date: 2026-09-03

"""
from typing import Sequence, Union

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

revision: str = "0001_outbox_events"
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

def upgrade() -> None:
    op.create_table(
        "events",
        sa.Column("id", sa.Uuid(), nullable=False),
        sa.Column(
            "created_at", sa.DateTime(timezone=True), nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column(
            "updated_at", sa.DateTime(timezone=True), nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column(
            "is_active", sa.Boolean(), nullable=False, server_default=sa.true()
        ),
        sa.Column("aggregate_id", postgresql.UUID(as_uuid=False), nullable=False),
        sa.Column("event_type", sa.String(length=128), nullable=False),
        sa.Column("payload", postgresql.JSONB(astext_type=sa.Text()), nullable=False),
        sa.Column("dispatched_at", sa.DateTime(timezone=True), nullable=True),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index("ix_events_aggregate_id", "events", ["aggregate_id"])
    op.create_index("ix_events_event_type", "events", ["event_type"])
    op.create_index("ix_events_dispatched_at", "events", ["dispatched_at"])

def downgrade() -> None:
    op.drop_index("ix_events_dispatched_at", table_name="events")
    op.drop_index("ix_events_event_type", table_name="events")
    op.drop_index("ix_events_aggregate_id", table_name="events")
    op.drop_table("events")
```

### 30. `src/migrations/versions/0002_identity_auth.py` — novo

```python
# src/migrations/versions/0002_identity_auth.py — novo
"""identity-auth: users, refresh_tokens, password_reset_tokens

Revision ID: 0002_identity_auth
Revises: 0001_outbox_events
Create Date: 2026-09-03

"""
from typing import Sequence, Union

import sqlalchemy as sa
from alembic import op

revision: str = "0002_identity_auth"
down_revision: Union[str, None] = "0001_outbox_events"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

def _base_columns() -> list[sa.Column]:
    # Novas instâncias de Column a cada chamada — não reutilizar entre tabelas.
    return [
        sa.Column("id", sa.Uuid(), nullable=False),
        sa.Column(
            "created_at", sa.DateTime(timezone=True), nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column(
            "updated_at", sa.DateTime(timezone=True), nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column(
            "is_active", sa.Boolean(), nullable=False, server_default=sa.true()
        ),
    ]

def upgrade() -> None:
    op.create_table(
        "users",
        *_base_columns(),
        sa.Column("name", sa.String(length=120), nullable=False),
        sa.Column("cpf", sa.String(length=11), nullable=False),
        sa.Column("email", sa.String(length=254), nullable=False),
        sa.Column("password_hash", sa.String(length=60), nullable=False),
        sa.Column(
            "role", sa.String(length=16), nullable=False, server_default="BUYER"
        ),
        sa.Column("security_stamp", sa.Uuid(), nullable=False),
        sa.PrimaryKeyConstraint("id"),
    )
    # Unicidade só entre contas ativas (soft delete não colide com um novo
    # cadastro do mesmo CPF/e-mail).
    op.create_index(
        "uq_users_email_active", "users", ["email"], unique=True,
        postgresql_where=sa.text("is_active"),
    )
    op.create_index(
        "uq_users_cpf_active", "users", ["cpf"], unique=True,
        postgresql_where=sa.text("is_active"),
    )

    op.create_table(
        "refresh_tokens",
        *_base_columns(),
        sa.Column("user_id", sa.Uuid(), nullable=False),
        sa.Column("token_hash", sa.String(length=64), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("rotated_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("used", sa.Boolean(), nullable=False, server_default=sa.false()),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index(
        "uq_refresh_tokens_token_hash", "refresh_tokens", ["token_hash"], unique=True
    )
    op.create_index("ix_refresh_tokens_user_id", "refresh_tokens", ["user_id"])

    op.create_table(
        "password_reset_tokens",
        *_base_columns(),
        sa.Column("user_id", sa.Uuid(), nullable=False),
        sa.Column("token_hash", sa.String(length=64), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("used_at", sa.DateTime(timezone=True), nullable=True),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index(
        "uq_password_reset_tokens_token_hash",
        "password_reset_tokens",
        ["token_hash"],
        unique=True,
    )
    op.create_index(
        "ix_password_reset_tokens_user_id", "password_reset_tokens", ["user_id"]
    )

def downgrade() -> None:
    op.drop_index(
        "ix_password_reset_tokens_user_id", table_name="password_reset_tokens"
    )
    op.drop_index(
        "uq_password_reset_tokens_token_hash", table_name="password_reset_tokens"
    )
    op.drop_table("password_reset_tokens")
    op.drop_index("ix_refresh_tokens_user_id", table_name="refresh_tokens")
    op.drop_index("uq_refresh_tokens_token_hash", table_name="refresh_tokens")
    op.drop_table("refresh_tokens")
    op.drop_index("uq_users_cpf_active", table_name="users")
    op.drop_index("uq_users_email_active", table_name="users")
    op.drop_table("users")
```

### 31. `src/migrations/env.py` — editar

Trocar o comentário-guia (linhas 15–16) pelos imports reais dos aggregates do
módulo, para o `Model.metadata` conhecer as tabelas em `--autogenerate`:

```python
# src/migrations/env.py — editar (após "import app.outbox.models")
import app.outbox.models  # noqa: F401,E402

# Cada feature acrescenta o import do próprio módulo aqui.
import app.modules.identity.domain.aggregates.user  # noqa: F401,E402
import app.modules.identity.domain.entities.refresh_token  # noqa: F401,E402
import app.modules.identity.domain.entities.password_reset_token  # noqa: F401,E402
```

### 32. `src/app/main.py` — editar

Adicionar o handler de `DomainError`, incluir o router de `identity` e importar
os handlers de outbox do módulo no boot:

```python
# src/app/main.py — editar

# 1) novos imports (junto aos demais do topo)
from app.core.domain.errors import DomainError
from app.modules.identity import handlers as _identity_handlers  # noqa: F401
from app.modules.identity.router import router as identity_router

# 2) logo após "app = FastAPI(...)" e o handler de RequestValidationError:
@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError):
    # Envelope único de erro 4xx: {"detail": [{"field", "message"}]}
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": [{"field": exc.field, "message": exc.message}]},
    )

# 3) após o add_middleware(CORSMiddleware, ...):
app.include_router(identity_router)
```

`_identity_handlers` é importado só pelo efeito colateral de `@register(...)`
rodar no boot — o relay já existente em `lifespan` passa a encontrar o handler de
`PasswordResetRequested`.

### 33. `src/app/config.py` — editar

Adicionar ao corpo de `Settings` (a URL do frontend para montar o link de
redefinição e a conexão SMTP que o e-mail de recuperação usa):

```python

    web_app_url: str = "http://localhost:3000"

    smtp_host: str = "localhost"
    smtp_port: int = 1025
    smtp_username: str = ""          # SENSITIVE
    smtp_password: str = ""          # SECRET
    email_sender: str = "no-reply@ludens.local"
```

### 34. `.env.example` — editar

Preencher as variáveis novas na seção da feature (substitui as linhas comentadas
de `identity-auth` e a de `notification-transactional-email` no bloco final):

```bash
# .env.example — editar

# --- JWT (identity-auth) ---
# SECRET — gere com: openssl rand -hex 32
JWT_SECRET_KEY=
# CONFIG
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7
# CONFIG — base pública do web.ludens (link do e-mail de redefinição)
WEB_APP_URL=http://localhost:3000

# --- E-mail transacional (identity-auth usa; notification-transactional-email amplia) ---
# CONFIG (dev: container MailHog/Mailpit em smtp:1025)
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USERNAME=            # SENSITIVE
SMTP_PASSWORD=            # SECRET
EMAIL_SENDER=no-reply@ludens.local
```

### 35. `scripts/seed_admin.py` — novo

```python
"""Cria a conta ADMIN fora do fluxo público (RF08/RF09).

Uso:
    ADMIN_CPF=52998224725 ADMIN_EMAIL=admin@ludens.local \\
    ADMIN_PASSWORD='troque-esta-senha' python scripts/seed_admin.py
"""
from __future__ import annotations

import os
import asyncio

from app.database import AsyncSessionLocal
from app.modules.identity.domain.aggregates.user import User
from app.modules.identity.domain.enumerations.role import Role
from app.modules.identity.domain.value_objects.cpf import CPF
from app.modules.identity.domain.value_objects.email import Email
from app.modules.identity.infrastructure.repositories import UserRepository
from app.modules.identity.infrastructure.services import PasswordHasherService

async def seed_admin() -> None:
    name = os.environ.get("ADMIN_NAME", "Administrador Ludens")
    cpf = os.environ.get("ADMIN_CPF", "")
    email = os.environ.get("ADMIN_EMAIL", "")
    password = os.environ.get("ADMIN_PASSWORD", "")

    if not (cpf and email and password):
        raise SystemExit("Defina ADMIN_CPF, ADMIN_EMAIL e ADMIN_PASSWORD no ambiente.")

    async with AsyncSessionLocal() as session:
        async with session.begin():
            user_repo = UserRepository(session)
            if await user_repo.find_by_email(email.strip().lower()) is not None:
                print(f"Admin {email} já existe; nada a fazer.")
                return

            admin = User.register(
                name=name,
                cpf=CPF(cpf),
                email=Email(email),
                password_hash=PasswordHasherService().hash(password),
                role=Role.ADMIN,
            )
            
            await user_repo.save(admin)
            print(f"Admin criado: {email}")

if __name__ == "__main__":
    asyncio.run(seed_admin())
```

### 36. `pyproject.toml` — editar

Primeiro módulo com rotas FastAPI: `Depends(...)`/`Security(...)` no default de
parâmetro dispara `B008` do bugbear. Adicionar a seção (o resto do arquivo fica
igual):

```toml
# pyproject.toml — editar (nova seção, após [tool.ruff.lint.per-file-ignores])

[tool.ruff.lint.flake8-bugbear]
extend-immutable-calls = [
    "fastapi.Depends",
    "fastapi.Query",
    "fastapi.Path",
    "fastapi.Header",
    "fastapi.Cookie",
    "fastapi.Security",
]
```

### 37. `.github/workflows/ci.yml` — editar

Injetar o segredo no job de lint/testes (o `config.py` tem default de dev, mas o
CI roda com o segredo real para exercitar a assinatura JWT de ponta a ponta):

```yaml
# .github/workflows/ci.yml — editar (job lint-and-test)
jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    env:
      JWT_SECRET_KEY: ${{ secrets.JWT_SECRET_KEY }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install
        run: pip install -e ".[dev]"
      - name: Lint (ruff)
        run: ruff check .
      - name: Tests (pytest)
        run: pytest -q
```

---

## 3. Onde cada regra de negócio entra

| Regra | Arquivo · função | Como |
| --- | --- | --- |
| RF09 — CPF com dígitos verificadores | `domain/value_objects/cpf.py` · `__post_init__` / `_is_valid_cpf` | módulo 11 nos dois DV; `DomainError("CPF inválido.", field="cpf")` → 422 |
| RF09 — unicidade de CPF e e-mail | `application/usecases/user_usecase.py` · `register` + `migrations/0002` índices parciais `uq_users_*_active` | `find_by_cpf` / `find_by_email` antes de criar → `ConflictError` (409); o índice parcial é a rede de segurança sob concorrência |
| RF09 — mensagem específica por campo no cadastro | `user_usecase.py` · `register` | "Este CPF já possui cadastro." (`field="cpf"`) / "Este e-mail já está em uso." (`field="email"`) |
| RF09 — login genérico | `auth_usecase.py` · `login` | e-mail inexistente e senha errada → o mesmo `UnauthorizedError("E-mail ou senha inválidos.")` |
| RF09 — resposta neutra em "esqueci a senha" | `api/routers/auth_router.py` · `forgot_password` + `auth_usecase.py` · `forgot_password` | rota sempre 202 com a mesma mensagem; o usecase retorna em silêncio quando o e-mail não existe |
| RF09 — token de redefinição uso único / 1 h | `domain/entities/password_reset_token.py` · `consume` + `auth_usecase.py` · `PASSWORD_RESET_TOKEN_TTL_HOURS` | `consume` recusa (`GoneError` 410) se `used_at` ou expirado; `issue` grava `expires_at = now + 1h` |
| RF09 — nova solicitação invalida as anteriores | `auth_usecase.py` · `forgot_password` + `password_reset_token.py` · `invalidate` | varre `find_active_for_user` e marca `used_at` em cada um antes de emitir o novo |
| RF09 — link enviado sem bloquear a resposta | `domain/aggregates/user.py` · `request_password_reset` (evento `PasswordResetRequested`) + `modules/identity/handlers.py` | `PasswordResetToken` (entidade filha, sem evento) só guarda o estado; quem publica é o `User`; efeito externo só no handler de outbox, o relay (~2 s) chama `send_transactional_email` |
| RF09 — logout / troca / redefinição derrubam todas as sessões | `domain/aggregates/user.py` · `rotate_security_stamp` (chamado por `logout` no usecase, e por `change_password`/`reset_password` no aggregate) | novo `security_stamp` (UUID) → todo access token anterior falha na checagem de stamp |
| RF09 — reuso de refresh token detectado | `auth_usecase.py` · `refresh` + `domain/entities/refresh_token.py` · `rotate` | `rotate` a cada uso marca `used=True`; um segundo uso do mesmo token → `user.rotate_security_stamp()` + 401 (derruba tudo) |
| RF09 — `GET /users/{id}` restrito por papel | `user_usecase.py` · `get_user` | `BUYER` só vê `target_id == requester_id` (senão `ForbiddenError` 403); `ADMIN` vê qualquer um, inclusive o próprio — sem exceção na regra |
| RF09 — `GET /users` paginado e restrito a `ADMIN` | `api/routers/user_router.py` · `list_users` (`Depends(require_admin)`) + `user_usecase.py` · `list_users` + `user_repository.py` · `list_paginated` | `BUYER`/`ADMIN` sem token → 401/403 antes do usecase; resposta traz `UserSummaryResponse` (sem CPF) + `page`/`size`/`total` |
| RNF01 — hash de senha | `infrastructure/services/password_hasher_service.py` | bcrypt com salt automático; `password_hash` nunca aparece em nenhum schema de response nem em log |
| RNF01 — `security_stamp` no claim e verificado a cada request | `token_service.py` · `issue_access` (claim) + `dependencies.py` · `get_current_user` | stamp do token comparado ao do `User`; divergência → 401 mesmo com JWT válido |
| RNF01 — nada sensível em log / erro / URL | `handlers.py` (log genérico), `token_service.py` (não loga token), `errors.py` (`DomainError` sem CPF/e-mail na mensagem), CPF guardado só em dígitos | mensagens de erro em linguagem de negócio; o token opaco vai em cookie `HttpOnly`, nunca em query string de rota da API |

---

## 4. DevOps

**Variável nova `JWT_SECRET_KEY`** (classificação `SECRET`): já listada em
`.env.example` sem valor. Gerar e configurar:

```bash
# local — grave em .env.local (fora do VCS)
echo "JWT_SECRET_KEY=$(openssl rand -hex 32)" >> .env.local

# CI — segredo do repositório
openssl rand -hex 32 | gh secret set JWT_SECRET_KEY --repo gcarvalhow/api.ludens
```

- **`ci.yml`**: adicionar o bloco `env:` com
  `JWT_SECRET_KEY: ${{ secrets.JWT_SECRET_KEY }}` ao job `lint-and-test` (arquivo
  35 acima).
- **Outras variáveis novas** (`WEB_APP_URL`, `SMTP_*`, `EMAIL_SENDER`): têm
  default de desenvolvimento em `config.py` e entram em `.env.example` como
  `CONFIG`/`SENSITIVE`/`SECRET` conforme a tabela. `SMTP_PASSWORD` só é
  necessária quando o SMTP de produção exigir auth — em dev, MailHog/Mailpit sem
  credencial. Registrar as três em
  `docs.ludens/backend/security/configuration.md` na seção "adicionadas por
  funcionalidade".
- **Migração**: `alembic upgrade head` aplica `0001_outbox_events` e
  `0002_identity_auth` (repositório ainda sem histórico de migration). Rodar
  contra um banco limpo antes de subir a app.

---

## 5. Passo a passo TBD (Backend)

```bash
git checkout master && git pull && git checkout -b feat/09-identity-auth

# commit 1 — infra compartilhada + domínio
git add src/app/core/domain/errors.py src/app/core/shared/schemas.py \
        src/app/modules/identity/__init__.py src/app/modules/identity/domain \
        src/app/modules/notification/__init__.py
git commit -m "feat(identity): modelar User, tokens e eventos de domínio"

# commit 2 — infrastructure (services + repositórios) + outbox
git add src/app/modules/identity/infrastructure \
        src/app/modules/notification/infrastructure \
        src/app/modules/notification/dependencies.py \
        src/app/modules/identity/handlers.py
git commit -m "feat(identity): hasher bcrypt, token service, repositórios e handler de e-mail"

# commit 3 — application (AuthUseCase + UserUseCase + schemas)
git add src/app/modules/identity/application
git commit -m "feat(identity): usecases de auth e user, schemas de request/response"

# commit 4 — api (auth_router + user_router) + migration + config + script
git add src/app/modules/identity/api src/app/modules/identity/router.py \
        src/app/modules/identity/dependencies.py src/migrations \
        src/app/main.py src/app/config.py .env.example pyproject.toml \
        scripts/seed_admin.py .github/workflows/ci.yml
git commit -m "feat(identity): expor rotas de /auth e /users, dependencies, migration e seed do admin"
```

Depois: `/team-ludens:tbd-pr` (senior-dev Modo 2 + `/code-review`) → push → PR
`Closes #<NN>` → merge (1 aprovação + CI verde). Antes do PR, localmente:
`ruff check .` · `alembic upgrade head` · `pytest -q` (inclui os casos de
`quality.md`).

---

## 6. Ordem entre as superfícies

Backend e QA (casos de domínio de `quality.md`) começam juntos a partir do
`logic.md`. Frontend começa em paralelo contra o contrato-alvo do
`integration.md` — `fetcher` com refresh e `AuthContext` não dependem do backend
pronto. O `integration.md` vira `canônico` e a integração real acontece depois do
merge do backend.

---

## 7. Bloqueios em aberto

Nenhum. Todas as decisões de produto estão fechadas no `spec.md` (§8) e no
`logic.md` (§5).

---

## 8. Ajustes feitos no `integration.md`

Ao fechar este documento, o `integration.md` (mantido `status: alvo`) foi
precisado nos pontos que estavam `<a definir globalmente>`:

- **Envelope de erro 4xx**: `{"detail": [{"field": string, "message": string}]}`
  — vale para 422 de validação (handler de `RequestValidationError` já
  existente) e para os erros de domínio (novo handler de `DomainError`).
- **Prefixo / versionamento**: sem prefixo e sem versionamento no N1 — as rotas
  são montadas em `/auth/...` e `/users/...`; base = `NEXT_PUBLIC_API_URL`.
- **`expiresIn`**: em segundos (`ACCESS_TOKEN_EXPIRE_MINUTES * 60`).
- **`POST /auth/password/change`**: corpo `{ currentPassword, newPassword }`
  (camelCase, como todo o resto do contrato).
- **`role`** no `/users/{id}` e no claim do JWT: `"BUYER"` | `"ADMIN"` (maiúsculas).
- **`POST /auth/logout`**: além de 204, envia `Set-Cookie` apagando
  `refresh_token` (`Path=/auth`).
- **`register` e a leitura de usuário saem de `/auth`**: `AuthUseCase` fica só
  com sessão (login/refresh/logout/senha); `UserUseCase` cobre cadastro e
  leitura. `POST /auth/register` → `POST /users`; sem `GET /auth/me` — vira
  `GET /users/{id}` (`BUYER` só o próprio, 403 em qualquer outro; `ADMIN`
  qualquer um). O frontend descobre o próprio `id` decodificando o claim `sub`
  do access token (payload do JWT, sem verificar assinatura — a verificação é
  sempre do backend).
- **`GET /users` paginado, `ADMIN`-only**: `?page=1&size=20` (default),
  `size` até 100. Resposta `{ items, page, size, total }` com
  `UserSummaryResponse` (sem CPF) — mesmo padrão de paginação de
  `identity-order-history` (`PagedOrders`).
