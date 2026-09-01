---
status: alvo
spec: booking-reservation
updated_at: 2026-09-01
responsavel: Igor (Backend)
---

# Integration Contract — Reserva temporária de ingressos

**Status:** alvo. **Módulo backend:** `booking`.

## Rotas

| Método | Caminho | Auth | Sucesso | Descrição |
|---|---|---|---|---|
| POST | `/reservations` | Bearer (comprador) | 201 | Abre uma reserva |
| GET | `/reservations/{id}` | Bearer (dono) | 200 | Estado da reserva + tempo restante |
| POST | `/reservations/{id}/cancel` | Bearer (dono) | 204 | Cancela reserva aberta |

## Request / Response

- `POST /reservations` → `{ sessionId, quantity (1..6), ticketType: "full"|"half" }`
  → 201 `{ id, sessionId, quantity, ticketType, status: "open", expiresAt }`.
- `GET /reservations/{id}` → 200
  `{ id, sessionId, quantity, ticketType, status, expiresAt, secondsLeft }`
  (`secondsLeft` calculado no servidor; ≤ 0 quando expirando/expirada).
- `POST /reservations/{id}/cancel` → 204.

## Erros esperados

| Status | Quando | Mensagem |
|---|---|---|
| 409 | disponibilidade insuficiente na sessão | "Não há ingressos suficientes para esta sessão." |
| 409 | limite por CPF excedido (RN01) | "Você já atingiu o limite de 6 ingressos por sessão." |
| 409 | sessão não está à venda | "Esta sessão não está mais à venda." |
| 404 | reserva inexistente ou de outro comprador | "Reserva não encontrada." |
| 409 | cancelar reserva que não está "open" | "Esta reserva não pode mais ser cancelada." |

## Estados de negócio

`open → confirmed` (pagamento aprovado) · `open → expired` (15 min) ·
`open → cancelled` (comprador, falha de pagamento, ou sessão cancelada). Só
`open` segura disponibilidade.

## Impacto de UX

Na criação, guardar `expiresAt` e rodar contador local; a cada X s (ou ao zerar)
chamar `GET /reservations/{id}` para revalidar. Ao ver `status != open`, mostrar
a mensagem final correspondente (expirou / cancelada / já paga). Recusa 409 na
criação é fluxo normal — voltar à sessão com a mensagem, não tela de erro.

## Semântica de disponibilidade

`GET` da sessão (feature `catalog-session-detail`) devolve
`availableCount = capacity − confirmados − reservas_abertas_não_vencidas`. Pode
estar até ~alguns segundos desatualizado; a verdade é decidida na criação da
reserva, sob trava.

## Lacunas / decisões em aberto

- Prefixo/base path, versionamento, envelope de erro — `<a definir globalmente>`.
