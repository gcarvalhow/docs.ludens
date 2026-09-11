---
status: done
spec: identity-auth
surface: quality
created_at: 2026-09-03
updated_at: 2026-09-04
---

# Cadastro e autenticação do comprador — Quality

> **Nota de reescopo (2026-09-11):** ver mesma nota em `backend.md`. Casos de
> cadastro saem pra `identity-user-management`; falta cobertura de alteração
> de e-mail (nova). Conteúdo abaixo ainda reflete o escopo antigo.

**Resumo:** cobertura de domínio (pytest, sem DB/HTTP) dos invariantes do módulo
`identity` — validação de CPF, evento e papel no registro, rotação de
`security_stamp` na troca/redefinição de senha, uso único e expiração do token de
redefinição, detecção de reuso de refresh token — mais os testes de integração
cross-surface do fluxo autenticado e o roteiro manual pré-entrega.
**RF:** RF09 · **RN:** — (reforça RNF01) · **Módulo backend:** `identity`
**Contrato:** `docs.ludens/specs/identity-auth/integration.md`

---

## 1. Definition of Ready — checagem

Contra `docs.ludens/team/quality.md`.

| Item DoR | Situação |
| --- | --- |
| História no formato "Como [papel], eu quero [funcionalidade] para que [benefício]" | ✅ `spec.md` §1 e RF09 (`requirements/functional.md`). |
| Critérios de aceite objetivos e verificáveis | ✅ `logic.md` §§1–3 e a tabela de erros do `integration.md` dão cada caso com entrada/saída. |
| Regras de negócio essenciais e exceções especificadas | ✅ `logic.md` §3 (hash bcrypt, login genérico, resposta neutra, `security_stamp`, token 1 h/uso único) e §5 (casos de borda: reuso de refresh, múltiplos links, e-mail fora do ar). Não há RN numérica; reforça RNF01. |
| Dependências técnicas mapeadas | ✅ Nenhuma feature dependente (é a base). Depende de: tabela `events` do outbox (primeira migration entra com esta feature), serviço de e-mail transacional (mínimo entregue aqui; `notification-transactional-email` amplia), `JWT_SECRET_KEY` no ambiente/CI. |
| Layout/protótipo da interface aprovado | ⚠️ Não há protótipo formal. Telas são padrão (form de e-mail/senha) e o `frontend.md` fixa cada campo, estado e mensagem — **não bloqueante** para N1; registrar screenshot no PR do frontend. |

**Veredito:** pronto para desenvolver. O único item aberto (protótipo) é
convenção de UI padrão, coberta pela especificação de tela no `frontend.md`.

---

## 2. Casos de teste de domínio (pytest)

Testes puros: instanciam a classe (`User`, aggregate root, ou `RefreshToken`/
`PasswordResetToken`, entidades filhas), chamam o método, verificam estado —
`dequeue_events()` só se aplica ao `User`, único que publica evento de domínio.
Sem DB, sem HTTP (`docs.ludens/backend/testing.md`).

