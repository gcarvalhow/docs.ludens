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

> **Atualizado 2026-09-17:** quatro eventos têm handler real hoje (ver
> `backend.md`) — os que já são emitidos por código que existe (`identity`).
> As demais linhas continuam descrevendo o desenho completo da feature, mas
> os módulos que emitiriam esses eventos (`booking`, `payment`, `catalog`
> além do que já existe) ainda não foram implementados — sem evento real,
> sem handler ainda.

| Evento (origem) | E-mail | Conteúdo | Handler implementado? |
| --- | --- | --- | --- |
| `PasswordResetRequested` (identity) | Redefinição de senha | link (`/redefinir-senha?token=...`), validade 1h | ✅ sim |
| `EmailChangeRequested` (identity) | Confirmação de troca de e-mail | link (`/confirmar-troca-de-email?token=...`), validade 1h, vai pro e-mail **atual** | ✅ sim |
| `EmailChanged` (identity) | Aviso de cortesia pós-troca | sem link, só informativo, vai pro e-mail **novo** | ✅ sim |
| `AccountDeletionRequested` (identity) | Confirmação de exclusão de conta | link (`/confirmar-exclusao-de-conta?token=...`), validade 1h | ✅ sim |
| `OrderPaid` (payment) | Compra confirmada | espetáculo, sessão, local, ingressos (tipo + código/QR) | ❌ não — `payment` não existe |
| `OrderRefunded` (payment) | Reembolso processado | pedido, valor reembolsado | ❌ não — `payment` não existe |
| `SessionCancelled` (catalog) | Sessão cancelada | sessão, orientação sobre o reembolso | ❌ não |
| `SessionRescheduled` (catalog) | Novo horário | sessão, horário antigo e novo | ❌ não |
| `TicketEmailResendRequested` (payment) | Reenvio da confirmação | igual ao "Compra confirmada" | ❌ não — `payment` não existe |

## Contrato do `EmailService`

`send(to: str, subject: str, html_body: str) -> None` — assíncrono; adapter
trocável por configuração (`EMAIL_BACKEND=acs|smtp`, RNF06); lança
`EmailServiceError` (nunca a exceção crua do transporte). Templates em
pt-BR, um por tipo, no código — quatro tipos existem hoje (redefinição de
senha, confirmação de troca de e-mail, aviso de cortesia pós-troca,
confirmação de exclusão de conta). Remetente de env; connection string do
ACS é `SECRET`, nunca em arquivo versionado. Ver `backend.md` para o código
completo.

## Impacto de UX

Só o toast "e-mail reenviado" após `POST /orders/{id}/resend-ticket` — e esse
toast/rota pertence a `payment`/`booking-ticket-issuance`, não a este módulo.

## Lacunas / decisões em aberto

- ~~Provedor de produção~~ — decidido 2026-09-11 (AWS SES), revisto
  2026-09-17 pra **Azure Communication Services** (ver `backend.md`).
  ~~Transporte de dev~~ — `aiosmtplib` (que tinha saído de `pyproject.toml`
  na decisão de 2026-09-11) voltou: é o transporte real do adapter de
  desenvolvimento (`SmtpEmailService`, contra um Mailpit em Docker).
- Layout dos e-mails (texto puro vs. HTML simples) — N1: HTML simples, sem
  imagens externas. Decidido e implementado para os quatro templates que já
  existem (redefinição de senha, troca de e-mail, aviso de cortesia,
  exclusão de conta); os demais entram junto do módulo que os disparar.
- Páginas de frontend que os links de confirmação abrem
  (`/confirmar-troca-de-email`, `/confirmar-exclusao-de-conta`) ainda não
  existem em `web.ludens` — ver `docs.ludens/team/overview.md`.
