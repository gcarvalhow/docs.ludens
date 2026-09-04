---
status: alvo
spec: identity-auth
updated_at: 2026-09-04
responsavel: Igor (Backend)
---

# Integration Contract — Cadastro e autenticação do comprador

**Status:** alvo (contrato para o frontend construir; vira `canônico` quando o
backend implementar). **Módulo backend:** `identity`.

## Rotas

| Método | Caminho | Auth | Sucesso | Descrição |
| --- | --- | --- | --- | --- |
| POST | `/auth/login` | pública | 200 | Autentica |
| POST | `/auth/refresh` | cookie de refresh | 200 | Novo par de tokens (rotaciona refresh) |
| POST | `/auth/logout` | Bearer | 204 | Regenera `security_stamp` |
| POST | `/auth/password/change` | Bearer | 204 | Troca de senha (senha atual + nova) |
| POST | `/auth/password/forgot` | pública | 202 | Sempre 202, resposta neutra |
| POST | `/auth/password/reset` | pública (token no corpo) | 204 | Define nova senha via token |
| POST | `/users` | pública | 201 | Cria conta e já autentica |
| GET | `/users` | Bearer, `ADMIN` | 200 | Lista paginada de usuários |
| GET | `/users/{id}` | Bearer | 200 | Dados do usuário `id` — `BUYER` só o próprio, `ADMIN` qualquer um |

`/auth/...` cobre sessão (`AuthUseCase`); `/users/...` cobre cadastro e leitura
de usuário (`UserUseCase`) — `register` e a leitura por `id`/listagem não são
concern de auth.

> Prefixo/base path e versionamento: **sem prefixo e sem versionamento no N1** —
> as rotas são montadas em `/auth/...` e `/users/...` na raiz; base =
> `NEXT_PUBLIC_API_URL` no frontend. Decisão registrada ao gerar
> `backend.md`/`frontend.md`; revisitar se uma convenção global for adotada
> (`docs.ludens/backend/integration/_template.md`).

## Request / Response (shapes principais)

Todo corpo trafega em **camelCase** (request e response). `expiresIn` é em
**segundos**. `role` é `"BUYER"` | `"ADMIN"` (maiúsculas).

- `POST /users` → body `{ name, cpf, email, password }` (CPF sem máscara,
  só dígitos; `password` ≥ 8). 201 → `{ accessToken, expiresIn }` +
  `Set-Cookie: refresh_token=...` (`HttpOnly; Secure; SameSite=Strict; Path=/auth`;
  `Secure` desligado quando `ENVIRONMENT=development`).
- `POST /auth/login` → `{ email, password }` → 200 mesmo shape + `Set-Cookie`.
- `POST /auth/refresh` → sem body; lê o cookie → 200 `{ accessToken, expiresIn }`,
  mais um novo `Set-Cookie` de refresh (rotação).
- `POST /auth/logout` → Bearer, sem body → 204 + `Set-Cookie` **apagando**
  `refresh_token` (`Path=/auth`).
- `POST /auth/password/change` → Bearer → `{ currentPassword, newPassword }`
  (`newPassword` ≥ 8) → 204.
- `GET /users/{id}` → 200 `{ id, name, email, cpf, role }`. `BUYER` só pode pedir
  o próprio `id` (403 em qualquer outro); `ADMIN` pode pedir qualquer `id`,
  inclusive o próprio — sem `/me`, o frontend descobre o próprio `id`
  decodificando o claim `sub` do access token (payload do JWT; não precisa
  verificar assinatura no cliente, isso é sempre feito pelo backend).
- `GET /users` → Bearer, só `ADMIN` (403 pra `BUYER`) → query `?page=1&size=20`
  (`page` ≥ 1, `size` 1–100) → 200
  `{ items: [{ id, name, email, role }], page, size, total }` — `items` não
  traz `cpf` (visão de listagem, não de detalhe).
- `POST /auth/password/forgot` → `{ email }` → **202 sempre**, body
  `{ message: "Se houver uma conta com esse e-mail, enviamos um link." }`.
- `POST /auth/password/reset` → `{ token, password }` (`password` ≥ 8) → 204.

## Erros esperados (linguagem de negócio)

| Rota | Status | Quando | Mensagem |
| --- | --- | --- | --- |
| register | 409 | CPF já cadastrado | "Este CPF já possui cadastro." |
| register | 409 | e-mail já cadastrado | "Este e-mail já está em uso." |
| register | 422 | CPF inválido | "CPF inválido." |
| login | 401 | e-mail ou senha errados | "E-mail ou senha inválidos." |
| refresh | 401 | cookie ausente/expirado/reusado | (frontend trata como anônimo) |
| password/change | 422 | senha atual errada | "A senha atual não confere." (`field: "currentPassword"`) |
| password/reset | 410 | token expirado ou já usado | "Este link não é mais válido, solicite um novo." |
| `/users/{id}` | 403 | `BUYER` pedindo `id` de outra pessoa | "Você só pode ver os próprios dados." |
| `/users` (listagem) | 403 | `BUYER` chamando (só `ADMIN`) | "Acesso restrito a administradores." |

**Envelope de erro (4xx):** `{ "detail": [ { "field": string, "message": string } ] }`
— vale para a validação de forma (422, handler de `RequestValidationError`) e
para os erros de domínio (`DomainError` e subclasses, novo handler em `main.py`).
`field` é o campo de formulário associado ou `"body"`/`"authorization"`. O
frontend usa `detail[0].message` no toast.

## Semântica de auth

- Access token: JWT HS256, `Authorization: Bearer`, expira em 30 min, claims
  `sub`, `role`, `security_stamp`, `type=access`, `exp`, `iat`.
- Refresh token: opaco, cookie `HttpOnly; Secure; SameSite=Strict; Path=/auth`,
  7 dias, rotacionado a cada `refresh`. Reuso detectado → `security_stamp`
  regenerado (derruba tudo).
- Qualquer token cujo `security_stamp` não bate o do comprador → 401. Logout,
  troca e redefinição de senha regeneram o `security_stamp`.

## Impacto de UX

401 numa chamada autenticada → o frontend tenta `refresh` uma vez; se falhar,
limpa o estado e manda para `/login`. Mensagens conforme a tabela de erros.

## Lacunas / decisões em aberto

- Nenhuma bloqueante. As convenções que estavam `<a definir globalmente>` foram
  fixadas para esta feature ao gerar `backend.md`/`frontend.md`/`quality.md`
  (mantido `status: alvo` até o backend implementar):
  - **Envelope de erro 4xx:** `{ "detail": [ { "field", "message" } ] }`.
  - **Prefixo / versionamento:** nenhum no N1; rotas em `/auth/...` e `/users/...`.
  - **`expiresIn`:** segundos. **`role`:** `"BUYER"` | `"ADMIN"`.
  - **`/auth/password/change`:** corpo `{ currentPassword, newPassword }`.
  - **`/auth/logout`:** 204 + `Set-Cookie` apagando `refresh_token`.
  - **`register` e leitura de usuário não são rota de `/auth`:** `POST /users`
    (cadastro), `GET /users/{id}` (substitui `/auth/me`, restrito por papel),
    `GET /users` (listagem paginada, só `ADMIN`).
- Se uma convenção global de prefixo/versionamento/erro for adotada depois, ela
  substitui o que está aqui e o contrato vira `canônico`.
