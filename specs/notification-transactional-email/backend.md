---
status: draft
spec: notification-transactional-email
surface: backend
created_at: 2026-09-11
---

# E-mails transacionais — Backend

> **Nota de escopo (2026-09-11):** este documento substitui o antigo
> `implementation-spec.md` (nunca terminado, ficava em "EmailService (SMTP
> async)"). Referência de forma (não de transporte) foi o `EmailService` do
> `api.societiza` (Vert Group): `SmtpClient` cru, `SendAsync(to, toName,
> subject, body, ct)`, log antes/depois, exceção se host não configurado — a
> interface simples é boa, o transporte SMTP genérico deles não é o que
> queremos aqui.
>
> **Nota de revisão (2026-09-17):** o provedor de produção mudou de **AWS
> SES** pra **Azure Communication Services (ACS)** — a hospedagem do projeto
> passou a ser 100% Azure (App Service via créditos do GitHub Student
> Developer Pack), então evitar depender de duas nuvens diferentes só pra
> e-mail. `boto3` sai, `azure-communication-email` entra. O adapter de
> desenvolvimento também mudou: em vez de só logar (`ConsoleEmailService`),
> agora envia de verdade pra um **Mailpit** local via Docker
> (`SmtpEmailService`, usando `aiosmtplib` — que já estava em
> `pyproject.toml` desde antes, agora finalmente em uso) — dá pra abrir
> `http://localhost:8025` e ver o e-mail de verdade, inclusive o link, sem
> precisar de conta Azure. Além disso, o escopo de handlers cresceu: a
> feature `identity-user-management` (issue `api.ludens#30`) mergeou e já
> emite, por código real, `EmailChangeRequested`, `EmailChanged` (aviso de
> cortesia pós-troca) e `AccountDeletionRequested`, além de
> `PasswordResetRequested` — os quatro ganham handler nesta entrega (issue
> `api.ludens#34`). Os outros cinco eventos de `logic.md` (`OrderPaid`,
> `OrderRefunded`, `SessionCancelled`, `SessionRescheduled`,
> `TicketEmailResendRequested`) continuam sem handler — pertencem a módulos
> (`booking`, `payment`, `catalog` além do que já existe) que ainda não
> existem.

**RF:** RF09 (parte de recuperação de senha) · **RN:** — · **Módulo
backend:** `notification` (novo módulo)
**Contrato:** este módulo não expõe rota própria (RF09 continua expondo
`POST /identity/password/forgot`); não há `integration.md` novo além do que
já existe aqui.
**Carregar antes:** skill `backend-architecture`,
`docs.ludens/backend/overview.md`, `docs.ludens/backend/conventions.md`,
`docs.ludens/backend/design/001-outbox-in-process.md`.

**Resumo:** módulo `notification` sem aggregate — só infraestrutura de envio.
Uma interface `EmailService` (RNF06: trocável por configuração) com dois
adapters: `AcsEmailService` (produção, `azure-communication-email` async,
exceção própria) e `SmtpEmailService` (desenvolvimento local, via
`aiosmtplib` contra um Mailpit em Docker — não precisa de credencial Azure
pra rodar o projeto). Quatro handlers registrados no outbox consomem
`PasswordResetRequested`, `EmailChangeRequested`, `EmailChanged` e
`AccountDeletionRequested` — os eventos que já existem no código e ficavam
sem handler — e enviam o e-mail correspondente via `EmailService`.

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | pacotes | `src/app/modules/notification/**/__init__.py` (vazios) | novo |
| 2 | infrastructure | `src/app/modules/notification/infrastructure/services/email_service.py` | novo |
| 3 | infrastructure | `src/app/modules/notification/infrastructure/services/acs_email_service.py` | novo |
| 4 | infrastructure | `src/app/modules/notification/infrastructure/services/smtp_email_service.py` | novo |
| 5 | infrastructure | `src/app/modules/notification/infrastructure/services/factory.py` | novo |
| 6 | infrastructure | `src/app/modules/notification/infrastructure/services/__init__.py` | novo |
| 7 | infrastructure | `src/app/modules/notification/infrastructure/templates.py` | novo |
| 8 | outbox | `src/app/modules/notification/handlers.py` | novo |
| 9 | api | `src/app/main.py` | editar |
| 10 | config | `src/app/config.py` | editar |
| 11 | config | `.env.example` / `.env.local` | editar |
| 12 | config | `pyproject.toml` | editar |
| 13 | infra dev | `docker/docker-compose.Development.yml` | editar (serviço `mailpit`) |

