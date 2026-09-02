---
status: approved
domain: payment
created_at: 2026-09-01
approved_at: 2026-09-01
---

# Cancelamento e reembolso

## 1. Visão da feature

Em "Minhas compras", a pessoa pode pedir o cancelamento de um pedido confirmado.
O sistema aplica sozinho a política de reembolso conforme o tempo que falta para
a sessão: 48h ou mais antes, devolve tudo; entre 48h e 24h, devolve metade;
menos de 24h, não devolve. O que der para reembolsar é estornado pelo mesmo
gateway do pagamento. Os ingressos do pedido são invalidados e voltam para a
disponibilidade da sessão.

## 2. Problema que resolve

Sem uma política automática, cada cancelamento vira uma negociação manual com o
teatro, sem consistência e sem rastro. E sem devolver os ingressos à sessão, um
lugar cancelado fica perdido.

## 3. Para quem é

- **Beneficiário direto:** o `Comprador` que precisa desistir.
- **Beneficiário indireto:** o teatro, que revende o lugar e não gasta tempo
  arbitrando reembolso.

## 4. Como melhora a experiência atual

**Antes:** ligar/escrever para o teatro, negociar valor, esperar.

**Depois:** um clique, a regra é aplicada na hora, o estorno é iniciado, o lugar
volta a ser vendido.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** `payment-pix-checkout` (o pedido pago e o
`gateway_charge_id`); `booking-ticket-issuance` (invalidação dos ingressos);
`identity-order-history` (RF06) é onde a ação vive na interface.

**O que habilita:** fecha o ciclo de pós-venda; é acionada também pelo
cancelamento de sessão pelo admin (RF08).

**Posição:** core, N1.

**RF/RN cobertos:** RF07 · RN02 (política de reembolso por janela de tempo).

## 6. O que não é (escopo negativo)

- **Não inclui** reembolso parcial por escolha do comprador (ex.: cancelar só 1
  de 3 ingressos) — no N1, cancela o pedido inteiro.
- **Não inclui** crédito/voucher para uso futuro em vez de estorno.
- **Não inclui** aprovação manual do teatro — a política é automática.
- **Não inclui** reembolso de pedido não pago (esse é só liberação de reserva,
  RF04).
- **Não inclui** taxa de cancelamento além do que a RN02 define.

## 7. Custos adicionais

Usa o **estorno do AbacatePay** — mesmo gateway do pagamento, sem provedor novo.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Política | RN02: ≥ 48h antes da sessão → 100%; entre 48h e 24h → 50%; < 24h → 0%. Calculada a partir de `agora` no momento da solicitação. |
| Fora de janela útil (< 24h, 0%) | A solicitação é **bloqueada** com mensagem explicativa (não gera pedido de estorno de R$ 0). |
| Escopo do cancelamento | Pedido inteiro (todos os ingressos). |
| Estorno | Iniciado automaticamente no gateway pelo valor calculado; o resultado do estorno chega por webhook/consulta. |
| Ingressos | Marcados como inválidos na hora; disponibilidade da sessão devolvida na hora (não espera o estorno concluir). |
| Cancelamento pela sessão (admin) | O admin cancela a sessão → todos os pedidos confirmados são reembolsados **como se fosse ≥ 48h** (100%), independentemente da data — a falha é do teatro, não do comprador. |
| Estado final | Pedido → "reembolsado" quando o estorno é confirmado; enquanto isso, "reembolso em processamento". |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
