---
status: draft
spec: notification-transactional-email
created_at: 2026-09-01
---

# E-mails transacionais — Implementation Spec

**Resumo:** módulo `notification` sem aggregate — só `EmailService` (SMTP async)
e handlers de outbox idempotentes para 6 eventos.
**RF:** RF05, RF09 (parte de e-mail) · **RN:** — · **Módulo backend:**
`notification` · **Feature frontend:** `account` (só o botão "reenviar") ·
**Contrato:** `specs/notification-transactional-email/integration.md`

**Depende de:** os eventos já existirem (`identity-auth`, `payment-*`,
`catalog-admin-management`, `booking-ticket-issuance`). Pode ser implementada em
paralelo com stubs dos eventos e ligada ao final.

Carregar antes: skill `backend-architecture` (`references/05` serviços,
`references/07` outbox).

---

## A. Backend — responsável: Igor

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | infrastructure | `src/app/modules/notification/infrastructure/services/email_service.py` | `EmailService` — `aiosmtplib`, `send(to, template, context)`; timeout; `EmailServiceError`; host/porta/user/senha/remetente de `settings` |
| 2 | infrastructure | `.../infrastructure/templates/` | um arquivo por template: `order_confirmed.html`, `password_reset.html`, `order_refunded.html`, `session_cancelled.html`, `session_rescheduled.html`. pt-BR, HTML simples sem imagem externa |
| 3 | infrastructure | `.../infrastructure/repositories/email_log_repository.py` (opcional) | `EmailLog` (to, template, ref_id, status, sent_at) — para idempotência e auditoria; usa `BaseRepository` (não é aggregate) |
| 4 | application | `.../application/renderers.py` | `render(template, context) -> (subject, html)` — monta assunto + corpo por tipo |
| 5 | handlers | `.../modules/notification/handlers.py` | `@register(...)` para: `OrderPaid`, `PasswordResetRequested`, `OrderRefunded`, `SessionCancelled`, `SessionRescheduled`, `TicketEmailResendRequested`. Cada handler: monta o contexto (pode ler dados via dependency exportada do módulo de origem), checa idempotência (`EmailLog` por `(to, template, ref_id)`, exceto `TicketEmailResendRequested`), chama `EmailService.send`, registra `EmailLog` |
| 6 | boot | `src/app/main.py` | importar `app.modules.notification.handlers` no startup para registrar os handlers |
| 7 | config | `src/app/config.py` | `smtp_host`, `smtp_port`, `smtp_username` (SENSITIVE), `smtp_password` (SECRET), `email_sender` (CONFIG) |
| 8 | fronteira | rotas de `resend-ticket` | `POST /orders/{id}/resend-ticket` fica em `payment` (dono do `Order`) e só emite `TicketEmailResendRequested` |

### Código a colar — `EmailService`

```python
class EmailServiceError(Exception):
    pass

class EmailService:
    def __init__(self) -> None:
        self._cfg = get_settings()

    async def send(self, to: str, template: str, context: dict) -> None:
        subject, html = render(template, context)
        message = EmailMessage()
        message["From"] = self._cfg.email_sender
        message["To"] = to
        message["Subject"] = subject
        message.set_content("Seu cliente de e-mail não suporta HTML.")
        message.add_alternative(html, subtype="html")
        try:
            await aiosmtplib.send(
                message, hostname=self._cfg.smtp_host, port=self._cfg.smtp_port,
                username=self._cfg.smtp_username, password=self._cfg.smtp_password,
                start_tls=True, timeout=15,
            )
        except (aiosmtplib.SMTPException, OSError) as exc:
            raise EmailServiceError(f"failed to send {template} to {to}") from exc
```

```python
# handler idempotente
@register("OrderPaid")
async def send_order_confirmation(payload: dict) -> None:
    async with AsyncSessionLocal() as session, session.begin():
        log_repo = EmailLogRepository(session)
        if await log_repo.exists(to=payload["buyer_email"], template="order_confirmed", ref_id=payload["order_id"]):
            return
        await EmailService().send(payload["buyer_email"], "order_confirmed", build_confirmation_context(payload))
        await log_repo.save(EmailLog(to=payload["buyer_email"], template="order_confirmed",
                                    ref_id=payload["order_id"], status="sent"))
```

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-notification-email
commit 1  feat(notification): EmailService SMTP + renderers e templates pt-BR
commit 2  feat(notification): handlers de outbox idempotentes para os 6 eventos
commit 3  feat(notification): EmailLog para idempotência + config SMTP
commit 4  feat(payment): rota resend-ticket emitindo TicketEmailResendRequested
/team-ludens:tbd-pr
```

---

## B. Frontend — responsável: Diego · feature `account`

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.js` | `orders.resendTicket(id)` |
| 2 | services | `src/features/account/services/order.service.js` | `resendTicket(orderId)` |
| 3 | mutations | `.../hooks/mutations/useOrderMutations.js` | `resendTicket` → toast "E-mail reenviado" |
| 4 | components | `.../components/OrderCard.jsx` | botão "Reenviar por e-mail" (pedido pago) |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-account-resend-ticket
commit 1  feat(account): ação de reenviar ingresso por e-mail
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_order_paid_envia_confirmacao_uma_vez` | handler processado 2x com o mesmo `order_id` → um único envio | RF05 |
| `test_falha_smtp_nao_propaga` | `EmailService.send` lança → handler não relança; evento fica para reprocesso | RF05 |
| `test_resend_ticket_reenvia_sempre` | `TicketEmailResendRequested` 3x → 3 envios | RF06 |
| `test_email_sem_dado_sensivel` | corpo do "compra confirmada" não contém CPF nem hash | RNF01 |
| `test_password_reset_link_valido_1h` | contexto do template traz o token e a validade | RF09 |

Roteiro manual (SMTP de teste / MailHog): comprar → confirmação chega com
ingressos em < 5 min; parar o SMTP e comprar → compra vale, e-mail chega quando o
SMTP volta; "reenviar" em Minhas compras → chega de novo; cancelar sessão com 2
compradores → os 2 recebem, sem duplicata.

```text
git checkout master && git pull && git checkout -b test/<NN>-notification-email
git commit -m "test(notification): cobrir idempotência, falha de SMTP e ausência de dado sensível"
```

## D. DevOps — Gabriel

- `.env.example`: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME` (SENSITIVE),
  `SMTP_PASSWORD` (SECRET), `EMAIL_SENDER` (CONFIG). Subir um MailHog no
  `docker-compose.Development.yml` para dev. Configurar o provedor real e os
  secrets no CI/produção.

## E. Ordem entre as fatias

Pode ser feita cedo com stubs dos eventos, mas só fica **útil de ponta a ponta**
depois que `payment-pix-checkout` e `booking-ticket-issuance` mergearem (evento
`OrderPaid` com os dados do ingresso). Recomendação: entrar logo após esses dois.

## F. Bloqueios em aberto

- Provedor SMTP de produção — decisão de DevOps (o código é agnóstico).
- Layout final dos e-mails — N1 usa HTML simples; refino é evolução.
