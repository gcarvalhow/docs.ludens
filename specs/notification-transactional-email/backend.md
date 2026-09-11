---
status: draft
spec: notification-transactional-email
surface: backend
created_at: 2026-09-11
---

# E-mails transacionais — Backend

> **Nota de escopo (2026-09-11):** este documento substitui o antigo
> `implementation-spec.md` (nunca terminado, ficava em "EmailService (SMTP
> async)"). Decisão desta rodada: o transporte é **AWS SES via boto3**, não
> SMTP — `aiosmtplib` (já em `pyproject.toml`, nunca usado) sai; `boto3` entra.
> Referência de forma (não de transporte) foi o `EmailService` do
> `api.societiza` (Vert Group): `SmtpClient` cru, `SendAsync(to, toName,
> subject, body, ct)`, log antes/depois, exceção se host não configurado — a
> interface simples é boa, o transporte SMTP genérico deles não é o que
> queremos aqui.
>
> **Escopo real desta entrega:** só o handler de `PasswordResetRequested`
> entra em código — é o **único** evento dos seis listados em `logic.md` que
> já é emitido por código real hoje (`identity-auth`, aggregate `User`). Os
> outros cinco (`OrderPaid`, `OrderRefunded`, `SessionCancelled`,
> `SessionRescheduled`, `TicketEmailResendRequested`) pertencem a módulos
> (`booking`, `payment`) que **não existem ainda** — não há evento real para
> registrar handler. Cada um ganha seu handler quando o módulo que o emite for
> implementado; a infraestrutura de e-mail (`EmailService`, adapters,
> templates, factory) já fica pronta pra eles reaproveitarem. O e-mail de
> confirmação de troca de e-mail (`identity-auth`, reescopo desta sessão)
> também é débito pendente — ver `identity-auth/backend.md`.

**RF:** RF09 (parte de recuperação de senha) · **RN:** — · **Módulo
backend:** `notification` (novo módulo)
**Contrato:** este módulo não expõe rota própria (RF09 continua expondo
`POST /auth/forgot-password`); não há `integration.md` novo.
**Carregar antes:** skill `backend-architecture`,
`docs.ludens/backend/overview.md`, `docs.ludens/backend/conventions.md`,
`docs.ludens/backend/design/001-outbox-in-process.md`.

**Resumo:** módulo `notification` sem aggregate — só infraestrutura de envio.
Uma interface `EmailService` (RNF06: trocável por configuração) com dois
adapters: `SesEmailService` (produção, boto3, timeout explícito, exceção
própria) e `ConsoleEmailService` (desenvolvimento local, só loga — não precisa
de credencial AWS pra rodar o projeto). Um handler registrado no outbox
consome `PasswordResetRequested` — o evento que já existe no código e fica
sem handler hoje (ver débito registrado em `identity-auth/backend.md §7`) — e
envia o e-mail via `EmailService`.

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | pacotes | `src/app/modules/notification/**/__init__.py` (vazios) | novo |
| 2 | infrastructure | `src/app/modules/notification/infrastructure/services/email_service.py` | novo |
| 3 | infrastructure | `src/app/modules/notification/infrastructure/services/ses_email_service.py` | novo |
| 4 | infrastructure | `src/app/modules/notification/infrastructure/services/console_email_service.py` | novo |
| 5 | infrastructure | `src/app/modules/notification/infrastructure/services/factory.py` | novo |
| 6 | infrastructure | `src/app/modules/notification/infrastructure/services/__init__.py` | novo |
| 7 | infrastructure | `src/app/modules/notification/infrastructure/templates.py` | novo |
| 8 | outbox | `src/app/modules/notification/handlers.py` | novo |
| 9 | api | `src/app/main.py` | editar |
| 10 | config | `src/app/config.py` | editar |
| 11 | config | `.env.example` | editar |
| 12 | config | `pyproject.toml` | editar |

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

### 3. `src/app/modules/notification/infrastructure/services/ses_email_service.py` — novo

