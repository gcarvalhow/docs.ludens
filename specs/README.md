# Specs de feature

Responsável: PO (Gabriel Carvalho) · Última revisão: 2026-09-01 · Status: vigente

Cada feature do Ludens nasce aqui como uma sequência de artefatos antes de
qualquer linha de código. O pipeline é operado pelo plugin Claude Code
[`gcarvalhow/team.ludens`](https://github.com/gcarvalhow/team.ludens) (skills
`feature-design`, `logic-design`, `feature-implementation-spec`); este diretório
guarda o resultado.

## Convenção de pastas

```text
specs/[domínio]-[conceito]/
  spec.md               ← o que a feature é e por quê (produto)
  logic.md              ← regras, estados, fluxos por perfil, contrato FE↔BE (negócio)
  integration.md        ← contrato backend → frontend (rotas, request/response, erros)
  implementation-spec.md ← mapa de todo o código a implementar, por responsável, com passo a passo TBD
```

- `[domínio]` é sempre um dos cinco módulos:
  `identity` · `catalog` · `booking` · `payment` · `notification`.
- `[conceito]` é kebab-case, no máximo 2 palavras, **em inglês**.
- Nome de pasta e de arquivo em inglês; **conteúdo dos documentos em português**.

Exemplos: `specs/booking-reservation/`, `specs/catalog-show-search/`,
`specs/payment-pix-checkout/`.

## Ordem dos artefatos (não pular etapas)

| # | Artefato | Quem gera | Gate antes de avançar |
| --- | --- | --- | --- |
| 1 | `spec.md` | skill `feature-design` (agente `product-thinking`) | **aprovação explícita do PO** (`status: approved`) |
| 2 | `logic.md` | skill `logic-design` | revisão conjunta dos tech leads de FE e BE (`status: reviewed`) |
| 3 | `integration.md` | skill `feature-implementation-spec` gera o **contrato-alvo**; o responsável de backend atualiza para o **canônico** ao fim da implementação | aprovação do responsável de backend |
| 4 | `implementation-spec.md` | skill `feature-implementation-spec` (delegando a `senior-dev` e `qa-engineer`) | nenhuma fatia com bloqueio em aberto |

Depois do artefato 4, cada fatia (Backend / Frontend / QA) vira uma issue no
GitHub Project [`@ludens`](https://github.com/orgs/gcarvalhow/projects/2) via
`/team-ludens:tbd-start` — sub-issue de uma issue-mãe por feature, com a checklist
de arquivos e passos copiada da `implementation-spec.md`.

## Mapa RF → spec (N1 / MVP)

| Pasta de spec | RF | Regras |
| --- | --- | --- |
| `identity-auth` | RF09 | — |
| `identity-order-history` | RF06 | — |
| `catalog-show-search` | RF01 | — |
| `catalog-session-detail` | RF02 | RN05 (leitura) |
| `catalog-admin-management` | RF08 | — |
| `booking-reservation` | RF03 | RN01, RN03, RN05 |
| `booking-ticket-issuance` | RF05 | RN04 |
| `payment-pix-checkout` | RF04 | — |
| `payment-cancellation-refund` | RF07 | RN02 |
| `notification-transactional-email` | RF05, RF09 | — |

## Status no frontmatter

`spec.md`: `draft` → `approved` → `in-progress` → `done` (ou `archived`).
`logic.md`: `draft` → `reviewed`.
`integration.md`: `alvo` → `canônico`.
`implementation-spec.md`: `draft` → `done`.
