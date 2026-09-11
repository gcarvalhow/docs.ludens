---
status: alvo
spec: catalog-admin-management
updated_at: 2026-09-11
responsavel: Igor (Backend)
---

# Integration Contract — Gestão de espetáculos e sessões

**Status:** alvo (vira `canônico` quando o backend implementar). **Módulo
backend:** `catalog`. **Auth:** todas as rotas exigem `require_admin` (403
quando `user.is_admin` é `False`), aplicado no `APIRouter` de `/admin`.

## Rotas

| Método | Caminho                       | Sucesso | Corpo de sucesso | Descrição                                                                               |
| ------ | ----------------------------- | ------- | ---------------- | --------------------------------------------------------------------------------------- |
| GET    | `/admin/shows`                | 200     | `AdminShowSummary[]` | Lista resumida (inclui rascunhos), sem sessões — ver §Semântica de leitura |
| GET    | `/admin/shows/{id}`           | 200     | `AdminShow`      | Detalhe de um espetáculo, com `sessions: AdminSession[]`, `tickets_sold`, `reserved_open` e `can_delete` por sessão |
| POST   | `/admin/shows`                | 201     | `AdminShow`      | Cria espetáculo (nasce `draft`)                                                         |
| PUT    | `/admin/shows/{id}`           | 200     | `AdminShow`      | Edita — corpo completo, sem `PATCH` (padronização)                                      |
| POST   | `/admin/shows/{id}/publish`   | 204     | —                | Publica                                                                                 |
| POST   | `/admin/shows/{id}/unpublish` | 204     | —                | Despublica                                                                              |
| DELETE | `/admin/shows/{id}`           | 204     | —                | Exclui (recusa se alguma sessão tem venda)                                              |
| POST   | `/admin/shows/{id}/sessions`  | 201     | `AdminSession`   | Cria sessão                                                                             |
| PUT    | `/admin/sessions/{id}`        | 200     | `AdminSession`   | Edita sessão — corpo completo, sem `PATCH` (padronização)                               |
| DELETE | `/admin/sessions/{id}`        | 204     | —                | Exclui sessão (recusa se `tickets_sold > 0`)                                            |
| POST   | `/admin/sessions/{id}/cancel` | 202     | —                | Cancela sessão (emite `SessionCancelled`)                                               |

Prefixo/base path e versionamento: `<a definir globalmente>` — ver
`docs.ludens/backend/integration/_template.md`. As rotas acima já assumem o
prefixo `/admin` dentro do módulo `catalog`.

## Shapes

Todos os campos em **snake_case** na entrada e na saída — mesma convenção do
contrato real de `identity-auth` (`UserResponse`, `RegisterRequest`). Não há
`CamelModel`/alias camelCase no backend real.

- **AdminShowSummary** (`GET /admin/shows`, lista): `{ id, title, synopsis,
  image_url, genre, status: "draft" | "published" }` — sem `sessions`. Pensada
  pra tela de listagem, onde o admin ainda não entrou num espetáculo
  específico (revisão 2026-09-11, ver notas abaixo).
- **AdminShow** (`GET /admin/shows/{id}`, `POST /admin/shows`,
  `PUT /admin/shows/{id}`): `{ id, title, synopsis, image_url, genre,
  status: "draft" | "published", sessions: AdminSession[] }` — mesmos campos
  de `AdminShowSummary` mais `sessions`. Em `POST`, `sessions` sempre vem
  vazio (o espetáculo acabou de nascer); em `PUT`, vem com as sessões atuais.
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
  - `can_delete`: `true` só quando `tickets_sold == 0`.
- **Criar espetáculo (POST /admin/shows) e editar (PUT /admin/shows/{id}):**
  mesmo corpo nos dois — `{ title, synopsis, genre }`, os três sempre
  obrigatórios (é `PUT`: substitui o registro inteiro, nunca parcial). Sem
  `image_url` no corpo em nenhum dos dois — não é editável nesta entrega.
