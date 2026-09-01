---
status: approved
domain: booking
created_at: 2026-09-01
approved_at: 2026-09-01
---

# Reserva temporária de ingressos

## 1. Visão da feature

Na página de uma sessão, a pessoa escolhe quantos ingressos quer e de que tipo
(inteira ou meia) e clica em reservar. O sistema segura esses ingressos para ela
por 15 minutos — tempo de concluir o pagamento — e mostra um contador regressivo.
Se ela pagar, a reserva vira compra. Se o tempo acabar ou ela desistir, os
ingressos voltam para a sessão automaticamente e ficam disponíveis para outra
pessoa.

Enquanto a reserva está ativa, aqueles lugares não aparecem como disponíveis para
mais ninguém — é o que garante que o mesmo assento nunca seja vendido duas vezes.

## 2. Problema que resolve

Sem reserva temporária, duas pessoas conseguem comprar o último ingresso ao mesmo
tempo (overbooking), ou a pessoa perde o lugar no meio do pagamento porque
alguém mais rápido comprou. É o problema central do Ludens: hoje não há um lugar
único que segure o assento durante a compra.

## 3. Para quem é

- **Beneficiário direto:** o `Comprador` autenticado, no momento do checkout.
- **Beneficiário indireto:** o teatro, que deixa de ter overbooking e clientes
  sem lugar na porta.

## 4. Como melhora a experiência atual

**Antes:** o lugar só é "seu" depois de pago — e pode sumir enquanto você digita
o Pix. Nenhuma garantia.

**Depois:** ao reservar, o lugar é bloqueado na hora, com um contador claro do
tempo que resta. Pagou, é seu. Não pagou, volta para a sessão sem intervenção de
ninguém.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** `identity-auth` (comprador autenticado, CPF);
`catalog-session-detail` (a sessão, sua capacidade e se está à venda).

**O que habilita:** `payment-pix-checkout` (RF04) — a cobrança nasce de uma
reserva; `booking-ticket-issuance` (RF05) — os ingressos são emitidos quando a
reserva é confirmada.

**Posição no produto:** core, N1. É a peça que sustenta o critério de sucesso
"nunca vender o mesmo assento duas vezes".

**RF/RN cobertos:** RF03 · RN01 (limite por CPF), RN03 (expiração 15 min),
RN05 (consistência atômica de disponibilidade).

## 6. O que não é (escopo negativo)

- **Não inclui** escolha de assento específico no mapa — o MVP reserva por
  quantidade dentro dos setores A/B/C, sem numeração de poltrona.
- **Não inclui** reserva por visitante não autenticado — precisa de conta (CPF
  para RN01).
- **Não inclui** lista de espera para sessão esgotada.
- **Não inclui** prorrogação do prazo de 15 min pelo comprador.
- **Não inclui** o pagamento em si nem a emissão do ingresso — são features
  separadas que consomem a reserva.
- **Não inclui** reserva de mais de uma sessão numa mesma transação (um checkout
  = uma sessão).

## 7. Custos adicionais

Nenhum custo externo identificado. Opera inteiramente sobre o PostgreSQL.

## 8. Decisões tomadas

| Ponto | Decisão |
|---|---|
| Unidade de reserva | Quantidade + tipo (inteira/meia), sem numeração de poltrona no N1. |
| Prazo | 15 minutos (RN03), configurável por variável de ambiente. |
| Expiração | Automática: um processo em background devolve os ingressos das reservas vencidas. O DB é a fonte de verdade — sobrevive a restart da aplicação. |
| Limite por CPF | 6 ingressos por sessão por CPF (RN01), contando ingressos já confirmados + reservas abertas do mesmo CPF na mesma sessão. |
| Concorrência | A linha da sessão é travada (`SELECT ... FOR UPDATE`) antes de recontar disponibilidade — duas reservas simultâneas nunca excedem a capacidade (RN05). |
| Disponibilidade exibida | Capacidade − ingressos confirmados − reservas abertas não vencidas. |
| Desistir | O comprador pode cancelar a própria reserva aberta antes de pagar; os ingressos voltam na hora. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
