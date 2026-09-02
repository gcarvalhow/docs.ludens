---
status: draft
spec: identity-auth
created_at: 2026-09-01
---

# Cadastro e autenticação do comprador — Implementation Spec

**Resumo:** módulo `identity` com o aggregate `Buyer`, dual-token JWT +
`security_stamp` + bcrypt, e recuperação de senha por token de uso único.
**RF:** RF09 · **RN:** — (reforça RNF01) · **Módulo backend:** `identity` ·
**Feature frontend:** `account` ·
**Contrato:** `specs/identity-auth/integration.md`

Carregar antes: skill `backend-architecture` (api.ludens), skill
`frontend-architecture` (web.ludens), `docs.ludens/backend/security/authentication.md`.

---

## A. Backend — responsável: Igor

Ordem: `domain/` → `application/` → `infrastructure/` → `api/` → `migrations/`.

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | domain | `src/app/modules/identity/domain/value_objects/cpf.py` | `CPF` frozen dataclass, valida dígitos verificadores no `__post_init__`, `DomainError` se inválido |
| 2 | domain | `src/app/modules/identity/domain/value_objects/email.py` | `Email` frozen, valida formato |
| 3 | domain | `src/app/modules/identity/domain/enumerations/role.py` | `Role(str, Enum)`: `BUYER`, `ADMIN` |
| 4 | domain | `src/app/modules/identity/domain/aggregates/buyer.py` | aggregate `Buyer`: campos `name`, `cpf`, `email`, `password_hash`, `role`, `security_stamp` (UUID). Métodos: `register()`, `change_password(new_hash)`, `reset_password(new_hash)`, `rotate_security_stamp()`. Eventos: `BuyerRegistered`, `BuyerPasswordChanged`, `BuyerSecurityStampRotated` |
| 5 | domain | `src/app/modules/identity/domain/aggregates/password_reset_token.py` | aggregate `PasswordResetToken`: `buyer_id`, `token_hash` (sha256), `expires_at`, `used_at`. Métodos: `issue()`, `consume()`. Regra: `consume()` recusa se `used_at` ou expirado |
| 6 | domain | `src/app/modules/identity/domain/aggregates/refresh_token.py` | `RefreshToken`: `buyer_id`, `token_hash`, `expires_at`, `rotated_at`, `used`. `rotate()` marca `used=True` e emite `RefreshTokenRotated` |
| 7 | domain | `src/app/modules/identity/domain/events/identity_events.py` | os eventos acima, `@dataclass(frozen=True)` estendendo `DomainEvent` |
| 8 | application | `.../application/schemas/request.py` | `RegisterRequest` (name, cpf: `str` só dígitos, email, password: `Field(min_length=8)`), `LoginRequest`, `ChangePasswordRequest`, `ForgotPasswordRequest`, `ResetPasswordRequest` |
| 9 | application | `.../application/schemas/response.py` | `TokenResponse` (`access_token`, `expires_in`), `BuyerResponse` (`id`, `name`, `email`, `cpf`, `role`), `MessageResponse` |
| 10 | application | `.../application/usecases/auth_usecase.py` | `register`, `login`, `refresh`, `logout`, `change_password`, `forgot_password`, `reset_password`. Hashing bcrypt e geração/verificação de JWT ficam em serviços de infra chamados aqui |
| 11 | infrastructure | `.../infrastructure/services/password_hasher.py` | `PasswordHasher` — `hash()`, `verify()` com bcrypt |
| 12 | infrastructure | `.../infrastructure/services/token_service.py` | `TokenService` — `issue_access(buyer)`, `decode_access(token)`, `new_opaque_token()`, `hash_opaque(token)` |
| 13 | infrastructure | `.../infrastructure/repositories/buyer_repository.py` | `BuyerRepository(AggregateRepository[Buyer])` + `find_by_email`, `find_by_cpf` |
| 14 | infrastructure | `.../infrastructure/repositories/*_token_repository.py` | repos de `RefreshToken` e `PasswordResetToken` |
| 15 | outbox | `.../modules/identity/handlers.py` + `outbox/registry.py` | `@register("PasswordResetRequested")` → handler chama `EmailService` de `notification` para enviar o link (idempotente por `token_hash`) |
| 16 | api | `.../api/routers/auth_router.py` | as 8 rotas do `integration.md`; `login`/`register`/`forgot`/`reset`/`refresh` públicas; `logout`/`change`/`me` com `Depends(get_current_buyer)` |
| 17 | api | `.../modules/identity/router.py` | agrega `auth_router` |
| 18 | api | `.../modules/identity/dependencies.py` | exporta `get_current_buyer` (decodifica Bearer, valida `security_stamp` contra o `Buyer`, 401) e `require_admin` (403 se `role != ADMIN`) |
| 19 | migration | `migrations/versions/xxxx_identity_auth.py` | tabelas `buyers`, `refresh_tokens`, `password_reset_tokens`; índice único parcial em `buyers.email`/`buyers.cpf` onde `is_active` |
| 20 | config | `src/app/config.py` | já tem `jwt_secret_key`, `access_token_expire_minutes`, `refresh_token_expire_days` |

