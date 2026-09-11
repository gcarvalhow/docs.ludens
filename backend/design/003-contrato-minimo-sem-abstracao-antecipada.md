# Design 003 — Contrato mínimo: sem VO para dinheiro, sem camelCase, sem métodos genéricos antecipados no repositório

> **Status:** implementado · **Última revisão:** 2026-09-11
> Formaliza uma decisão que só existia em mensagem de commit
> (`gcarvalhow/api.ludens@78d75e3`, `@941a520`) e em nota de revisão dentro de
> `specs/catalog-admin-management/backend.md`.

## Contexto

`identity-auth` (RF09) foi o primeiro módulo de negócio implementado e
mergeado em `api.ludens`. `catalog-admin-management` (RF08, PR #17) foi
desenhado e implementado de forma independente por outra pessoa (Igor
Seberino), sem revisitar o código real de `identity` antes de escrever a
spec. O resultado, descoberto em code review, foi três abstrações
introduzidas sem nenhum precedente ou caso de uso real que as justificasse:

1. Um value object `Money` (`domain/value_objects/money.py`) para representar
   preço, com `from_reais`, `.reais`, `.half()` — quando o único uso real era
   "guardar um preço em centavos e dividir por 2".
2. Um `CamelModel` (`core/shared/schemas.py`, `alias_generator=to_camel`) para
   expor o contrato REST em camelCase — quando `identity-auth`, já mergeado,
   usa `pydantic.BaseModel` puro, snake_case.
3. Métodos genéricos `find_by_id`/`find_by_id_for_update` adicionados ao
   `BaseRepository` do `core` — quando o único caso de uso real (travar uma
   `Session` sob concorrência) é específico de **um** agregado, não um padrão
   geral de todo repositório.

A revisão da PR (doc corrigido no commit `78d75e3`, código corrigido no
commit `941a520`) reverteu os três pelo mesmo motivo: nenhum tinha um segundo
caso de uso real que justificasse a generalização. Sem essa formalização, o
próximo módulo (ou a skill `backend-architecture` do plugin `team.ludens`, que
ainda ensina `Money` como padrão) reintroduz o mesmo desvio.

## Decisão

1. **Sem value object para um tipo primitivo com uma operação simples.**
   Dinheiro é `*_cents: int` — uma coluna e, quando precisa de valor derivado,
   uma `@property` (`Session.half_price_cents`). Não vira um VO com
   aritmética própria **enquanto não houver um caso de uso real** que exija
   isso (câmbio, múltiplas operações compostas, formatação regionalizada
   etc.). Se esse caso aparecer, esta decisão é revisitada — não é proibição
   permanente de VOs de dinheiro, é rejeição de generalização sem uso real.
2. **Contrato REST é sempre `pydantic.BaseModel` puro, snake_case.** Sem
   `CamelModel`/`alias_generator`/camelCase como padrão. Uma exceção pontual
   (ex.: `fromDate` em `catalog-show-search`, por ser query param de URL
   pública) é documentada caso a caso na spec da própria feature — nunca vira
   infraestrutura compartilhada (`core/shared/schemas.py`) de novo.
3. **`BaseRepository`/`AggregateRepository` do `core` só têm os cinco métodos
   genéricos** (`find_by`, `find_all`, `find_all_by`, `exists_by`, `save`).
   Nenhum método de conveniência (`find_by_id`, `find_by_id_for_update`, ou
   qualquer outro) é adicionado à base. Um repositório especializado
   implementa o que seu próprio agregado precisar — `SessionRepository`
   define `find_by_id_for_update` só nele, porque só `Session` precisa da
   trava; `RefreshTokenRepository` define `deactivate_all_for_user` só nele,
   pela mesma razão.

## Consequências

- Menos código genérico sem segundo usuário real — quem lê o `BaseRepository`
  vê exatamente os métodos que todo agregado usa, nada especulativo.
- Toda spec nova precisa checar o `BaseRepository`/schemas reais antes de
  assumir um método ou uma convenção de contrato — não pode copiar de um
  `backend.md` de outra feature sem conferir contra o código.
- `backend/conventions.md` referencia este ADR na seção "Repositório base
  mínimo" e na seção "Contrato REST", pra quem quiser o "porquê" por trás da
  regra.
- A skill `backend-architecture` (plugin `team.ludens`, repositório separado)
  ainda ensina `Money` como padrão de value object
  (`references/03-domain-layer.md`) e `find_by_id`/`find_by_id_for_update`
  como métodos genéricos do `BaseRepository`
  (`references/02-core-layer.md`) — incompatível com esta decisão. Correção
  fica para uma sessão separada, contra o repositório `team.ludens`.