Não há `domain/`, `application/`, `api/routers/` nem migration — o módulo não
tem aggregate nem estado próprio (`notification` = "handlers de evento; sem
agregado", `docs.ludens/backend/overview.md`), e não expõe rota HTTP.

---

## 2. Código

### 1. Pacotes — `__init__.py` vazios

```python
# src/app/modules/notification/__init__.py — novo
```

```python
# src/app/modules/notification/infrastructure/__init__.py — novo
```

### 2. `src/app/modules/notification/infrastructure/services/email_service.py` — novo

```python
from typing import Protocol

class EmailServiceError(Exception):
    pass

class EmailService(Protocol):
    async def send(self, to: str, subject: str, html_body: str) -> None:
        ...
```

Só `to`/`subject`/`html_body` — os eventos de domínio hoje não carregam o
nome do destinatário (`PasswordResetRequested` só tem `id`, `email`, `token`,
`expires_at`), e o módulo `notification` não consulta o `UserRepository` de
`identity` diretamente (módulos não importam infraestrutura interna uns dos
outros — `docs.ludens/backend/overview.md`). Os templates tratam o
destinatário de forma genérica ("Olá,"), não personalizada por nome.

### 3. `src/app/modules/notification/infrastructure/services/acs_email_service.py` — novo

```python
from azure.communication.email.aio import EmailClient
from azure.core.exceptions import AzureError

from app.config import settings

from app.modules.notification.infrastructure.services.email_service import EmailServiceError

class AcsEmailService:
    async def send(self, to: str, subject: str, html_body: str) -> None:
        message = {
            "senderAddress": settings.acs_sender_address,
            "recipients": {"to": [{"address": to}]},
            "content": {"subject": subject, "html": html_body},
        }

        try:
            async with EmailClient.from_connection_string(
                settings.acs_connection_string, connection_timeout=5, read_timeout=10
            ) as client:
                poller = await client.begin_send(message)
                await poller.result()
        except AzureError as exc:
            raise EmailServiceError(f"Falha ao enviar e-mail via ACS para {to}: {exc}") from exc
```

`AzureError` (não só `HttpResponseError`) — é a base comum de erro de resposta
HTTP, autenticação (connection string malformada) e requisição/rede do SDK;
capturar só `HttpResponseError` deixaria vazar exceção crua em falha de auth
ou de rede, quebrando a garantia de "nunca a exceção crua do transporte".

Pontos de produção deliberados:

- `azure-communication-email` já expõe um cliente assíncrono nativo
  (`azure.communication.email.aio.EmailClient`) — sem precisar de
  `asyncio.to_thread` como seria necessário com um SDK só síncrono.
- Um `EmailClient` novo por chamada (`async with ... as client`), em vez de
  cachear um cliente de longa duração: é a opção mais simples e evita bug de
  ciclo de vida de sessão HTTP async reentrante; o volume esperado (dezenas a
  poucas centenas de e-mails/mês) não justifica otimizar isso agora (ADR 003
  — sem abstração/otimização antecipada). Revisitar só se o volume real
  exigir.
- Nenhuma exceção crua do SDK escapa do adapter — sempre vira
  `EmailServiceError`, com o e-mail de destino na mensagem (nunca o corpo do
  e-mail, que pode conter o token).
- Credencial (`ACS_CONNECTION_STRING`) é `SECRET` — nunca commitada; em
  produção fica em App Service Application Settings / Key Vault, nunca em
  `.env` versionado. Ver §4 DevOps.
- Timeout explícito (`connection_timeout=5`, `read_timeout=10`) — regra
  obrigatória da skill `backend-architecture` pra toda chamada de rede em
  `infrastructure/services/`; sem isso, um ACS lento/parado atrasa o lote
  inteiro do relay.

### 4. `src/app/modules/notification/infrastructure/services/smtp_email_service.py` — novo

```python
import aiosmtplib

from email.message import EmailMessage

from app.config import settings

from app.modules.notification.infrastructure.services.email_service import EmailServiceError

class SmtpEmailService:
    async def send(self, to: str, subject: str, html_body: str) -> None:
        message = EmailMessage()
        message["From"] = f"{settings.email_from_name} <{settings.email_from_address}>"
        message["To"] = to
        message["Subject"] = subject
        message.set_content(html_body, subtype="html")

        try:
            await aiosmtplib.send(message, hostname=settings.smtp_host, port=settings.smtp_port, timeout=10)
        except aiosmtplib.SMTPException as exc:
            raise EmailServiceError(f"Falha ao enviar e-mail via SMTP para {to}: {exc}") from exc
```

Adapter de desenvolvimento local: envia de verdade (não só loga) pra um
Mailpit rodando em Docker (`docker/docker-compose.Development.yml`, serviço
`mailpit`, UI em `http://localhost:8025`) — nenhuma credencial Azure é
necessária pra rodar o projeto. É o padrão (`EMAIL_BACKEND=smtp` no
`.env.example`). Quem quiser ver o e-mail de verdade — inclusive o link
clicável — durante o desenvolvimento abre a UI do Mailpit, não precisa ler
log de aplicação.

### 5. `src/app/modules/notification/infrastructure/services/factory.py` — novo

```python
from functools import lru_cache

from app.config import settings

from app.modules.notification.infrastructure.services.email_service import EmailService
from app.modules.notification.infrastructure.services.acs_email_service import AcsEmailService
from app.modules.notification.infrastructure.services.smtp_email_service import SmtpEmailService

@lru_cache
def get_email_service() -> EmailService:
    if settings.email_backend == "acs":
        return AcsEmailService()

    return SmtpEmailService()
```

`lru_cache` sem argumento — uma única instância por processo. Como o
`AcsEmailService` não guarda cliente nenhum como atributo (cria um por
chamada, ver item 3), cachear a instância aqui é só pra não recriar o objeto
Python à toa, não pra reaproveitar conexão.

### 6. `src/app/modules/notification/infrastructure/services/__init__.py` — novo

```python
from .email_service import EmailService, EmailServiceError
from .acs_email_service import AcsEmailService
from .smtp_email_service import SmtpEmailService
from .factory import get_email_service

__all__ = ["EmailService", "EmailServiceError", "AcsEmailService", "SmtpEmailService", "get_email_service"]
```

### 7. `src/app/modules/notification/infrastructure/templates.py` — novo

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

Um por tipo de e-mail (RN de `logic.md` §4: "templates em pt-BR, um por tipo,
versionados no código"). Os cinco tipos restantes de `logic.md` entram aqui
quando os módulos que os disparam existirem — não fabricar template pra
evento que não existe. `email_changed_courtesy_email` não recebe URL — é
puramente informativo, sem link nem ação (ver `identity-user-management/logic.md`
§1, "aviso de cortesia enviado ao endereço novo depois da troca confirmada").

### 8. `src/app/modules/notification/handlers.py` — novo

```python
from app.config import settings
from app.outbox.registry import register

from app.modules.notification.infrastructure.services import get_email_service
from app.modules.notification.infrastructure.templates import (
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

`payload` é o dict que `AggregateRepository._serialize` gravou. Os links de
`EmailChangeRequested`/`AccountDeletionRequested` apontam pra rotas de
frontend ainda não implementadas em `web.ludens`
(`/confirmar-troca-de-email`, `/confirmar-exclusao-de-conta`) — a página
recebe o `token` via query string no `GET` e chama, via JS, o
`PATCH`/`DELETE` real do backend (mesmo padrão de `/redefinir-senha`); ver
débito em `docs.ludens/team/tech-debt.md`. Nenhum handler trata exceção — se
`EmailService.send` levantar `EmailServiceError`, ela sobe pro relay
(`app/outbox/relay.py`), que já loga e deixa `dispatched_at` sem marcar,
retentando no próximo ciclo. Não duplicar essa lógica aqui.

**Débito consciente (não idempotente de verdade), nos quatro handlers:** se o
relay processar o lote, o e-mail sair, e o processo cair *antes* de marcar
`dispatched_at`, o mesmo e-mail é reenviado no próximo ciclo — a pessoa
recebe duas cópias com o mesmo link (mesmo token, ainda válido). Não é dano
de segurança nem de correção, só duplicidade rara de notificação;
implementar um marcador de idempotência por evento é desproporcional ao
risco no MVP. Registrado como débito técnico, não escondido — ver §7.

**Verificado localmente (2026-09-17):** os quatro handlers foram exercitados
de ponta a ponta contra um Mailpit real (`docker compose -f
docker/docker-compose.Development.yml up -d mailpit`) — os quatro e-mails
chegaram com assunto, destinatário e link corretos (conferido via
`GET http://localhost:8025/api/v1/messages`).

### 9. `src/app/main.py` — editar

```python
# adicionar entre os imports de routers e a criação do FastAPI(...)
from app.modules.notification import handlers as notification_handlers  # noqa: F401
```

O import por si só já registra o handler — o decorator `@register(...)` roda
na importação do módulo. `notification` não tem `router.py` (sem rota HTTP),
então não entra em `app.include_router(...)`; só precisa ser importado uma
vez pra o registro acontecer antes do primeiro ciclo do relay.

### 10. `src/app/config.py` — editar

```python
# adicionar ao corpo da classe Settings, após outbox_relay_interval_seconds
email_backend: Literal["acs", "smtp"] = "smtp"
email_from_address: str = "no-reply@ludens.local"
email_from_name: str = "Ludens"
acs_connection_string: str = ""
acs_sender_address: str = ""
smtp_host: str = "localhost"
smtp_port: int = 1025
frontend_base_url: str = "http://localhost:3000"
```

### 11. `.env.example` / `.env.local` — editar

```bash
# --- E-mail transacional (notification) --- (CONFIG, exceto connection string ACS)
# EMAIL_BACKEND=smtp envia de verdade pro Mailpit local (docker-compose, sem
# precisar de conta Azure) — veja a UI em http://localhost:8025. Trocar pra
# "acs" em produção.
EMAIL_BACKEND=smtp
EMAIL_FROM_ADDRESS=no-reply@ludens.local
EMAIL_FROM_NAME=Ludens
SMTP_HOST=ludens-mailpit-dev
SMTP_PORT=1025
# Azure Communication Services (produção) — connection string é SECRET, nunca
# commitar; ACS_SENDER_ADDRESS é o remetente verificado no recurso ACS.
ACS_CONNECTION_STRING=
ACS_SENDER_ADDRESS=

# --- URL do frontend (link de e-mails) --- (CONFIG)
FRONTEND_BASE_URL=http://localhost:3000
```

### 12. `pyproject.toml` — editar

```toml
# dependencies: aiosmtplib>=3.0 sai do "nunca usado" — passa a ser usado de
# verdade pelo SmtpEmailService. Adicionar:
"azure-communication-email>=1.0",
# EmailClient assíncrono (azure.communication.email.aio) usa aiohttp como
# transporte HTTP — não é dependência transitiva de azure-communication-email
# nem de azure-core; sem isso, from_connection_string() falha em runtime com
# ModuleNotFoundError. Só foi pego rodando o smoke test de verdade, não na
# revisão estática de conformidade — vale lição pra próximos adapters async
# do SDK Azure.
"aiohttp>=3.9",
```

### 13. `docker/docker-compose.Development.yml` — editar

```yaml
# adicionar ao services:, mesma network ludens-dev do serviço postgres
  mailpit:
    image: axllent/mailpit:latest
    container_name: ludens-mailpit-dev
    ports:
      - "1025:1025" # SMTP
      - "8025:8025" # UI web
    networks:
      - ludens-dev
```

---

## 3. Onde cada regra de negócio entra

| Regra | Arquivo · função | Como |
| --- | --- | --- |
| Link de redefinição/confirmação expira em 1h, uso único (`identity-auth`, `identity-user-management`) | `handlers.py` · os quatro handlers | Não recalcula validade — só monta a URL com o `token` que já veio pronto do evento; a expiração é checada no módulo `identity`, no consumo do token. |
| Falha de envio não derruba a transação de origem (RF05/ADR 001) | `handlers.py` + `app/outbox/relay.py` (já existe) | Handlers rodam fora da transação de `identity`; exceção não propaga pro usecase que originou o evento. |
| Nenhum dado sensível além do necessário no e-mail (RNF01) | `templates.py` | Corpo só tem o link com token (ou, no aviso de cortesia, nenhum dado) — sem CPF, sem hash, sem dado de pagamento. |
| Dependência externa trocável por configuração (RNF06) | `factory.py` · `get_email_service` | `EMAIL_BACKEND` decide o adapter; nenhum código de domínio/aplicação conhece ACS. |
| Aviso de cortesia só após confirmação, nunca no request (`identity-user-management/logic.md` §1) | `handlers.py` · `handle_email_changed` | Só reage ao evento `EmailChanged` (pós-confirmação); `EmailChangeRequested` (pré-confirmação) vai só pro e-mail antigo, nunca pro novo. |

---

## 4. DevOps

- Variáveis novas em `src/app/config.py` / `.env.example`: `EMAIL_BACKEND`,
  `EMAIL_FROM_ADDRESS`, `EMAIL_FROM_NAME`, `SMTP_HOST`, `SMTP_PORT`,
  `FRONTEND_BASE_URL` — todas `CONFIG`. `ACS_CONNECTION_STRING` é `SECRET` —
  entra em `Settings` (diferente do antigo desenho com boto3/IAM role), mas
  **nunca** commitada em `.env.example`/`.env.local`; em produção fica em
  App Service Application Settings ou Key Vault. `ACS_SENDER_ADDRESS` é
  `CONFIG` (endereço, não segredo).
- **Pré-requisitos de infraestrutura, fora do código** (registrar como
  checklist de deploy — Workstream de Terraform/infra, não implementar aqui):
  - Provisionar o recurso **Azure Communication Services** (Email
    Communication Service) e obter a connection string.
  - Verificar o **domínio remetente** no ACS (registro SPF/DKIM) — sem isso,
    entregabilidade cai e provedores marcam como spam.
  - Guardar `ACS_CONNECTION_STRING` no cofre de segredos do deploy (App
    Service Application Settings / Key Vault), nunca em arquivo versionado.
- **Custo real (não é grátis, é irrelevante no volume do teatro):** ACS Email
  cobra por e-mail enviado (faixa de centavos de dólar por 1.000 e-mails,
  variável por região). Para o volume esperado de um teatro comunitário
  (dezenas a poucas centenas de e-mails/mês), isso fica na casa de centavos
  de dólar por mês — não é zero, mas é desprezível. Não anunciar como
  "grátis" sem essa ressalva.
- Nenhum segredo novo de CI (`.github/workflows/ci.yml`) — o adapter padrão de
  desenvolvimento é o `SmtpEmailService` contra o Mailpit do
  docker-compose, sem precisar de conta Azure nem de credencial no pipeline.

---

## 5. Passo a passo TBD (Backend)

```bash
git checkout master && git pull && git checkout -b feat/34-email-service
# commit 1 — infraestrutura de e-mail
git add src/app/modules/notification && git commit -m "feat(notification): adicionar EmailService com adapters ACS e SMTP/Mailpit"
# commit 2 — handlers + registro no outbox
git add src/app/modules/notification/handlers.py src/app/main.py && git commit -m "feat(notification): consumir eventos de identity e enviar e-mail de verdade"
# commit 3 — config, dependências e docker-compose
git add src/app/config.py .env.example .env.local pyproject.toml docker/docker-compose.Development.yml && git commit -m "chore(notification): configurar ACS/SMTP e adicionar Mailpit ao docker-compose"
```

Depois: `/team-ludens:tbd-pr` (senior-dev Modo 2 + `/code-review`) → push → PR
`Closes #34` → merge (1 aprovação + CI verde).

---

## 6. Ordem entre as superfícies

Sem frontend próprio — nenhuma tela nova, nenhum contrato de API novo (as
rotas de `identity` que emitem esses eventos já existem e não mudam). QA
(casos de domínio) pode começar em paralelo ao backend a partir deste
documento. Há, porém, um débito de frontend **fora** desta feature: as
páginas que os links de `EmailChangeRequested`/`AccountDeletionRequested`
apontam (`/confirmar-troca-de-email`, `/confirmar-exclusao-de-conta`) ainda
não existem em `web.ludens` — ver `docs.ludens/team/tech-debt.md`.

---

## 7. Débitos técnicos registrados

- **Handlers não são idempotentes de verdade** (ver §2, arquivo 8) — risco
  aceito de e-mail duplicado numa janela de crash muito específica do relay.
  Não implementado marcador de idempotência por evento nesta entrega.
- **Cinco dos seis tipos de e-mail de `logic.md` continuam sem handler**
  (`OrderPaid`, `OrderRefunded`, `SessionCancelled`, `SessionRescheduled`,
  `TicketEmailResendRequested`) — os módulos `booking` e `payment` que os
  emitiriam ainda não existem. Cada um entra junto do `backend.md` do módulo
  que o disparar.
- **Páginas de frontend pra abrir os links de confirmação** (troca de e-mail,
  exclusão de conta) ainda não existem em `web.ludens` — só o backend está
  pronto; sem essas páginas, o link do e-mail não tem pra onde ir. Ver
  `docs.ludens/team/tech-debt.md`.
- **Verificação de domínio remetente e provisionamento do recurso ACS** são
  passos de infraestrutura que este documento não executa — são
  pré-requisito de deploy, listados em §4, cobertos pela Workstream de
  Terraform/infra do plano técnico de fechamento (deploy + identity +
  paginação + infra).
