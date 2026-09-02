---
status: alvo
spec: notification-transactional-email
updated_at: 2026-09-01
responsavel: Igor (Backend)
---

# Integration Contract — E-mails transacionais

**Status:** alvo. **Módulo backend:** `notification` (sem aggregate, sem router
próprio). Nenhuma rota nova além de:

| Método | Caminho | Auth | Sucesso | Mora em |
| --- | --- | --- | --- | --- |
| POST | `/orders/{id}/resend-ticket` | Bearer (dono) | 202 | `payment` (emite `TicketEmailResendRequested`) |

## Eventos consumidos (handlers de outbox)

| Evento (origem) | E-mail | Conteúdo |
| --- | --- | --- |
| `OrderPaid` (payment) | Compra confirmada | espetáculo, sessão, local, ingressos (tipo + código/QR) |
| `PasswordResetRequested` (identity) | Redefinição de senha | link (`/redefinir-senha?token=...`), validade 1h |
| `OrderRefunded` (payment) | Reembolso processado | pedido, valor reembolsado |
| `SessionCancelled` (catalog) | Sessão cancelada | sessão, orientação sobre o reembolso |
| `SessionRescheduled` (catalog) | Novo horário | sessão, horário antigo e novo |
| `TicketEmailResendRequested` (payment) | Reenvio da confirmação | igual ao "Compra confirmada" |

## Contrato do `EmailService`

`send(to: str, template: str, context: dict) -> None` — SMTP async; timeout;
lança `EmailServiceError` (capturado no handler; nunca sobe). Templates em pt-BR,
um por tipo, no código. Remetente e credenciais de env.

## Impacto de UX

Só o toast "e-mail reenviado" após `POST /orders/{id}/resend-ticket`.

## Lacunas / decisões em aberto

- Provedor SMTP concreto (host/porta/credenciais) — decisão de DevOps; o código
  fala SMTP genérico.
- Layout dos e-mails (texto puro vs. HTML simples) — N1: HTML simples, sem
  imagens externas.
