---
status: draft
spec: notification-transactional-email
surface: quality
created_at: 2026-09-11
---

# E-mails transacionais — Quality

**Resumo:** cobertura de domínio (pytest, sem rede/DB real) do módulo
`notification` — os dois adapters de `EmailService` (SES via boto3 mockado,
console/log), a factory que escolhe entre eles por configuração, o template
de redefinição de senha, e o handler que consome `PasswordResetRequested` e
monta a URL do link. Nenhum teste depende de credencial AWS real nem de rede.
**RF:** RF09 (parte) · **RN:** — (reforça RNF01, RNF06) · **Módulo backend:**
`notification`
**Contrato:** nenhum (`notification` não expõe rota própria).

---

## 1. Definition of Ready — checagem

Contra `docs.ludens/team/quality.md`.

| Item DoR | Situação |
| --- | --- |
| Critérios de aceite testáveis | ✅ `logic.md` §3 (regras) e `backend.md` §3 (onde cada regra entra) dão entrada/saída de cada caso. |
| Contrato (`integration.md`) definido | ✅ N/A — sem rota própria; `frontend.md` já documenta por quê. |
| RN citadas com valores aprovados | ⚠️ Não há RN numérica própria; reforça RNF01 (dado sensível) e RNF06 (troca de provedor por configuração). |
| Dependências de outras features resolvidas | ✅ Depende só do evento `PasswordResetRequested`, já emitido por `identity-auth` (aggregate `User`, em produção). Os outros cinco tipos de e-mail de `logic.md` ficam de fora desta entrega — módulos que os emitem não existem (ver `backend.md` §7). |

**Veredito:** pronto para desenvolver, no escopo reduzido (só
`PasswordResetRequested`) que `backend.md` declara explicitamente.

---

## 2. Casos de teste de domínio (pytest)

Testes puros/com dublê: nenhum hit de rede real ao SES, nenhuma credencial
AWS necessária. `boto3.client` é substituído por um dublê em memória.

| Caso | Cenário (estado → ação → asserção) | RN/RF |
| --- | --- | --- |
| `test_console_email_service_loga_e_nao_lanca` | `ConsoleEmailService()` → `send(...)` → não lança, mensagem aparece no log | RNF06 |
| `test_factory_retorna_console_por_padrao` | `settings.email_backend="console"` (default) → `get_email_service()` → instância de `ConsoleEmailService` | RNF06 |
| `test_factory_retorna_ses_quando_configurado` | `settings.email_backend="ses"` → `get_email_service()` → instância de `SesEmailService` | RNF06 |
| `test_factory_reaproveita_a_mesma_instancia` | duas chamadas seguidas de `get_email_service()` (mesmo backend) → mesmo objeto (`is`) | — |
| `test_ses_email_service_envia_com_sucesso` | client SES dublê sem erro → `SesEmailService().send(...)` → `send_email` chamado com `Destination`/`Subject`/`Body` corretos | RF09 |
| `test_ses_email_service_traduz_erro_boto_em_email_service_error` | client SES dublê lança `ClientError` → `send(...)` → `EmailServiceError`, sem exceção crua do botocore | RF09 · RNF01 |
| `test_password_reset_email_template_contem_link_e_nao_expoe_dado_sensivel` | `password_reset_email(url)` → corpo contém a URL, não contém CPF/senha/hash | RNF01 |
| `test_handler_monta_url_com_frontend_base_url_e_token_e_envia` | payload `{"email", "token", ...}` + `EmailService` dublê → `handle_password_reset_requested(payload)` → dublê recebe `to=email`, corpo com `FRONTEND_BASE_URL + "/redefinir-senha?token=" + token` | RF09 |

### `tests/modules/notification/__init__.py` — novo

```python
# tests/modules/notification/__init__.py — novo
```

### `tests/modules/notification/test_console_email_service.py` — novo

