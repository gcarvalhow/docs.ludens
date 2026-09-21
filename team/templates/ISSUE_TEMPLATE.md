# Template de Issue

Preencha isto antes de abrir a issue no GitHub Project
[`@ludens`](https://github.com/orgs/gcarvalhow/projects/2). Uma issue só entra
em desenvolvimento quando bate o
[Definition of Ready](../quality.md#definition-of-ready-dor-pronto-para-desenvolver)
completo.

## Título

Curto, no formato `<verbo no infinitivo> <o quê>` (ex.: "Implementar busca de
espetáculos por gênero").

## História de usuário

Como [papel], eu quero [funcionalidade] para que [benefício].

## Critérios de aceitação

Lista objetiva e verificável do que precisa ser verdade pra considerar a
tarefa pronta. Cada item testável isoladamente.

## Regras de negócio e exceções

Regras essenciais que a tarefa precisa respeitar (ex.: limite de ingressos por
CPF, política de reembolso, expiração da reserva) e os casos de exceção
esperados. Referencie a regra formal em
[`product/overview.md`](../../product/overview.md#regras-de-negócio-rn01rn05)
quando existir.

## Dependências técnicas

O que a tarefa precisa que já exista (gateway de pagamento, esquema de banco,
envio de e-mail, endpoint de outro módulo).

## Escopo

Campos do Project
[`@ludens`](https://github.com/orgs/gcarvalhow/projects/2):

* **Área:** `backend` | `frontend` | `infra` | `docs`
* **Tipo:** `feature` | `task` | `refactor` | `bug`
* **Prioridade:** `0` a `3` (3 é a mais importante)
* **Status:** `Backlog` (padrão ao abrir)

## Labels

* `module: <identity|catalog|booking|payment|notification>`, quando a issue
  pertence a um módulo específico.
* `N1` | `N2` | `N3`, nível de entrega da feature, quando aplicável.
* `débito técnico`, quando a issue registra um atalho assumido conforme a
  [política de registro](../tech-debt.md#política-de-registro).
