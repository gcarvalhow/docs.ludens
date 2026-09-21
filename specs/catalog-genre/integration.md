---
status: canônico
spec: catalog-genre
updated_at: 2026-09-21
responsavel: Igor (Backend)
---

# Integration Contract — Gêneros do catálogo

**Status:** canônico (reflete o código real já mergeado, ver `backend.md`).
**Módulo backend:** `catalog`.

## Rotas

| Método | Caminho | Auth | Sucesso | Descrição |
| --- | --- | --- | --- | --- |
| GET | `/catalog/genres` | pública | 200 | Lista todos os gêneros ativos |
| POST | `/catalog/genres` | admin | 201 | Cria um gênero |
| PUT | `/catalog/genres/{id}` | admin | 200 | Edita nome do gênero |
| DELETE | `/catalog/genres/{id}` | admin | 204 | Desativa o gênero (lógico, não apaga a linha) |

Todas em `genre_router.py`, prefixo `/catalog/genres` — não há um router
`admin` separado do router público, a mesma rota `GET` serve os dois casos.

## Request/Response

```json
// GenreRequest (POST, PUT)
{ "name": "string, 1-80 chars" }

// GenreResponse (POST, PUT, item de GET)
{ "id": "uuid", "name": "string" }
```

Sem campo de ícone; não existe no domínio.

`GET /catalog/shows` e `GET /catalog/shows/{id}` (contrato de
`catalog-show-search`) trazem o gênero do espetáculo como par de campos
planos, não como objeto:

```json
{ "genre_id": "uuid", "genre": "string (nome do gênero)" }
```

## Erros esperados

| Status | Quando | Mensagem |
| --- | --- | --- |
| 409 | Nome já usado por outro gênero ativo | "Já existe um gênero ativo com este nome." |
| 409 | Excluir gênero referenciado por algum espetáculo | "Existem espetáculos usando este gênero; altere-os para outro gênero antes de excluir." |
| 404 | `PUT`/`DELETE` com `id` inexistente | "Gênero não encontrado." |
| 401/403 | `POST`/`PUT`/`DELETE` sem sessão admin | erro padrão de auth |

## Impacto de UX

Excluir um gênero em uso é bloqueado no servidor (409), não checado
antecipadamente no cliente — o formulário de exclusão não precisa perguntar
"esse gênero está em uso?" antes de tentar; só trata o 409 se vier.

## Semântica de dados

Nome é a identidade do gênero: duas linhas ativas nunca têm o mesmo `name`
(comparação de string exata, sem normalização de acento/maiúscula). Exclusão
é lógica (`is_active = false`); um `genre_id` referenciado por um espetáculo
continua resolvendo o nome de exibição mesmo depois de desativado.

## Lacunas / decisões em aberto

Nenhuma. O contrato acima é o que está mergeado.