### Código a colar (recorte principal)

```python
# src/app/modules/identity/domain/aggregates/buyer.py
class Buyer(AggregateRoot, Model):
    __tablename__ = "buyers"

    name: Mapped[str] = mapped_column(String(120), nullable=False)
    cpf: Mapped[str] = mapped_column(String(11), nullable=False)
    email: Mapped[str] = mapped_column(String(254), nullable=False)
    password_hash: Mapped[str] = mapped_column(String(60), nullable=False)
    role: Mapped[Role] = mapped_column(default=Role.BUYER, nullable=False)
    security_stamp: Mapped[UUID] = mapped_column(default=uuid4, nullable=False)

    @classmethod
    def register(cls, name: str, cpf: CPF, email: Email, password_hash: str) -> "Buyer":
        buyer = cls()
        buyer.raise_event(lambda v: BuyerRegistered(
            version=v, id=buyer.id, name=name, cpf=cpf.value, email=email.value,
            password_hash=password_hash, role=Role.BUYER,
        ))
        return buyer

    def change_password(self, new_hash: str) -> None:
        self.raise_event(lambda v: BuyerPasswordChanged(version=v, id=self.id, password_hash=new_hash))
        self.rotate_security_stamp()

    def rotate_security_stamp(self) -> None:
        self.raise_event(lambda v: BuyerSecurityStampRotated(version=v, id=self.id, security_stamp=uuid4()))

    def _when_BuyerRegistered(self, e): ...
    def _when_BuyerPasswordChanged(self, e): self.password_hash = e.password_hash
    def _when_BuyerSecurityStampRotated(self, e): self.security_stamp = e.security_stamp
```

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-identity-auth
commit 1  feat(identity): modelar Buyer, tokens e eventos de domínio
commit 2  feat(identity): usecase de auth + schemas
commit 3  feat(identity): hasher bcrypt, token service e repositórios
commit 4  feat(identity): expor rotas de auth, dependencies e migration
/team-ludens:tbd-pr   (roda senior-dev Modo 2 + /code-review antes do push)
```

---

## B. Frontend — responsável: Diego · feature `account`

> **Stack:** Next.js (App Router) + TypeScript. Convenções completas na skill `frontend-architecture` do `team.ludens`: `services/` fica sob `server/`, tipos em `server/types/` (`z.infer`), rotas em `src/app/<rota>/page.tsx` (Server Component), `'use client'` só onde há hook/estado/handler, barrels `index.ts`. Os caminhos abaixo são o mapa da feature — ajuste a extensão/pasta ao padrão da skill.

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.ts` | grupo `auth`: register, login, refresh, logout, me, passwordForgot, passwordReset |
| 2 | schemas | `src/features/account/schemas/auth.schema.ts` | Zod: `registerSchema` (cpf com refine de 11 dígitos, email, password min 8), `loginSchema`, `changePasswordSchema`, `forgotSchema`, `resetSchema`, `tokenResponseSchema`, `buyerSchema` |
| 3 | services | `src/features/account/services/auth.service.ts` | `register`, `login`, `refresh`, `logout`, `fetchMe`, `forgotPassword`, `resetPassword` — `fetcher` com `withCredentials` para o cookie |
| 4 | lib | `src/lib/fetcher.ts` | wrapper fetch/axios: injeta `Authorization`, em 401 tenta `refresh` uma vez e repete; `withCredentials: true` |
| 5 | context | `src/features/account/contexts/AuthContext.tsx` | guarda o access token em memória (nunca `localStorage`), expõe `buyer`, `isAuthenticated`, `login`, `logout` |
| 6 | queries | `src/features/account/hooks/queries/query-options.ts` + `useAccountQueries.ts` | `me` query (`enabled` quando há token) |
| 7 | mutations | `src/features/account/hooks/mutations/useAuthMutations.ts` | `register`, `login`, `logout`, `changePassword`, `forgotPassword`, `resetPassword` — cada uma com toast; sucesso de login/register popula o `AuthContext` |
| 8 | forms | `src/features/account/hooks/forms/{useRegisterForm,useLoginForm,useResetPasswordForm}.ts` | resolver Zod = schema de request; máscara de CPF só na exibição |
| 9 | components | `src/features/account/components/{RegisterForm,LoginForm,ForgotPasswordForm,ResetPasswordForm,RequireAuth}.tsx` | `RequireAuth` protege rotas autenticadas; redireciona a `/login` guardando o destino |
| 10 | components/ui | `src/features/account/components/ui/AuthCard.tsx` | layout puro de formulário de auth |
| 11 | rotas | `src/app/` (App Router: uma `page.tsx` por rota) | `/login`, `/registro`, `/recuperar-senha`, `/redefinir-senha` |
| 12 | barrels | `index.ts` em toda subpasta + na raiz da feature | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-account-auth
commit 1  feat(account): endpoints, schemas e services de auth + fetcher com refresh
commit 2  feat(account): AuthContext, queries e mutations de auth
commit 3  feat(account): formulários e telas de login/registro/recuperação + RequireAuth
commit 4  chore(account): barrels index.ts da feature
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