- **Criar sessão (POST /admin/shows/{id}/sessions) e editar
  (PUT /admin/sessions/{id}):** mesmo corpo nos dois — `{ starts_at, venue,
  capacity, full_price }`, os quatro sempre obrigatórios (`starts_at` com
  offset e futuro; `capacity` > 0; `full_price` > 0).

## Erros esperados

Envelope real: **`{ "detail": "<mensagem>" }`** para erro de negócio (401 /
403 / 404 / 409 / 422-de-domínio — mapeado por subclasse de `DomainError` numa
lista central em `main.py`); **`{ "detail": [ { "field", "message" } ] }`**
para 422 de forma (validação de schema). O frontend (`apiErrorMessage`, via
`ApiError` — `frontend.md` §2) lê as duas formas.

| Status | Quando                                                                      | Mensagem (`detail`)                                                           |
| ------ | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| 403    | usuário autenticado não é admin (`is_admin=false`), ou sem token            | (produzida por `require_admin` de `identity-auth`)                            |
| 404    | `show_id` / `session_id` inexistente ou inativo (inclui `GET /admin/shows/{id}`) | "Espetáculo não encontrado." / "Sessão não encontrada."           |
| 409    | `DELETE /admin/sessions/{id}` com `tickets_sold > 0`                        | "Cancele a sessão em vez de excluir."                                         |
| 409    | `DELETE /admin/shows/{id}` com sessão vendida                               | "Cancele as sessões com ingressos vendidos antes de excluir o espetáculo."    |
| 409    | `PUT` de sessão com `capacity < tickets_sold + reserved_open`               | "Já há ingressos comprometidos nesta sessão."                                 |
| 409    | `cancel` de sessão já cancelada / `PUT` de sessão cancelada                 | "A sessão já está cancelada." / "Não é possível editar uma sessão cancelada." |
| 422    | `starts_at` no passado (na criação ou edição)                               | "A data da sessão deve ser futura."                                           |
| 422    | `starts_at` sem offset, `capacity <= 0`, `full_price <= 0`, campo em branco | lista `{ field, message }`                                                    |

## Eventos

`POST /admin/sessions/{id}/cancel` → **202** e emite `SessionCancelled`
(payload: `id`, `show_id`, `starts_at`, `cancelled_at`), gravado na tabela
`events` na mesma transação. Consumido por `payment` (reembolso em massa —
RF07, política RN02 aplicada a partir do momento do cancelamento; o admin não
define o valor) e por `notification` (aviso aos compradores). Latência de
entrega ~2 s (relay do Outbox).

Alteração de horário de sessão vendida (`PUT` com `tickets_sold > 0`): a
resposta traz `tickets_sold` para o frontend avisar o admin; o template do
e-mail de aviso fica em `notification-transactional-email`.

## Semântica de leitura

- `GET /admin/shows` devolve **array de resumo** (`AdminShowSummary`, sem
  `sessions`), não paginado nesta fatia, ordenado por `created_at` desc,
  incluindo rascunhos. Pra ver ou gerenciar as sessões de um espetáculo
  específico, o frontend busca `GET /admin/shows/{id}` — mesmo padrão de
  navegação lista→detalhe do catálogo público (RF01→RF02), aplicado aqui à
  área do admin (revisão 2026-09-11).
- `GET /admin/shows/{id}` devolve o espetáculo com `sessions` — futuras e
  passadas, sem filtro (diferente da vitrine pública, que só mostra sessões
  futuras à venda; aqui o admin precisa ver o histórico completo pra
  gerenciar).
- `is_on_sale` (usado pela vitrine, não exposto aqui): espetáculo `published` E
  sessão `on_sale` E `starts_at` futura.

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

## Lacunas / decisões em aberto

- Prefixo/base path e versionamento de API — `<a definir globalmente>`.
- `tickets_sold` / `reserved_open` dependem das tabelas de `booking`
  (`tickets` / `reservations`); enquanto `booking` não é mergeado, retornam `0`
  (ver `backend.md` §7).
