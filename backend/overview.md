# Ludens — Arquitetura da API (`api.ludens`)

> **Status:** proposto — **não há código implementado** · **Última revisão:** 2026-08-28
> Este documento fixa apenas a **base** do backend: os padrões e a estrutura
> comum que todo módulo vai seguir. O detalhe de cada funcionalidade (rotas,
> agregados, schema, fluxos) **não é documentado aqui por antecipação** — entra
> por **spec, uma funcionalidade por vez**, conforme for implementada, e este
> documento é atualizado junto.
>
> Os padrões vêm do backend da plataforma DOM Med (`api.hub.dommed`), que usa
> monólito modular + DDD da mesma forma. Diferenças assumidas para o Ludens:
> **sem Event Sourcing** e **sem broker de mensagens** — ver
> [ADR 001](design/001-outbox-in-process.md).
>
> As **regras de código** que operacionalizam estes padrões vivem na skill
> `backend-architecture` do plugin
> [`gcarvalhow/team.ludens`](https://github.com/gcarvalhow/team.ludens) (10
> arquivos de referência, um por camada). Este documento é a fonte de verdade
> viva do estado atual; a skill explica o padrão e o porquê e aponta de volta
> para cá.

## Stack

FastAPI + PostgreSQL, executado em Docker. SQLAlchemy async, Alembic para
migrations, `pydantic-settings` para configuração (ver
[security/configuration.md](security/configuration.md)).

## Padrões fundamentais

**Domain-Driven Design (DDD).** Cada área de negócio vive isolada em seu módulo,
com suas regras. Módulos não importam o `domain`/`infrastructure` interno uns dos
outros — a comunicação é por evento no outbox ou por dependência pública
exportada. Ver [ADR 002](design/002-monolito-modular.md).

**Outbox in-process.** Um caso de uso nunca chama efeito colateral externo
(e-mail, estorno) diretamente. O efeito é gravado como um `Event` no PostgreSQL
na mesma transação que muda o estado; um relay em background lê a tabela `events`
e chama handlers registrados no próprio processo — sem broker. Ver
[ADR 001](design/001-outbox-in-process.md).

**Sem Event Sourcing e sem CQRS.** O estado de verdade está nas tabelas de
domínio; `events` é só a fila de saída de efeitos. Não há read model separado.

## Core (`src/app/core/`)

Base reutilizada por todos os módulos. Modelada como no backend da DOM Med.

### `Model` base

```python
class Model(DeclarativeBase):
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), onupdate=func.now())
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
```

Toda tabela herda de `Model`. **Não existe `DELETE` real** em entidade de
domínio — a remoção é `is_active = False`, e nenhuma leitura da API expõe
registro inativo.

### `AggregateRoot`

```python
class AggregateRoot:
    def raise_event(self, factory: Callable[[int], DomainEvent]) -> None:
        self._version += 1
        event = factory(self._version)
        self._apply(event)          # muta o estado do agregado imediatamente
        self._events.append(event)  # enfileira para persistir depois

    def dequeue_events(self) -> list[DomainEvent]:
        events, self._events = list(self._events), []
        return events
```

`raise_event` → estado muta + evento enfileirado → o repositório de agregado
chama `dequeue_events()` no `save()` → persiste os eventos na tabela `events` →
o relay executa os handlers. Entidades que não são agregados usam
`BaseRepository` puro, sem drenar eventos.

### Repositórios e tradução de erro

`BaseRepository` / `AggregateRepository` genéricos sobre SQLAlchemy async.
Violação de regra de negócio vira erro de domínio (`DomainError`), traduzido pela
camada de API para o status HTTP adequado (ver
[code-style.md](code-style.md)).

## Módulos previstos

Um módulo por área de negócio (nome em **inglês**, `snake_case`). O conteúdo de
cada um é definido pela sua spec quando a funcionalidade for construída.

| Módulo | Área | Requisitos que cobre |
| --- | --- | --- |
| `identity` | Cadastro e autenticação do comprador | RF06, RF09 |
| `catalog` | Espetáculos e sessões; disponibilidade | RF01, RF02, RF08 |
| `booking` | Reserva temporária, controle de estoque, emissão de ingresso | RF03, RF05 · RN01, RN03, RN05 |
| `payment` | Cobrança Pix (AbacatePay) e pedidos | RF04, RF07 · RN02 |
| `notification` | E-mails transacionais (handlers de evento; sem agregado) | RF05, RF09 |

Cada módulo segue a mesma anatomia interna — ver
[ADR 002](design/002-monolito-modular.md).

## Estrutura de arquivos (base)

Esqueleto do repositório `api.ludens`. O conteúdo interno de cada módulo é
definido pela sua spec.

```text
api.ludens/
├── src/
│   └── app/
│       ├── config.py           # Settings: pydantic-settings → .env.local
│       ├── database.py         # async engine + sessionmaker
│       ├── dependencies.py     # get_db
│       ├── main.py             # FastAPI app + lifespan (registra o relay)
│       ├── core/
│       │   ├── domain/                     # Model, AggregateRoot, DomainEvent, DomainError
│       │   ├── infrastructure/repositories/ # BaseRepository, AggregateRepository
│       │   └── shared/                      # respostas comuns, health, tradução de erro
│       ├── outbox/
│       │   ├── models.py       # Event
│       │   ├── relay.py        # relay in-process
│       │   └── registry.py     # event_type → [handler]
│       └── modules/
│           ├── identity/
│           ├── catalog/
│           ├── booking/
│           ├── payment/
│           └── notification/
├── migrations/versions/        # Alembic
├── docker/
│   ├── docker-compose.Development.yml
│   └── docker-compose.Production.yml
├── Dockerfile
├── pyproject.toml
└── .env.example
```

## Como a documentação evolui

1. Uma funcionalidade entra por **spec** (o que faz, regras, contrato).
2. Implementação no `api.ludens`.
3. A spec e o **contrato de integração** com o frontend
   ([modelo](integration/_template.md)) passam a refletir o código; este
   `overview.md` é atualizado se um padrão base mudar.

Nada abaixo do nível "padrão + lista de módulos" é assumido antes de existir
código.