| Caso | Cenário (estado → ação → asserção) | RN/RF |
| --- | --- | --- |
| `test_registra_user_com_cpf_valido_emite_evento_e_papel_buyer` | sem conta → `User.register(CPF válido)` → evento `UserRegistered`, `role=BUYER`, `id`/`security_stamp` preenchidos | RF09 |
| `test_change_password_rotaciona_security_stamp` | user registrado → `change_password` → eventos `UserPasswordChanged` + `UserSecurityStampRotated`, stamp muda | RF09 · RNF01 |
| `test_reset_password_tambem_rotaciona_security_stamp` | user registrado → `reset_password` → mesma dupla de eventos, stamp muda | RF09 · RNF01 |
| `test_request_password_reset_emite_evento_com_token_em_claro` | user registrado → `request_password_reset(...)` → evento `PasswordResetRequested` com `reset_token` em claro | RF09 |
| `test_rejeita_digito_verificador_invalido` | string com DV errado → `CPF(...)` → `DomainError` (`field="cpf"`, `422`) | RF09 |
| `test_rejeita_sequencia_repetida` / `test_rejeita_quantidade_de_digitos_errada` | `"11111111111"` / `"529982247"` → `DomainError` | RF09 |
| `test_consume_valido_marca_used_at_uma_vez` | token válido (`PasswordResetToken`, entidade filha) → `consume(now)` → `used_at` setado, sem evento próprio | RF09 |
| `test_consume_segunda_vez_falha` | token já consumido → `consume` de novo → `GoneError` (410) | RF09 |
| `test_consume_token_expirado_falha` | `expires_at` no passado → `consume` → `GoneError` | RF09 |
| `test_invalidate_impede_consumo_posterior` | `invalidate(now)` (nova solicitação) → `consume` → `GoneError` | RF09 |
| `test_rotate_marca_used` | refresh token novo (`RefreshToken`, entidade filha) → `rotate(now)` → `used=True`, sem evento próprio | RNF01 |
| `test_rotate_de_token_ja_rotacionado_e_reuso_detectado` | token já rotacionado → `rotate` de novo → `UnauthorizedError` (o usecase reage rotacionando o `security_stamp`) | RNF01 |

### `tests/modules/__init__.py` e `tests/modules/identity/__init__.py` — novos

```text
# ambos vazios
tests/modules/__init__.py
tests/modules/identity/__init__.py
```

### `tests/modules/identity/test_cpf.py` — novo

```python
# tests/modules/identity/test_cpf.py — novo
import pytest

from app.core.domain.errors import DomainError
from app.modules.identity.domain.value_objects.cpf import CPF


def test_aceita_cpf_valido_e_guarda_so_digitos():
    assert CPF("529.982.247-25").value == "52998224725"
    assert CPF("52998224725").value == "52998224725"


def test_rejeita_digito_verificador_invalido():
    with pytest.raises(DomainError) as exc_info:
        CPF("52998224724")
    assert exc_info.value.field == "cpf"
    assert exc_info.value.status_code == 422


def test_rejeita_sequencia_repetida():
    with pytest.raises(DomainError):
        CPF("11111111111")


def test_rejeita_quantidade_de_digitos_errada():
    with pytest.raises(DomainError):
        CPF("529982247")
```

### `tests/modules/identity/test_email.py` — novo

```python
# tests/modules/identity/test_email.py — novo
import pytest

from app.core.domain.errors import DomainError
from app.modules.identity.domain.value_objects.email import Email


def test_normaliza_para_minusculas_e_apara_espacos():
    assert Email("  Ana.Souza@Example.COM ").value == "ana.souza@example.com"


def test_rejeita_formato_invalido():
    with pytest.raises(DomainError) as exc_info:
        Email("ana(at)example.com")
    assert exc_info.value.field == "email"
```

### `tests/modules/identity/test_user.py` — novo

