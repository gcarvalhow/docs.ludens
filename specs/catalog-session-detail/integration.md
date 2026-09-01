---
status: alvo
spec: catalog-session-detail
updated_at: 2026-09-01
responsavel: Igor (Backend)
---

# Integration Contract — Detalhe da sessão

**Status:** alvo. **Módulo backend:** `catalog`.

## Rotas

| Método | Caminho | Auth | Sucesso |
| --- | --- | --- | --- |
| GET | `/shows/{showId}` | pública | 200 |
| GET | `/sessions/{sessionId}` | pública | 200 |

## Response

- `GET /shows/{showId}` → `{ id, title, synopsis, imageUrl, genre,
  sessions: [SessionSummary] }` (só sessões futuras).
- `GET /sessions/{sessionId}` → `SessionDetail`:
  `{ id, show: {id, title}, startsAt, venue, capacity, availableCount,
  status: "on_sale"|"sold_out"|"closed"|"cancelled",
  ticketTypes: [{ type: "full"|"half", price }] }`.

## Regras aplicadas no servidor

`availableCount = capacity − confirmados − reservas abertas não vencidas`;
`status` derivado de `is_on_sale` + `startsAt` + cancelamento; `price` de `half`
= 50% de `full`.

## Erros

- 404 sessão/espetáculo inexistente ou inativo.

## Impacto de UX

*Polling* de `GET /sessions/{id}` a cada ~15 s enquanto a página está aberta.
`status != "on_sale"` ou `availableCount <= 0` desabilita o botão de reservar e
mostra o rótulo do estado.

## Contrato interno (para o módulo `booking`)

`catalog/dependencies.py` exporta:

- `lock_session_for_update(session, session_id) -> SessionRef | None` —
  `SELECT ... FOR UPDATE` na linha da sessão.
- `count_confirmed_tickets_for_session(session, session_id) -> int`.

`SessionRef` = `@dataclass(frozen=True)` `{ id, capacity, starts_at, is_on_sale }`.
Nunca expõe o aggregate `Session`.

## Lacunas / decisões em aberto

- Prefixo/base path, versionamento, envelope de erro — `<a definir globalmente>`.
