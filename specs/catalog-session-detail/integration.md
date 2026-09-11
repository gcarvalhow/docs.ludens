---
status: alvo
spec: catalog-session-detail
updated_at: 2026-09-10
responsavel: Igor (Backend)
---

# Integration Contract — Detalhe da sessão

**Status:** alvo. **Módulo backend:** `catalog`.

## Rotas

| Método | Caminho | Auth | Sucesso |
| --- | --- | --- | --- |
| GET | `/shows/{show_id}` | pública | 200 |
| GET | `/sessions/{session_id}` | pública | 200 |

## Response

Campos em **snake_case** — mesma grafia do contrato de `identity-auth` e de
`catalog-show-search`.

- `GET /shows/{show_id}` → `ShowDetail`: `{ id, title, synopsis, image_url,
  genre, sessions: [SessionSummary] }` (`sessions` só as futuras à venda).
  `SessionSummary`: `{ id, starts_at, venue }`.
- `GET /sessions/{session_id}` → `SessionDetail`:
  `{ id, show: { id, title }, starts_at, venue, capacity, available_count,
  status: "on_sale"|"sold_out"|"closed"|"cancelled",
  ticket_types: [{ type: "full"|"half", price }] }`.

## Regras aplicadas no servidor

`available_count = capacity − confirmados − reservas abertas não vencidas`
(via `SeatCountsRepository`, compartilhado com `catalog-admin-management`;
0 enquanto `booking` não existe); `status` derivado de cancelamento →
horário → disponibilidade, nessa ordem; `price` de `half` = 50% de `full`
(vem pronto do aggregate `Session`, nunca recalculado aqui).

## Erros

- 404 sessão/espetáculo inexistente, inativo, **ou despublicado** (decisão de
  implementação: um espetáculo em rascunho não é navegável por link direto —
  mesma inferência de `catalog-show-search`).

## Impacto de UX

*Polling* de `GET /sessions/{id}` a cada ~15 s enquanto a página está aberta
(`staleTime: 5s`, `refetchInterval: 15s`). `status != "on_sale"` ou
`available_count <= 0` desabilita o botão de reservar e mostra o rótulo do
estado.

## Contrato interno (para o módulo `booking`)

`catalog/dependencies.py` exporta:

- `lock_session_for_update(session, session_id) -> SessionRef | None` —
  `SELECT ... FOR UPDATE` na linha da sessão, via
  `SessionRepository.find_by_id_for_update` (novo método — ver `backend.md`
  §1 sobre não duplicar com `catalog-admin-management`).
- `count_confirmed_tickets_for_session(session, session_id) -> int` — lê
  `tickets` (tabela de `booking`) por SQL direto, com fallback a 0 enquanto
  essa tabela não existir (mesma exceção controlada de `SeatCountsRepository`).

`SessionRef` = `@dataclass(frozen=True)` `{ id, capacity, starts_at,
is_on_sale }`. Nunca expõe o aggregate `Session`.

## Lacunas / decisões em aberto

- Prefixo/base path, versionamento, envelope de erro — mesma pendência
  global de `catalog-show-search` (`{"detail": "..."}` já é o padrão real,
  só falta registrar formalmente em um documento único).
- **Arquitetura de `count_confirmed_tickets_for_session`**: ler a tabela
  `tickets` de `booking` por SQL direto (feito) vs. `booking` expor essa
  contagem — decisão de arquitetura que não bloqueia esta fatia.
