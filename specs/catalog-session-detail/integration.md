---
status: canônico
spec: catalog-session-detail
updated_at: 2026-09-21
responsavel: Igor (Backend)
---

# Integration Contract — Detalhe da sessão

**Status:** canônico — reflete o código real já mergeado, ver
`catalog-admin-management/backend.md` (mesmos arquivos/classes). **Módulo
backend:** `catalog`.

## Rotas

Sem prefixo solto — as duas rotas vivem nos mesmos routers que
`catalog-admin-management` já documenta, sob `/catalog/shows`/
`/catalog/sessions`:

| Método | Caminho | Auth | Sucesso |
| --- | --- | --- | --- |
| GET | `/catalog/shows/{show_id}` | opcional (`is_admin` muda a resposta) | 200 |
| GET | `/catalog/sessions/{session_id}` | opcional (`is_admin` muda a resposta) | 200 |

## Response

Campos em **snake_case** — mesma grafia do contrato de `identity-auth` e de
`catalog-show-search`. Resposta pública abaixo (`is_admin=false`); com token
de admin, a mesma rota devolve `AdminShow`/`AdminSession` — ver
`catalog-admin-management/integration.md`.

- `GET /catalog/shows/{show_id}` → `ShowDetail`: `{ id, title, synopsis,
  image_url, genre_id, genre, sessions: [SessionSummary] }` (`genre_id` desde
  `catalog-genre`; `sessions` só as futuras). `SessionSummary`:
  `{ id, starts_at, venue, capacity, available_count, status }` — carrega
  disponibilidade e status por sessão, não só `{id, starts_at, venue}`.
- `GET /catalog/sessions/{session_id}` → `SessionDetail`:
  `{ id, show: { id, title }, starts_at, venue, capacity, available_count,
  status: "on_sale"|"sold_out"|"closed"|"cancelled",
  ticket_types: [{ type: "full"|"half", price }] }`.

## Regras aplicadas no servidor

`available_count = capacity − confirmados − reservas abertas não vencidas`
(`Session.available_count`, domain). **Hoje sempre `0`**: a contagem real
(`SeatCounts`) é `(0, 0)` fixo em todo lugar — não existe repositório/SQL
fazendo essa conta ainda (ver `catalog-admin-management/backend.md` §7), não
só "enquanto `booking` não existe". `status` derivado de cancelamento →
horário → disponibilidade, nessa ordem (`Session.status_at`); `price` de
`half` = 50% de `full` (vem pronto do aggregate `Session`, nunca recalculado
aqui).

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

**Ainda não existe no código real** — `catalog/dependencies.py` nunca foi
criado, porque `booking-reservation` (o consumidor) não mergeou. O que segue
é planejamento, não contrato canônico; exporta (quando criado):

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
