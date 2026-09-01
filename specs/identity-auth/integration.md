---
status: alvo
spec: identity-auth
updated_at: 2026-09-01
responsavel: Igor (Backend)
---

# Integration Contract — Cadastro e autenticação do comprador

**Status:** alvo (contrato para o frontend construir; vira `canônico` quando o
backend implementar). **Módulo backend:** `identity`.

## Rotas

| Método | Caminho | Auth | Sucesso | Descrição |
|---|---|---|---|---|
| POST | `/auth/register` | pública | 201 | Cria conta e já autentica |
| POST | `/auth/login` | pública | 200 | Autentica |
| POST | `/auth/refresh` | cookie de refresh | 200 | Novo par de tokens (rotaciona refresh) |
| POST | `/auth/logout` | Bearer | 204 | Regenera `security_stamp` |
| POST | `/auth/password/change` | Bearer | 204 | Troca de senha (senha atual + nova) |
| POST | `/auth/password/forgot` | pública | 202 | Sempre 202, resposta neutra |
| POST | `/auth/password/reset` | pública (token no corpo) | 204 | Define nova senha via token |
| GET | `/auth/me` | Bearer | 200 | Dados do comprador autenticado |

> Prefixo/base path e versionamento: `<a definir globalmente>` — ver
> `docs.ludens/backend/integration/_template.md`.

## Request / Response (shapes principais)

- `POST /auth/register` → body `{ name, cpf, email, password }` (CPF sem máscara,
  só dígitos). 201 → `{ accessToken, expiresIn }` + `Set-Cookie: refresh_token=...`.
- `POST /auth/login` → `{ email, password }` → 200 mesmo shape.
- `POST /auth/refresh` → sem body; lê o cookie → 200 `{ accessToken, expiresIn }`
  + novo `Set-Cookie`.
- `GET /auth/me` → 200 `{ id, name, email, cpf, role }` (só do próprio usuário).
- `POST /auth/password/forgot` → `{ email }` → **202 sempre**, body
  `{ message: "Se houver uma conta com esse e-mail, enviamos um link." }`.
- `POST /auth/password/reset` → `{ token, password }` → 204.

## Erros esperados (linguagem de negócio)

| Rota | Status | Quando | Mensagem |
|---|---|---|---|
| register | 409 | CPF já cadastrado | "Este CPF já possui cadastro." |
| register | 409 | e-mail já cadastrado | "Este e-mail já está em uso." |
| register | 422 | CPF inválido | "CPF inválido." |
| login | 401 | e-mail ou senha errados | "E-mail ou senha inválidos." |
| refresh | 401 | cookie ausente/expirado/reusado | (frontend trata como anônimo) |
| password/change | 422 | senha atual errada | "A senha atual não confere." |
| password/reset | 410 | token expirado ou já usado | "Este link não é mais válido, solicite um novo." |

Envelope de erro: `<forma exata a definir globalmente>`.

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

- Prefixo de rota / base path, versionamento de API, forma do corpo de erro —
  `<a definir globalmente>`.
