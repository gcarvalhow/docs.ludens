# Decisões de Design (ADRs)

Um **ADR** (Architecture Decision Record) registra uma decisão de arquitetura ou
processo: o contexto que a motivou, a decisão tomada e as consequências. É um
registro imutável — se uma decisão muda, escreve-se um ADR novo que substitui o
anterior (e o antigo passa a `substituído`).

Use o [template](template.md) para um ADR novo. Numere em sequência,
`NNN-slug.md`.

## Status possíveis

| Status | Significado |
| --- | --- |
| `proposto` | Escrito, ainda não decidido |
| `proposto — pendente de aprovação do PO` | Aguardando decisão do Product Owner (Aprovador na RACI) |
| `aceito` | Decisão vigente |
| `substituído por ADR-NNN` | Não vale mais; ver o ADR indicado |
| `revertido` | Foi aceito e depois desfeito |

## Índice

| ADR | Título | Status |
| --- | --- | --- |
| [001](001-monolito-modular-com-ddd.md) | Monólito modular com DDD | aceito |
| [002](002-stack-fastapi-react-postgresql-docker.md) | Stack: FastAPI + React + PostgreSQL + Docker | aceito |
| [003](003-estrategia-de-branches.md) | Estratégia de branches | proposto — pendente de aprovação do PO |
| [004](004-escopo-da-regra-de-meia-entrada.md) | Escopo da regra de meia-entrada | proposto — pendente de aprovação do PO |
