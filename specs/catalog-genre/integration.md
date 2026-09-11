---
status: alvo
spec: catalog-genre
updated_at: 2026-09-11
responsavel: gerado via feature-implementation-spec
---

# Integration Contract — Gêneros do catálogo

**Status:** alvo (vira `canônico` quando o backend implementar). **Módulo backend:** `catalog`.

## 1. Identificação e status da feature

Gênero de espetáculo deixa de ser texto livre e vira entidade própria
(`Genre`), criada exclusivamente pelo admin. Status: alvo — nenhuma linha de
código desta feature está implementada ainda (backend.md/frontend.md desta
pasta são a spec de implementação).

## 2. Objetivo de negócio

O admin cadastra gêneros curados (com ícone) em vez de digitar texto livre em
cada espetáculo — fortalece o filtro por gênero da vitrine (RF01) e ajusta um
detalhe do critério de aceite de RF08.

## 3. Escopo atual vs. futuro

**Cobre agora:** criação de gênero pelo admin; seleção de gênero (não mais
texto livre) no cadastro/edição de espetáculo; migração automática dos
gêneros em texto livre já existentes.

**Fica para depois:** editar/excluir gênero (spec §6); múltiplos gêneros por
espetáculo; relatório de uso de gênero; filtro por mais de um gênero
simultâneo.

## 4. Semântica de produto

Um "gênero" é uma categoria curada e reutilizável — não um campo livre do
espetáculo. Duas listas de gênero coexistem com propósitos diferentes: a
lista **completa** (tudo que o admin já cadastrou, usada no formulário de
espetáculo) e a lista de **filtro público** (só gêneros com pelo menos um
espetáculo publicado e com sessão futura à venda, usada na busca/vitrine).

## 5. Dependências

Depende de `catalog-admin-management` (RF08) já implementado. Sem serviço
externo novo. Sem variável de ambiente nova.

## 6. Rotas

| Método | Caminho | Auth | Sucesso | Descrição |
|---|---|---|---|---|
| GET | `/admin/genres` | `require_admin` | 200 | Lista completa de gêneros cadastrados |
| POST | `/admin/genres` | `require_admin` | 201 | Cria gênero |
| GET | `/genres` | pública | 200 | Lista de filtro — só gêneros com espetáculo publicado e sessão futura à venda |
| GET | `/shows` | pública | 200 | (já existe, RF01) filtro agora por `genre_id` (query, UUID), não mais `genre` (slug) |
| POST | `/admin/shows` | `require_admin` | 201 | (já existe, RF08) corpo troca `genre: str` por `genre_id: UUID` |
| PUT | `/admin/shows/{id}` | `require_admin` | 200 | (já existe, RF08) idem |

Prefixo/base path e versionamento: `<a definir globalmente>`, mesma pendência
de todas as specs de `catalog`.

## 7. Contrato de request + transforms obrigatórios no frontend

- **`POST /admin/genres`:** `{ name: string, icon: string }` — `name` 1–80
  chars; `icon` 1–80 chars, `^[a-z][a-z0-9-]*$` (kebab-case). Sem transform —
  frontend manda o `name` como o admin digitou (o backend normaliza).
- **`POST /admin/shows` / `PUT /admin/shows/{id}`:** campo `genre_id: UUID`
  substitui `genre: string`. Nenhum transform — é o `id` do gênero
  selecionado no `<Select>`.
- **`GET /shows`:** query param `genre_id` (UUID) substitui `genre` (slug).

## 8. Contrato de response

- **`GenreResponse`** (mesmo shape em `GET /admin/genres`, `GET /genres` e
  embutido em toda resposta de show): `{ id: UUID, name: string, icon: string }`.
  `name` já vem normalizado pelo servidor (sem acento, minúsculo) — é a forma
  canônica **e** de exibição ao mesmo tempo (não há um segundo campo "label"
  com grafia original).
- **`AdminShowSummaryResponse`** (`GET /admin/shows`): `{ id, title, synopsis,
  image_url, genre: GenreResponse, status }` — sem `sessions`.
- **`AdminShowResponse`** (`GET /admin/shows/{id}`, `POST`, `PUT`): tudo
  acima + `sessions: AdminSessionResponse[]`.
