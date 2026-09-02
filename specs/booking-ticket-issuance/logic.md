---
status: reviewed
spec: booking-ticket-issuance
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# Emissão do ingresso — Lógica de Negócio

## 1. Fluxo por perfil

### Comprador

1. Pagamento aprovado → a tela de confirmação mostra: espetáculo, sessão,
   quantidade, tipo, e a lista de ingressos, cada um com o código e o QR.
2. Em até 5 minutos (RF05), chega um e-mail com os mesmos dados.
3. Se o e-mail não chegar: os ingressos estão em "Minhas compras" (RF06), com um
   botão "reenviar por e-mail".
4. Um ingresso reembolsado aparece marcado como "cancelado" e o QR não vale mais.

### Admin

Não emite ingressos manualmente. Vê a quantidade vendida por sessão (RF08).

## 2. Estados e transições

### Ingresso (Ticket)

**Estados:** válido · inválido (cancelado/reembolsado).

- **inexistente → válido:** emitido na confirmação do pagamento.
- **válido → inválido:** o pedido correspondente é reembolsado (RF07), ou a
  sessão é cancelada (RF08 → RF07).
- Não há re-emissão: reenviar o e-mail não gera código novo, reenvia o mesmo.

## 3. Regras de negócio

- Um `Ticket` por unidade de `quantity` da reserva confirmada.
- O código é único no sistema, opaco e não sequencial (não permite adivinhar
  outro código a partir do seu).
- Emissão acontece **na mesma transação** que `order.mark_paid()` e
  `reservation.confirm()` — se a emissão falhar, nada é confirmado (rollback).
- O `Ticket` guarda `type` (`full`/`half`). **Não guarda nem exige número de
  documento de estudante** (RN04).
- Preço registrado no ingresso = preço unitário do tipo na sessão no momento da
  compra (histórico, não muda se o admin editar a sessão depois).
- Ingresso inválido: o código deixa de ser aceito (importante para a validação
  na porta, N2).
- O e-mail de confirmação é disparado por evento (`OrderPaid`) e processado fora
  da transação; falha de envio não afeta o `Ticket` nem o `Order` (RF05).

## 4. Pontos de integração

```text
Frontend precisa saber:
  - O shape do Ticket na resposta do pedido: { id, type, code, qrData, status }
    (qrData = string a renderizar como QR, pode ser o próprio code)
  - Que a tela de confirmação e "Minhas compras" (RF06) usam o mesmo shape
  - Que "reenviar e-mail" é uma ação do pedido, não do ingresso

Backend precisa garantir:
  - Emissão de N tickets na MESMA transação do pagamento aprovado
  - code único (constraint no banco) e opaco
  - Ticket.session_id preenchido (a contagem de disponibilidade de catalog usa
    isso — ver catalog-session-detail)
  - Expor em booking/dependencies.py:
      confirm_reservation(session, reservation_id) -> None
      issue_tickets_for_reservation(session, reservation_id) -> list[TicketRef]
    para o webhook de payment chamar dentro da transação dele
  - Ao reembolsar (evento vindo de payment), marcar os tickets do pedido como
    inválidos e devolver disponibilidade
```

## 5. Casos de borda

**Reserva confirmada mas sessão foi cancelada no intervalo.** Não deveria
acontecer (cancelar sessão cancela reservas abertas e reembolsa pedidos), mas se
acontecer, os tickets emitidos entram no fluxo de invalidação + reembolso.
`[fechada]`

**Colisão de código.** A constraint de unicidade no banco recusa; o usecase
gera outro e tenta de novo (até um limite pequeno de tentativas). `[fechada]`

**Reenvio de e-mail solicitado 5 vezes.** Cada clique enfileira um evento de
reenvio; o handler é idempotente por (pedido, tipo de e-mail) na janela — ou
simplesmente reenvia o mesmo conteúdo (não há dano). Decisão: reenvia, sem
limite rígido no N1. `[fechada]`

**Meia-entrada sem documento.** É o comportamento correto: o ingresso é emitido
com `type=half` e nenhum campo de documento. A checagem do documento é na porta,
manual. `[fechada]`
