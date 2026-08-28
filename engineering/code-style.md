# Guia de Estilo e Código

> **Responsável:** Desenvolvedores Frontend e Backend · **Aprovação:** PO
> **Última revisão:** 2026-08-28 · **Status:** vigente
> Versão operacional do [Acordo de Manutenibilidade §5](../team/acordo-de-manutenibilidade.md).

## Convenções por stack

### Backend (Python / FastAPI) — PEP 8

- `snake_case` para variáveis e funções.
- `PascalCase` para classes.
- `snake_case` para módulos/arquivos (ex.: `ticket_service.py`).
- Lint e formatação: **Ruff** (`ruff check .`), obrigatório na pipeline.

### Frontend (React)

- `camelCase` para variáveis e funções.
- `PascalCase` para componentes e arquivos de componente (ex.: `CheckoutPage.jsx`).
- Lint e formatação: **ESLint** + **Prettier**, obrigatórios na pipeline
  (`npm run lint`).

### Idioma do código

- Identificadores (classes, métodos, variáveis) em **Inglês** — usar os termos do
  [glossário](../requirements/glossary.md) (`Show`, `Session`, `Ticket`,
  `Reservation`, `Order`, `Buyer`).
- Comentários em **Português**.

## Boas práticas de manutenibilidade

- **Fronteiras de módulo.** Respeitar as fronteiras entre módulos (monólito
  modular) e isolar as regras de domínio (DDD). Nenhum módulo importa o
  `domain`/`infrastructure` interno de outro — a comunicação é por contrato
  público explícito. Ver [ADR 001](../architecture/design/001-monolito-modular-com-ddd.md).
- **DRY.** Evitar duplicação com funções e módulos reutilizáveis.
- **Responsabilidade única.** Funções/métodos com uma responsabilidade e **até
  ~30 linhas** sem justificativa técnica.
- **Sem `except`/`catch` vazio.** Exceções são tratadas ou registradas. Violação
  de regra de negócio vira erro de domínio (ex.: `DomainError`) e é traduzida
  pela camada de API no status HTTP adequado (ex.: `422`).
- **Nenhum segredo versionado.** Credenciais em variáveis de ambiente; `.env`
  fora do controle de versão; versionar apenas `.env.example` com valores em
  branco ou de desenvolvimento.

## Idioma da documentação e dos artefatos de processo

- Issues, histórias de usuário e mensagens de commit: **português**.
- Nomes de branch: **inglês** (ver [fluxo Git](git-flow.md)).
