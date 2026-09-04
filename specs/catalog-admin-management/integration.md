---
status: alvo
spec: catalog-admin-management
updated_at: 2026-09-03
responsavel: Igor (Backend)
---

# Integration Contract — Gestão de espetáculos e sessões

**Status:** alvo (vira `canônico` quando o backend implementar). **Módulo
backend:** `catalog`. **Auth:** todas as rotas exigem `require_admin` (403 se
`role != ADMIN`), aplicado no `APIRouter` de `/admin`.

## Rotas

| Método | Caminho                       | Sucesso | Corpo de sucesso | Descrição                                                                            |
| ------ | ----------------------------- | ------- | ---------------- | ------------------------------------------------------------------------------------ |
| GET    | `/admin/shows`                | 200     | `AdminShow[]`    | Lista (inclui rascunhos), com `ticketsSold`, `reservedOpen` e `canDelete` por sessão |
| POST   | `/admin/shows`                | 201     | `AdminShow`      | Cria espetáculo (nasce `draft`)                                                      |
| PATCH  | `/admin/shows/{id}`           | 200     | `AdminShow`      | Edita (campos parciais)                                                              |
| POST   | `/admin/shows/{id}/publish`   | 204     | —                | Publica                                                                              |
| POST   | `/admin/shows/{id}/unpublish` | 204     | —                | Despublica                                                                           |
| DELETE | `/admin/shows/{id}`           | 204     | —                | Exclui (recusa se alguma sessão tem venda)                                           |
| POST   | `/admin/shows/{id}/sessions`  | 201     | `AdminSession`   | Cria sessão                                                                          |
| PATCH  | `/admin/sessions/{id}`        | 200     | `AdminSession`   | Edita sessão                                                                         |
| DELETE | `/admin/sessions/{id}`        | 204     | —                | Exclui sessão (recusa se `ticketsSold > 0`)                                          |
| POST   | `/admin/sessions/{id}/cancel` | 202     | —                | Cancela sessão (emite `SessionCancelled`)                                            |

Prefixo/base path e versionamento: `<a definir globalmente>` — ver
`docs.ludens/backend/integration/_template.md`. As rotas acima já assumem o
prefixo `/admin` dentro do módulo `catalog`.

## Shapes

Todos os campos em **camelCase** na entrada e na saída (o backend usa
`CamelModel` — `alias_generator=to_camel`, `populate_by_name=True`).

- **AdminShow:** `{ id, title, synopsis, imageUrl, genre,
  status: "draft" | "published", sessions: AdminSession[] }`.
- **AdminSession:** `{ id, showId, startsAt, venue, capacity, fullPrice,
  halfPrice, status: "on_sale" | "closed" | "cancelled", ticketsSold,
  reservedOpen, canDelete }`.
  - `startsAt`: ISO 8601 **com offset** (ex.: `2026-10-01T20:00:00-03:00`).
  - `fullPrice` / `halfPrice`: número em reais (ex.: `80` ou `79.9`).
    `halfPrice` é sempre derivado (50% de `fullPrice`, truncado ao centavo —
    RN04); nunca é enviado pelo admin.
  - `status`: derivado — `"cancelled"` se cancelada; senão `"closed"` se
    `startsAt` já passou; senão `"on_sale"`. (O estado `draft`/`published` é do
    espetáculo, não da sessão.)
  - `canDelete`: `true` só quando `ticketsSold == 0`.
- **Criar espetáculo (POST /admin/shows):** `{ title, synopsis, imageUrl,
  genre }`.
- **Editar espetáculo (PATCH):** qualquer subconjunto de `{ title, synopsis,
  imageUrl, genre }`.
- **Criar sessão (POST /admin/shows/{id}/sessions):** `{ startsAt, venue,
  capacity, fullPrice }` (`startsAt` com offset e futuro; `capacity` > 0;
  `fullPrice` > 0).
- **Editar sessão (PATCH):** qualquer subconjunto de `{ startsAt, venue,
  capacity, fullPrice }`.

