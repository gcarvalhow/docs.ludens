# CI/CD

> **Responsável:** DevOps · **Aprovação:** PO · **Última revisão:** 2026-08-28 · **Status:** vigente
> Versão operacional do [Acordo de Manutenibilidade §6.3](../team/acordo-de-manutenibilidade.md).

## Princípio

**Nenhum Pull Request é aprovado se o linter acusar erros críticos ou se os
testes automatizados falharem.** O ambiente conteinerizado (Docker) é o padrão de
execução — a aplicação deve construir e subir via Docker sem erros.

## Pipeline por repositório

Cada repositório de código tem seu próprio workflow de CI, disparado em `push` e
`pull_request` para `main`.

### `api.ludens` — Backend

| Etapa | Comando | Ferramenta |
| --- | --- | --- |
| Setup | Python 3.12 | `actions/setup-python` |
| Instalar | `pip install -e ".[dev]"` | — |
| Lint | `ruff check .` | Ruff |
| Testes | `pytest -q` | Pytest |
| Build | `docker build` do backend | Docker |

### `web.ludens` — Frontend

| Etapa | Comando | Ferramenta |
| --- | --- | --- |
| Setup | Node 20 | `actions/setup-node` |
| Instalar | `npm ci` (fallback `npm install`) | — |
| Lint | `npm run lint` | ESLint + Prettier |
| Build | `npm run build` | Vite |

### `docs.ludens` — Documentação

Sem CI obrigatório (repositório só de conteúdo). Recomenda-se rodar localmente
antes do commit:

```bash
npx markdownlint-cli2 "**/*.md"
npx markdown-link-check README.md   # ou lychee
```

## Portões de merge

- Lint verde (Ruff / ESLint).
- Testes verdes (Pytest).
- Build Docker sem erros.
- Pelo menos **1 aprovação** de outro desenvolvedor no Pull Request.

## Origem

O monorepo original tinha um único `ci.yml` com dois jobs (`backend`,
`frontend`). Na separação em três repositórios, cada job vai para o `.github/`
do repositório de código correspondente.
