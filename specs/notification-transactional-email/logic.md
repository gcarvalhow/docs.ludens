---
status: reviewed
spec: notification-transactional-email
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# E-mails transacionais — Lógica de Negócio

## 1. Fluxo por perfil

Não há interação de usuário com esta feature — ela reage a eventos. O que o
usuário percebe:

- **Comprador:** recebe o e-mail de confirmação com os ingressos poucos minutos
  após pagar; o link de redefinição de senha quando solicita; um aviso quando o
  reembolso é processado; um aviso quando o teatro cancela ou remarca uma sessão
  que ele comprou. Pode pedir o reenvio da confirmação em "Minhas compras".
- **Admin:** ao cancelar/remarcar uma sessão vendida, a interface avisa "os
  compradores serão notificados por e-mail".

## 2. Estados e transições

### Envio de e-mail (registro opcional)

**Estados:** pendente · enviado · falho.

- **inexistente → pendente:** um handler de evento decide enviar.
- **pendente → enviado:** o `EmailService` retorna sucesso.
- **pendente → falho:** o `EmailService` lança erro → o relay reprocessa o
  evento na próxima volta (o handler é idempotente).

Não há estado no fluxo de negócio principal — a origem (pedido, token, sessão)
segue seu curso independentemente.

## 3. Regras de negócio

- Cada tipo de e-mail é disparado por exatamente um evento de domínio:
  - `OrderPaid` → "compra confirmada" (dados da sessão + lista de ingressos com
    código/QR).
  - `PasswordResetRequested` → "redefinição de senha" (link, validade 1h).
  - `OrderRefunded` → "reembolso processado" (valor, pedido).
  - `SessionCancelled` → "sessão cancelada" (para cada comprador da sessão).
  - `SessionRescheduled` → "novo horário da sessão".
  - `TicketEmailResendRequested` → reenvia "compra confirmada".
- Todo handler é **idempotente**: reenviar o mesmo e-mail para o mesmo
  (destinatário, tipo, referência) não deve gerar dois envios — checar um marcador
  antes de enviar (ou aceitar o reenvio explícito no caso de
  `TicketEmailResendRequested`).
- Falha de envio **nunca** propaga para desfazer a transação de origem — o
  handler roda fora dela, e o relay é at-least-once (RF05).
- O e-mail não carrega dado sensível além do necessário: sem CPF, sem hash, sem
  dado de pagamento (RNF01). O código do ingresso é aceitável (é o artefato da
  compra).
- Prazo alvo: confirmação entregue em ≤ 5 min após `OrderPaid` (RF05).

## 4. Pontos de integração

```text
Frontend precisa saber:
  - Nada de novo além de: a ação "reenviar ingresso" em Minhas compras chama
    POST /orders/{id}/resend-ticket e mostra um toast "e-mail reenviado"

Backend precisa garantir:
  - Handlers registrados no outbox para os eventos listados
  - EmailService (SMTP async) com timeout, exceção própria (EmailServiceError),
    credenciais só de env, remetente configurável
  - Idempotência por (destinatário, tipo, referência); marcador persistido ou
    consulta ao estado de origem
  - Nenhuma exceção do EmailService sobe além do handler
  - Templates em pt-BR, um por tipo, versionados no código
```

## 5. Casos de borda

**SMTP fora do ar por 20 minutos.** Os eventos ficam com `dispatched_at NULL`; o
relay reprocessa quando o SMTP volta. A confirmação pode passar dos 5 min alvo —
aceitável e transparente (o ingresso já está em "Minhas compras"). `[fechada]`

**Sessão cancelada com 300 compradores.** `SessionCancelled` é um evento; o
handler itera os compradores e envia um e-mail por pessoa, com marcador de
idempotência por (comprador, "sessão cancelada", sessão). Se o relay reprocessar,
ninguém recebe duas vezes. `[fechada]`

**Comprador pede reenvio 3 vezes.** `TicketEmailResendRequested` é o único tipo
em que o reenvio é intencional — envia as 3 vezes (sem dano; é o mesmo
conteúdo). `[fechada]`

**E-mail de redefinição para conta que não existe.** Não chega a gerar evento — o
usecase de "esqueci a senha" só emite `PasswordResetRequested` quando a conta
existe; a resposta ao usuário é neutra de qualquer forma (identity-auth).
`[fechada]`
