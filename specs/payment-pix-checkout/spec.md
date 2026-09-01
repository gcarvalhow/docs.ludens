---
status: approved
domain: payment
created_at: 2026-09-01
approved_at: 2026-09-01
---

# Pagamento Pix e criação do pedido

## 1. Visão da feature

Com a reserva ativa, a pessoa vê o valor total e um QR Code Pix (ou o código
copia-e-cola). Ela paga pelo app do banco. A tela mostra "aguardando
confirmação" e, assim que o pagamento é aprovado, vira a compra: um pedido é
criado no nome dela e os ingressos são emitidos. Se o pagamento falhar, for
cancelado ou a reserva expirar antes de pagar, a reserva é liberada na hora e os
ingressos voltam para a sessão.

## 2. Problema que resolve

Hoje a venda online não tem um meio de pagamento integrado — o dinheiro é
combinado por fora e a confirmação é manual. Sem um pagamento que conclui a
compra sozinho, não há como garantir "encontrou → escolheu → reservou → pagou →
confirmado" sem intervenção do teatro (RF04).

## 3. Para quem é

- **Beneficiário direto:** o `Comprador`, no checkout.
- **Beneficiário indireto:** o teatro, que recebe o pagamento conciliado e não
  precisa confirmar compra manualmente.

## 4. Como melhora a experiência atual

**Antes:** combinar pagamento por fora, enviar comprovante, esperar o teatro
confirmar.

**Depois:** pagar por Pix na hora e receber a confirmação automática em minutos.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** `booking-reservation` (a cobrança nasce de uma
reserva aberta); `identity-auth` (comprador autenticado).

**O que habilita:** `booking-ticket-issuance` (RF05) — os ingressos são emitidos
na confirmação do pagamento; `identity-order-history` (RF06);
`payment-cancellation-refund` (RF07) — o estorno usa o mesmo gateway.

**Posição:** core, N1.

**RF/RN cobertos:** RF04.

## 6. O que não é (escopo negativo)

- **Não inclui** cartão de crédito, boleto ou qualquer meio além de Pix.
- **Não inclui** persistir qualquer dado bancário ou identificador do pagador —
  delegado ao AbacatePay (RNF01).
- **Não inclui** antifraude, split de pagamento, ou repasse a terceiros.
- **Não inclui** parcelamento.
- **Não inclui** reembolso (feature separada, RF07).

## 7. Custos adicionais

**Depende do gateway AbacatePay (Pix)** — já previsto no escopo do produto e nas
premissas (conta AbacatePay disponível). Sem provedor novo além desse. A criação
da cobrança exige uma chamada síncrona ao AbacatePay no checkout.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Meio de pagamento | Pix via AbacatePay, apenas. |
| Criação da cobrança | Síncrona no checkout (a pessoa precisa ver o QR na hora), **depois** de o pedido já estar salvo como "pendente". Falha aqui deixa o pedido pendente com mensagem, sem derrubar a reserva. |
| Confirmação | Via webhook do AbacatePay. Ao confirmar: pedido vira "pago", reserva vira "confirmada", ingressos emitidos — tudo na mesma transação. |
| Falha / cancelamento / expiração | Libera a reserva imediatamente (RF04); ingressos voltam à disponibilidade. |
| Valor | Total = quantidade × preço do tipo de ingresso da sessão (inteira ou meia). |
| Dados persistidos | Só `order` (id, comprador, sessão, quantidade, tipo, total, status) e o id da cobrança no gateway. Nada do pagador. |
| Idempotência do webhook | O mesmo evento do gateway pode chegar mais de uma vez — processar duas vezes não pode emitir ingresso em dobro. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