### Casos de teste de domínio (pytest)

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_registra_buyer_com_cpf_valido` | CPF válido → `Buyer` criado, evento `BuyerRegistered`, `role=BUYER` | RF09 |
| `test_cpf_invalido_recusa` | CPF com DV errado → `DomainError` | RF09 |
| `test_troca_senha_rotaciona_security_stamp` | `change_password` → `security_stamp` muda, evento emitido | RNF01 |
| `test_reset_token_uso_unico` | `consume()` duas vezes → segunda levanta `DomainError` | RF09 |
| `test_reset_token_expirado` | `expires_at` no passado → `consume()` recusa | RF09 |
| `test_refresh_token_reuso_detectado` | usar token já `rotated` → `DomainError` (dispara rotação de stamp no usecase) | RNF01 |

### Roteiro manual

Cadastro → login → fechar e reabrir a aba (sessão mantida via refresh) → trocar
senha (todos os dispositivos deslogam) → "esqueci a senha" com e-mail
inexistente (mesma mensagem) → e-mail existente → link → nova senha → login.
Verificar que nenhuma resposta traz CPF/e-mail/hash de terceiro e que nenhum log
tem senha/CPF.

### Passo a passo TBD (QA)

```text
git checkout master && git pull && git checkout -b test/<NN>-identity-auth
git commit -m "test(identity): cobrir registro, security_stamp e tokens de auth"
```

---

## D. DevOps — Gabriel

- `.env.example` já tem `JWT_SECRET_KEY` (SECRET, `openssl rand -hex 32`) — gerar
  e configurar no ambiente e nos secrets do CI.

## E. Ordem entre as fatias

Backend + QA (casos de domínio) começam juntos. Frontend começa contra o
contrato-alvo; o `fetcher` com refresh e o `AuthContext` não dependem do backend
pronto para serem escritos. Integração real após o merge do backend.

## F. Bloqueios em aberto

Nenhum.