```python
# tests/modules/notification/test_console_email_service.py — novo
import logging

from app.modules.notification.infrastructure.services import ConsoleEmailService


async def test_console_email_service_loga_e_nao_lanca(caplog):
    service = ConsoleEmailService()

    with caplog.at_level(logging.INFO):
        await service.send("comprador@example.com", "Assunto de teste", "<p>corpo</p>")

    assert "comprador@example.com" in caplog.text
    assert "Assunto de teste" in caplog.text
```

### `tests/modules/notification/test_factory.py` — novo

```python
# tests/modules/notification/test_factory.py — novo
import pytest

from app.config import settings
from app.modules.notification.infrastructure.services import get_email_service
from app.modules.notification.infrastructure.services.console_email_service import ConsoleEmailService
from app.modules.notification.infrastructure.services.ses_email_service import SesEmailService


@pytest.fixture(autouse=True)
def _limpa_cache_da_factory():
    get_email_service.cache_clear()
    yield
    get_email_service.cache_clear()


def test_factory_retorna_console_por_padrao(monkeypatch):
    monkeypatch.setattr(settings, "email_backend", "console")

    assert isinstance(get_email_service(), ConsoleEmailService)


def test_factory_retorna_ses_quando_configurado(monkeypatch):
    monkeypatch.setattr(settings, "email_backend", "ses")
    monkeypatch.setattr("boto3.client", lambda *args, **kwargs: object())

    assert isinstance(get_email_service(), SesEmailService)


def test_factory_reaproveita_a_mesma_instancia(monkeypatch):
    monkeypatch.setattr(settings, "email_backend", "console")

    assert get_email_service() is get_email_service()
```

### `tests/modules/notification/test_ses_email_service.py` — novo

```python
# tests/modules/notification/test_ses_email_service.py — novo
import pytest
from botocore.exceptions import ClientError

from app.modules.notification.infrastructure.services.email_service import EmailServiceError
from app.modules.notification.infrastructure.services.ses_email_service import SesEmailService


class _FakeSesClient:
    def __init__(self, *, should_fail: bool = False) -> None:
        self.should_fail = should_fail
        self.calls: list[dict] = []

    def send_email(self, **kwargs):
        self.calls.append(kwargs)

        if self.should_fail:
            raise ClientError({"Error": {"Code": "MessageRejected", "Message": "recusado"}}, "SendEmail")


async def test_ses_email_service_envia_com_sucesso(monkeypatch):
    fake_client = _FakeSesClient()
    monkeypatch.setattr("boto3.client", lambda *args, **kwargs: fake_client)

    service = SesEmailService()
    await service.send("comprador@example.com", "Assunto", "<p>corpo</p>")

    assert len(fake_client.calls) == 1
    call = fake_client.calls[0]
    assert call["Destination"] == {"ToAddresses": ["comprador@example.com"]}
    assert call["Message"]["Subject"]["Data"] == "Assunto"
    assert call["Message"]["Body"]["Html"]["Data"] == "<p>corpo</p>"


async def test_ses_email_service_traduz_erro_boto_em_email_service_error(monkeypatch):
    fake_client = _FakeSesClient(should_fail=True)
    monkeypatch.setattr("boto3.client", lambda *args, **kwargs: fake_client)

    service = SesEmailService()

    with pytest.raises(EmailServiceError):
        await service.send("comprador@example.com", "Assunto", "<p>corpo</p>")
```

### `tests/modules/notification/test_templates.py` — novo

```python
# tests/modules/notification/test_templates.py — novo
from app.modules.notification.infrastructure.templates import password_reset_email


def test_password_reset_email_template_contem_link_e_nao_expoe_dado_sensivel():
    url = "http://localhost:3000/redefinir-senha?token=abc123"
    subject, html_body = password_reset_email(url)

    assert url in html_body
    assert "senha" in subject.lower()

    dados_sensiveis = ["cpf", "password_hash", "cartão", "pix"]
    corpo_minusculo = html_body.lower()
    assert not any(dado in corpo_minusculo for dado in dados_sensiveis)
```

### `tests/modules/notification/test_handlers.py` — novo

