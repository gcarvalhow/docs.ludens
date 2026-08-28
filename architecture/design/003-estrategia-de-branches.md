# ADR 003 — Estratégia de branches

> **Status:** proposto — pendente de aprovação do PO
> **Data:** 2026-08-28 · **Responsável:** DevOps · **Decisor:** Product Owner (Gabriel Carvalho)

## Contexto

Há uma divergência entre os documentos do projeto e o estado real do
repositório sobre a estratégia de branches:

| Fonte | O que diz |
| --- | --- |
| [Acordo de Manutenibilidade §6.1](../../team/acordo-de-manutenibilidade.md) | **Trunk Based Development (TBD)**: `main` é a única branch de longa duração; branches de curta duração saem de `main` e voltam em poucos dias; integração via Pull Request; funcionalidades incompletas atrás de feature flags. |
| README do monorepo original (`plataforma-ingressos-teatro`) | **Duas branches de longa duração**: `main` (estável, implantável) e `develop` (integração). Features saem de `develop`; ao fim do ciclo, `develop` é promovida a `main` via PR de release. |
| Repositório de código real | Contém `main`, `develop` e `feature/ajustes-home`. |

O Acordo de Manutenibilidade é o documento **assinado** pela equipe no Processo
18. O fluxo com `develop` foi introduzido depois, apenas no README, sem emenda ao
Acordo.

## Decisão

**Adotar Trunk Based Development, com `main` como única branch de longa
duração**, conforme o Acordo assinado. Concretamente:

- Branches de curta duração saem de `main` e voltam para `main` via Pull
  Request, com no máximo ~2 dias de vida.
- Prefixos de branch: `feature/`, `fix/`, `refactor/`, `chore/`.
- Nada de commit direto em `main`.
- Trabalho incompleto fica protegido por feature flag quando necessário.
- A branch `develop` é **descontinuada** nos três repositórios.

Isto alinha o projeto ao Acordo e ao padrão já usado nos outros repositórios do
mesmo mantenedor.

## Alternativas consideradas

- **Manter `main` + `develop`** (git-flow reduzido) — rejeitado: contradiz o
  Acordo assinado, adiciona uma etapa de promoção sem ganho real para uma
  equipe pequena com ciclo curto, e favorece branches longas (divergência).
- **Emendar o Acordo para oficializar `develop`** — possível, é decisão do PO;
  não é a recomendação, pelos motivos acima.

## Consequências

- [`engineering/git-flow.md`](../../engineering/git-flow.md) descreve TBD puro,
  sem `develop`.
- Os repositórios `api.ludens`, `web.ludens` e `docs.ludens` passam a ter só
  `main` como branch protegida; `develop` é removida.
- O template de Pull Request e a proteção de branch apontam para `main`.
- A pipeline de CI roda em `push` e `pull_request` para `main`.

## Ações decorrentes

- [ ] PO aprova ou ajusta esta decisão
- [ ] Remover `develop` de `api.ludens` e `web.ludens`; migrar PRs abertos para `main`
- [ ] Ajustar proteção de branch e workflows de CI para `main`
- [ ] Conferir que `engineering/git-flow.md` não menciona `develop`
