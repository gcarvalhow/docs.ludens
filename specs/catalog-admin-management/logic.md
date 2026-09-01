---
status: reviewed
spec: catalog-admin-management
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# Gestão de espetáculos e sessões — Lógica de Negócio

## 1. Fluxo por perfil

### Admin

**Espetáculo:** cria (título, sinopse, imagem URL, categoria) → edita → publica /
despublica → exclui (só se não tiver sessão com venda).

**Sessão (dentro de um espetáculo):** cria (data/hora, local, capacidade, preço
inteira) → edita → exclui (só sem venda) ou **cancela** (com venda → dispara
reembolso, RF07).

Ações proibidas e o que acontece:

- Excluir sessão com ingressos vendidos → recusado ("Cancele a sessão em vez de
  excluir; os compradores serão reembolsados").
- Reduzir capacidade abaixo de `confirmados + reservas abertas` → recusado
  ("Já há N ingressos comprometidos nesta sessão").
- Criar sessão com data no passado → recusado.
- Publicar espetáculo sem nenhuma sessão futura → permitido, mas ele não aparece
  na vitrine até ter uma.

### Comprador / Visitante

Sem acesso a esta área. As rotas exigem papel `ADMIN` (`require_admin`) → 403.

## 2. Estados e transições

### Espetáculo

**Estados:** rascunho · publicado · inativo (excluído).

- rascunho ↔ publicado: ações `publish` / `unpublish`.
- publicado/rascunho → inativo: `delete` (só sem sessão vendida).

### Sessão

**Estados:** à venda · encerrada (data passou) · cancelada · inativa (excluída).

- à venda → cancelada: `cancel` (dispara reembolso de todos os pedidos
  confirmados da sessão).
- à venda → inativa: `delete` (só sem venda).
- à venda → encerrada: automático quando `starts_at` passa.

## 3. Regras de negócio

- CRUD de espetáculo e de sessão exige papel `ADMIN`.
- Sessão só some da vitrine quando: despublicada (espetáculo), cancelada, ou
  encerrada.
- Excluir sessão com ≥ 1 ingresso confirmado → proibido; usar `cancel`.
- `cancel` de sessão → todas as reservas abertas viram canceladas; todos os
  pedidos confirmados entram no fluxo de reembolso (RF07, política RN02 aplicada
  a partir do momento do cancelamento — o admin não define o valor).
- Capacidade nunca abaixo de `confirmados + reservas abertas não vencidas`.
- Preço de meia = 50% do preço de inteira da sessão (derivado, não digitado).
- Data de sessão sempre no futuro na criação.

## 4. Pontos de integração

```text
Frontend precisa saber:
  - Shapes de Show e Session para os formulários (campos, validações de forma)
  - Que ações de sessão são "cancel" e "delete" e quando cada uma está
    disponível (a resposta traz canDelete: bool, ticketsSold: int)
  - As mensagens de recusa (excluir com venda / reduzir capacidade / data passada)

Backend precisa garantir:
  - require_admin em todas as rotas
  - delete de sessão recusado se houver ingresso confirmado
  - cancel de sessão emite SessionCancelled (consumido por payment para reembolso
    em massa) na mesma transação
  - capacidade validada contra confirmados + reservas abertas
  - is_on_sale derivado de: espetáculo publicado E sessão não cancelada E não
    encerrada
```

## 5. Casos de borda

**Cancelar sessão com 200 pedidos confirmados.** `SessionCancelled` é um único
evento; o handler de `payment` itera os pedidos e agenda um estorno por pedido,
de forma idempotente (RF07). O admin recebe confirmação imediata; os estornos
correm em background. `[fechada]`

**Editar horário de sessão já vendida.** Permitido, mas a resposta sinaliza
`ticketsSold > 0` para o frontend avisar o admin que compradores serão
notificados da mudança (via `notification`). `[fechada]`

**Despublicar espetáculo com sessão à venda e reservas abertas.** As reservas
abertas continuam válidas até expirar ou serem pagas; o espetáculo só some da
vitrine. Novas reservas ficam bloqueadas (sessão deixa de estar `on_sale`).
`[fechada]`
