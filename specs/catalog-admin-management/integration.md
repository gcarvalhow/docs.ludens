---
status: canônico
spec: catalog-admin-management
updated_at: 2026-09-21
responsavel: Igor (Backend)
---

# Integration Contract — Gestão de espetáculos e sessões

**Status:** canônico — reflete o código real já mergeado, ver `backend.md`.
**Módulo backend:** `catalog`. **Auth:** as rotas de escrita exigem
`require_admin` (403 quando `user.is_admin` é `False`), aplicado por rota, não
no router inteiro — o mesmo router acumula rotas de leitura pública sem
`require_admin` (ver abaixo).

## Rotas

Sem prefixo `/admin`: as rotas vivem direto em `/catalog/shows` e
`/catalog/sessions`, e o mesmo router mistura escrita (admin) com leitura
(pública ou admin, conforme quem está autenticado). Esta spec documenta o
write-path, que é dela; o `GET`/busca (`GET /catalog/shows`,
`GET /catalog/shows/{id}`, `GET /catalog/sessions/{id}`) é código real também,
mas contrato completo em `catalog-show-search/integration.md` e
`catalog-session-detail/integration.md`.

| Método | Caminho                         | Auth            | Sucesso | Corpo de sucesso     | Descrição                                                 |
| ------ | ------------------------------- | --------------- | ------- | -------------------- | --------------------------------------------------------- |
| POST   | `/catalog/shows`                | `require_admin` | 201     | `AdminShow`          | Cria espetáculo (nasce `draft`), `sessions` vazio         |
| PUT    | `/catalog/shows/{id}`           | `require_admin` | 200     | `AdminShow`          | Edita — corpo completo, sem `PATCH` (padronização)        |
| POST   | `/catalog/shows/{id}/publish`   | `require_admin` | 204     | —                    | Publica                                                   |
| POST   | `/catalog/shows/{id}/unpublish` | `require_admin` | 204     | —                    | Despublica                                                |
| DELETE | `/catalog/shows/{id}`           | `require_admin` | 204     | —                    | Exclui (recusa se alguma sessão tem venda)                |
| POST   | `/catalog/sessions/`            | `require_admin` | 201     | `AdminSession`       | Cria sessão — `show_id` vai no corpo, não na URL          |
| PUT    | `/catalog/sessions/{id}`        | `require_admin` | 200     | `AdminSession`       | Edita sessão — corpo completo, sem `PATCH`                |
| DELETE | `/catalog/sessions/{id}`        | `require_admin` | 204     | —                    | Exclui sessão (recusa se `tickets_sold > 0`)              |
| POST   | `/catalog/sessions/{id}/cancel` | `require_admin` | 202     | —                    | Cancela sessão (emite `SessionCancelled`)                 |
| GET    | `/catalog/shows`                | opcional        | 200     | `Page[...]`, varia   | Busca/listagem — ver `catalog-show-search/integration.md` |
| GET    | `/catalog/shows/{id}`           | opcional        | 200     | varia por `is_admin` | Detalhe — ver `catalog-session-detail/integration.md`     |
| GET    | `/catalog/sessions/{id}`        | opcional        | 200     | varia por `is_admin` | Detalhe — ver `catalog-session-detail/integration.md`     |

Prefixo/base path e versionamento: `<a definir globalmente>` — ver
`docs.ludens/backend/integration/_template.md`.

## Shapes

Todos os campos em **snake_case** na entrada e na saída — mesma convenção do
contrato real de `identity-auth` (`UserResponse`, `RegisterRequest`). Não há
`CamelModel`/alias camelCase no backend real.

- **AdminShowSummary** (`GET /catalog/shows` como admin, sem sessões):
  `{ id, title, synopsis, image_url, genre_id, genre, status: "draft" |
  "published" }`. `genre_id` (UUID) e `genre` (nome já resolvido) sempre
  juntos — desde `catalog-genre`; não é mais um campo de texto livre.
- **AdminShow** (`POST /catalog/shows`, `PUT /catalog/shows/{id}`, e o
  detalhe admin de `GET /catalog/shows/{id}`): mesmos campos de
  `AdminShowSummary` mais `sessions: AdminSession[]`. Em `POST`, `sessions`
  sempre vem vazio; em `PUT`/detalhe, vem com as sessões atuais.
  `image_url` é sempre atribuída pelo servidor (pool padrão) — nunca vem do
  request de criação/edição (débito técnico: sem upload real nesta entrega).
