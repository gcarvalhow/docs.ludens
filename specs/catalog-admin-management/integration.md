---
status: alvo
spec: catalog-admin-management
updated_at: 2026-09-01
responsavel: Igor (Backend)
---

# Integration Contract — Gestão de espetáculos e sessões

**Status:** alvo. **Módulo backend:** `catalog`. **Auth:** todas exigem
`require_admin` (403 se `role != ADMIN`).

## Rotas

| Método | Caminho | Sucesso | Descrição |
| --- | --- | --- | --- |
| POST | `/admin/shows` | 201 | Cria espetáculo |
| PATCH | `/admin/shows/{id}` | 200 | Edita |
| POST | `/admin/shows/{id}/publish` | 204 | Publica |
| POST | `/admin/shows/{id}/unpublish` | 204 | Despublica |
| DELETE | `/admin/shows/{id}` | 204 | Exclui (sem sessão vendida) |
| POST | `/admin/shows/{id}/sessions` | 201 | Cria sessão |
| PATCH | `/admin/sessions/{id}` | 200 | Edita sessão |
| DELETE | `/admin/sessions/{id}` | 204 | Exclui sessão (sem venda) |
| POST | `/admin/sessions/{id}/cancel` | 202 | Cancela sessão (dispara reembolso) |
| GET | `/admin/shows` | 200 | Lista (inclui rascunhos), com `ticketsSold` e `canDelete` por sessão |

## Shapes

- Show: `{ id, title, synopsis, imageUrl, genre, status: "draft"|"published" }`.
- Session (admin): `{ id, showId, startsAt, venue, capacity, fullPrice,
  halfPrice (derivado), status, ticketsSold, reservedOpen, canDelete }`.
- Criar sessão: `{ startsAt, venue, capacity, fullPrice }`.

## Erros esperados

| Status | Quando | Mensagem |
| --- | --- | --- |
| 403 | não-admin | "Acesso restrito." |
| 409 | DELETE sessão com `ticketsSold > 0` | "Cancele a sessão em vez de excluir." |
| 409 | `capacity` < `ticketsSold + reservedOpen` | "Já há ingressos comprometidos nesta sessão." |
| 422 | `startsAt` no passado | "A data da sessão deve ser futura." |

## Eventos

`POST /admin/sessions/{id}/cancel` → 202 e emite `SessionCancelled` (consumido
por `payment` para reembolso em massa e por `notification` para avisar os
compradores).

## Lacunas / decisões em aberto

- Prefixo/base path, versionamento, envelope de erro — `<a definir globalmente>`.
- Notificação de alteração de horário de sessão vendida: qual template — deixar
  para a spec de `notification-transactional-email`.
