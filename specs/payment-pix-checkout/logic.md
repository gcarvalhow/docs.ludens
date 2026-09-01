---
status: reviewed
spec: payment-pix-checkout
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# Pagamento Pix e criação do pedido — Lógica de Negócio

## 1. Fluxo por perfil

### Comprador

1. No checkout (reserva aberta), pede para pagar.
2. O sistema cria um pedido "pendente" ligado à reserva, chama o gateway e
   devolve o QR Code Pix + código copia-e-cola + o valor total.
3. A tela mostra "aguardando confirmação" e consulta o status periodicamente.
4. **Pagamento aprovado** (webhook): pedido → "pago", reserva → "confirmada",
   ingressos emitidos; a tela leva para a confirmação com os ingressos (RF05).
5. **Pagamento não aprovado / cancelado / reserva expira antes**: pedido →
   "falho", reserva → "cancelada", ingressos voltam; a tela informa e oferece
   reservar de novo.
6. Se a criação da cobrança falhar (gateway fora do ar): o pedido fica
   "pendente" com uma mensagem "não foi possível iniciar o pagamento, tente de
   novo"; a reserva continua valendo até expirar.

### Admin

Não participa do pagamento. Vê o resultado no histórico/estado do pedido.

## 2. Estados e transições

### Pedido (Order)

**Estados:** pendente · pago · falho · reembolsado.

- **inexistente → pendente:** comprador inicia o pagamento.
- **pendente → pago:** webhook de aprovação (idempotente).
- **pendente → falho:** webhook de recusa/cancelamento, ou a reserva vinculada
  expira/é cancelada antes do pagamento.
- **pago → reembolsado:** fluxo de cancelamento e reembolso (RF07).
- `pago`, `falho`, `reembolsado` (após concluído) são terminais para este fluxo.

### Cobrança no gateway (referência)

Criada · aprovada · expirada/recusada. O sistema não modela isso como aggregate
próprio — guarda só o `gateway_charge_id` no pedido e reage ao webhook.

## 3. Regras de negócio

- Um pedido nasce sempre de uma **reserva aberta** do próprio comprador; a
  reserva é travada/relida ao criar o pedido para não pagar uma reserva expirada.
- Total = `quantidade × preço(tipo, sessão)`. Meia = 50% do inteira (RN04).
- A cobrança no gateway é criada **depois** de o pedido "pendente" estar salvo,
  numa chamada síncrona; falha da chamada não reverte o pedido nem a reserva —
  vira mensagem.
- O webhook é **idempotente**: se o pedido já está "pago", reprocessar o mesmo
  evento não faz nada (não emite ingresso de novo, não confirma reserva de novo).
- Nenhum dado do pagador é persistido nem logado (RNF01).
- Aprovação que chega para uma reserva que já não está "aberta" (expirou no
  intervalo) → o pagamento é tratado como pago-indevido: pedido "pago" mas
  imediatamente encaminhado ao estorno automático (RF04/RF07), e a tela informa.
- Falha/recusa → `ReservationCancelled` disparado (libera os ingressos na hora).

## 4. Pontos de integração

```text
Frontend precisa saber:
  - A resposta de "iniciar pagamento": { orderId, status, pixQrCode (imagem/base64
    ou url), pixCopyPaste, amount, expiresAt }
  - Que deve fazer polling de GET do pedido/reserva até status sair de "pendente"
  - Que "pago" leva à confirmação (RF05); "falho" volta à sessão com CTA de
    reservar de novo; erro de criação da cobrança mostra "tente de novo"
  - Nunca renderizar/guardar dado do pagador

Backend precisa garantir:
  - Criar o pedido "pendente" e SÓ ENTÃO chamar o gateway (síncrono), com timeout
  - Webhook idempotente (dedupe por gateway_charge_id + estado do pedido)
  - Ao aprovar: order.mark_paid() + reservation.confirm() + emissão de tickets na
    MESMA transação; evento OrderPaid para o e-mail (outbox)
  - Ao recusar/expirar: order.mark_failed() + reservation.expire()/cancel()
  - Verificar assinatura do webhook (ABACATEPAY_WEBHOOK_SECRET)
  - Nenhum dado de pagador persistido; segredos só em env
```

## 5. Casos de borda

**Webhook chega duas vezes.** Dedupe por `gateway_charge_id`: o segundo é
ignorado se o pedido já está no estado final correspondente. `[fechada]`

**Reserva expira 1 s antes do webhook de aprovação.** Pedido é marcado "pago"
(o dinheiro entrou) e imediatamente entra na fila de estorno automático; a
pessoa é avisada de que a reserva expirou e o valor será devolvido. `[fechada]`

**Gateway aprova, mas a emissão de ingresso falha na mesma transação.** A
transação inteira faz rollback: o pedido não fica "pago" sem ingresso. O webhook
será reentregue pelo gateway (ou o relcrédito de status por polling reprocessa).
`[fechada]`

**Comprador fecha a aba após pagar.** O webhook confirma no backend de qualquer
jeito; ao voltar, "Minhas compras" (RF06) já mostra o pedido pago com os
ingressos. `[fechada]`

**Chamada de criação da cobrança demora além do timeout.** O pedido fica
"pendente"; a tela mostra "não foi possível iniciar o pagamento"; a pessoa tenta
de novo (novo `gateway_charge_id`), a reserva ainda vale. `[fechada]`