```python
# tests/modules/identity/test_user.py — novo
from datetime import datetime, timedelta, timezone

from app.modules.identity.domain.aggregates.user import User
from app.modules.identity.domain.enumerations.role import Role
from app.modules.identity.domain.events.identity_events import (
    PasswordResetRequested,
    UserRegistered,
)
from app.modules.identity.domain.value_objects.cpf import CPF
from app.modules.identity.domain.value_objects.email import Email

VALID_CPF = "52998224725"


def _register() -> User:
    return User.register(
        "Ana Souza", CPF(VALID_CPF), Email("ana@example.com"), "hash-1"
    )


def test_registra_user_com_cpf_valido_emite_evento_e_papel_buyer():
    user = _register()

    events = user.dequeue_events()
    assert [type(e).__name__ for e in events] == ["UserRegistered"]

    registered = events[0]
    assert isinstance(registered, UserRegistered)
    assert registered.cpf == VALID_CPF
    assert registered.role == Role.BUYER.value

    assert user.role is Role.BUYER
    assert user.cpf == VALID_CPF
    assert user.email == "ana@example.com"
    assert user.password_hash == "hash-1"
    assert user.id is not None
    assert user.security_stamp is not None


def test_change_password_rotaciona_security_stamp():
    user = _register()
    original_stamp = user.security_stamp
    user.dequeue_events()

    user.change_password("hash-2")

    assert [type(e).__name__ for e in user.dequeue_events()] == [
        "UserPasswordChanged",
        "UserSecurityStampRotated",
    ]
    assert user.password_hash == "hash-2"
    assert user.security_stamp != original_stamp


def test_reset_password_tambem_rotaciona_security_stamp():
    user = _register()
    original_stamp = user.security_stamp
    user.dequeue_events()

    user.reset_password("hash-3")

    assert [type(e).__name__ for e in user.dequeue_events()] == [
        "UserPasswordChanged",
        "UserSecurityStampRotated",
    ]
    assert user.password_hash == "hash-3"
    assert user.security_stamp != original_stamp


def test_request_password_reset_emite_evento_com_token_em_claro():
    # `PasswordResetToken` (entidade filha) não emite evento — quem publica
    # `PasswordResetRequested` é o `User`, o aggregate root.
    user = _register()
    user.dequeue_events()
    expires_at = datetime.now(timezone.utc) + timedelta(hours=1)

    user.request_password_reset(
        token_hash="hash", raw_token="raw-token", expires_at=expires_at
    )

    events = user.dequeue_events()
    assert [type(e).__name__ for e in events] == ["PasswordResetRequested"]
    assert isinstance(events[0], PasswordResetRequested)
    assert events[0].reset_token == "raw-token"
    assert events[0].email == user.email
    assert events[0].id == user.id
```

### `tests/modules/identity/test_password_reset_token.py` — novo

`PasswordResetToken` é entidade filha do `User` (não aggregate root) — não emite
evento próprio; os testes verificam só o estado da linha. O evento
`PasswordResetRequested` é coberto em `test_user.py`
(`test_request_password_reset_emite_evento_com_token_em_claro`).

```python
# tests/modules/identity/test_password_reset_token.py — novo
from datetime import datetime, timedelta, timezone
from uuid import uuid4

import pytest

from app.core.domain.errors import GoneError
from app.modules.identity.domain.entities.password_reset_token import (
    PasswordResetToken,
)


def _issue(ttl_minutes: int = 60) -> PasswordResetToken:
    now = datetime.now(timezone.utc)
    return PasswordResetToken.issue(
        user_id=uuid4(), token_hash="hash", expires_at=now + timedelta(minutes=ttl_minutes)
    )


def test_issue_cria_token_nao_consumido():
    token = _issue()

    assert token.used_at is None
    assert token.id is not None


def test_consume_valido_marca_used_at_uma_vez():
    token = _issue()
    now = datetime.now(timezone.utc)

    token.consume(now)

    assert token.used_at == now


def test_consume_segunda_vez_falha():
    token = _issue()
    now = datetime.now(timezone.utc)
    token.consume(now)

    with pytest.raises(GoneError):
        token.consume(now + timedelta(seconds=1))


def test_consume_token_expirado_falha():
    token = _issue(ttl_minutes=-1)  # emitido já expirado

    with pytest.raises(GoneError):
        token.consume(datetime.now(timezone.utc))


def test_invalidate_impede_consumo_posterior():
    token = _issue()
    now = datetime.now(timezone.utc)

    token.invalidate(now)

    with pytest.raises(GoneError):
        token.consume(now + timedelta(seconds=1))
```

### `tests/modules/identity/test_refresh_token.py` — novo

`RefreshToken` é entidade filha do `User` (não aggregate root) — emitir/rotacionar
um token é bookkeeping interno de sessão, sem consumidor no outbox; os testes
verificam só o estado da linha.

