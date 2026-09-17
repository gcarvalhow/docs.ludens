---
status: alvo
spec: notification-transactional-email
updated_at: 2026-09-11
responsavel: Igor (Backend)
---

# Integration Contract — E-mails transacionais

**Status:** alvo. **Módulo backend:** `notification` (sem aggregate, sem router
próprio). Nenhuma rota nova além de:

| Método | Caminho | Auth | Sucesso | Mora em |
| --- | --- | --- | --- | --- |
| POST | `/orders/{id}/resend-ticket` | Bearer (dono) | 202 | `payment` (emite `TicketEmailResendRequested`) |

## Eventos consumidos (handlers de outbox)

> **Atualizado 2026-09-11:** só a primeira linha tem handler real (ver
> `backend.md`) — é o único evento hoje emitido por código que existe
> (`identity-auth`). As demais linhas continuam descrevendo o desenho
> completo da feature, mas os módulos que emitiriam esses eventos
> (`booking`, `payment`, `catalog` além do que já existe) ainda não foram
> implementados — sem evento real, sem handler ainda.

| Evento (origem) | E-mail | Conteúdo | Handler implementado? |
| --- | --- | --- | --- |
| `PasswordResetRequested` (identity) | Redefinição de senha | link (`/redefinir-senha?token=...`), validade 1h | ✅ sim |
| `OrderPaid` (payment) | Compra confirmada | espetáculo, sessão, local, ingressos (tipo + código/QR) | ❌ não — `payment` não existe |
| `OrderRefunded` (payment) | Reembolso processado | pedido, valor reembolsado | ❌ não — `payment` não existe |
| `SessionCancelled` (catalog) | Sessão cancelada | sessão, orientação sobre o reembolso | ❌ não |
| `SessionRescheduled` (catalog) | Novo horário | sessão, horário antigo e novo | ❌ não |
| `TicketEmailResendRequested` (payment) | Reenvio da confirmação | igual ao "Compra confirmada" | ❌ não — `payment` não existe |

## Contrato do `EmailService`

`send(to: str, subject: str, html_body: str) -> None` — assíncrono; adapter
trocável por configuração (`EMAIL_BACKEND=ses|console`, RNF06); timeout
explícito no adapter SES; lança `EmailServiceError` (nunca a exceção crua do
transporte). Templates em pt-BR, um por tipo, no código — só o de redefinição
de senha existe hoje. Remetente e credenciais de env (credencial AWS nunca em
`.env`, resolvida por IAM role). Ver `backend.md` para o código completo.

## Impacto de UX

Só o toast "e-mail reenviado" após `POST /orders/{id}/resend-ticket` — e esse
toast/rota pertence a `payment`/`booking-ticket-issuance`, não a este módulo.

## Lacunas / decisões em aberto

- ~~Provedor SMTP concreto~~ — decidido nesta rodada: **AWS SES via boto3**,
  não SMTP (ver `backend.md`). `aiosmtplib` removido de `pyproject.toml`.
- Layout dos e-mails (texto puro vs. HTML simples) — N1: HTML simples, sem
  imagens externas. Decidido e implementado para o template de redefinição de
  senha; os demais templates entram junto do módulo que os disparar.
