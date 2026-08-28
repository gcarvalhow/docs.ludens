# ADR 001 — Monólito modular com DDD

> **Status:** aceito
> **Data:** 2026-07-31 · **Responsável:** Desenvolvedor Backend · **Decisor:** Engenheiro de Requisitos / PO
> Formaliza a decisão já registrada no [Acordo de Manutenibilidade](../../team/acordo-de-manutenibilidade.md) e no escopo do Processo 18.

## Contexto

A plataforma cobre cinco contextos de domínio distintos (Catálogo, Bilheteria/
Reserva, Pagamento, Conta, Notificação) que evoluem de forma relativamente
independente. É um projeto acadêmico com equipe pequena (6 integrantes) e ciclo
curto.

- Microsserviços dariam isolamento forte, mas introduzem complexidade
  operacional desproporcional para o estágio e o objetivo didático — múltiplos
  deploys, service discovery, transações distribuídas.
- Um monólito sem estrutura interna criaria acoplamento entre contextos:
  qualquer mudança poderia afetar as demais e o código cresceria sem fronteiras.

## Decisão

A API (`api.ludens`) é um **único serviço deployável**, organizado internamente
em **módulos isolados por bounded context**. Cada módulo encapsula seu próprio
domínio: roteamento (camada de apresentação), casos de uso (aplicação), acesso a
dados (infraestrutura) e as regras de negócio (domínio).

- As **regras de negócio vivem na camada de domínio**, sem conhecer HTTP, banco
  ou frameworks. As rotas apenas orquestram.
- Módulos **não importam o domínio interno uns dos outros** — a comunicação é por
  contrato explícito (função pública exportada) ou evento.
- A estrutura já está exemplificada no módulo `tickets`
  (`app/domain/tickets/`, `app/main.py`).

## Alternativas consideradas

- **Microsserviços** — descartado: complexidade operacional sem retorno no
  escopo acadêmico.
- **Monólito em camadas técnicas (sem módulos de domínio)** — descartado: não
  isola os contextos, acoplamento cresce com o tempo.

## Consequências

- Deploy simples: um container, sem orquestração entre serviços.
- Fronteiras explícitas: a anatomia interna padrão torna previsível onde cada
  coisa vive.
- Regras de domínio testáveis isoladamente — é o foco da
  [estratégia de testes](../../engineering/testing.md).
- Não é possível escalar um contexto individualmente sem escalar todo o serviço
  — aceitável no escopo atual.

## Ações decorrentes

- [x] Estrutura de pastas do `api.ludens` seguindo a anatomia módulo → domain /
  application / infrastructure / api
- [ ] `architecture/overview.md` mantém a lista de módulos atualizada conforme o
  código evolui
