---
status: canônico
spec: catalog-show-search
updated_at: 2026-09-21
responsavel: Igor (Backend)
---

# Integration Contract — Busca e filtro de espetáculos

**Status:** canônico — reflete o código real já mergeado, ver `backend.md`.
**Módulo backend:** `catalog`.

## Rotas

| Método | Caminho | Auth | Sucesso |
| --- | --- | --- | --- |
| GET | `/catalog/shows` | pública (opcional; admin vê mais campos) | 200 |
| GET | `/catalog/genres` | pública | 200 |

`GET /catalog/shows` é compartilhada com a listagem administrativa de
`catalog-admin-management`: com sessão de admin, devolve o resumo completo
(inclusive `status: draft`); sem sessão ou sessão de comprador, devolve só o
card de espetáculo publicado.

## Request / Response

Campos em **snake_case** — mesma grafia do contrato de `identity-auth`; não
existe camada de tradução camelCase no backend real.

* `GET /catalog/shows?from_date=YYYY-MM-DD&genre_id=<uuid>&page=1&size=12` →
  200 `Page<ShowCard>`: `{ items, page, size, total }`. Sem sessão de admin:
  `ShowCard = { id, title, synopsis_short, image_url, genre_id, genre,
  upcoming_dates: [ISO 8601] (até 5), price_min, price_max }`. Com sessão de
  admin: `AdminShowSummary = { id, title, synopsis, image_url, genre_id,
  genre, status: "draft" | "published" }`.
* `GET /catalog/genres` → 200 `list<Genre>`: `{ id, name }` (ver
  [`catalog-genre`](../catalog-genre/integration.md)) — não uma lista de
  string.

## Regras aplicadas no servidor

Só espetáculo **publicado** com ≥ 1 sessão futura à venda entra na busca
pública; `upcoming_dates` (até 5 datas), `price_min`, `price_max` consideram
só sessões futuras; ordenação por próxima sessão ascendente; `from_date` no
passado é tratado como agora, calculado em horário de Brasília (não UTC).

## Erros esperados

* 422 em `from_date`/`genre_id` mal formatado, corpo `{ "detail": [{ "field",
  "message" }] }` (envelope real de erro de forma do FastAPI, via
  `format_validation_errors`).
* Sem erro de negócio (é leitura pública, sem regra que rejeite a requisição).

## Impacto de UX

Estados loading / vazio ("Nenhum espetáculo em cartaz para esse filtro.") /
erro. Sem autenticação obrigatória. Meta ≤ 2 s p95.

## Lacunas / decisões em aberto

* Prefixo/base path e versionamento continuam sem decisão global; hoje as
  rotas reais não têm prefixo `/api` nem versão.

## Ajustes feitos no `integration.md`

* `genre` (texto livre) virou `genre_id` (UUID), e a lista de gêneros passou a
  vir de `GET /catalog/genres` (objetos `{id, name}`), não de um
  `GET /genres` próprio retornando string.
* `fromDate` (camelCase) foi removido; o parâmetro real é `from_date`.
* A rota real é `GET /catalog/shows`, não `GET /shows`, e é compartilhada com
  a listagem administrativa.
* A resposta é o `Page<T>` genérico do backend, não um formato próprio desta
  feature.
