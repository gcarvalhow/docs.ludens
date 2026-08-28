# Design 002 — Monólito modular

> **Status:** proposto · **Última revisão:** 2026-08-28
> Padrão herdado do backend da plataforma DOM Med (`api.hub.dommed`), que usa a
> mesma abordagem. Adaptado ao domínio do Ludens.

## Contexto

O Ludens precisa suportar áreas de negócio que evoluem de forma independente —
identidade, catálogo, reserva, pagamento e notificação. Microsserviços resolveriam o
isolamento, mas introduzem complexidade operacional desproporcional para o
estágio e o objetivo didático do projeto: múltiplos deploys independentes,
service discovery, observabilidade distribuída, transações entre serviços. O
ganho real só aparece quando times e volumes são grandes o suficiente para
amortizá-la.

Um monólito sem estrutura interna, por outro lado, criaria acoplamento entre
áreas: uma mudança em pagamento poderia afetar reserva, e o código cresceria sem
fronteiras claras.

## Decisão

A API é um **único serviço deployável**, mas internamente organizado em módulos
isolados. Cada módulo encapsula seu próprio domínio — roteamento, lógica de
aplicação, acesso a dados, schemas e eventos — sem dependências cruzadas entre
módulos.

Módulos previstos (nome em **inglês**). O domínio interno de cada um — agregados,
rotas, eventos — é definido pela sua **spec**, quando a funcionalidade for
construída; não é assumido aqui.

- **`identity/`** — cadastro, autenticação e histórico do comprador. Cobre RF06, RF09.
- **`catalog/`** — espetáculos e sessões; disponibilidade em tempo real. Cobre RF01, RF02, RF08.
- **`booking/`** — reserva temporária, controle de disponibilidade e emissão de
  ingresso. Cobre RF03, RF05; aplica RN01, RN03, RN05.
- **`payment/`** — cobrança Pix (AbacatePay) e pedidos. Cobre RF04, RF07; aplica RN02.
- **`notification/`** — e-mails transacionais (handlers de evento; sem agregado).
  Cobre RF05, RF09.

Todo módulo segue a mesma anatomia interna:

```text
modulo/
├── router.py               ← agrega os sub-roteadores; único ponto de entrada para o main.py
├── dependencies.py         ← injeção de dependência específica do módulo, exportada para outros
├── api/
│   └── routers/            ← endpoints HTTP organizados por recurso
├── domain/
│   ├── aggregates/         ← aggregate root: lógica de negócio e emissão de eventos de domínio
│   ├── entities/           ← entidades filhas do agregado, quando existem
│   ├── value_objects/      ← value objects, quando existem
│   ├── enumerations/       ← enums do domínio
│   └── events/             ← classes de evento de domínio publicadas por este módulo
├── infrastructure/
│   ├── repositories/       ← persistência via SQLAlchemy async
│   └── services/           ← integrações externas, quando existem (ex.: gateway de pagamento)
└── application/
    ├── schemas/            ← DTOs de request e response (Pydantic)
    └── usecases/           ← orquestração: valida, chama domínio, persiste, retorna resposta
```

`entities/`, `value_objects/` e `services/` são subpastas opcionais — só existem
nos módulos que precisam delas. `aggregates/`, `enumerations/`, `events/`,
`repositories/`, `schemas/` e `usecases/` são pacotes (com `__init__.py`
reexportando), não arquivos únicos.

A separação entre `usecases/` e `domain/` é intencional: o domínio não conhece
HTTP nem banco de dados — só regras de negócio. Quem coordena a persistência é o
usecase.

O `src/app/main.py` registra os roteadores de cada módulo de forma explícita —
não há descoberta automática. Todo módulo novo precisa ser incluído com
`app.include_router(...)`.

Módulos não importam o domínio interno uns dos outros. A comunicação acontece de
duas formas:

- **Eventos no outbox** (ver [Design 001](001-outbox-in-process.md)) — para
  efeitos assíncronos. O agregado produz um evento, o repositório o persiste na
  tabela `events`, o relay chama o handler in-process. Ex. (a formalizar em
  spec): `payment` emite um evento de pagamento confirmado; `notification` reage
  enviando o e-mail com o ingresso.
- **Dependências explícitas exportadas** — para consultas síncronas necessárias
  numa operação. Uma função pública que outro módulo importa e chama na mesma
  requisição. Ex. (a formalizar em spec): `identity` exporta o "comprador
  autenticado" para todo router autenticado; `catalog` exporta uma referência de
  sessão para o `booking` validar e ler capacidade.

O contrato entre módulos, num caso ou no outro, é sempre uma superfície pública e
explícita — nunca um import direto do `domain`/`infrastructure` interno de outro
módulo.

Para adicionar um novo módulo:

1. Criar o diretório em `src/app/modules/<nome>/` seguindo a anatomia acima.
2. Definir seus eventos de domínio em `domain/events/`.
3. Expor um `router.py` que agrega seus sub-roteadores.
4. Registrar o roteador em `src/app/main.py`.

Nenhum módulo existente deve ser modificado.

## Consequências

- **Deploy simples**: um único contêiner, sem orquestração entre serviços.
- **Isolamento real**: uma mudança em um módulo não toca os demais.
- **Fronteiras explícitas**: a anatomia interna padrão torna previsível onde cada
  coisa vive em qualquer módulo.
- **Escala limitada**: não é possível escalar um módulo individualmente sem
  escalar todo o serviço — aceitável no volume esperado.
