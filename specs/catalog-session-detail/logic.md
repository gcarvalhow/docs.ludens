---
status: reviewed
spec: catalog-session-detail
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# Detalhe da sessão — Lógica de Negócio

## 1. Fluxo por perfil

### Visitante

1. Abre `/sessoes/{sessionId}`. Vê espetáculo, data, horário, local, tipos de
   ingresso (inteira/meia) com preços, e a quantidade disponível.
2. A quantidade disponível é atualizada por *polling* enquanto a página fica
   aberta.
3. Se disponível > 0: pode escolher quantidade/tipo e reservar (RF03). Se
   disponível = 0: vê "esgotado", sem botão de reservar.
4. Sessão já começada ou cancelada: página informa o status e não permite
   reserva.

### Comprador

Mesma página, com o botão de reservar levando ao fluxo autenticado.

### Admin

Vê a página normal; a edição da sessão é por outra área (RF08).

## 2. Estados e transições

### Sessão (do ponto de vista do visitante)

**Estados:** à venda · esgotada · encerrada · cancelada.

- **à venda → esgotada:** disponível chega a 0 (por reservas/compras). Volta a
  "à venda" se reservas expiram ou compras são canceladas.
- **à venda / esgotada → encerrada:** a data/hora da sessão passou.
- **qualquer → cancelada:** admin cancela a sessão (RF08 → RF07).

## 3. Regras de negócio

- Disponível = capacidade − confirmados − reservas abertas não vencidas.
- "Esgotado" é `disponível <= 0`, e bloqueia reserva.
- Sessão encerrada ou cancelada bloqueia reserva.
- O número de disponíveis é uma leitura consistente no instante da resposta; não
  é garantia — a reserva revalida sob trava (RN05).
- Preço de meia = 50% do preço de inteira da sessão (RN04 / regra de catálogo).

## 4. Pontos de integração

```text
Frontend precisa saber:
  - O shape do detalhe: espetáculo, startsAt, venue, ticketTypes[{type, price}],
    capacity, availableCount, status ("on_sale"|"sold_out"|"closed"|"cancelled")
  - Que availableCount deve ser re-consultado por polling (~15 s) enquanto a
    página está aberta
  - Que status != "on_sale" ou availableCount <= 0 desabilita o botão de reservar

Backend precisa garantir:
  - availableCount calculado como capacity − confirmados − reservas abertas não
    vencidas, em ≤ 1 s p95 (RNF02) — índices em tickets(session_id,status) e
    reservations(session_id,status,expires_at)
  - status derivado de is_on_sale + startsAt + cancelamento
  - Exportar para o módulo booking (dependencies.py):
      lock_session_for_update(session, session_id) -> SessionRef  (SELECT ... FOR UPDATE)
      count_confirmed_tickets_for_session(session, session_id) -> int
    SessionRef = frozen dataclass {id, capacity, starts_at, is_on_sale}
```

## 5. Casos de borda

**availableCount negativo transitório.** Não deve acontecer (a reserva trava a
linha), mas se acontecer por dado legado, o frontend trata `<= 0` como esgotado.
`[fechada]`

**Sessão fica esgotada e volta a ter vaga em segundos** (reserva alheia
expirou). O *polling* pega na próxima volta; nenhuma ação especial. `[fechada]`

**Meia-entrada:** o preço mostrado é 50% do inteira; a exigência (ou não) de
documento é assunto da emissão do ingresso — aqui só o preço. `[fechada]`
