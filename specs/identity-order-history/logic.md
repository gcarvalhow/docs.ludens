---
status: reviewed
spec: identity-order-history
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# Histórico de compras — Lógica de Negócio

## 1. Fluxo por perfil

### Comprador

1. Abre "Minhas compras". Vê a lista dos próprios pedidos, mais recente primeiro,
   com paginação simples.
2. Cada linha: espetáculo, data/hora da sessão, quantidade, tipo, valor, status.
3. Abre um pedido → vê os ingressos (código/QR, status de cada um) e as ações
   disponíveis:
   - **Reenviar por e-mail** — sempre, para pedido pago.
   - **Cancelar** — só se o pedido está pago e a sessão é futura (a tela chama o
     preview de reembolso de RF07 e mostra a política antes de confirmar).
4. Um pedido pendente ou falho aparece com o status, sem ingressos e sem ações.
5. Tentar abrir um pedido que não é seu → "Pedido não encontrado".

### Admin

Não usa esta tela para ver compras de terceiros. As rotas são escopadas ao
comprador autenticado.

## 2. Estados e transições

Sem máquina de estados própria — reflete o estado do `Order` (de
`payment-pix-checkout` e `payment-cancellation-refund`) e do `Ticket` (de
`booking-ticket-issuance`). Os status exibidos:

| Status exibido | Deriva de |
| --- | --- |
| Pagamento pendente | `Order.status = pending` |
| Confirmado | `Order.status = paid` |
| Reembolso em processamento | `Order.status = refund_processing` |
| Reembolsado | `Order.status = refunded` |
| Cancelado | `Order.status = failed` (pagamento não concluído) |

## 3. Regras de negócio

- A listagem e o detalhe só retornam pedidos cujo `buyer_id` é o do comprador
  autenticado. Nunca um pedido de terceiro, mesmo com o id correto.
- O detalhe inclui os ingressos apenas para pedidos pagos/reembolsados.
- A ação "Cancelar" só é oferecida quando `Order.status = paid` e a sessão é
  futura — a validação final é do backend (RF07).
- A ação "Reenviar por e-mail" só para `Order.status = paid`.
- Ordenação: `Order.created_at` desc.
- Nenhum dado pessoal de terceiro; o próprio CPF/e-mail não precisam aparecer
  aqui (RNF01).

## 4. Pontos de integração

```text
Frontend precisa saber:
  - GET /me/orders -> { items: [OrderSummary], page, size, total }
    OrderSummary: { id, showTitle, sessionStartsAt, quantity, ticketType,
                    amount, status }
  - GET /me/orders/{id} -> OrderDetail (OrderSummary + tickets[] + canCancel: bool)
  - Que canCancel já vem calculado (status=paid e sessão futura); o botão real de
    cancelar chama o fluxo de RF07
  - Que reenviar chama POST /orders/{id}/resend-ticket (RF05/notification)

Backend precisa garantir:
  - Filtro por buyer autenticado em toda rota; 404 para pedido de terceiro
  - status do pedido mapeado para os rótulos de exibição
  - tickets embutidos só quando pago/reembolsado
  - canCancel = status paid AND session.starts_at > now
  - paginação (page, size) com ordenação por created_at desc
```

## 5. Casos de borda

**Comprador tem 200 pedidos.** Paginação simples (page/size); sem scroll infinito
no N1. `[fechada]`

**Pedido cuja sessão foi cancelada pelo teatro.** Aparece como "Reembolsado" (ou
"Reembolso em processamento") — o cancelamento de sessão dispara o reembolso
(RF07/RF08). Os ingressos aparecem "cancelados". `[fechada]`

**Pedido pago mas ingressos ainda não visíveis (janela de ~2 s do relato).** O
`GET /me/orders/{id}` lê o estado real do banco; os ingressos são emitidos na
mesma transação do pagamento aprovado, então se o pedido está "pago" os ingressos
já existem. `[fechada]`

**Comprador abre "Minhas compras" logo após reservar, sem pagar.** Não há pedido
ainda (o pedido nasce no checkout); a reserva aberta não aparece aqui — ela vive
na tela de checkout. `[fechada]`
