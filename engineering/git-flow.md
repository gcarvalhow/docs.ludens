# Fluxo Git

> **Responsável:** DevOps · **Aprovação:** PO · **Última revisão:** 2026-08-28
> **Status:** vigente — reflete o [Acordo de Manutenibilidade §6](../team/acordo-de-manutenibilidade.md) e o [ADR 003](../architecture/design/003-estrategia-de-branches.md) (pendente de aprovação do PO).

## Estratégia de branches — Trunk Based Development

- **`main`** é a **única branch de longa duração**: sempre estável, integrável e
  apta a implantação. Nada é commitado diretamente nela.
- Branches de **curta duração** saem de `main`, têm escopo pequeno e voltam para
  `main` via Pull Request em **poucos dias** (máximo ~2 dias de vida).
- Trabalho incompleto fica protegido por **feature flag** quando necessário.

> A branch `develop` foi descontinuada — ver
> [ADR 003](../architecture/design/003-estrategia-de-branches.md). Se você tem
> uma branch baseada em `develop`, rebaseie sobre `main`.

### Nome da branch

`<tipo>/<slug-curto-em-ingles>` — 2–3 palavras, tradução do sentido (não
transliteração).

| Tipo | Uso | Exemplo |
| --- | --- | --- |
| `feature/` | Nova funcionalidade | `feature/ticket-reservation` |
| `fix/` | Correção de bug | `fix/reservation-expiry` |
| `refactor/` | Alteração sem mudar comportamento externo | `refactor/order-service` |
| `chore/` | Manutenção/configuração | `chore/ci-config` |

## Conventional Commits

Formato: `tipo(escopo opcional): descrição no imperativo`.

| Tipo | Uso |
| --- | --- |
| `feat` | Nova funcionalidade |
| `fix` | Correção de bug |
| `refactor` | Alteração que não muda o comportamento externo (manutenção preventiva) |
| `docs` | Documentação |
| `test` | Criação ou ajuste de testes |
| `chore` | Manutenção/configuração sem impacto em regra de negócio |

Regras:
- Mensagem em **português**, no **imperativo**, objetiva.
- Commits **pequenos e frequentes** — um commit é uma mudança coesa.

Exemplos:

```text
feat(tickets): adiciona reserva de ingresso com expiração
fix(checkout): corrige cálculo de valor total com desconto
docs(readme): documenta fluxo de branches
test(domain): cobre limite de ingressos por CPF
```

## Fluxo de trabalho (backlog → entrega)

1. **Backlog.** O PO organiza e prioriza as histórias no GitHub Project da
   organização. Um card só entra em desenvolvimento após atender ao
   [Definition of Ready](quality.md).
2. **Branch.** O responsável cria uma branch de curta duração a partir de `main`
   atualizada.

   ```bash
   git checkout main
   git pull
   git checkout -b feature/ticket-reservation
   ```

3. **Desenvolvimento.** Commits pequenos em Conventional Commits; código conforme
   o [guia de estilo](code-style.md).
4. **Antes do PR**, rodar linters e testes localmente:

   ```bash
   # Backend (api.ludens)
   cd backend && ruff check . && pytest

   # Frontend (web.ludens)
   cd frontend && npm run lint
   ```

5. **Pull Request para `main`.** Preencher o template do repositório e vincular a
   issue (`Closes #123`).
6. **Code Review.** Aprovação de **no mínimo outro desenvolvedor**. A pipeline
   (lint + testes + build Docker) deve estar **verde**.
7. **Merge** em `main` após aprovação.
8. **Done.** O card só fecha quando atende a **todos** os itens do
   [Definition of Done](quality.md).

## Regras de Pull Request

- Nunca fazer merge do próprio PR sem revisão de outro integrante.
- PR não é aprovado com pipeline vermelha (lint ou testes falhando).
- PR pequeno e focado — facilita a revisão e reduz risco de divergência.
- Sem push direto em `main`.
- Sem segredos no código — usar `.env` (ver `.env.example`).
