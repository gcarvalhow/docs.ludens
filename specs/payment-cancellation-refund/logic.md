---
status: reviewed
spec: payment-cancellation-refund
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# Cancelamento e reembolso — Lógica de Negócio

## 1. Fluxo por perfil

### Comprador

1. Em "Minhas compras", num pedido confirmado, pede cancelamento.
2. O sistema calcula a janela: horas entre agora e o início da sessão.
   - ≥ 48h → reembolso 100%.
   - 48h a 24h → reembolso 50%.
   - < 24h → **solicitação bloqueada** com mensagem ("Cancelamentos com menos de
     24h da sessão não têm reembolso e não podem ser feitos pelo site — procure
     o teatro").
3. Confirma. O pedido vai para "reembolso em processamento"; os ingressos ficam
   "cancelados" na hora; a disponibilidade da sessão volta na hora.
4. O estorno é iniciado no gateway. Quando confirma, o pedido vira "reembolsado"
   e a pessoa é avisada.

### Admin

1. Cancela uma sessão (RF08). Para cada pedido confirmado dessa sessão, o sistema
   inicia reembolso **de 100%** (a política de janela não se aplica — a causa é o
   teatro), invalida os ingressos e libera a disponibilidade (que deixa de
   importar, sessão cancelada).
2. Os compradores são notificados (feature de notificação).

## 2. Estados e transições

### Pedido (Order) — continuação de payment-pix-checkout

**Estados relevantes:** pago · reembolso-solicitado · reembolsado.

- **pago → reembolso-solicitado:** comprador cancela dentro da janela, ou admin
  cancela a sessão.
- **reembolso-solicitado → reembolsado:** o estorno é confirmado (webhook/consulta).
- `reembolsado` é terminal.
- Um pedido `pago` cuja sessão já passou não pode mais ser cancelado pelo
  comprador.

### Ingresso

`válido → inválido` no momento da solicitação (não espera o estorno).

## 3. Regras de negócio

- Só pedido **pago** e de sessão **futura** pode ser cancelado pelo comprador.
- Valor do reembolso (RN02), horas = (início da sessão − agora):
  - horas ≥ 48 → `total`
  - 24 ≤ horas < 48 → `total * 0,5`
  - horas < 24 → **bloqueia a solicitação** (não cria estorno de 0).
- O cálculo é feito no domínio do `Order`, sem tocar gateway nem outro aggregate.
- Ingressos do pedido são invalidados e a disponibilidade da sessão é devolvida
  **na mesma transação** que registra a solicitação de reembolso.
- O estorno no gateway é um **efeito de outbox** (evento `OrderRefundRequested`),
  idempotente — reprocessar não estorna duas vezes.
- Cancelamento por sessão (admin): reembolso de 100% sempre, um `OrderRefundRequested`
  por pedido, disparados pelo handler de `SessionCancelled`.
- Nenhum dado do pagador é usado — o estorno referencia o `gateway_charge_id`.

## 4. Pontos de integração

```text
Frontend precisa saber:
  - Antes de confirmar, pode chamar um "preview": { refundAmount, policyLabel }
    ("Reembolso integral" / "Reembolso de 50%" / "Sem reembolso — não permitido")
  - Que < 24h retorna 409 com a mensagem — mostrar como aviso, não erro técnico
  - Que após confirmar o pedido fica "reembolso em processamento" e depois
    "reembolsado" (polling do pedido, RF06)

Backend precisa garantir:
  - Cálculo RN02 no domínio do Order; bloqueio de < 24h antes de qualquer efeito
  - invalidação de tickets + devolução de disponibilidade na mesma transação que
    marca reembolso-solicitado
  - OrderRefundRequested no outbox; handler chama PaymentGateway.refund(charge_id,
    amount) de forma idempotente
  - SessionCancelled -> um OrderRefundRequested de 100% por pedido confirmado
  - Ao confirmar o estorno, Order -> reembolsado; evento OrderRefunded (e-mail)
```

## 5. Casos de borda

**Comprador cancela exatamente na marca das 48h.** `horas >= 48` → 100%. A
borda pertence à faixa de cima. `[fechada]`

**Estorno recusado pelo gateway.** O pedido fica "reembolso em processamento"; o
handler é reprocessado pelo relay (at-least-once). Se persistir, é um item para o
teatro tratar manualmente — o pedido não volta a "pago" (os ingressos já foram
invalidados). Registrar log/alerta. `[fechada]`

**Admin cancela sessão com pedido cujo comprador já tinha pedido reembolso de
50%.** O pedido já está "reembolso-solicitado"; o cancelamento da sessão não
sobrepõe para 100% um estorno já em andamento. Decisão: se ainda não foi ao
gateway, ajusta para 100%; se já foi, o teatro complementa manualmente.
`[fechada]`

**Sessão começa enquanto o comprador está na tela de cancelamento.** A
confirmação revalida "sessão futura"; se já começou, recusa com "a sessão já
começou". `[fechada]`
