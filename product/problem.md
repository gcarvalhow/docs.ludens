# Definição do Problema

> **Responsável:** Product Owner (Gabriel Carvalho) · **Última revisão:** 2026-08-28 · **Status:** vigente

## Contexto

Um grupo de teatro comunitário e independente produz peças e apresenta sessões
para o público.
Hoje a venda de ingressos acontece em dois canais desconectados: a bilheteria
física do teatro e vendas informais pela internet.
Não existe um sistema que seja a fonte única da disponibilidade de assentos de
cada sessão.

## O problema

Como os dois canais não compartilham o estado de disponibilidade, **o mesmo
assento pode ser vendido duas vezes** — uma na bilheteria física e outra pela
internet, para a mesma sessão.
O resultado é overbooking, público sem lugar na porta do evento, retrabalho
manual de conferência e perda de confiança na venda online.

Sem uma plataforma dedicada, o teatro também não tem:

- **Disponibilidade confiável em tempo real** — não há como saber, num dado
  instante, quantos ingressos de uma sessão ainda podem ser vendidos
- **Reserva com garantia** — o comprador não consegue segurar os ingressos pelo
  tempo necessário para concluir o pagamento sem risco de perdê-los
- **Registro estruturado das vendas** — pedidos, cancelamentos e reembolsos
  ficam espalhados e não são consultáveis de forma consistente
- **Base para relatórios de ocupação** — medir a taxa de ocupação de sala por
  sessão depende de contagem manual

## Critério de sucesso

- **Fonte única de disponibilidade**: a plataforma é o único lugar que define
  quantos ingressos de uma sessão estão disponíveis; a bilheteria física passa a
  operar sobre ela
- **Sem venda em duplicidade**: duas compras concorrentes nunca conseguem,
  juntas, exceder a capacidade da sessão, mesmo sob acesso simultâneo
- **Compra completa pela internet**: um comprador encontra o espetáculo,
  seleciona a sessão e o tipo de ingresso (inteira/meia), reserva, paga e recebe
  a confirmação sem intervenção manual do teatro
- **Reserva temporária confiável**: os ingressos ficam bloqueados durante o
  checkout e voltam automaticamente à disponibilidade se o pagamento não for
  concluído no prazo
- **Rastreabilidade**: todo pedido tem status consultável (confirmado, cancelado,
  reembolsado) e um ingresso com identificador único validável na entrada
- **Ocupação mensurável**: a taxa de ocupação de sala por sessão pode ser
  extraída da plataforma, sem contagem manual (evolução N2/N3)

## Fora do alcance do problema

A validação presencial de meia-entrada (conferência do documento de estudante na
porta) e a operação de caixa da bilheteria física permanecem processos manuais
do teatro. A plataforma registra a intenção de compra de meia-entrada, mas não
substitui a conferência no local — ver
[RN04](../requirements/business-rules.md#rn04--meia-entrada).