- **`ShowCardResponse`** (`GET /shows`, RF01): `genre` também vira
  `GenreResponse` (era `string`).
- **`ShowDetailResponse`** (`GET /shows/{id}`, RF02): idem.

### 8.1 Nulabilidade

`genre` nunca é `null` em nenhuma resposta de show — todo espetáculo sempre
referencia um gênero válido (garantia estrutural: FK `NOT NULL`).

## 9. Estados de negócio e transições

Gênero não tem ciclo de vida (sem estados) — uma vez criado, existe
permanentemente nesta entrega. A única "transição" observável é indireta:
visibilidade no filtro público, derivada do estado dos espetáculos que usam
o gênero (aparece quando ≥ 1 espetáculo publicado com sessão futura à venda
usa aquele gênero; some quando não há mais nenhum).

## 10. Erros esperados e casos de borda

| Status | Quando | Mensagem |
|---|---|---|
| 403 | `POST`/`GET /admin/genres` sem admin | (produzida por `require_admin`) |
| 409 | `POST /admin/genres` com nome já existente (normalizado) | "Já existe um gênero com esse nome." |
| 404 | `POST`/`PUT /admin/shows` com `genre_id` inexistente | "Gênero não encontrado." |
| 422 | `POST /admin/genres` sem nome ou sem ícone (forma) | lista `{ field, message }` |

## 11. Semântica de autenticação/autorização

`GET /admin/genres` e `POST /admin/genres` exigem `require_admin` (mesmo
padrão 403 de todo `/admin/*`). `GET /genres` continua público, sem
autenticação — igual ao comportamento de hoje.

## 12. Impacto de UX

Formulário de espetáculo depende de `GET /admin/genres` não vir vazio —
se vier, o formulário orienta o admin a cadastrar um gênero antes (não
mostra um seletor vazio sem explicação). Criar gênero invalida a query da
lista completa — o gênero novo aparece no formulário de espetáculo
imediatamente, sem reload.

## 13. Frescor de dados, ids estáveis, evolução de contrato

`id` de `Genre` é estável e nunca reaproveitado (sem exclusão física). A
migração de dados (`0003_catalog_genre.py`, ver `backend.md`) gera `id`s
novos para os gêneros derivados do texto livre existente — nenhum client
deveria ter persistido um `id` de gênero antes desta feature (não existiam).

**Evolução de contrato:** esta feature quebra o contrato de duas rotas já em
produção (`GET /shows` e as respostas de show do RF08) — não é aditivo puro.
Ver §16.

## 14. Observabilidade

Nada específico desta feature — mesmo padrão do resto do módulo `catalog`
(sem `traceId` formalizado ainda, pendência global).

## 15. Exemplos de payload

```json
// POST /admin/genres — request
{ "name": "Comédia", "icon": "laugh" }

// POST /admin/genres — response (201)
{ "id": "b6e1...", "name": "comedia", "icon": "laugh" }

// POST /admin/genres — erro de duplicidade (409)
{ "detail": "Já existe um gênero com esse nome." }

// GET /admin/shows — response (200), item da lista
{
  "id": "a1f0...",
  "title": "A Bela e a Fera",
  "synopsis": "Um clássico do teatro comunitário.",
  "image_url": "/images/show-placeholders/1.jpg",
  "genre": { "id": "b6e1...", "name": "infantil", "icon": "baby" },
  "status": "published"
}
```

## 16. Lacunas e decisões em aberto

- **[bloqueio] Ícone de gênero migrado automaticamente** — sem decisão de
  produto (ver `backend.md`/`quality.md` §7). Não rodar a migration contra
  dado real sem isso.
- **[bloqueio, fora desta pasta] `docs.ludens/specs/catalog-show-search/integration.md`
  fica desatualizado por esta feature** — `GET /shows?genre=<slug>` vira
  `?genre_id=<uuid>`, `ShowCardResponse.genre` deixa de ser `string`. Alguém
  precisa atualizar aquele documento antes/junto desta feature ir a produção;
  não foi editado aqui por isolamento de escopo desta sessão.
- Prefixo/base path e versionamento de API — pendência global, não desta
  feature.