```python
import asyncio

import boto3
from botocore.config import Config as BotoConfig
from botocore.exceptions import BotoCoreError, ClientError

from app.config import settings

from app.modules.notification.infrastructure.services.email_service import EmailServiceError

class SesEmailService:
    def __init__(self) -> None:
        self._client = boto3.client(
            "ses",
            region_name=settings.aws_region,
            config=BotoConfig(connect_timeout=5, read_timeout=10, retries={"max_attempts": 2, "mode": "standard"}),
        )

    async def send(self, to: str, subject: str, html_body: str) -> None:
        try:
            await asyncio.to_thread(self._send_sync, to, subject, html_body)
        except (BotoCoreError, ClientError) as exc:
            raise EmailServiceError(f"Falha ao enviar e-mail via SES para {to}: {exc}") from exc

    def _send_sync(self, to: str, subject: str, html_body: str) -> None:
        self._client.send_email(
            Source=f"{settings.email_from_name} <{settings.email_from_address}>",
            Destination={"ToAddresses": [to]},
            Message={
                "Subject": {"Data": subject, "Charset": "UTF-8"},
                "Body": {"Html": {"Data": html_body, "Charset": "UTF-8"}},
            },
        )
```

Pontos de produção deliberados:

- `boto3` é síncrono — a chamada real roda em `asyncio.to_thread`, senão
  bloqueia o loop de eventos que também serve requisições HTTP e o próprio
  relay do outbox.
- Timeout explícito (`connect_timeout=5`, `read_timeout=10`) — sem isso, uma
  SES lenta/parada trava a thread indefinidamente e atrasa o lote inteiro do
  relay.
- `max_attempts=2` no nível do boto3 (retry de rede transitório) — deliberadamente
  baixo porque o outbox **já** retenta o evento inteiro no próximo ciclo do
  relay se o handler falhar (ver `handlers.py` abaixo); não empilhar duas
  camadas agressivas de retry.
- Nenhuma exceção crua do `botocore` escapa do adapter — sempre vira
  `EmailServiceError`, com o e-mail de destino na mensagem (nunca o corpo do
  e-mail, que pode conter o token).
- Credenciais AWS **não** passam por `Settings`/`.env` — o `boto3.client`
  resolve pela cadeia padrão (variável de ambiente `AWS_ACCESS_KEY_ID`/
  `AWS_SECRET_ACCESS_KEY`, ou role da instância/task em produção). Isso é
  deliberado: em produção, a forma mais segura é uma IAM role anexada ao
  contêiner, sem chave estática nenhuma no `.env` — ver §4 DevOps.

### 4. `src/app/modules/notification/infrastructure/services/console_email_service.py` — novo

```python
import logging

logger = logging.getLogger(__name__)

class ConsoleEmailService:
    async def send(self, to: str, subject: str, html_body: str) -> None:
        logger.info(
            "E-mail (dev, não enviado de verdade) — Para: %s | Assunto: %s\n%s",
            to, subject, html_body,
        )
```

Adapter de desenvolvimento local: nenhuma credencial AWS é necessária pra
rodar o projeto — é o padrão (`EMAIL_BACKEND=console` no `.env.example`). Só
loga; quem quiser ver o link de verdade durante o desenvolvimento lê o log da
aplicação.

### 5. `src/app/modules/notification/infrastructure/services/factory.py` — novo

```python
from functools import lru_cache

from app.config import settings

from app.modules.notification.infrastructure.services.email_service import EmailService
from app.modules.notification.infrastructure.services.ses_email_service import SesEmailService
from app.modules.notification.infrastructure.services.console_email_service import ConsoleEmailService

@lru_cache
def get_email_service() -> EmailService:
    if settings.email_backend == "ses":
        return SesEmailService()

    return ConsoleEmailService()
```

`lru_cache` sem argumento — uma única instância por processo, reaproveitando
o cliente `boto3` (a AWS recomenda não recriar o client a cada chamada).

### 6. `src/app/modules/notification/infrastructure/services/__init__.py` — novo

```python
from .email_service import EmailService, EmailServiceError
from .ses_email_service import SesEmailService
from .console_email_service import ConsoleEmailService
from .factory import get_email_service

__all__ = ["EmailService", "EmailServiceError", "SesEmailService", "ConsoleEmailService", "get_email_service"]
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
```

