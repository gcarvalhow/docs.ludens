---
status: draft
spec: catalog-show-search
created_at: 2026-09-01
---

# Busca e filtro de espetáculos — Implementation Spec

**Resumo:** vitrine pública de espetáculos com sessão futura, filtro por data e
gênero. Leitura pura, sem aggregate novo (usa `Show`/`Session` de
`catalog-admin-management`).
**RF:** RF01 · **RN:** — · **Módulo backend:** `catalog` · **Feature frontend:**
`catalog` · **Contrato:** `specs/catalog-show-search/integration.md`

**Depende de:** `catalog-admin-management` (aggregates `Show` e `Session` +
migration) mergeado.

---

## A. Backend — responsável: Igor

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | application | `src/app/modules/catalog/application/schemas/response.py` | `ShowCardResponse`, `GenreResponse`, `PagedShows` |
| 2 | application | `.../application/usecases/show_search_usecase.py` | `search(from_date, genre, page, size)` e `list_genres()` — só leitura |
| 3 | infrastructure | `.../infrastructure/repositories/show_repository.py` | adicionar `search_with_upcoming(from_date, genre, page, size)` — `JOIN` com `sessions`, `WHERE session.starts_at >= greatest(:from_date, now()) AND session.is_on_sale`, agrega `min/max(price)` e `array_agg(distinct starts_at)`; `list_genres_in_catalog()` |
| 4 | api | `.../api/routers/show_router.py` | `GET /shows`, `GET /genres` (sem auth) |
| 5 | api | `.../modules/catalog/router.py` | incluir `show_router` (se ainda não incluído por `catalog-admin-management`) |
| 6 | — | — | nenhuma migration nova; garantir índice `sessions(starts_at, is_on_sale)` na migration de `catalog-admin-management` |

### Passo a passo TBD (Backend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-show-search
commit 1  feat(catalog): schemas e usecase de busca de espetáculos
commit 2  feat(catalog): query de espetáculos com sessão futura + gêneros
commit 3  feat(catalog): expor GET /shows e GET /genres
/team-ludens:tbd-pr
```

---

## B. Frontend — responsável: Diego · feature `catalog`

> **Stack:** Next.js (App Router) + TypeScript. Convenções completas na skill `frontend-architecture` do `team.ludens`: `services/` fica sob `server/`, tipos em `server/types/` (`z.infer`), rotas em `src/app/<rota>/page.tsx` (Server Component), `'use client'` só onde há hook/estado/handler, barrels `index.ts`. Os caminhos abaixo são o mapa da feature — ajuste a extensão/pasta ao padrão da skill.

| # | Camada | Caminho | O que fazer |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.ts` | `catalog.shows.list`, `catalog.genres` |
| 2 | schemas | `src/features/catalog/schemas/show.schema.ts` | Zod: `showCardSchema`, `pagedShowsSchema`, `genreSchema` |
| 3 | services | `src/features/catalog/services/show.service.ts` | `fetchShows(filters)`, `fetchGenres()` |
| 4 | queries | `.../hooks/queries/query-options.ts` + `useCatalogQueries.ts` | `showList(filters)` com key incluindo os filtros; `genres` |
| 5 | hooks | `.../hooks/useShowFilters.ts` | estado dos filtros (fromDate, genre, page) sincronizado com a query string da URL |
| 6 | components | `.../components/ShowGrid.tsx` | conecta `useCatalogQueries` + `useShowFilters`; trata loading/empty/error |
| 7 | components | `.../components/ShowFilters.tsx` | seletor de data + gênero |
| 8 | components/ui | `.../components/ui/ShowCard.tsx` | apresentacional: imagem, título, sinopse curta, datas, faixa de preço |
| 9 | rotas | `src/app/` (App Router: uma `page.tsx` por rota) | `/` → `ShowGrid` |
| 10 | barrels | `index.ts` em toda subpasta + raiz | obrigatório |

### Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-show-search
commit 1  feat(catalog): endpoints, schemas e services de busca
commit 2  feat(catalog): queries e hook de filtros de espetáculo
commit 3  feat(catalog): grade, filtros e card de espetáculo
commit 4  chore(catalog): barrels index.ts
npm run lint && npm run build
/team-ludens:tbd-pr
```

---

## C. QA — responsável: Adrian

### Casos de teste

| Caso | Cenário | Cobre |
| --- | --- | --- |
| `test_espetaculo_sem_sessao_futura_nao_aparece` | show com só sessões passadas → fora da lista | RF01 |
| `test_faixa_de_preco_considera_so_sessao_futura` | preços variados; sessão passada mais barata não puxa o priceMin | RF01 |
| `test_filtro_por_data_e_genero` | combinação de filtros retorna só o esperado | RF01 |
| `test_filtro_data_no_passado_vira_hoje` | `fromDate` = ontem → resultado igual a `fromDate` = hoje | RF01 |

### Roteiro manual

Cadastrar 3 espetáculos (um só com sessão passada) → home mostra 2 → filtrar por
gênero e por data → estado vazio quando nada casa. Medir tempo de resposta de
`/shows` com ~50 espetáculos (meta ≤ 2 s p95).

### Passo a passo TBD (QA)

```text
git checkout master && git pull && git checkout -b test/<NN>-catalog-show-search
git commit -m "test(catalog): cobrir exibição, faixa de preço e filtros da vitrine"
```

## D. DevOps — Gabriel

Nada. Sem variável nova, sem segredo.

## E. Ordem entre as fatias

Depende do merge de `catalog-admin-management`. Backend + QA juntos; frontend
contra o contrato-alvo.

## F. Bloqueios em aberto

Nenhum.
