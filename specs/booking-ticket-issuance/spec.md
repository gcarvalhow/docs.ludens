---
status: approved
domain: booking
created_at: 2026-09-01
approved_at: 2026-09-01
---

# Emissão do ingresso e confirmação da compra

## 1. Visão da feature

Quando o pagamento é aprovado, a reserva vira compra e o sistema emite os
ingressos: cada um com um identificador único (código / QR) que serve para
validar a entrada no evento. A pessoa vê os ingressos na tela de confirmação e
recebe um e-mail com eles em poucos minutos. Se o e-mail não chegar, os ingressos
continuam disponíveis na área "Minhas compras" — a compra vale do mesmo jeito.

## 2. Problema que resolve

Sem um ingresso com identificador único e validável, o teatro não tem como saber,
na porta, quem realmente comprou — e o comprador não tem prova da compra. Hoje a
"prova" é um comprovante de pagamento informal, que não diz nada sobre sessão nem
sobre lugar.

## 3. Para quem é

- **Beneficiário direto:** o `Comprador`, que recebe algo concreto ("comprei").
- **Beneficiário indireto:** o teatro, que passa a ter um artefato validável por
  sessão — base para a validação na porta (N2).

## 4. Como melhora a experiência atual

**Antes:** comprovante de pagamento genérico, nenhuma ligação clara com a sessão,
nenhuma forma de validar na entrada.

**Depois:** um ingresso por lugar comprado, com código único, na tela e no
e-mail, sempre recuperável na conta.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** `payment-pix-checkout` (a emissão acontece na
confirmação do pagamento); `booking-reservation` (a reserva confirmada é a origem
dos ingressos).

**O que habilita:** `identity-order-history` (RF06) lista esses ingressos; a
validação na porta (N2) usará o código; `payment-cancellation-refund` (RF07)
invalida ingressos ao reembolsar.

**Posição:** core, N1.

**RF/RN cobertos:** RF05 · RN04 (meia-entrada **não** exige número de documento
de estudante).

## 6. O que não é (escopo negativo)

- **Não inclui** a validação do ingresso na entrada (leitura do QR na porta) —
  isso é N2.
- **Não inclui** a validação presencial do documento de estudante para a
  meia-entrada — presencial, fora do escopo (RN04); o sistema só registra a
  intenção de meia.
- **Não inclui** transferência/revenda de ingresso entre pessoas.
- **Não inclui** PDF estilizado do ingresso — no N1, o e-mail traz o código/QR
  e os dados da sessão; um layout mais elaborado é evolução.
- **Não inclui** o envio do e-mail em si — é feito pela feature
  `notification-transactional-email` (handler de evento).

## 7. Custos adicionais

Nenhum custo externo próprio. A geração de QR é local (biblioteca). O envio do
e-mail usa o serviço de e-mail transacional já previsto.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Quantos ingressos | Um `Ticket` por unidade da reserva (`quantity`). |
| Identificador | Código único por ingresso (string opaca não sequencial) + QR derivado do código. |
| Momento da emissão | Na mesma transação que marca o pedido como pago e a reserva como confirmada. |
| Meia-entrada | O ingresso registra o tipo (`half`), **sem** exigir número de documento — RN04. A validação do documento é presencial. |
| Falha de e-mail | Não invalida a compra (RF05); o ingresso fica na área do usuário; dá para reenviar o e-mail de lá (RF06). |
| Invalidação | Um ingresso reembolsado (RF07) é marcado como inválido e o código deixa de valer. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
