# Guia de Estilo e Código — Backend

> **Responsável:** Desenvolvedor Backend (Igor Thiago Seberino) · **Aprovação:** PO (Gabriel Carvalho)
> **Última revisão:** 2026-08-28 · **Status:** vigente
> Versão operacional do [Acordo de Manutenibilidade §5](../team/maintainability.md).
> O guia de estilo do frontend virá na pasta `frontend/`.

## Convenções de código (Python / FastAPI) — PEP 8

- `snake_case` para variáveis e funções.
- `PascalCase` para classes.
- `snake_case` para módulos/arquivos (ex.: `ticket_service.py`).
- **Sem formatador/linter automatizado.** Nenhuma ferramenta reformata o código
  na pipeline — `pytest -q` é o único portão automatizado do backend hoje (ver
  [testes e CI](../backend/testing.md)).

## Estilo existente é absoluto

Espaçamento, quebra de linha, separação entre blocos e organização de um
arquivo já escrito devem ser respeitados **exatamente como estão**. Uma
implementação nova nunca reformata, reordena import ou "limpa" código já
existente por iniciativa própria — só toca o que a mudança pedida exige.
Isso vale tanto para pessoas quanto para qualquer assistente de IA trabalhando
no repo: nunca rodar um formatter/linter de forma automática sobre código já
escrito, e nunca reescrever um arquivo inteiro a partir de uma versão em cache
— sempre reler o arquivo atual antes de editar.

## Idioma do código

- Identificadores (classes, métodos, variáveis, módulos) em **Inglês** — usar os
  termos do domínio: `Show`, `Session`, `Ticket`, `Reservation`, `Order`,
  `User`.
- Comentários em **Português**.

## Boas práticas de manutenibilidade

- **Fronteiras de módulo.** Respeitar as fronteiras entre módulos (monólito
  modular) e isolar as regras de domínio (DDD). Nenhum módulo importa o
  `domain`/`infrastructure` interno de outro — a comunicação é por contrato
  público explícito.
- **DRY.** Evitar duplicação com funções e módulos reutilizáveis.
- **Responsabilidade única.** Funções/métodos com uma responsabilidade e **até
  ~30 linhas** sem justificativa técnica.
- **Sem `except` vazio.** Exceções são tratadas ou registradas. Violação de regra
  de negócio vira erro de domínio (ex.: `DomainError`) e é traduzida pela camada
  de API no status HTTP adequado (ex.: `422`).
- **Nenhum segredo versionado.** Credenciais em variáveis de ambiente; `.env`
  fora do controle de versão; versionar apenas `.env.example` com valores em
  branco ou de desenvolvimento.

## Idioma da documentação e dos artefatos de processo

- Issues, histórias de usuário e mensagens de commit: **português**.
- Nomes de branch: **inglês** (ver [ambiente de desenvolvimento](../team/development.md#convenções-de-contribuição)).