## Erros esperados

Envelope: **`{ "detail": "<mensagem>" }`** para erro de negócio (403 / 404 /
409 / 422-de-domínio); **`{ "detail": [ { "field", "message" } ] }`** para 422
de forma (validação de schema). Divergência de "envelope global a definir"
resolvida aqui — o frontend lê as duas formas.

| Status | Quando                                                                    | Mensagem (`detail`)                                                           |
| ------ | ------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| 403    | token sem `role=ADMIN` (ou ausente)                                       | (produzida por `require_admin` de `identity-auth`)                            |
| 404    | `showId` / `sessionId` inexistente ou inativo                             | "Espetáculo não encontrado." / "Sessão não encontrada."                       |
| 409    | `DELETE /admin/sessions/{id}` com `ticketsSold > 0`                       | "Cancele a sessão em vez de excluir."                                         |
| 409    | `DELETE /admin/shows/{id}` com sessão vendida                             | "Cancele as sessões com ingressos vendidos antes de excluir o espetáculo."    |
| 409    | `PATCH` de sessão com `capacity < ticketsSold + reservedOpen`             | "Já há ingressos comprometidos nesta sessão."                                 |
| 409    | `cancel` de sessão já cancelada / `PATCH` de sessão cancelada             | "A sessão já está cancelada." / "Não é possível editar uma sessão cancelada." |
| 422    | `startsAt` no passado (na criação ou edição)                              | "A data da sessão deve ser futura."                                           |
| 422    | `startsAt` sem offset, `capacity <= 0`, `fullPrice <= 0`, campo em branco | lista `{ field, message }`                                                    |

## Eventos

`POST /admin/sessions/{id}/cancel` → **202** e emite `SessionCancelled`
(payload: `id`, `showId`, `startsAt`, `cancelledAt`), gravado na tabela `events`
na mesma transação. Consumido por `payment` (reembolso em massa — RF07, política
RN02 aplicada a partir do momento do cancelamento; o admin não define o valor) e
por `notification` (aviso aos compradores). Latência de entrega ~2 s (relay do
Outbox).

Alteração de horário de sessão vendida (`PATCH` com `ticketsSold > 0`): a
resposta traz `ticketsSold` para o frontend avisar o admin; o template do
e-mail de aviso fica em `notification-transactional-email`.

## Semântica de leitura

- `GET /admin/shows` devolve **array** (não paginado nesta fatia), ordenado por
  `createdAt` desc, incluindo rascunhos e sessões futuras e passadas.
- `is_on_sale` (usado pela vitrine, não exposto aqui): espetáculo `published` E
  sessão `on_sale` E `startsAt` futura.

## Notas desta revisão (2026-09-03)

Divergências fechadas ao gerar `backend.md` / `frontend.md`:

- **Casing:** todo o I/O em camelCase (antes o doc só citava `imageUrl`/`genre`
  soltos).
- **Envelope de erro:** definido como `{ "detail": ... }` (string para negócio,
  lista para forma) — antes era "a definir globalmente". Continua pendente só o
  **prefixo/base path e versionamento** globais.
- **`status` de sessão no admin:** valores `on_sale | closed | cancelled`
  (derivados), explicitados.
- **Corpo de sucesso:** `POST /admin/shows` → `AdminShow`; `POST .../sessions` →
  `AdminSession`; `GET /admin/shows` → `AdminShow[]` com `sessions` aninhadas.
- **`reservedOpen`** adicionado ao shape de `AdminSession` (o doc antigo citava
  no texto, faltava no shape).
- **404** adicionado à tabela de erros (recurso inexistente).

## Lacunas / decisões em aberto

- Prefixo/base path e versionamento de API — `<a definir globalmente>`.
- `ticketsSold` / `reservedOpen` dependem das tabelas de `booking`
  (`tickets` / `reservations`); enquanto `booking` não é mergeado, retornam `0`
  (ver `backend.md` §7).
