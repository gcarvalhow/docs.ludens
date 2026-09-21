---
status: canônico
spec: identity-auth
updated_at: 2026-09-21
responsavel: Igor (Backend)
---

# Integration Contract — Cadastro e autenticação do comprador

> **Nota de reescopo (2026-09-11):** rotas de cadastro/leitura de usuário saem
> pra `identity-user-management/integration.md`; falta adicionar aqui a rota
> de alteração de e-mail (nova). Conteúdo abaixo ainda reflete o escopo
> antigo (ver também a revisão de 2026-09-17 logo abaixo, sobre o prefixo de
> rota e o casing do contrato).
>
> **Revisão de 2026-09-21:** o prefixo mudou de novo desde 2026-09-17. O
> router de autenticação foi movido para `/identity/authentication` (não
> `/identity` puro) pra não colidir com o prefixo de `user_router.py`
> (`/identity/users`). Todas as rotas abaixo já foram corrigidas pra
> `/identity/authentication/...`.

**Status:** canônico (reflete o código real já mergeado — ver `backend.md`).
**Módulo backend:** `identity`.

## Rotas

| Método | Caminho | Auth | Sucesso | Descrição |
| --- | --- | --- | --- | --- |
| POST | `/identity/authentication/login` | pública | 200 | Autentica |
| POST | `/identity/authentication/refresh` | cookie de refresh | 200 | Novo par de tokens (rotaciona refresh) |
| POST | `/identity/authentication/logout` | Bearer | 204 | Regenera `security_stamp` |
| POST | `/identity/authentication/password/change` | Bearer | 204 | Troca de senha (senha atual + nova) |
| POST | `/identity/authentication/password/forgot` | pública | 202 | Sempre 202, resposta neutra |
| POST | `/identity/authentication/password/reset` | pública (token no corpo) | 204 | Define nova senha via token |
| POST | `/identity/users` | pública | 201 | Cria conta e já autentica |
| GET | `/identity/users/{id}` | Bearer | 200 | Dados do usuário `id` — usuário comum só o próprio, admin (`is_admin`) qualquer um |