```python
# tests/modules/identity/test_refresh_token.py — novo
from datetime import datetime, timedelta, timezone
from uuid import uuid4

import pytest

from app.core.domain.errors import DomainError, UnauthorizedError
from app.modules.identity.domain.entities.refresh_token import RefreshToken


def _issue() -> RefreshToken:
    now = datetime.now(timezone.utc)
    return RefreshToken.issue(uuid4(), "hash", now + timedelta(days=7))


def test_issue_cria_token_nao_usado():
    token = _issue()

    assert token.used is False
    assert token.id is not None


def test_rotate_marca_used():
    token = _issue()
    now = datetime.now(timezone.utc)

    token.rotate(now)

    assert token.used is True
    assert token.rotated_at == now


def test_rotate_de_token_ja_rotacionado_e_reuso_detectado():
    token = _issue()
    now = datetime.now(timezone.utc)
    token.rotate(now)

    with pytest.raises(UnauthorizedError):
        token.rotate(now + timedelta(seconds=1))
    assert issubclass(UnauthorizedError, DomainError)
```

---

## 3. Testes de integração cross-surface

Fluxo que atravessa backend (`api.ludens`) + frontend (`web.ludens`) após o merge
das duas fatias. Executáveis como teste de API (httpx contra o container) e/ou
Playwright.

| Fluxo | Verifica |
| --- | --- |
| cadastro → `201 { accessToken, expiresIn }` + `Set-Cookie: refresh_token` | header `Authorization` aceito na sequência; cookie `HttpOnly; SameSite=Strict; Path=/auth`; senha nunca no corpo |
| `GET /users/{id}` com o access token do cadastro → `200 { id, name, email, cpf, role: "BUYER" }` | `security_stamp` do token bate com o do `User`; retorna só os dados do próprio usuário |
| access token expira → `GET /users/{id}` responde `401` → frontend chama `POST /auth/refresh` (cookie) → novo par → repete a chamada com sucesso | o `fetcher` faz o refresh **uma vez** e repete; o refresh anterior fica `used=True` |
| reusar o refresh token anterior (já rotacionado) em `POST /auth/refresh` | `401`; `security_stamp` do `User` é regenerado → o access token vigente também para de valer (derruba tudo) |
| `POST /auth/logout` (Bearer) → `204` + `Set-Cookie` apagando `refresh_token` → nova chamada autenticada e novo `refresh` → ambos `401` | logout invalida **todos** os dispositivos (regeneração de `security_stamp`); frontend limpa estado e vai para `/login` |
| `POST /auth/password/change` `{ currentPassword, newPassword }` com senha atual errada | `422` `{ detail: [{ field: "currentPassword", message: "A senha atual não confere." }] }`; nenhuma sessão derrubada |
| `POST /auth/password/change` com senha atual correta → `204` | `security_stamp` regenerado; o dispositivo que trocou precisa logar de novo (frontend redireciona para `/login`) |
| `POST /auth/password/forgot` com e-mail cadastrado e com e-mail inexistente | **os dois** → `202` com a mesma mensagem; só o caso cadastrado grava `PasswordResetToken` e enfileira `PasswordResetRequested` |
| `PasswordResetRequested` no outbox → relay (~2 s) → e-mail com link `WEB_APP_URL/redefinir-senha?token=...` | handler idempotente; derrubar o SMTP não invalida nada — o relay retenta |
| `POST /auth/password/reset` `{ token, password }` com token válido → `204`; reusar o mesmo token → `410` "Este link não é mais válido, solicite um novo." | uso único; `security_stamp` regenerado; frontend leva ao `/login` |
| duas solicitações de recuperação seguidas → só o link mais recente funciona | a segunda `invalidate` a primeira (`used_at` setado) |
| CORS: request do `web.ludens` (origin em `ALLOWED_ORIGINS`) com `credentials: 'include'` | `Access-Control-Allow-Credentials: true`; cookie de refresh trafega |
| `RequireAuth` numa rota protegida sem sessão | redireciona para `/login?next=<rota>`; após login volta para `<rota>` |

---

## 4. Roteiro de teste manual pré-entrega

Ambiente: `docker compose ... up -d` (Postgres) · `alembic upgrade head` ·
container SMTP de dev (MailHog/Mailpit) · `api.ludens` no ar · `web.ludens`
(`npm run dev`) apontando `NEXT_PUBLIC_API_URL` para a API.