- **AdminSession:** `{ id, show_id, starts_at, venue, capacity, full_price,
  half_price, status: "on_sale" | "closed" | "cancelled", tickets_sold,
  reserved_open, can_delete }`.
  - `starts_at`: ISO 8601 **com offset** (ex.: `2026-10-01T20:00:00-03:00`).
  - `full_price` / `half_price`: número em reais (ex.: `80` ou `79.9`).
    `half_price` é sempre derivado (50% de `full_price`, truncado ao centavo —
    RN04); nunca é enviado pelo admin.
  - `status`: derivado — `"cancelled"` se cancelada; senão `"closed"` se
    `starts_at` já passou; senão `"on_sale"`. (O estado `draft`/`published` é do
    espetáculo, não da sessão.)
  - `can_delete`: `true` só quando `tickets_sold == 0`. **Hoje `tickets_sold`
    é sempre `0`** — a contagem real ainda não foi implementada (ver
    `backend.md` §7); `can_delete` fica sempre `true` na prática.
- **Criar espetáculo (POST) e editar (PUT):** mesmo corpo nos dois —
  `{ title, synopsis, genre_id }`, os três sempre obrigatórios (é `PUT`:
  substitui o registro inteiro, nunca parcial). `genre_id` precisa existir
  (404 "Gênero não encontrado." senão). Sem `image_url` no corpo em nenhum dos
  dois — não é editável nesta entrega.
- **Criar sessão (POST /catalog/sessions/) e editar (PUT
  /catalog/sessions/{id}):** mesmo corpo nos dois — `{ show_id, starts_at,
  venue, capacity, full_price }`. Em `PUT`, `show_id` vai no corpo mas é
  **ignorado** (a sessão não muda de espetáculo depois de criada) — só existe
  ali porque o schema é compartilhado entre criação e edição.

## Erros esperados

Envelope real: **`{ "detail": "<mensagem>" }`** para erro de negócio (401 /
403 / 404 / 409 / 422-de-domínio — mapeado por subclasse de `DomainError` numa
lista central em `main.py`); **`{ "detail": [ { "field", "message" } ] }`**
para 422 de forma (validação de schema). O frontend (`apiErrorMessage`, via
`ApiError` — `frontend.md` §2) lê as duas formas.

| Status | Quando                                                                                 | Mensagem (`detail`)                                                           |
| ------ | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| 403    | usuário autenticado não é admin (`is_admin=false`), ou sem token, numa rota de escrita | (produzida por `require_admin` de `identity-auth`)                            |
| 404    | `show_id` / `session_id` inexistente ou inativo                                        | "Espetáculo não encontrado." / "Sessão não encontrada."                       |
| 404    | `genre_id` inexistente ou inativo, em `POST`/`PUT` de espetáculo                       | "Gênero não encontrado."                                                      |
| 409    | `DELETE /catalog/sessions/{id}` com `tickets_sold > 0`                                 | "Cancele a sessão em vez de excluir."                                         |
| 409    | `DELETE /catalog/shows/{id}` com sessão vendida                                        | "Cancele as sessões com ingressos vendidos antes de excluir o espetáculo."    |
| 409    | `PUT` de sessão com `capacity < tickets_sold + reserved_open`                          | "Já há ingressos comprometidos nesta sessão."                                 |
| 409    | `cancel` de sessão já cancelada / `PUT` de sessão cancelada                            | "A sessão já está cancelada." / "Não é possível editar uma sessão cancelada." |
| 422    | `starts_at` no passado (na criação ou edição)                                          | "A data da sessão deve ser futura."                                           |
| 422    | `capacity <= 0`, na criação                                                            | "A capacidade deve ser maior que zero."                                       |
| 422    | `starts_at` sem offset, `capacity <= 0`/`> 100000`, `full_price <= 0`, campo em branco | lista `{ field, message }`                                                    |

## Eventos

`POST /catalog/sessions/{id}/cancel` → **202** e emite `SessionCancelled`
(payload: `id`, `show_id`, `starts_at`, `cancelled_at`), gravado na tabela
`events` na mesma transação. Consumido por `payment` (reembolso em massa —
RF07, política RN02 aplicada a partir do momento do cancelamento; o admin não
define o valor) e por `notification` (aviso aos compradores). Latência de
entrega ~2 s (relay do Outbox).

Alteração de horário de sessão vendida (`PUT` com `tickets_sold > 0`): a
resposta traz `tickets_sold` para o frontend avisar o admin; o template do
e-mail de aviso fica em `notification-transactional-email`.

## Semântica de leitura

A listagem admin deixou de ser um `GET /admin/shows` próprio: virou o mesmo
`GET /catalog/shows` que a vitrine pública usa, com resposta paginada
(`Page[AdminShowSummary]` para admin, `Page[ShowCardResponse]` para
visitante/comprador) conforme `is_admin`. Contrato completo, filtros e
paginação em `catalog-show-search/integration.md`. `GET /catalog/shows/{id}`
e `GET /catalog/sessions/{id}` seguem o mesmo princípio (resposta varia por
`is_admin`; admin vê `sessions`/histórico completo, sem filtro de data;
visitante só vê sessões futuras à venda) — contrato completo em
`catalog-session-detail/integration.md`.

