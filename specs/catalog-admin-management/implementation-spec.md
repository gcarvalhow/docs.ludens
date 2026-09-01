---
status: draft
spec: catalog-admin-management
created_at: 2026-09-01
---

# Gestão de espetáculos e sessões — Implementation Spec

**Resumo:** módulo `catalog` — aggregates `Show` e `Session`, CRUD admin, regra
"cancela, não exclui" para sessão vendida, evento `SessionCancelled`.
**RF:** RF08 · **RN:** — (aciona RN02/RF07) · **Módulo backend:** `catalog` ·
**Feature frontend:** `catalog` (admin) ·
**Contrato:** `specs/catalog-admin-management/integration.md`

**Depende de:** `identity-auth` mergeado (`require_admin`).
**É base de:** `catalog-show-search`, `catalog-session-detail`,
`booking-reservation`.

---

## A. Backend — responsável: Igor

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | domain | `.../catalog/domain/enumerations/show_status.py`, `session_status.py` | `ShowStatus`: `DRAFT`, `PUBLISHED`; `SessionStatus`: `ON_SALE`, `CANCELLED` (encerrada é derivada de `starts_at`) |
| 2 | domain | `.../domain/aggregates/show.py` | aggregate `Show` — `title`, `synopsis`, `image_url`, `genre`, `status`. Métodos `create`, `update`, `publish`, `unpublish`, `deactivate`. Eventos `ShowCreated/Updated/Published/Unpublished` |
| 3 | domain | `.../domain/aggregates/session.py` | aggregate `Session` — `show_id`, `starts_at`, `venue`, `capacity`, `full_price`, `status`. Métodos `create`, `update`, `cancel`, `deactivate`. `is_on_sale` property = `status==ON_SALE and starts_at>now and show publicado` (a parte "show publicado" é resolvida no usecase/leitura). Eventos `SessionCreated/Updated/Cancelled` |
| 4 | domain | `.../domain/value_objects/money.py` | `Money` (evita float em preço); `half_price = full_price * 0.5` |
| 5 | domain | `.../domain/events/catalog_events.py` | os eventos acima |
| 6 | application | `.../application/schemas/request.py` | `CreateShowRequest`, `UpdateShowRequest`, `CreateSessionRequest` (`starts_at` futuro), `UpdateSessionRequest` |
| 7 | application | `.../application/schemas/response.py` | `AdminShowResponse`, `AdminSessionResponse` (com `tickets_sold`, `reserved_open`, `can_delete`) |
| 8 | application | `.../application/usecases/show_admin_usecase.py`, `session_admin_usecase.py` | CRUD + `publish/unpublish`, `cancel_session`, `delete_session` (recusa se `tickets_sold>0`), validação de capacidade |
| 9 | infrastructure | `.../infrastructure/repositories/{show,session}_repository.py` | `AggregateRepository`; `SessionRepository.count_sold_and_open(session_id)` |
| 10 | api | `.../api/routers/admin_catalog_router.py` | as 10 rotas do contrato, todas `Depends(require_admin)` |
| 11 | api | `.../modules/catalog/router.py` | agrega `admin_catalog_router` |
| 12 | migration | `migrations/versions/xxxx_catalog.py` | tabelas `shows`, `sessions`; índices `sessions(show_id)`, `sessions(starts_at, status)` |

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-admin-management
commit 1  feat(catalog): modelar Show, Session, Money e eventos
commit 2  feat(catalog): usecases de gestão + schemas (cancela nao exclui, capacidade)
commit 3  feat(catalog): repositórios com contagem de vendas
commit 4  feat(catalog): rotas admin (require_admin), router e migration
/team-ludens:tbd-pr
```

---

## B. Frontend — responsável: Diego · feature `catalog` (área admin)

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.js` | grupo `catalog.admin`: shows CRUD + publish/unpublish, sessions CRUD + cancel |
| 2 | schemas | `src/features/catalog/schemas/admin.schema.js` | Zod: `showFormSchema`, `sessionFormSchema` (startsAt futuro), `adminShowSchema`, `adminSessionSchema` |
| 3 | services | `src/features/catalog/services/admin-catalog.service.js` | funções por rota |
| 4 | queries | `.../hooks/queries/query-options.js` | `adminShowList` |
| 5 | mutations | `.../hooks/mutations/useAdminCatalogMutations.js` | create/update/publish/unpublish/deleteShow, create/update/cancel/deleteSession — invalida `adminShowList` + toasts; erro 409 de delete → toast "cancele em vez de excluir" |
| 6 | forms | `.../hooks/forms/{useShowForm,useSessionForm}.js` | resolver = schema de request |
| 7 | components | `.../components/admin/{ShowList,ShowForm,SessionForm,SessionRow}.jsx` | `SessionRow` mostra `ticketsSold` e habilita "excluir" só se `canDelete` |
| 8 | components/ui | `.../components/ui/ConfirmCancelSessionDialog.jsx` | confirma cancelamento avisando do reembolso |
| 9 | rotas | `src/App.jsx` | `/admin/espetaculos` protegida por `RequireAuth` + checagem de `role === 'ADMIN'` |
| 10 | barrels | `index.js` | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-admin
commit 1  feat(catalog): endpoints, schemas e services da área admin
commit 2  feat(catalog): queries, mutations e forms de espetáculo/sessão
commit 3  feat(catalog): telas de gestão com regra de cancelar vs excluir
commit 4  chore(catalog): barrels
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_excluir_sessao_com_venda_recusa` | sessão com 1 ingresso confirmado → `delete` levanta `DomainError` | RF08 |
| `test_cancelar_sessao_emite_evento` | `cancel()` → `status=CANCELLED`, evento `SessionCancelled` | RF08 / RF07 |
| `test_reduzir_capacidade_abaixo_do_vendido_recusa` | capacidade 10, 8 comprometidos, set 5 → recusa | RF08 |
| `test_criar_sessao_data_passada_recusa` | `starts_at` ontem → recusa | RF08 |
| `test_meia_e_metade_do_inteira` | `full_price=100` → `half_price=50` | RN04 |
| `test_rota_admin_sem_papel_admin_403` | `get_current_buyer` com `role=BUYER` → 403 | RF08 |

Roteiro manual: cadastrar espetáculo + sessão, publicar, ver na vitrine; vender 1
ingresso; tentar excluir a sessão (recusado) → cancelar → comprador aparece na
fila de reembolso. Tentar rota admin como comprador comum → 403.

```text
git checkout master && git pull && git checkout -b test/<NN>-catalog-admin
git commit -m "test(catalog): cobrir cancelar-nao-excluir, capacidade e acesso admin"
```

## D. DevOps — Gabriel

Nada.

## E. Ordem entre as fatias

**Primeira feature de `catalog` a entrar** (base das demais). Depende só do merge
de `identity-auth`. Backend + QA juntos; frontend contra o alvo.

## F. Bloqueios em aberto

Nenhum. (O template de e-mail de "sessão cancelada" / "horário alterado" é
detalhado em `notification-transactional-email`.)