1. **Cadastro.** Abrir `/registro`, preencher nome, CPF **válido** com máscara,
   e-mail e senha (≥ 8). Enviar. → Redireciona para `/` autenticado; toast "Conta
   criada". DevTools → Application → Cookies: existe `refresh_token`
   `HttpOnly`, `SameSite=Strict`, `Path=/auth`. Nenhuma resposta no Network traz
   `password`/hash.
2. **CPF inválido.** Repetir o cadastro com CPF de DV errado (ex.: `52998224724`).
   → Erro abaixo do campo CPF: "CPF inválido."; conta não criada.
3. **CPF/e-mail duplicado.** Cadastrar de novo com o mesmo CPF → "Este CPF já
   possui cadastro."; com o mesmo e-mail e CPF diferente → "Este e-mail já está em
   uso."
4. **Sessão que se mantém.** Fechar a aba e reabrir `/` (ou `/minhas-compras`
   quando existir). → Continua autenticado sem digitar senha (o `AuthProvider`
   chamou `POST /auth/refresh` no load). No Network: uma chamada a `/auth/refresh`
   → `200`.
5. **Login inválido.** `/login`, e-mail certo + senha errada → "E-mail ou senha
   inválidos."; e-mail inexistente + qualquer senha → **a mesma** mensagem.
6. **Login válido.** Entrar. → Redireciona para `/` (ou para `?next=` se veio de
   rota protegida); toast "Login efetuado".
7. **Troca de senha (logado).** Acionar a troca com senha atual **errada** →
   "A senha atual não confere.". Com a senha atual correta → sucesso; a aba é
   deslogada e vai para `/login`. Tentar usar a aba antiga (se houver outra
   aberta) → primeira chamada autenticada cai em `401` → vai para `/login`.
8. **Esqueci a senha — e-mail inexistente.** `/recuperar-senha`, informar um
   e-mail que não existe. → Mensagem "Se houver uma conta com esse e-mail,
   enviamos um link...". Caixa do MailHog: **nenhum** e-mail.
9. **Esqueci a senha — e-mail existente.** Informar o e-mail cadastrado. →
   Mesma mensagem. Em ~2 s aparece um e-mail no MailHog com o link
   `/redefinir-senha?token=...`.
10. **Vários links.** Repetir o passo 9 mais uma vez. Abrir o link **antigo** →
    "Este link não é mais válido, solicite um novo." Abrir o link **novo** →
    formulário de nova senha.
11. **Redefinir senha.** Definir nova senha (≥ 8). → Sucesso; vai para `/login`.
    Reabrir o mesmo link → "Este link não é mais válido...". Logar com a nova
    senha → ok. Logar com a senha antiga → "E-mail ou senha inválidos.".
12. **Link sem token.** Abrir `/redefinir-senha` sem `?token=` → tela "Link
    inválido" com CTA para solicitar outro.
13. **Resiliência do e-mail.** Derrubar o container SMTP e repetir o passo 9. →
    A resposta da API continua `202` e imediata. Subir o SMTP de novo → o relay
    entrega o e-mail atrasado (evento ainda `dispatched_at IS NULL`).
14. **Restart.** Com um `PasswordResetRequested` ainda pendente (SMTP no ar),
    reiniciar o `api.ludens`. → Ao subir, o relay processa o evento pendente e o
    e-mail chega; nada se perde.
15. **Sem vazamento.** Varrer os logs do `api.ludens` do roteiro inteiro: nenhum
    CPF, e-mail, senha, hash ou token em claro. Varrer as respostas do Network:
    `/users/{id}` só devolve os dados do próprio usuário.

Resultado esperado de cada passo é o descrito na própria linha; qualquer desvio
vira bug registrado conforme `docs.ludens/backend/testing.md` (severidade **alta**
se expor dado sensível ou permitir login/token inválido).

---

## 5. Riscos e pontos de atenção