`/identity/...` (tag `01.Identity - Auth`) cobre sessão (`AuthUseCase`);
`/identity/users/...` (tag `02.Identity - User`) cobre cadastro e leitura de
usuário (`UserUseCase`) — cadastro e a leitura por `id` não são concern de
auth. Cadastro é `POST /identity/users` puro, sem sufixo `/register` — a
razão original para reservar esse path (2026-09-11: "libera para uma
eventual listagem futura") caiu quando a listagem paginada de usuários foi
implementada como `GET /identity/users` (feature `identity-user-management`,
2026-09-17); manter `/register` como sufixo de cadastro deixaria de fazer
sentido. Ver `specs/identity-user-management/` para o restante do contrato de
usuário (perfil, troca de e-mail, exclusão de conta, listagem admin).

> Prefixo/base path e versionamento: **sem versionamento no N1** — as rotas do
> módulo `identity` são montadas sob `/identity/...` na raiz; base =
> `NEXT_PUBLIC_API_URL` no frontend. Decisão registrada ao gerar
> `backend.md`/`frontend.md`; revisitar se uma convenção global for adotada
> (`docs.ludens/backend/integration/_template.md`).

## Request / Response (shapes principais)

Todo corpo trafega em **snake_case** (request e response) — sem exceção, não há
camada de conversão para camelCase. `expires_in` é em **segundos**. Não existe
campo `role`: permissão é o booleano `is_admin` (`true`/`false`), tanto no
`UserResponse` quanto no claim do JWT.

- `POST /identity/users` → body `{ name, cpf, email, password }` (CPF
  sem máscara, só dígitos; `password` ≥ 8). 201 → `{ access_token, expires_in }`
  - `Set-Cookie: refresh_token=...` (`HttpOnly; Secure; SameSite=Strict; Path=/identity`;
  `Secure` desligado quando `ENVIRONMENT=development`).
- `POST /identity/authentication/login` → `{ email, password }` → 200 mesmo shape + `Set-Cookie`.
- `POST /identity/authentication/refresh` → sem body; lê o cookie → 200 `{ access_token, expires_in }`,
  mais um novo `Set-Cookie` de refresh (rotação).
- `POST /identity/authentication/logout` → Bearer, sem body → 204 + `Set-Cookie` **apagando**
  `refresh_token` (`Path=/identity`).
- `POST /identity/authentication/password/change` → Bearer → `{ current_password, new_password }`
  (`new_password` ≥ 8) → 204.
- `GET /identity/users/{id}` → 200 `{ id, name, email, cpf, is_admin }`.
  Usuário comum só pode pedir o próprio `id` (403 em qualquer outro); admin
  pode pedir qualquer `id`, inclusive o próprio — sem `/me`, o frontend
  descobre o próprio `id` decodificando o claim `sub` do access token (payload
  do JWT; não precisa verificar assinatura no cliente, isso é sempre feito
  pelo backend).
- `POST /identity/authentication/password/forgot` → `{ email }` → **202 sempre**, body
  `{ message: "Se houver uma conta com esse e-mail, enviamos um link." }`.
- `POST /identity/authentication/password/reset` → `{ token, password }` (`password` ≥ 8) → 204.

## Erros esperados (linguagem de negócio)

| Rota | Status | Quando | Mensagem |
| --- | --- | --- | --- |
| register | 409 | CPF já cadastrado | "Este CPF já possui cadastro." |
| register | 409 | e-mail já cadastrado | "Este e-mail já está em uso." |
| register | 422 | CPF inválido | "CPF inválido." |
| login | 401 | e-mail ou senha errados | "E-mail ou senha inválidos." |
| refresh | 401 | cookie ausente/expirado/reusado | (frontend trata como anônimo) |
| password/change | 422 | senha atual errada | "A senha atual não confere." (`field: "current_password"`) |
| password/reset | 410 | token expirado ou já usado | "Este link não é mais válido, solicite um novo." |
| `/identity/users/{id}` | 403 | usuário comum pedindo `id` de outra pessoa | "Acesso restrito ao próprio usuário." |

**Envelope de erro (4xx):** são **dois formatos diferentes**, não um só:
- Erro de validação de forma (422, handler de `RequestValidationError`):
  `{ "detail": [ { "field": string, "message": string } ] }` — `field` é o
  campo de formulário associado ou `"body"`. O frontend usa
  `detail[0].message` no toast.
- Erro de domínio (409/401/403/404/410, `DomainError` e subclasses, handler em
  `main.py`): `{ "detail": string }` — mensagem única, sem lista, sem `field`.
  O frontend usa `detail` diretamente no toast.

## Semântica de auth

- Access token: JWT HS256, `Authorization: Bearer`, expira em 30 min, claims
  `sub`, `is_admin`, `security_stamp`, `type=access`, `exp`, `iat`.
- Refresh token: opaco, cookie `HttpOnly; Secure; SameSite=Strict; Path=/identity`,
  7 dias, rotacionado a cada `refresh`. Reuso detectado → `security_stamp`
  regenerado (derruba tudo).
- Qualquer token cujo `security_stamp` não bate o do comprador → 401. Logout,
  troca e redefinição de senha regeneram o `security_stamp`.

## Impacto de UX

401 numa chamada autenticada → o frontend tenta `refresh` uma vez; se falhar,
limpa o estado e manda para `/login`. Mensagens conforme a tabela de erros.

## Lacunas / decisões em aberto

- Nenhuma bloqueante. As convenções que estavam `<a definir globalmente>` foram
  fixadas para esta feature ao gerar `backend.md`/`frontend.md`:
  - **Envelope de erro 4xx:** dois formatos — lista `{ field, message }` para
    422 de validação, mensagem única `{ detail: string }` para erro de
    domínio.
  - **Prefixo / versionamento:** nenhum versionamento no N1; rotas do módulo
    `identity` montadas sob `/identity/...`.
  - **Casing:** contrato inteiro em snake_case, sem exceção. **`expires_in`:**
    segundos. Sem campo `role`: permissão é o booleano `is_admin`.
  - **`/identity/authentication/password/change`:** corpo `{ current_password, new_password }`.
  - **`/identity/authentication/logout`:** 204 + `Set-Cookie` apagando `refresh_token`.
  - **`register` e leitura de usuário não são rota de sessão:** `POST
    /identity/users` (cadastro), `GET /identity/users/{id}`
    (substitui `/auth/me`, restrito por papel). Listagem paginada existe em
    `GET /identity/users` (admin) — ver `identity-user-management`.
- **Frontend ainda não atualizado:** `web.ludens` foi construído contra o
  contrato antigo (`/auth/...`, `/users/...`). Ver
  [`team/overview.md`](../../team/overview.md#débitos-conhecidos-hoje) para o
  débito técnico registrado.
- Se uma convenção global de prefixo/versionamento/erro for adotada depois, ela
  substitui o que está aqui.
