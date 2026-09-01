---
status: alvo
spec: catalog-show-search
updated_at: 2026-09-01
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

- `GET /shows?fromDate=YYYY-MM-DD&genre=<slug>&page=1&size=12` → 200
  `{ items: [ShowCard], page, size, total }`.
- `ShowCard`: `{ id, title, synopsisShort, imageUrl, genre, upcomingDates: [ISO],
  priceMin, priceMax }`.
- `GET /genres` → 200 `[{ slug, label }]` (só gêneros com espetáculo em cartaz).

## Regras aplicadas no servidor

Só espetáculos com sessão futura à venda; `upcomingDates`, `priceMin`, `priceMax`
consideram só sessões futuras; ordenação por próxima sessão ascendente;
`fromDate` no passado é tratado como hoje.

## Erros esperados

- 422 em `fromDate` mal formatado. Sem outros erros de negócio (é leitura
  pública).

## Impacto de UX

Estados loading / vazio ("Nenhum espetáculo em cartaz para esse filtro") / erro.
Sem autenticação. Meta ≤ 2 s p95.

## Lacunas / decisões em aberto

- Prefixo/base path, versionamento — `<a definir globalmente>`.