- **`security_stamp` só no access token, não no refresh.** O refresh é opaco e
  validado por hash contra a linha em `refresh_tokens`; quem "derruba" o refresh
  é a detecção de reuso (rotação) e a regeneração do `security_stamp` que o
  usecase dispara. Teste explícito: `test_rotate_de_token_ja_rotacionado...` +
  o fluxo cross-surface de reuso.
- **Token de redefinição em claro na linha de `events`.** `PasswordResetRequested`
  carrega `reset_token` em claro (o handler precisa dele para montar o link). Só
  o **hash** vai para `password_reset_tokens`. A linha de `events` é interna e o
  token é de uso único e expira em 1 h — aceitável para o N1; se `events` virar
  auditoria externa, mover o link para um cofre efêmero.
- **Reidratação do `AggregateRoot` pelo SQLAlchemy.** `identity` é o primeiro
  módulo de negócio a carregar um aggregate do banco e mutá-lo em seguida (`User`
  vindo de `login`/`change_password`, não só recém-criado) — se `_events`/`_version`
  não sobreviverem a esse caminho (o `__init__` normal não roda na reidratação do
  ORM), `raise_event` quebra em qualquer método de mudança de estado. Isso é
  responsabilidade do `AggregateRoot` em `core/domain/aggregate.py`, não deste
  módulo, mas cobrir aqui com um teste de usecase (Postgres real): carregar o
  `User`, `change_password`, `save` — o evento `UserPasswordChanged` precisa
  aparecer na tabela `events`.
- **`bcrypt` trunca em 72 bytes.** O `PasswordHasher` corta explicitamente; senhas
  muito longas passam a ser equivalentes ao prefixo de 72 bytes. Documentado; sem
  impacto prático (limite de 128 no schema, quase sempre ASCII).
- **Primeira migration do repositório.** `0001_outbox_events` cria a tabela
  `events` que o `AggregateRepository` já pressupõe. Rodar `alembic upgrade head`
  contra banco limpo antes de qualquer teste de usecase/integração.
- **`JWT_SECRET_KEY` no CI.** Sem o secret, o job usa o default de dev do
  `config.py` — os testes passam, mas não exercitam a assinatura com o segredo
  real. O `env:` no `ci.yml` fecha isso.
- **Relógio.** `RefreshToken.is_expired` e `PasswordResetToken.consume` comparam
  contra `datetime.now(timezone.utc)` — todos os `datetime` são timezone-aware;
  um `expires_at` naïve quebraria a comparação. Os aggregates só recebem `now`/
  `expires_at` do usecase, que usa `datetime.now(timezone.utc)`.
- **Frontend sem shadcn/ui ainda.** As telas usam `<input>`/`<button>` nativos +
  Tailwind. Quando o design system entrar, trocar em `components/` sem tocar em
  hooks/services.

---

## 6. Passo a passo TBD (QA)

```bash
git checkout master && git pull && git checkout -b test/09-identity-auth

git add tests/modules/__init__.py tests/modules/identity
git commit -m "test(identity): cobrir CPF, security_stamp e tokens de auth"
```

`pytest -q` verde localmente (roda junto com `tests/test_core.py` e
`tests/test_smoke.py`) → `/team-ludens:tbd-pr`. Os testes de domínio entram na
mesma issue/PR da fatia Backend ou numa sub-issue de QA em `api.ludens`, conforme
`/team-ludens:tbd-start`.

---

## 7. Bloqueios em aberto

Nenhum. Decisões de produto fechadas no `spec.md` (§8) e no `logic.md` (§5).

---

## 8. Ajustes feitos no `integration.md`

Registrados em conjunto com `backend.md` §8 e `frontend.md` §8 (o `integration.md`
segue `status: alvo`):

- Envelope de erro 4xx: `{ "detail": [ { "field": string, "message": string } ] }`.
- `POST /auth/password/change` → corpo `{ currentPassword, newPassword }`.
- `expiresIn` em segundos; `role` = `"BUYER"` | `"ADMIN"`.
- Sem prefixo/versionamento de rota no N1.
- `POST /auth/logout` também apaga o cookie `refresh_token`.