Um por tipo de e-mail (RN de `logic.md` §4: "templates em pt-BR, um por tipo,
versionados no código"). Os outros cinco tipos entram aqui quando os módulos
que os disparam existirem — não fabricar template pra evento que não existe.

### 8. `src/app/modules/notification/handlers.py` — novo

```python
from app.config import settings
from app.outbox.registry import register

from app.modules.notification.infrastructure.services import get_email_service
from app.modules.notification.infrastructure.templates import password_reset_email

@register("PasswordResetRequested")
async def handle_password_reset_requested(payload: dict) -> None:
    reset_url = f"{settings.frontend_base_url}/redefinir-senha?token={payload['token']}"
    subject, html_body = password_reset_email(reset_url)

    await get_email_service().send(payload["email"], subject, html_body)
```

`payload` é o dict que `AggregateRepository._serialize` gravou —
`{"id": "...", "email": "...", "token": "...", "expires_at": "..."}` (ver
`core/infrastructure/repositories/repository.py` no código real). O handler
não trata exceção — se `EmailService.send` levantar `EmailServiceError`, ela
sobe pro relay (`app/outbox/relay.py`), que já loga e deixa `dispatched_at`
sem marcar, retentando no próximo ciclo. Não duplicar essa lógica aqui.

**Débito consciente (não idempotente de verdade):** se o relay processar o
lote, o e-mail sair, e o processo cair *antes* de marcar `dispatched_at`, o
mesmo reset é reenviado no próximo ciclo — a pessoa recebe dois e-mails com o
mesmo link (mesmo token, ainda válido). Não é dano de segurança nem de
correção, só duplicidade rara de notificação; implementar um marcador de
idempotência por evento é desproporcional ao risco no MVP. Registrado como
débito técnico, não escondido — ver §7.

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
email_backend: Literal["ses", "console"] = "console"
email_from_address: str = ""
email_from_name: str = "Ludens"
aws_region: str = "us-east-1"
frontend_base_url: str = "http://localhost:3000"
```

### 11. `.env.example` — editar

```bash
# --- E-mail transacional (notification) --- (CONFIG, exceto credencial AWS)
# EMAIL_BACKEND=console não envia de verdade (só loga) — use em dev, sem
# precisar de conta AWS. Trocar pra "ses" em produção.
EMAIL_BACKEND=console
EMAIL_FROM_ADDRESS=
EMAIL_FROM_NAME=Ludens
AWS_REGION=us-east-1
# Credenciais AWS (AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY) NÃO vão aqui —
# em produção usar IAM role anexada ao contêiner/instância; boto3 resolve
# sozinho. Só defina essas duas var de ambiente manualmente se for testar o
# adapter SES localmente sem role (nunca commitar).

# --- URL do frontend (link de e-mails) --- (CONFIG)
FRONTEND_BASE_URL=http://localhost:3000
```

### 12. `pyproject.toml` — editar

```toml
# dependencies: remover "aiosmtplib>=3.0" (nunca usado — SMTP não é mais o
# transporte escolhido), adicionar:
"boto3>=1.34",
```

---

## 3. Onde cada regra de negócio entra

| Regra | Arquivo · função | Como |
| --- | --- | --- |
| Link de redefinição expira em 1h, uso único (`identity-auth`) | `handlers.py` · `handle_password_reset_requested` | Não recalcula validade — só monta a URL com o `token` que já veio pronto do evento; a expiração é checada em `identity-auth`, no consumo do token. |
| Falha de envio não derruba a transação de origem (RF05/ADR 001) | `handlers.py` + `app/outbox/relay.py` (já existe) | Handler roda fora da transação de `identity-auth`; exceção não propaga pro usecase que originou o evento. |
| Nenhum dado sensível além do necessário no e-mail (RNF01) | `templates.py` · `password_reset_email` | Corpo só tem o link com token — sem CPF, sem hash, sem dado de pagamento. |
| Dependência externa trocável por configuração (RNF06) | `factory.py` · `get_email_service` | `EMAIL_BACKEND` decide o adapter; nenhum código de domínio/aplicação conhece SES. |

---

## 4. DevOps

- Variáveis novas em `src/app/config.py` / `.env.example`: `EMAIL_BACKEND`,
  `EMAIL_FROM_ADDRESS`, `EMAIL_FROM_NAME`, `AWS_REGION`, `FRONTEND_BASE_URL`
  — todas `CONFIG`. As credenciais AWS (`AWS_ACCESS_KEY_ID`/
  `AWS_SECRET_ACCESS_KEY`) são `SECRET` mas **não** entram no `.env` da
  aplicação nem em `Settings` — resolvidas pelo boto3 via IAM role em
  produção (ECS task role / instance profile) ou pela cadeia padrão de
  credenciais em dev, se alguém optar por testar o adapter SES localmente.
- **Pré-requisitos de infraestrutura, fora do código** (registrar como
  checklist de deploy, não implementar aqui):
  - Conta SES nasce em **sandbox**: só envia para endereços/domínios
    verificados, limite baixo de volume/taxa. Pedir **production access** à
    AWS (support case, aprovação em até ~24h) antes de qualquer envio real a
    destinatários não verificados.
  - Verificar o **domínio remetente** (registro SPF e DKIM via SES) — sem
    isso, entregabilidade cai e provedores marcam como spam.
  - IAM: criar uma policy mínima (`ses:SendEmail`, `ses:SendRawEmail`) e
    anexar como role ao ambiente de execução — nunca uma chave de usuário IAM
    de longa duração num `.env` de produção.
- **Custo real (não é grátis, é irrelevante no volume do teatro):** SES cobra
  ~US$0,10 por 1.000 e-mails enviados (fora do free tier de 12 meses da AWS
  quando a origem é EC2, que não se aplica necessariamente a este deploy).
  Para o volume esperado de um teatro comunitário (dezenas a poucas centenas
  de e-mails/mês), isso fica na casa de centavos de dólar por mês — não é
  zero, mas é desprezível. Não anunciar como "grátis" sem essa ressalva.
- Nenhum segredo novo de CI (`.github/workflows/ci.yml`) — os testes de
  `quality.md` rodam contra `ConsoleEmailService`, sem precisar de conta AWS
  nem de credencial no pipeline.

---

## 5. Passo a passo TBD (Backend)

```bash
git checkout master && git pull && git checkout -b feat/<NN>-notification-email-service
# commit 1 — infraestrutura de e-mail
git add src/app/modules/notification && git commit -m "feat(notification): adicionar EmailService com adapters SES e console"
# commit 2 — handler + registro no outbox
git add src/app/modules/notification/handlers.py src/app/main.py && git commit -m "feat(notification): consumir PasswordResetRequested e enviar e-mail de redefinição"
# commit 3 — config e dependências
git add src/app/config.py .env.example pyproject.toml && git commit -m "chore(notification): configurar SES e remover aiosmtplib não usado"
```

Depois: `/team-ludens:tbd-pr` (senior-dev Modo 2 + `/code-review`) → push → PR
`Closes #<NN>` → merge (1 aprovação + CI verde).

---

## 6. Ordem entre as superfícies

Sem frontend próprio — nenhuma tela nova, nenhum contrato de API novo (o
`POST /auth/forgot-password` de `identity-auth` já existe e não muda). QA
(casos de domínio) pode começar em paralelo ao backend a partir deste
documento; não há dependência de merge do frontend porque não há frontend
nesta feature.

---

## 7. Débitos técnicos registrados

- **Handler não é idempotente de verdade** (ver §2, arquivo 8) — risco aceito
  de e-mail duplicado numa janela de crash muito específica do relay. Não
  implementado marcador de idempotência por evento nesta entrega.
- **Cinco dos seis tipos de e-mail de `logic.md` continuam sem handler**
  (`OrderPaid`, `OrderRefunded`, `SessionCancelled`, `SessionRescheduled`,
  `TicketEmailResendRequested`) — os módulos `booking` e `payment` que os
  emitiriam ainda não existem. Cada um entra junto do `backend.md` do módulo
  que o disparar.
- **E-mail de confirmação de troca de e-mail** (`identity-auth`, reescopo
  2026-09-11) também não tem handler — o evento correspondente nem existe
  ainda no código (`identity-auth/backend.md` ainda reflete o escopo antigo,
  sem alteração de e-mail). Entra junto do rework de `identity-auth`.
- **Verificação de domínio, saída do sandbox do SES, e IAM role** são passos
  de infraestrutura que este documento não executa — são pré-requisito de
  deploy, listados em §4, não código.
