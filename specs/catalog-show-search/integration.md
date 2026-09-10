---
status: alvo
spec: catalog-show-search
updated_at: 2026-09-10
responsavel: Igor (Backend)
---

# Integration Contract — Busca e filtro de espetáculos

**Status:** alvo. **Módulo backend:** `catalog`.

## Rotas

| Método | Caminho | Auth | Sucesso |
| --- | --- | --- | --- |
| GET | `/shows` | pública | 200 |
| GET | `/genres` | pública | 200 |

## Request / Response

Campos em **snake_case** — mesma grafia do contrato de `identity-auth`
(`UserResponse`, `RegisterRequest`); não existe camadas de tradução
camelCase no backend real.

- `GET /shows?fromDate=YYYY-MM-DD&genre=<texto>&page=1&size=12` → 200
  `{ items: [ShowCard], page, size, total }`. `fromDate` é o único parâmetro
  em camelCase (é query string de URL pública, não corpo JSON).
- `ShowCard`: `{ id, title, synopsis_short, image_url, genre, upcoming_dates:
  [ISO 8601], price_min, price_max }`.
- `GET /genres` → 200 `["drama", "comédia", ...]` — lista simples de string,
  só os gêneros com espetáculo visível na vitrine. (Sem par `{slug, label}`:
  `Show.genre` é um campo de texto livre no domínio, sem conceito de slug.)

## Regras aplicadas no servidor

Só espetáculo **publicado** com ≥ 1 sessão futura à venda; `upcoming_dates`,
`price_min`, `price_max` consideram só sessões futuras; ordenação por próxima
sessão ascendente; `fromDate` no passado é tratado como hoje.

## Erros esperados

- 422 em `fromDate` mal formatado — corpo `{ "detail": [{ "field", "message" }] }`
  (envelope real de erro de forma do FastAPI, via `format_validation_errors`).
- Sem erro de negócio (é leitura pública, sem regra que rejeite a requisição).

## Impacto de UX

Estados loading / vazio ("Nenhum espetáculo em cartaz para esse filtro.") /
erro. Sem autenticação (`skipAuth: true` em toda chamada). Meta ≤ 2 s p95.

## Lacunas / decisões em aberto

- Prefixo/base path, versionamento — ainda `<a definir globalmente>` (mesma
  pendência global registrada em `identity-auth`; hoje as rotas reais não têm
  prefixo `/api` nem versão).
