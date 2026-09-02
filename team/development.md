# Ambiente de Desenvolvimento

> **Responsável:** DevOps (Gabriel Carvalho) · **Última revisão:** 2026-08-28 · **Status:** proposto
> Descreve como subir o projeto localmente. O backend ainda não está
> implementado — os comandos abaixo são o alvo, alinhados a
> [`backend/overview.md`](../backend/overview.md) e
> [`backend/security/configuration.md`](../backend/security/configuration.md).

## Repositórios

| Repositório | O que é |
| --- | --- |
| [`gcarvalhow/docs.ludens`](https://github.com/gcarvalhow/docs.ludens) | Esta documentação |
| [`gcarvalhow/api.ludens`](https://github.com/gcarvalhow/api.ludens) | Backend — FastAPI + PostgreSQL |
| [`gcarvalhow/web.ludens`](https://github.com/gcarvalhow/web.ludens) | Frontend — Next.js (App Router, TypeScript) |
| [`gcarvalhow/team.ludens`](https://github.com/gcarvalhow/team.ludens) | Plugin Claude Code — papéis, pipeline de spec, fluxo TBD |

## Plugin do time (`team.ludens`)

O pipeline de produto→engenharia→specs e o fluxo Trunk-Based Development são
operados pelo plugin Claude Code
[`gcarvalhow/team.ludens`](https://github.com/gcarvalhow/team.ludens). Cada repo
já declara o que habilita no próprio `.claude/settings.json` (commitado):
`docs.ludens` usa `core`; `api.ludens` usa `core` + `backend`; `web.ludens` usa
`core` + `frontend`.

Depois do prompt de *trust* do Claude Code, instale de fato (um plugin por
comando) e abra uma **sessão nova**:

```bash
claude plugin marketplace add gcarvalhow/team.ludens
claude plugin install core@team-ludens --scope project
claude plugin install backend@team-ludens --scope project   # só api.ludens
claude plugin install frontend@team-ludens --scope project  # só web.ludens
claude plugin list                                           # confirme "enabled"
```

Rode `/team-ludens:setup` para validar `git`/`gh`/`python`. O pipeline completo
(`feature-design` → `logic-design` → `feature-implementation-spec` → `tbd-start`
→ `tbd-commit` → `tbd-pr`) está descrito no README do `team.ludens` e em
[`../specs/README.md`](../specs/README.md).

## Pré-requisitos

- **Docker** e **Docker Compose** — ambiente padrão de execução, local e de
  pipeline.
- **Python 3.12** e **Node 20** — só se for rodar backend/frontend fora do
  contêiner.
- [`gh` CLI](https://cli.github.com/) autenticado — para abrir PRs.

## Backend (`api.ludens`)

```bash
git clone https://github.com/gcarvalhow/api.ludens
cd api.ludens

cp .env.example .env.local
# preencher os SECRET/SENSITIVE — ver backend/security/configuration.md

docker compose -f docker/docker-compose.Development.yml up -d
docker compose -f docker/docker-compose.Development.yml exec api alembic upgrade head
docker compose -f docker/docker-compose.Development.yml exec api python scripts/seed_admin.py
```

A API sobe em `http://localhost:8000`; docs OpenAPI em `/docs`.

Recriar o banco do zero (mudança de schema sem migration incremental):

```bash
docker compose -f docker/docker-compose.Development.yml down -v
docker compose -f docker/docker-compose.Development.yml up -d
docker compose -f docker/docker-compose.Development.yml exec api alembic upgrade head
```

### Lint e testes

```bash
ruff check .
pytest -q
```

Ambos rodam na pipeline e são portão de merge — ver
[`backend/testing.md`](../backend/testing.md).

## Frontend (`web.ludens`)

```bash
git clone https://github.com/gcarvalhow/web.ludens
cd web.ludens
npm install
npm run dev      # http://localhost:3000
npm run lint
npm run build
```

Stack: **Next.js (App Router) + TypeScript** — as regras de código estão na skill
`frontend-architecture` do plugin `team.ludens`. O detalhamento do frontend
(design, camadas de integração) virá numa pasta
`frontend/` própria.

## Documentação (`docs.ludens`)

Antes de commitar:

```bash
npx markdownlint-cli2 "**/*.md"
npx markdown-link-check README.md   # ou lychee, para checar links
```

## Convenções de contribuição

- Branches de curta duração a partir de `master`; PR pequeno; **1 aprovação** de
  outro desenvolvedor; pipeline verde (lint + testes + build Docker).
- Commits em [Conventional Commits](https://www.conventionalcommits.org/pt-br/),
  mensagem em português no imperativo.
- Sem segredo versionado — só `.env.example` com valores em branco.
