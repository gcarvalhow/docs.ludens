# Decisões de design (ADRs) — Backend

> **Última revisão:** 2026-08-28

Registro das decisões de arquitetura do backend (`api.ludens`). Cada ADR é um
arquivo `NNN-slug.md` com uma linha de **Status**:

- `proposto` — decidido no papel, ainda não implementado
- `implementado` — reflete o código
- `revertido` — abandonado; mantido como registro histórico

| ADR | Assunto | Status |
| --- | --- | --- |
| [001](001-outbox-in-process.md) | Padrão Outbox in-process para efeitos colaterais (sem broker) | proposto |
| [002](002-monolito-modular.md) | Monólito modular com DDD; um módulo por área de negócio | proposto |

Vários padrões vêm do backend da plataforma DOM Med (`api.hub.dommed`), que usa
monólito modular + DDD da mesma forma. A diferença assumida para o Ludens: **sem
Event Sourcing e sem broker de mensagens** — ver [ADR 001](001-outbox-in-process.md).