```python
# tests/modules/notification/test_handlers.py — novo
from app.config import settings
from app.modules.notification.handlers import handle_password_reset_requested


class _FakeEmailService:
    def __init__(self) -> None:
        self.sent: list[tuple[str, str, str]] = []

    async def send(self, to: str, subject: str, html_body: str) -> None:
        self.sent.append((to, subject, html_body))


async def test_handler_monta_url_com_frontend_base_url_e_token_e_envia(monkeypatch):
    fake_service = _FakeEmailService()
    monkeypatch.setattr(
        "app.modules.notification.handlers.get_email_service",
        lambda: fake_service,
    )

    payload = {
        "id": "11111111-1111-1111-1111-111111111111",
        "email": "comprador@example.com",
        "token": "token-de-teste",
        "expires_at": "2026-09-11 16:00:00+00:00",
    }

    await handle_password_reset_requested(payload)

    assert len(fake_service.sent) == 1
    to, subject, html_body = fake_service.sent[0]
    assert to == "comprador@example.com"
    assert "redefinição" in subject.lower()

    esperado = f"{settings.frontend_base_url}/redefinir-senha?token=token-de-teste"
    assert esperado in html_body
```

---

## 3. Testes de integração cross-surface

Sem frontend nesta feature (ver `frontend.md`). O único fluxo cross-surface
real é backend-para-backend, via outbox — não é um teste de HTTP:

| Fluxo | Verifica |
| --- | --- |
| `identity-auth` emite `PasswordResetRequested` → outbox grava evento → relay chama `handle_password_reset_requested` → `EmailService.send` é chamado | O handler está de fato registrado (`@register("PasswordResetRequested")` roda no import de `app/main.py`) e o payload gravado por `AggregateRepository._serialize` tem as chaves que o handler espera (`email`, `token`). Cobrir com um teste de integração leve em `identity-auth` (fora deste documento) ou manualmente (§4). |

## 4. Roteiro de teste manual pré-entrega

1. Local, com `EMAIL_BACKEND=console` (padrão do `.env.example`): chamar
   `POST /auth/forgot-password` com um e-mail cadastrado → esperado: resposta
   neutra imediata; em até ~`OUTBOX_RELAY_INTERVAL_SECONDS` (2s), o log da
   aplicação mostra o e-mail "enviado" com o link de redefinição.
2. Repetir com um e-mail **não** cadastrado → esperado: mesma resposta
   neutra; nenhum log de e-mail aparece (nenhum evento foi emitido).
3. Copiar o link do log, completar a redefinição de senha → esperado: senha
   trocada, `security_stamp` rotacionado (login antigo invalidado).
4. Com `EMAIL_BACKEND=ses` e credenciais reais de uma conta SES em sandbox,
   repetir o passo 1 usando um e-mail **verificado** na conta SES → esperado:
   e-mail chega de verdade, com o link funcional.
5. Derrubar a rede/DNS momentaneamente (ou apontar `AWS_REGION` errado) com
   `EMAIL_BACKEND=ses` → esperado: `EmailServiceError` no log do relay, o
   evento continua com `dispatched_at NULL`, e é retentado no próximo ciclo
   sem derrubar a API nem a resposta do `forgot-password`.

## 5. Riscos e pontos de atenção

- Handler não é idempotente de verdade (débito registrado em `backend.md`
  §7) — teste de duplicidade sob crash do relay não é coberto aqui (exigiria
  simular queda do processo no meio do lote; desproporcional ao risco).
- Testes de `SesEmailService` cobrem só a tradução de erro e o formato da
  chamada — não substituem um teste manual real contra uma conta SES em
  sandbox (passo 4 do roteiro) antes do primeiro deploy em produção.
- Sem cobertura automatizada dos outros cinco tipos de e-mail — não existem
  ainda (ver `backend.md` §7); ficam para quando os módulos que os emitem
  forem implementados.

## 6. Passo a passo TBD (QA)

```bash
git checkout master && git pull && git checkout -b test/<NN>-notification-email-service
git add tests/modules/notification && git commit -m "test(notification): cobrir adapters de e-mail, factory e handler de redefinição de senha"
```

## 7. Bloqueios em aberto

Nenhum.
