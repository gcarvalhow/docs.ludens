---
status: done
spec: identity-auth
surface: backend
created_at: 2026-09-03
updated_at: 2026-09-17
---

# Cadastro e autenticação do comprador — Backend

> **Nota de reescopo (2026-09-11):** `spec.md`/`logic.md` de `identity-auth`
> foram trimmados — cadastro e leitura de usuário saíram para
> `identity-user-management`. O código abaixo (ainda em produção, ainda
> correto) reflete o escopo **antigo**: note que já existe uma separação
> `AuthUseCase` (sessão) × `UserUseCase` (cadastro/leitura) nos arquivos #24 e
> #25 — a divisão de spec só está formalizando uma fronteira que o código já
> tinha. Pendente: mover os arquivos de `UserUseCase`/`user_router.py` (#25,
> #28) pro `backend.md` de `identity-user-management`, e adicionar aqui o
> fluxo de alteração de e-mail (novo). Ver `feature-implementation-spec`.

**Resumo:** módulo `identity` com o aggregate `User` (CPF validado, e-mail, hash
bcrypt, `is_admin: bool`, `security_stamp`), dual-token JWT (access HS256 +
refresh opaco SHA-256 rotacionado a cada uso), recuperação de senha por token
de uso único com validade de 1 hora. Duas usecases — `AuthUseCase` (sessão:
login/refresh/logout/senha) e `UserUseCase` (cadastro e leitura) — expostas em
dois routers: 6 rotas em `/identity` (tag `01.Identity - Auth`) e 2 em
`/identity/users` (tag `02.Identity - User`). Dependências
`get_current_user` / `require_admin` exportadas para os demais módulos.
**RF:** RF09 · **RN:** — (reforça RNF01) · **Módulo backend:** `identity`
**Contrato:** `docs.ludens/specs/identity-auth/integration.md`
**Carregar antes:** skill `backend-architecture` (todos os `references/`),
`docs.ludens/backend/overview.md`, `docs.ludens/backend/conventions.md`,
`docs.ludens/backend/security/authentication.md`,
`docs.ludens/backend/security/configuration.md`.

> **Revisão de 2026-09-11:** a versão anterior deste documento não batia com o
> código real já mergeado (PR #8, `gcarvalhow/api.ludens`) — apesar de ser este
> o primeiro módulo do repositório, o documento nunca foi atualizado depois da
> própria implementação divergir dele. Corrigido nesta revisão: `role: enum
> Role` → `is_admin: bool` (a classe `Role` não existe); schemas em camelCase
> via `CamelModel` → `pydantic.BaseModel` puro, snake_case (`CamelModel` não
> existe no projeto); `DomainError(message, *, status_code=, field=)` → classe
> real só tem `message`, subclasses vazias (`ConflictError`, `AuthError` — não
> `UnauthorizedError` —, `ForbiddenError`, `GoneError`, `NotFoundError`),
> mapeamento por subclasse numa lista central em `main.py`; `find_by_id`/
> `find_by_email`/`find_by_cpf`/`list_paginated` como métodos próprios de
> `UserRepository` → o repositório real não tem **nenhum** método próprio, é
> `AggregateRepository[User]` puro, tudo via `find_by(campo, valor)` genérico;
> `_user_repo`/`_refresh_token_repo` → `_user_repository`/`_refresh_repository`
> (nome real); `PasswordHasherService` → `PasswordService` (nome e arquivo
> reais); `CurrentUser` (dataclass própria) → não existe, as dependencies
> retornam o próprio aggregate `User`; import de erro por submódulo
> (`app.core.domain.errors`) → sempre pelo pacote (`app.core.domain`).
>
> Duas seções inteiras foram **removidas** por não existirem no código real
> nem terem RF que as sustente: a listagem paginada de usuários (`GET /users`,
> `PagedUsersResponse`, `list_users` — não há requisito em
> `requirements/functional.md` que peça isso, e o `UserRepository` real não
> tem `list_paginated`) e o envio de e-mail de redefinição de senha via módulo
> `notification` + `identity/handlers.py` (não existem no código real — o
> evento `PasswordResetRequested` é levantado pelo aggregate, mas **nenhum
> handler o consome ainda**; o outbox relay processa o evento e ele fica
> `dispatched_at = NULL` para sempre). RF09 pede recuperação "por e-mail" —
> isso está **pendente como débito técnico**, não implementado, apesar do
> `status: done` deste documento referir-se ao fluxo de sessão/cadastro, que
> está completo. Ver §7.

> **Revisão de 2026-09-17:** os routers do módulo mudaram de prefixo/tag para
> seguir o novo padrão do backend (`prefix`/`tags` numerados por router).
> `auth_router.py`: `prefix="/auth", tags=["Identity"]` →
> `prefix="/identity", tags=["01.Identity - Auth"]`. `user_router.py`:
> `prefix="/users", tags=["Identity"]` → `prefix="/identity/users",
> tags=["02.Identity - User"]`. O `Path` do cookie de refresh acompanhou a
> mudança (`REFRESH_PATH`: `/auth` → `/identity`). Todas as rotas abaixo estão
> atualizadas para os novos caminhos; `web.ludens` ainda não foi atualizado —
> ver débito técnico em `docs.ludens/team/tech-debt.md`.

Este é o primeiro módulo de negócio do repositório. Ele também introduz dois
arquivos de infraestrutura compartilhada que qualquer feature seguinte
reaproveita: `core/domain/errors.py` (`DomainError` + subclasses) e
`core/shared/errors.py` (`format_validation_errors`, usado pelo handler de
`RequestValidationError` do Pydantic em `main.py`).

---

## 1. Arquivos (ordem de dependência)

`infrastructure/` (repositórios e services) vem antes de `application/` — o
usecase depende dos dois, não o contrário. Cada subpacote de domínio
(`value_objects`, `events`, `aggregates`, `entities`) e de infraestrutura
(`services`, `repositories`) tem seu próprio `__init__.py` agregador com
`__all__` — ver [`backend/conventions.md`](../../backend/conventions.md).

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | core | `src/app/core/domain/errors.py` | novo |
| 2 | core | `src/app/core/shared/errors.py` | novo |
| 3 | pacotes | `src/app/modules/identity/**/__init__.py` (vazios) | novo |
| 4 | domain | `src/app/modules/identity/domain/value_objects/cpf.py` | novo |
| 5 | domain | `src/app/modules/identity/domain/value_objects/email.py` | novo |
| 6 | domain | `src/app/modules/identity/domain/value_objects/__init__.py` | novo |
| 7 | domain | `src/app/modules/identity/domain/events/domain_events.py` | novo |
| 8 | domain | `src/app/modules/identity/domain/events/__init__.py` | novo |
| 9 | domain | `src/app/modules/identity/domain/aggregates/user.py` | novo |
| 10 | domain | `src/app/modules/identity/domain/aggregates/__init__.py` | novo |
| 11 | domain | `src/app/modules/identity/domain/entities/refresh_token.py` | novo |
| 12 | domain | `src/app/modules/identity/domain/entities/password_reset_token.py` | novo |
| 13 | domain | `src/app/modules/identity/domain/entities/__init__.py` | novo |
| 14 | infrastructure | `src/app/modules/identity/infrastructure/services/password_service.py` | novo |
| 15 | infrastructure | `src/app/modules/identity/infrastructure/services/token_service.py` | novo |
| 16 | infrastructure | `src/app/modules/identity/infrastructure/services/__init__.py` | novo |
| 17 | infrastructure | `src/app/modules/identity/infrastructure/repositories/user_repository.py` | novo |
| 18 | infrastructure | `src/app/modules/identity/infrastructure/repositories/refresh_token_repository.py` | novo |
| 19 | infrastructure | `src/app/modules/identity/infrastructure/repositories/password_reset_token_repository.py` | novo |
| 20 | infrastructure | `src/app/modules/identity/infrastructure/repositories/__init__.py` | novo |
| 21 | application | `src/app/modules/identity/application/schemas/request.py` | novo |
| 22 | application | `src/app/modules/identity/application/schemas/response.py` | novo |
| 23 | application | `src/app/modules/identity/application/usecases/utils/session.py` | novo |
| 24 | application | `src/app/modules/identity/application/usecases/auth_usecase.py` | novo |
| 25 | application | `src/app/modules/identity/application/usecases/user_usecase.py` | novo |
| 26 | api | `src/app/modules/identity/api/routers/utils/cookies.py` | novo |
| 27 | api | `src/app/modules/identity/api/routers/auth_router.py` | novo |
| 28 | api | `src/app/modules/identity/api/routers/user_router.py` | novo |
| 29 | api | `src/app/modules/identity/router.py` | novo |
| 30 | api | `src/app/modules/identity/dependencies.py` | novo |
| 31 | migration | `src/migrations/versions/0001_identity_auth.py` | novo |
| 32 | migration | `src/migrations/env.py` | editar |
| 33 | api | `src/app/main.py` | editar |
| 34 | config | `src/app/config.py` | editar |
| 35 | config | `.env.example` | editar |

---

## 2. Código

### 1. `src/app/core/domain/errors.py` — novo

```python
class DomainError(Exception):
    def __init__(self, message: str) -> None:
        super().__init__(message)
        self.message = message

class ConflictError(DomainError):
    pass

class AuthError(DomainError):
    pass

class ForbiddenError(DomainError):
    pass

class GoneError(DomainError):
    pass

class NotFoundError(DomainError):
    pass
```

Sem `status_code`/`field` na classe — o mapeamento pra HTTP é uma lista central
em `main.py` (arquivo 33), por subclasse, com fallback `422` pra `DomainError`
crua.

### 2. `src/app/core/shared/errors.py` — novo

```python
from collections.abc import Sequence

def format_validation_errors(errors: Sequence[dict]) -> list[dict]:
    result = []

    for error in errors:
        loc = [str(part) for part in error.get("loc", []) if part != "body"]
        field = ".".join(loc) if loc else "body"
        error_type = error.get("type", "")
        ctx = error.get("ctx", {})

        if error_type == "missing":
            message = "Campo obrigatório"
        elif error_type == "string_too_short":
            min_len = ctx.get("min_length", 1)
            message = "Não pode ficar em branco" if min_len <= 1 else f"Mínimo de {min_len} caracteres"
        elif error_type == "string_too_long":
            message = f"Máximo de {ctx.get('max_length', '')} caracteres"
        elif "email" in error_type or "email" in error.get("msg", "").lower():
            message = "E-mail inválido"
        elif "uuid" in error_type:
            message = "ID inválido"
        elif error_type == "enum":
            message = f"Valor inválido. Opções: {ctx.get('expected', '')}"
        elif "int" in error_type:
            message = "Deve ser um número inteiro"
        elif "float" in error_type or "decimal" in error_type:
            message = "Deve ser um número"
        elif "bool" in error_type:
            message = "Deve ser verdadeiro ou falso"
        else:
            message = error.get("msg", "Valor inválido")

        result.append({"field": field, "message": message})

    return result
```

Usado só pelo handler de `RequestValidationError` (validação Pydantic, 422) em
`main.py` — **não** é usado pelo handler de `DomainError`, que tem envelope
diferente (ver arquivo 33 e §8).

### 3. Pacotes — `__init__.py` vazios

```text
# todos novos, conteúdo vazio
src/app/modules/identity/__init__.py
src/app/modules/identity/domain/__init__.py
src/app/modules/identity/application/__init__.py
src/app/modules/identity/application/schemas/__init__.py
src/app/modules/identity/application/usecases/__init__.py
src/app/modules/identity/application/usecases/utils/__init__.py
src/app/modules/identity/infrastructure/__init__.py
src/app/modules/identity/api/__init__.py
src/app/modules/identity/api/routers/__init__.py
src/app/modules/identity/api/routers/utils/__init__.py
```

### 4. `src/app/modules/identity/domain/value_objects/cpf.py` — novo

```python
from dataclasses import dataclass

from app.core.domain import DomainError

@dataclass(frozen=True)
class CPF:
    value: str

    def __post_init__(self) -> None:
        if not _is_valid(self.value):
            raise DomainError("CPF inválido.")

def _is_valid(cpf: str) -> bool:
    if not cpf.isdigit() or len(cpf) != 11:
        return False

    if cpf == cpf[0] * 11:
        return False

    for length in (9, 10):
        total = sum(int(cpf[i]) * ((length + 1) - i) for i in range(length))
        check = (total * 10) % 11
        check = 0 if check == 10 else check

        if check != int(cpf[length]):
            return False

    return True
```

### 5. `src/app/modules/identity/domain/value_objects/email.py` — novo

```python
import re
from dataclasses import dataclass

from app.core.domain import DomainError

_EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self) -> None:
        if not _EMAIL_RE.match(self.value):
            raise DomainError("E-mail inválido.")
```

### 6. `src/app/modules/identity/domain/value_objects/__init__.py` — novo

```python
from .cpf import CPF
from .email import Email

__all__ = ["CPF", "Email"]
```

### 7. `src/app/modules/identity/domain/events/domain_events.py` — novo

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
```

`PasswordResetRequested.token` é o token **em claro** — vive só nesta linha de
`events`, nunca numa tabela de domínio (a tabela guarda o hash). Hoje nenhum
handler consome este evento (§7).

### 8. `src/app/modules/identity/domain/events/__init__.py` — novo

```python
from .domain_events import (
    PasswordResetRequested,
    UserPasswordChanged,
    UserRegistered,
    UserSecurityStampRotated,
)

__all__ = [
    "PasswordResetRequested",
    "UserPasswordChanged",
    "UserRegistered",
    "UserSecurityStampRotated",
]
```

### 9. `src/app/modules/identity/domain/aggregates/user.py` — novo

```python
from __future__ import annotations

from datetime import datetime
from uuid import UUID, uuid4

from sqlalchemy import Boolean, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain import AggregateRoot, DomainEvent, Model
from app.modules.identity.domain.events import (
    PasswordResetRequested,
    UserPasswordChanged,
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
```

### 10. `src/app/modules/identity/domain/aggregates/__init__.py` — novo

```python
from .user import User

__all__ = ["User"]
```

### 11. `src/app/modules/identity/domain/entities/refresh_token.py` — novo

```python
from datetime import datetime, timezone
from uuid import UUID

from sqlalchemy import Boolean, DateTime, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain import Model

class RefreshToken(Model):
    __tablename__ = "refresh_tokens"

    user_id: Mapped[UUID] = mapped_column(nullable=False, index=True)
    token_hash: Mapped[str] = mapped_column(String(64), nullable=False, index=True)
    expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    used: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    rotated_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    def mark_rotated(self) -> None:
        self.used = True
        self.rotated_at = datetime.now(timezone.utc)
```

Entidade filha simples — sem evento próprio, sem `AggregateRepository`. É
`Model` puro, persistida via `BaseRepository[RefreshToken]`.

### 12. `src/app/modules/identity/domain/entities/password_reset_token.py` — novo

```python
from datetime import datetime
from uuid import UUID

from sqlalchemy import DateTime, String
from sqlalchemy.orm import Mapped, mapped_column

from app.core.domain import GoneError, Model

class PasswordResetToken(Model):
    __tablename__ = "password_reset_tokens"

    user_id: Mapped[UUID] = mapped_column(nullable=False, index=True)
    token_hash: Mapped[str] = mapped_column(String(64), nullable=False, index=True)
    expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    used_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    def consume(self, now: datetime) -> None:
        if self.used_at is not None or self.expires_at <= now:
            raise GoneError("Este link não é mais válido, solicite um novo.")

        self.used_at = now
```

### 13. `src/app/modules/identity/domain/entities/__init__.py` — novo

```python
from .password_reset_token import PasswordResetToken
from .refresh_token import RefreshToken

__all__ = ["PasswordResetToken", "RefreshToken"]
```

### 14. `src/app/modules/identity/infrastructure/services/password_service.py` — novo

```python
import bcrypt

class PasswordService:
    def hash(self, plain: str) -> str:
        return bcrypt.hashpw(plain.encode("utf-8"), bcrypt.gensalt()).decode("utf-8")

    def verify(self, plain: str, hashed: str) -> bool:
        try:
            return bcrypt.checkpw(plain.encode("utf-8"), hashed.encode("utf-8"))
        except ValueError:
            return False
```

### 15. `src/app/modules/identity/infrastructure/services/token_service.py` — novo

```python
import jwt
import hashlib
import secrets
from datetime import datetime, timedelta, timezone

from app.config import settings
from app.core.domain import AuthError
from app.modules.identity.domain.aggregates import User

_ALGORITHM = "HS256"

class TokenService:
    def issue_access(self, user: User) -> tuple[str, int]:
        expires_in = settings.access_token_expire_minutes * 60
        now = datetime.now(timezone.utc)

        payload = {
            "sub": str(user.id),
            "is_admin": user.is_admin,
            "security_stamp": str(user.security_stamp),
            "type": "access",
            "iat": int(now.timestamp()),
            "exp": int((now + timedelta(seconds=expires_in)).timestamp()),
        }

        return jwt.encode(payload, settings.jwt_secret_key, algorithm=_ALGORITHM), expires_in

    def decode_access(self, token: str) -> dict:
        try:
            payload = jwt.decode(token, settings.jwt_secret_key, algorithms=[_ALGORITHM])
        except jwt.PyJWTError as exc:
            raise AuthError("Sessão inválida ou expirada.") from exc

        if payload.get("type") != "access":
            raise AuthError("Sessão inválida ou expirada.")

        return payload

    def new_opaque_token(self) -> str:
        return secrets.token_urlsafe(32)

    def hash_opaque(self, token: str) -> str:
        return hashlib.sha256(token.encode("utf-8")).hexdigest()
```

Note que `is_admin` **vai no claim do JWT** (diferente do que a checagem de
autorização usa em runtime): `get_current_user` sempre rebusca o `User` no
banco por `sub` e decide por `user.is_admin` real, não pelo claim — o claim é
só um dado a mais no token, não a fonte de verdade da autorização.

### 16. `src/app/modules/identity/infrastructure/services/__init__.py` — novo

```python
from .password_service import PasswordService
from .token_service import TokenService

__all__ = ["PasswordService", "TokenService"]
```

### 17. `src/app/modules/identity/infrastructure/repositories/user_repository.py` — novo

```python
from app.core.infrastructure.repositories import AggregateRepository
from app.modules.identity.domain.aggregates import User

class UserRepository(AggregateRepository[User]):
    model = User
```

Sem método próprio — `find_by("email"|"cpf"|"id", valor)` genérico cobre tudo
(o repositório base não ganha `find_by_id`; ver
[`backend/conventions.md`](../../backend/conventions.md)).

### 18. `src/app/modules/identity/infrastructure/repositories/refresh_token_repository.py` — novo

```python
from uuid import UUID

from app.core.infrastructure.repositories import BaseRepository
from app.modules.identity.domain.entities.refresh_token import RefreshToken

class RefreshTokenRepository(BaseRepository[RefreshToken]):
    model = RefreshToken

    async def deactivate_all_for_user(self, user_id: UUID) -> None:
        for token in await self.find_all_by(user_id=user_id):
            token.is_active = False
```

`deactivate_all_for_user` é uma consulta multi-linha que o `find_by` genérico
não cobre (não é um lookup de campo único) — por isso vira método próprio,
igual ao racional de `SessionRepository.find_by_id_for_update` em
`catalog-admin-management`.

### 19. `src/app/modules/identity/infrastructure/repositories/password_reset_token_repository.py` — novo

```python
from datetime import datetime
from uuid import UUID

from app.core.infrastructure.repositories import BaseRepository
from app.modules.identity.domain.entities.password_reset_token import PasswordResetToken

class PasswordResetTokenRepository(BaseRepository[PasswordResetToken]):
    model = PasswordResetToken

    async def invalidate_all_for_user(self, user_id: UUID, now: datetime) -> None:
        for token in await self.find_all_by(user_id=user_id):
            if token.used_at is None:
                token.used_at = now
```

### 20. `src/app/modules/identity/infrastructure/repositories/__init__.py` — novo

```python
from .password_reset_token_repository import PasswordResetTokenRepository
from .refresh_token_repository import RefreshTokenRepository
from .user_repository import UserRepository

__all__ = [
    "UserRepository",
    "PasswordResetTokenRepository",
    "RefreshTokenRepository",
]
```

### 21. `src/app/modules/identity/application/schemas/request.py` — novo

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
```

`pydantic.BaseModel` puro — sem `CamelModel`, sem alias generator. O contrato
inteiro é snake_case (ver §8).

### 22. `src/app/modules/identity/application/schemas/response.py` — novo

```python
from uuid import UUID
from pydantic import BaseModel

class TokenResponse(BaseModel):
    access_token: str
    expires_in: int

class UserResponse(BaseModel):
    id: UUID
    name: str
    email: str
    cpf: str
    is_admin: bool
```

Só estes dois — não existe `UserDetailResponse`/`UserSummaryResponse`/
`PagedUsersResponse`/`MessageResponse`. A resposta de `forgot_password` é um
`dict[str, str]` cru, montada direto no usecase (arquivo 24).

### 23. `src/app/modules/identity/application/usecases/utils/session.py` — novo

```python
from datetime import datetime, timedelta, timezone

from app.config import settings
from app.modules.identity.application.schemas.response import TokenResponse
from app.modules.identity.domain.aggregates import User
from app.modules.identity.domain.entities.refresh_token import RefreshToken
from app.modules.identity.infrastructure.repositories.refresh_token_repository import RefreshTokenRepository
from app.modules.identity.infrastructure.services.token_service import TokenService

async def issue_session(
    user: User, token_service: TokenService, refresh_repository: RefreshTokenRepository
) -> tuple[TokenResponse, str]:
    access, expires_in = token_service.issue_access(user)
    raw_refresh = token_service.new_opaque_token()

    expires_at = datetime.now(timezone.utc) + timedelta(days=settings.refresh_token_expire_days)

    await refresh_repository.save(
        RefreshToken(
            user_id=user.id,
            token_hash=token_service.hash_opaque(raw_refresh),
            expires_at=expires_at,
        )
    )

    return TokenResponse(access_token=access, expires_in=expires_in), raw_refresh
```

Extraído pra `utils/` porque **dois** usecases emitem sessão: `AuthUseCase`
(login/refresh) e `UserUseCase` (registro loga o usuário direto).

### 24. `src/app/modules/identity/application/usecases/auth_usecase.py` — novo

```python
from uuid import UUID
from datetime import datetime, timedelta, timezone

from sqlalchemy.ext.asyncio import AsyncSession

from app.database import AsyncSessionLocal
from app.core.domain import AuthError, DomainError, GoneError

from app.modules.identity.domain.aggregates import User
from app.modules.identity.domain.entities.password_reset_token import PasswordResetToken
from app.modules.identity.application.schemas.response import TokenResponse
from app.modules.identity.application.usecases.utils.session import issue_session

from app.modules.identity.infrastructure.services import PasswordService, TokenService

from app.modules.identity.application.schemas.request import (
    LoginRequest,
    ResetPasswordRequest,
    ChangePasswordRequest,
    ForgotPasswordRequest
)

from app.modules.identity.infrastructure.repositories import (
    PasswordResetTokenRepository,
    RefreshTokenRepository,
    UserRepository
)

class AuthUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

        self._user_repository = UserRepository(session)
        self._refresh_repository = RefreshTokenRepository(session)
        self._password_reset_repository = PasswordResetTokenRepository(session)

        self._token_service = TokenService()
        self._password_service = PasswordService()

    async def login(self, req: LoginRequest) -> tuple[TokenResponse, str]:
        user = await self._user_repository.find_by("email", req.email.strip().lower())
        if user is None or not self._password_service.verify(req.password, user.password_hash):
            raise AuthError("E-mail ou senha inválidos.")

        return await issue_session(user, self._token_service, self._refresh_repository)

    async def logout(self, user: User) -> None:
        user.rotate_security_stamp()

        await self._user_repository.save(user)
        await self._refresh_repository.deactivate_all_for_user(user.id)

    async def refresh(self, raw_refresh: str | None) -> tuple[TokenResponse, str]:
        record = None

        if raw_refresh:
            token_hash = self._token_service.hash_opaque(raw_refresh)
            record = await self._refresh_repository.find_by("token_hash", token_hash)

        now = datetime.now(timezone.utc)
        if record is None or record.expires_at <= now:
            raise AuthError("Sessão expirada.")

        if record.used:
            await self._revoke_all_sessions(record.user_id)
            raise AuthError("Sessão expirada.")

        user = await self._user_repository.find_by("id", record.user_id)
        if user is None:
            raise AuthError("Sessão expirada.")

        record.mark_rotated()
        return await issue_session(user, self._token_service, self._refresh_repository)

    async def change_password(self, user: User, req: ChangePasswordRequest) -> None:
        if not self._password_service.verify(req.current_password, user.password_hash):
            raise DomainError("A senha atual não confere.")

        user.change_password(self._password_service.hash(req.new_password))

        await self._user_repository.save(user)
        await self._refresh_repository.deactivate_all_for_user(user.id)

    async def forgot_password(self, req: ForgotPasswordRequest) -> dict[str, str]:
        user = await self._user_repository.find_by("email", req.email.strip().lower())
        if user is not None:
            now = datetime.now(timezone.utc)
            await self._password_reset_repository.invalidate_all_for_user(user.id, now)

            raw = self._token_service.new_opaque_token()
            expires_at = now + timedelta(hours=1)

            await self._password_reset_repository.save(
                PasswordResetToken(
                    user_id=user.id,
                    token_hash=self._token_service.hash_opaque(raw),
                    expires_at=expires_at,
                )
            )

            user.request_password_reset(raw, expires_at)
            await self._user_repository.save(user)

        return {"message": "Se houver uma conta com esse e-mail, enviamos um link."}

    async def reset_password(self, req: ResetPasswordRequest) -> None:
        token_hash = self._token_service.hash_opaque(req.token)
        record = await self._password_reset_repository.find_by("token_hash", token_hash)

        if record is None:
            raise GoneError("Este link não é mais válido, solicite um novo.")

        record.consume(datetime.now(timezone.utc))
        user = await self._user_repository.find_by("id", record.user_id)

        if user is None:
            raise GoneError("Este link não é mais válido, solicite um novo.")

        user.reset_password(self._password_service.hash(req.password))

        await self._user_repository.save(user)
        await self._refresh_repository.deactivate_all_for_user(user.id)

    async def _revoke_all_sessions(self, user_id: UUID) -> None:
        async with AsyncSessionLocal() as session, session.begin():
            users = UserRepository(session)
            user = await users.find_by("id", user_id)

            if user is not None:
                user.rotate_security_stamp()

                await users.save(user)
                await RefreshTokenRepository(session).deactivate_all_for_user(user_id)
```

Diferença de comportamento relevante em relação a uma versão anterior deste
documento: `logout`/`change_password`/`reset_password` não só rotacionam o
`security_stamp` — também **desativam todos os refresh tokens do usuário**
(`deactivate_all_for_user`), derrubando qualquer sessão paralela de fato, não
só invalidando o access token corrente. `refresh` sobre um token já usado
detecta reuso e revoga tudo numa sessão de banco própria
(`_revoke_all_sessions`), porque nesse ponto o `AsyncSession` da requisição
pode já estar num estado que não convém reaproveitar para uma revogação em
massa.

### 25. `src/app/modules/identity/application/usecases/user_usecase.py` — novo

```python
from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain import ConflictError, NotFoundError

from app.modules.identity.domain.aggregates import User
from app.modules.identity.domain.value_objects import CPF, Email
from app.modules.identity.application.schemas.request import RegisterRequest
from app.modules.identity.application.schemas.response import TokenResponse, UserResponse
from app.modules.identity.application.usecases.utils.session import issue_session

from app.modules.identity.infrastructure.services import PasswordService, TokenService

from app.modules.identity.infrastructure.repositories import (
    RefreshTokenRepository,
    UserRepository
)

class UserUseCase:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

        self._user_repository = UserRepository(session)
        self._refresh_repository = RefreshTokenRepository(session)

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

        return UserResponse(
            id=user.id, name=user.name, email=user.email, cpf=user.cpf, is_admin=user.is_admin
        )
```

Sem mensagem por campo (`field=`) nos `ConflictError` — a classe real não tem
esse conceito. `get_by_id` não checa quem está pedindo — a checagem "próprio
usuário ou admin" fica no **router** (arquivo 28), não aqui: o usecase não
conhece papel/autorização, só busca e traduz `None` → `NotFoundError`.

### 26. `src/app/modules/identity/api/routers/utils/cookies.py` — novo

```python
from fastapi import Response

from app.config import settings

REFRESH_COOKIE = "refresh_token"
REFRESH_PATH = "/identity"

def set_refresh_cookie(response: Response, raw_refresh: str) -> None:
    response.set_cookie(
        key=REFRESH_COOKIE,
        value=raw_refresh,
        httponly=True,
        secure=settings.environment != "development",
        samesite="strict",
        path=REFRESH_PATH,
        max_age=settings.refresh_token_expire_days * 24 * 3600,
    )
```

### 27. `src/app/modules/identity/api/routers/auth_router.py` — novo

```python
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import APIRouter, Depends, Request, Response, status

from app.dependencies import get_db

from app.modules.identity.dependencies import get_current_user

from app.modules.identity.domain.aggregates import User
from app.modules.identity.application.usecases.auth_usecase import AuthUseCase

from app.modules.identity.application.schemas.request import (
    ChangePasswordRequest,
    ForgotPasswordRequest,
    LoginRequest,
    ResetPasswordRequest,
)

from app.modules.identity.application.schemas.response import TokenResponse
from app.modules.identity.api.routers.utils.cookies import REFRESH_COOKIE, REFRESH_PATH, set_refresh_cookie

router = APIRouter(prefix="/identity", tags=["01.Identity - Auth"])

@router.post("/login", response_model=TokenResponse)
async def login(body: LoginRequest, response: Response, session: AsyncSession = Depends(get_db)) -> TokenResponse:
    token, raw_refresh = await AuthUseCase(session).login(body)
    set_refresh_cookie(response, raw_refresh)

    return token

@router.post("/logout", status_code=status.HTTP_204_NO_CONTENT)
async def logout(response: Response, session: AsyncSession = Depends(get_db), user: User = Depends(get_current_user)) -> None:
    await AuthUseCase(session).logout(user)
    response.delete_cookie(REFRESH_COOKIE, path=REFRESH_PATH)

@router.post("/refresh", response_model=TokenResponse)
async def refresh(request: Request, response: Response, session: AsyncSession = Depends(get_db)) -> TokenResponse:
    token, raw_refresh = await AuthUseCase(session).refresh(request.cookies.get(REFRESH_COOKIE))
    set_refresh_cookie(response, raw_refresh)

    return token

@router.post("/password/change", status_code=status.HTTP_204_NO_CONTENT)
async def change_password(body: ChangePasswordRequest, session: AsyncSession = Depends(get_db), user: User = Depends(get_current_user)) -> None:
    await AuthUseCase(session).change_password(user, body)

@router.post("/password/forgot", response_model=dict[str, str], status_code=status.HTTP_202_ACCEPTED)
async def forgot_password(body: ForgotPasswordRequest, session: AsyncSession = Depends(get_db)) -> dict[str, str]:
    return await AuthUseCase(session).forgot_password(body)

@router.post("/password/reset", status_code=status.HTTP_204_NO_CONTENT)
async def reset_password(body: ResetPasswordRequest, session: AsyncSession = Depends(get_db)) -> None:
    await AuthUseCase(session).reset_password(body)
```

### 28. `src/app/modules/identity/api/routers/user_router.py` — novo

```python
from uuid import UUID

from fastapi import APIRouter, Depends, Response, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain import ForbiddenError

from app.dependencies import get_db
from app.modules.identity.dependencies import get_current_user

from app.modules.identity.domain.aggregates import User
from app.modules.identity.application.schemas.request import RegisterRequest
from app.modules.identity.application.schemas.response import TokenResponse, UserResponse

from app.modules.identity.api.routers.utils.cookies import set_refresh_cookie
from app.modules.identity.application.usecases.user_usecase import UserUseCase

router = APIRouter(prefix="/identity/users", tags=["02.Identity - User"])

@router.post("/register", response_model=TokenResponse, status_code=status.HTTP_201_CREATED)
async def register(body: RegisterRequest, response: Response, session: AsyncSession = Depends(get_db)) -> TokenResponse:
    token, raw_refresh = await UserUseCase(session).register(body)
    set_refresh_cookie(response, raw_refresh)

    return token

@router.get("/{user_id}", response_model=UserResponse)
async def get(user_id: UUID, session: AsyncSession = Depends(get_db), current_user: User = Depends(get_current_user)) -> UserResponse:
    if not current_user.is_admin and current_user.id != user_id:
        raise ForbiddenError("Acesso restrito ao próprio usuário.")

    return await UserUseCase(session).get_by_id(user_id)
```

Cadastro é `POST /identity/users/register` (não `POST /identity/users` puro) —
libera `POST /identity/users` pra uma eventual listagem futura, se algum dia
houver RF pra isso. A
checagem "próprio usuário ou admin" mora aqui, no router — nunca dentro do
usecase (mesmo racional de `require_admin` em `catalog-admin-management`:
usecase não tem sufixo nem lógica de papel).

### 29. `src/app/modules/identity/router.py` — novo

```python
from fastapi import APIRouter

from app.modules.identity.api.routers.auth_router import router as auth_router
from app.modules.identity.api.routers.user_router import router as user_router

router = APIRouter()
router.include_router(auth_router)
router.include_router(user_router)
```

### 30. `src/app/modules/identity/dependencies.py` — novo

```python
from uuid import UUID

from fastapi import Depends
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db
from app.core.domain import AuthError, ForbiddenError
from app.modules.identity.domain.aggregates import User

from app.modules.identity.infrastructure.services import TokenService
from app.modules.identity.infrastructure.repositories import UserRepository

_bearer = HTTPBearer(auto_error=False)

async def get_current_user(credentials: HTTPAuthorizationCredentials | None = Depends(_bearer), session: AsyncSession = Depends(get_db)) -> User:
    if credentials is None or not credentials.credentials:
        raise AuthError("Não autenticado.")

    payload = TokenService().decode_access(credentials.credentials)
    user = await UserRepository(session).find_by("id", UUID(payload["sub"]))

    if user is None or str(user.security_stamp) != payload.get("security_stamp"):
        raise AuthError("Sessão inválida ou expirada.")

    return user

async def require_admin(user: User = Depends(get_current_user)) -> User:
    if not user.is_admin:
        raise ForbiddenError("Acesso restrito a administradores.")

    return user
```

Sem `CurrentUser` (dataclass própria) — as duas dependencies retornam o
próprio aggregate `User`, buscado de novo a cada request. `require_admin` é o
que `catalog-admin-management` importa daqui (`Depends(require_admin)` nas
rotas de admin).

### 31. `src/migrations/versions/0001_identity_auth.py` — novo

```python
"""identity-auth: events (outbox) + users, refresh_tokens, password_reset_tokens

Revision ID: 0001_identity_auth
Revises:
Create Date: 2026-09-04

Primeira migration do repo. Alem das tabelas do modulo identity, cria a tabela
`events` do outbox (core), que passa a existir com a primeira feature — o
env.py ja mapeia app.outbox.models no metadata. Os indices unicos parciais
(email/cpf onde is_active) sao escritos a mao: o autogenerate do Alembic nao os
gera corretamente.
"""

from typing import Sequence, Union

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

revision: str = "0001_identity_auth"
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

def _model_columns() -> list[sa.Column]:
    # Colunas herdadas de core Model, iguais em toda tabela.
    return [
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("is_active", sa.Boolean(), nullable=False),
    ]

def upgrade() -> None:
    op.create_table(
        "events",
        *_model_columns(),
        sa.Column("aggregate_id", postgresql.UUID(as_uuid=False), nullable=False),
        sa.Column("event_type", sa.String(length=128), nullable=False),
        sa.Column("payload", postgresql.JSONB(astext_type=sa.Text()), nullable=False),
        sa.Column("dispatched_at", sa.DateTime(timezone=True), nullable=True),
    )
    op.create_index("ix_events_aggregate_id", "events", ["aggregate_id"])
    op.create_index("ix_events_event_type", "events", ["event_type"])

    op.create_table(
        "users",
        *_model_columns(),
        sa.Column("name", sa.String(length=120), nullable=False),
        sa.Column("cpf", sa.String(length=11), nullable=False),
        sa.Column("email", sa.String(length=254), nullable=False),
        sa.Column("password_hash", sa.String(length=60), nullable=False),
        sa.Column("is_admin", sa.Boolean(), nullable=False),
        sa.Column("security_stamp", postgresql.UUID(as_uuid=True), nullable=False),
    )
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
        *_model_columns(),
        sa.Column("user_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("token_hash", sa.String(length=64), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("used", sa.Boolean(), nullable=False),
        sa.Column("rotated_at", sa.DateTime(timezone=True), nullable=True),
    )
    op.create_index("ix_refresh_tokens_user_id", "refresh_tokens", ["user_id"])
    op.create_index("ix_refresh_tokens_token_hash", "refresh_tokens", ["token_hash"])

    op.create_table(
        "password_reset_tokens",
        *_model_columns(),
        sa.Column("user_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("token_hash", sa.String(length=64), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("used_at", sa.DateTime(timezone=True), nullable=True),
    )
    op.create_index("ix_password_reset_tokens_user_id", "password_reset_tokens", ["user_id"])
    op.create_index("ix_password_reset_tokens_token_hash", "password_reset_tokens", ["token_hash"])

def downgrade() -> None:
    op.drop_table("password_reset_tokens")
    op.drop_table("refresh_tokens")
    op.drop_table("users")
    op.drop_table("events")
```

Uma única migration — não duas (`events` e `identity` saem juntas, não em
`0001_outbox_events.py` + `0002_identity_auth.py` separados).

### 32. `src/migrations/env.py` — editar

```python
# src/migrations/env.py — editar (após "import app.outbox.models")
import app.outbox.models  # noqa: F401,E402

# Cada feature acrescenta o import do próprio módulo aqui.
import app.modules.identity.domain.aggregates.user  # noqa: F401,E402
import app.modules.identity.domain.entities.password_reset_token  # noqa: F401,E402
import app.modules.identity.domain.entities.refresh_token  # noqa: F401,E402
```

### 33. `src/app/main.py` — editar

```python
# 1) novos imports (junto aos demais do topo)
from app.core.shared import format_validation_errors
from app.modules.identity.router import router as identity_router
from app.core.domain import AuthError, ConflictError, DomainError, ForbiddenError, GoneError, NotFoundError

# 2) handler de RequestValidationError (422 de validação Pydantic)
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content={"detail": format_validation_errors(exc.errors())},
    )

# 3) mapeamento de erro de domínio por subclasse + handler único
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

# 4) após o add_middleware(CORSMiddleware, ...):
app.include_router(identity_router)
```

Envelope de `DomainError` é **`{"detail": "mensagem em string"}`** — diferente
do envelope de validação Pydantic (`{"detail": [{"field", "message"}]}`, via
`format_validation_errors`). São dois formatos distintos pro frontend tratar
(ver §8).

### 34. `src/app/config.py` — editar

Adicionar ao corpo de `Settings` (JWT e duração das sessões):

```python
    jwt_secret_key: str
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7
```

### 35. `.env.example` — editar

```bash
# --- JWT (identity-auth) ---
# SECRET — gere com: openssl rand -hex 32
JWT_SECRET_KEY=
# CONFIG
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7
```

---

## 3. Onde cada regra de negócio entra

| Regra | Arquivo · função | Como |
| --- | --- | --- |
| RF09 — CPF com dígitos verificadores | `domain/value_objects/cpf.py` · `__post_init__` / `_is_valid` | módulo 11 nos dois DV; `DomainError("CPF inválido.")` → 422 |
| RF09 — unicidade de CPF e e-mail | `application/usecases/user_usecase.py` · `register` + `migrations/0001` índices parciais `uq_users_*_active` | `exists_by("cpf"/"email", ...)` antes de criar → `ConflictError` (409); o índice parcial é a rede de segurança sob concorrência |
| RF09 — login genérico | `auth_usecase.py` · `login` | e-mail inexistente e senha errada → o mesmo `AuthError("E-mail ou senha inválidos.")` |
| RF09 — resposta neutra em "esqueci a senha" | `api/routers/auth_router.py` · `forgot_password` + `auth_usecase.py` · `forgot_password` | rota sempre 202 com a mesma mensagem; o usecase retorna a mesma mensagem mesmo quando o e-mail não existe |
| RF09 — token de redefinição uso único / 1 h | `domain/entities/password_reset_token.py` · `consume` + `auth_usecase.py` · `forgot_password` | `consume` recusa (`GoneError` 410) se `used_at` ou expirado; `forgot_password` grava `expires_at = now + 1h` |
| RF09 — nova solicitação invalida as anteriores | `auth_usecase.py` · `forgot_password` + `password_reset_token_repository.py` · `invalidate_all_for_user` | invalida tudo que ainda não foi usado antes de emitir o novo token |
| RF09 — logout / troca / redefinição derrubam todas as sessões | `domain/aggregates/user.py` · `rotate_security_stamp` + `infrastructure/repositories/refresh_token_repository.py` · `deactivate_all_for_user` | novo `security_stamp` invalida qualquer access token anterior; `deactivate_all_for_user` também desativa todo refresh token do usuário |
| RF09 — reuso de refresh token detectado | `auth_usecase.py` · `refresh` / `_revoke_all_sessions` + `domain/entities/refresh_token.py` · `mark_rotated` | `mark_rotated` marca `used=True` a cada uso; reuso do mesmo token → revoga tudo numa sessão de banco própria + 401 |
| RF09 — `GET /identity/users/{id}` restrito a si mesmo ou admin | `api/routers/user_router.py` · `get` | `current_user.is_admin` ou `current_user.id == user_id`, senão `ForbiddenError` 403 — checagem no router, não no usecase |
| RNF01 — hash de senha | `infrastructure/services/password_service.py` | bcrypt com salt automático; `password_hash` nunca aparece em nenhum schema de response nem em log |
| RNF01 — `security_stamp` no claim e verificado a cada request | `token_service.py` · `issue_access` (claim) + `dependencies.py` · `get_current_user` | stamp do token comparado ao do `User` buscado de novo no banco; divergência → 401 mesmo com JWT válido |
| RNF01 — nada sensível em log / erro / URL | `errors.py` (`DomainError` sem CPF/e-mail na mensagem), CPF guardado só em dígitos | mensagens de erro em linguagem de negócio; o refresh token opaco vai em cookie `HttpOnly`, nunca em query string de rota da API |

---

## 4. DevOps

**Variável nova `JWT_SECRET_KEY`** (classificação `SECRET`): já listada em
`.env.example` sem valor. Gerar e configurar localmente:

```bash
echo "JWT_SECRET_KEY=$(openssl rand -hex 32)" >> .env.local
```

**Revisão de 2026-09-17:** o job de testes do CI (`.github/workflows/ci.yml`)
foi removido junto com `tests/` — a pipeline hoje só builda a imagem Docker,
sem consumir `JWT_SECRET_KEY`. O valor continua necessário só em
`.env.local` para rodar localmente; quando a suite for reconstruída, o job de
testes volta a precisar de um valor fixo de desenvolvimento direto no
workflow (`JWT_SECRET_KEY: ci-only-not-a-real-secret`) — não é secret de
repositório, porque não assina nada fora do próprio pipeline.

- **Migração**: `alembic upgrade head` aplica `0001_identity_auth` (primeira
  migration do repositório — cria `events`, `users`, `refresh_tokens`,
  `password_reset_tokens` numa tacada só). Rodar contra um banco limpo antes
  de subir a app.

---

## 5. Passo a passo TBD (Backend)

```bash
git checkout master && git pull && git checkout -b feat/09-identity-auth

# commit 1 — infra compartilhada + domínio
git add src/app/core/domain/errors.py src/app/core/shared/errors.py \
        src/app/modules/identity/__init__.py src/app/modules/identity/domain
git commit -m "feat(identity): modelar User, tokens e eventos de domínio"

# commit 2 — infrastructure (services + repositórios)
git add src/app/modules/identity/infrastructure
git commit -m "feat(identity): hasher bcrypt, token service e repositórios"

# commit 3 — application (AuthUseCase + UserUseCase + schemas)
git add src/app/modules/identity/application
git commit -m "feat(identity): usecases de auth e user, schemas de request/response"

# commit 4 — api (auth_router + user_router) + migration + config
git add src/app/modules/identity/api src/app/modules/identity/router.py \
        src/app/modules/identity/dependencies.py src/migrations \
        src/app/main.py src/app/config.py .env.example
git commit -m "feat(identity): expor rotas de /auth e /users, dependencies e migration"
```

Depois: `/team-ludens:tbd-pr` (senior-dev Modo 2 + `/code-review`) → push → PR
`Closes #<NN>` → merge (1 aprovação + CI verde). Antes do PR, localmente:
`alembic upgrade head` — sem
lint automatizado (o projeto não usa Ruff nem outro formatter, ver
[`code-style.md`](../../backend/code-style.md)).

---

## 6. Ordem entre as superfícies

Backend começa a partir do
`logic.md`. Frontend começa em paralelo contra o contrato-alvo do
`integration.md` — `fetcher` com refresh e `AuthContext` não dependem do backend
pronto. O `integration.md` vira `canônico` e a integração real acontece depois do
merge do backend.

---

## 7. Débitos técnicos registrados

O fluxo de sessão/cadastro (login, refresh, logout, troca de senha, cadastro,
leitura de usuário) está **completo e mergeado**. Dois pedaços do RF09 ficaram
para trás e não têm código correspondente hoje:

- **Envio do e-mail de redefinição de senha.** O aggregate `User` levanta
  `PasswordResetRequested` (arquivo 7) e o outbox relay processa a fila de
  `events`, mas **nenhum handler está registrado** pra esse tipo de evento —
  não existe módulo `notification`, nem `identity/handlers.py`, nem envio de
  e-mail real (SMTP) em nenhum lugar do código. O evento fica em
  `dispatched_at = NULL` indefinidamente. Reconstruir isso quando o e-mail
  transacional entrar em escopo: criar o handler consumindo
  `PasswordResetRequested` (usa `payload["token"]`/`payload["email"]` já
  gravados no evento — não precisa mudar nada no `identity`).
- **Criação de conta admin.** Não existe `scripts/seed_admin.py` nem qualquer
  outro mecanismo de bootstrap — a primeira conta `is_admin=True` precisa ser
  promovida manualmente no banco. Sem isso, `catalog-admin-management` (e
  qualquer outra rota atrás de `require_admin`) fica inacessível num ambiente
  novo sem uma intervenção manual.

Nenhum dos dois bloqueia o que já está mergeado — são lacunas a fechar antes
de depender deles em produção.

---

## 8. Ajustes feitos no `integration.md`

Ao fechar este documento (revisão de 2026-09-11), o `integration.md` foi
precisado nos pontos que estavam `<a definir globalmente>`. **Atualização de
2026-09-17: os pontos abaixo foram conferidos e aplicados ao `integration.md`
real, que agora está `status: canônico`** — incluindo a troca de prefixo para
`/identity/...` (§ revisão de 2026-09-17 acima):

- **Envelope de erro**: são **dois formatos diferentes**, não um só. Erro de
  validação Pydantic (422): `{"detail": [{"field": string, "message":
  string}]}`. Erro de domínio (409/401/403/404/410): `{"detail": "mensagem em
  string"}` — sem lista, sem campo. O frontend precisa tratar os dois formatos
  separadamente.
- **Prefixo / versionamento**: sem versionamento no N1 — as rotas montadas em
  `/identity/...` e `/identity/users/...`; base = `NEXT_PUBLIC_API_URL`.
- **`expires_in`**: em segundos (`ACCESS_TOKEN_EXPIRE_MINUTES * 60`), campo
  snake_case (não `expiresIn`) — contrato inteiro é snake_case, sem exceção.
- **`is_admin`** no `/identity/users/{id}` e no claim do JWT: `true`/`false` —
  não há `role`/`Role` em nenhum lugar do contrato.
- **`POST /identity/logout`**: além de 204, envia `Set-Cookie` apagando
  `refresh_token` (`Path=/identity`).
- **`register` e a leitura de usuário saem da sessão**: `AuthUseCase` fica só
  com sessão (login/refresh/logout/senha); `UserUseCase` cobre cadastro e
  leitura. `POST /identity/users/register`; sem `GET /auth/me` — vira `GET
  /identity/users/{id}` (usuário comum só o próprio, 403 em qualquer outro;
  admin qualquer um). O frontend descobre o próprio `id` decodificando o claim
  `sub` do access token (payload do JWT, sem verificar assinatura — a
  verificação é sempre do backend).
- **Sem listagem de usuários no contrato** — não existe `GET
  /identity/users` (nem paginado, nem de outra forma). Se isso virar um
  requisito real no futuro, entra como uma spec nova, com RF próprio.