## Notas desta revisão (2026-09-10)

Revisão de alinhamento com o código real de `identity-auth` (já mergeado) —
substitui a revisão de 2026-09-03, que assumia convenções inexistentes no
projeto:

- **Casing:** trocado de camelCase (assumia um `CamelModel` que não existe)
  para **snake_case**, igual ao contrato real de `identity-auth`.
- **Erro de negócio:** mapeado por subclasse de `DomainError`
  (`ConflictError`→409, `NotFoundError`→404 etc.) numa lista central em
  `main.py` — não por um parâmetro `status_code` na exceção, que a classe real
  não tem.
- Mantido do resto: envelope `{ "detail": ... }`, `status` derivado de
  sessão, corpos de sucesso por rota, `reserved_open` no shape.
- **Padronização de método (mesma data):** `PATCH` trocado por `PUT` nas duas
  rotas de edição — decisão do PO de não usar `PATCH` em lugar nenhum da API.
  Como consequência, os corpos de edição deixam de ser parciais: usam o
  mesmo schema (e os mesmos campos, todos obrigatórios) da criação.
- **`is_admin`, não `Role` (mesma data):** o `User` real de `identity-auth`
  guarda `is_admin: bool` — não existe enum `Role` no código. Trocado
  `role != ADMIN` por `is_admin=false` em toda menção a 403.

## Notas desta revisão (2026-09-11)

`GET /admin/shows` deixava de ser uma listagem e virava um dump de todo o
catálogo com todas as sessões embutidas — sem paginação, sem filtro, tudo de
uma vez. Além do custo de payload crescendo sem controle conforme o catálogo
acumula sessões passadas e canceladas, quebrava o modelo mental que o admin já
aprende no resto do produto (lista resumida → entra no item específico pra ver
detalhe), o mesmo que o RF01→RF02 já estabelece pro visitante.

- **Rota nova:** `GET /admin/shows/{id}`, mesmo formato de erro (404) das
  demais rotas por id deste módulo.
- **`AdminShow` dividido em dois shapes:** `AdminShowSummary` (lista, sem
  `sessions`) e `AdminShow` (detalhe — `GET /admin/shows/{id}`, `POST`, `PUT`
  — com `sessions`). Nenhum campo mudou de nome ou de tipo, só a presença de
  `sessions` na lista.
- **Não afeta:** as rotas de sessão (`/admin/shows/{id}/sessions`,
  `/admin/sessions/{id}`, `.../cancel`) continuam iguais — o corpo delas
  sempre foi por sessão individual, nunca por lista.
- Esta revisão acontece com o PR de frontend desta feature ainda em revisão
  (não mergeado) — a tela de listagem do admin (que hoje já busca sessões
  embutidas) precisa ser ajustada pra esse novo shape antes do merge.

## Notas desta revisão (2026-09-21)

Reconciliação com o código real, já mergeado, das fatias seguintes:

- **Sem prefixo `/admin`:** as rotas vivem em `/catalog/shows` e
  `/catalog/sessions`; `require_admin` é aplicado por rota, não no router
  inteiro (o router acumula leitura pública sem essa exigência).
- **`GET /admin/shows` e `GET /admin/shows/{id}` (da revisão de 2026-09-11)
  nunca chegaram a existir como rotas próprias de admin** — o read-path virou
  o mesmo endpoint que a vitrine pública usa (`catalog-show-search`,
  `catalog-session-detail`), com resposta que varia por `is_admin`.
- **`genre` (string) virou `genre_id` (UUID, FK)** em `ShowRequest` e em todo
  shape que carrega gênero — mudança de `catalog-genre`.
- **Criar sessão não é mais aninhado em `/shows/{id}/sessions`:** é
  `POST /catalog/sessions/` com `show_id` no corpo.
- **`tickets_sold`/`reserved_open` continuam sempre `0`** — não só porque
  `booking` não mergeou, mas porque a contagem real (`SeatCountsRepository`)
  nunca chegou a ser implementada (ver `backend.md` §7). `can_delete` fica
  sempre `true` na prática, por enquanto.

## Lacunas / decisões em aberto

- Prefixo/base path e versionamento de API — `<a definir globalmente>`.
- `tickets_sold` / `reserved_open` dependem de uma contagem real que ainda
  não existe (nem o repositório, nem as tabelas de `booking`); enquanto isso
  não muda, sempre retornam `0` (ver `backend.md` §7).
