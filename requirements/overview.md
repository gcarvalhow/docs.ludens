# Especificação de Requisitos (ERS) — Visão Geral

> **Status:** vigente · **Última revisão:** 2026-08-28
>
> **Papéis (RACI)**
>
> - **Responsável (R):** Renato Colin Neto — Engenheiro de Requisitos
> - **Aprovação (A):** Gabriel Carvalho — Product Owner
> - **Consulta (C):** Adrian Cesar Gonçalves — QA

Este é o índice da Especificação de Requisitos. O conteúdo está dividido em:

- [Requisitos funcionais e não funcionais](functional.md) — RF01–RF09,
  RNF01–RNF06 e o quadro consolidado de dependências técnicas
- [Regras de negócio](business-rules.md) — RN01–RN05

## 1. Introdução

### 1.1. Objetivo

Especificar os requisitos funcionais, as regras de negócio e os requisitos não
funcionais da plataforma de venda de ingressos para um teatro comunitário,
servindo de base para a construção do backlog, a validação do
[Definition of Ready](../team/quality.md) e a modelagem dos módulos de
domínio (DDD) pelo time de backend.

### 1.2. Escopo

O sistema cobre o fluxo completo do comprador — busca de espetáculos, seleção de
sessão e ingresso, reserva, pagamento, confirmação e gestão de compras — e o
fluxo administrativo de cadastro de espetáculos e sessões pelo Product Owner.
Estão fora do escopo a bilheteria física presencial e a validação de
meia-entrada no local, que permanecem processos manuais do teatro. Ver
[escopo do produto](../product/scope.md).

## 2. Requisitos funcionais

Ver [functional.md](functional.md) — RF01 a RF09, cada um com história de
usuário, critérios de aceitação e dependências técnicas.

## 3. Regras de negócio

Ver [business-rules.md](business-rules.md) — RN01 a RN05, todas **aprovadas pelo
PO em 2026-08-28** (ver [§ 6](#6-decisões-do-product-owner)).

## 4. Requisitos não funcionais (RNF)

Redigidos em [functional.md § Requisitos não funcionais](functional.md#requisitos-não-funcionais):
RNF01 segurança e proteção de dados, RNF02 desempenho, RNF03 disponibilidade e
confiabilidade, RNF04 usabilidade e acessibilidade, RNF05 manutenibilidade,
RNF06 portabilidade e operação. Todos **aprovados pelo PO em 2026-08-28** (ver
[§ 6](#6-decisões-do-product-owner)).

## 5. Dependências técnicas por requisito

Quadro consolidado em
[functional.md § Dependências técnicas por requisito](functional.md#dependências-técnicas-por-requisito),
montado a partir das dependências listadas em cada RF. Serve de insumo para a
priorização do backlog junto ao Backend e ao DevOps.

## 6. Decisões do Product Owner

Conforme a [Matriz RACI](../team/overview.md), o Product Owner é o Aprovador (A)
dos requisitos e regras de negócio. Registro das decisões tomadas:

### 2026-08-28 — aprovação geral da ERS

O PO aprovou todos os requisitos funcionais (RF01–RF09), não funcionais
(RNF01–RNF06) e regras de negócio (RN01–RN05). Itens que estavam em aberto:

- **RN01** — limite de **6 ingressos por CPF** por sessão.
- **RN02** — reembolso **integral até 48h** antes da sessão, **50% entre 48h e
  24h**, **sem reembolso a menos de 24h**.
- **RN03** — reserva não paga **expira em 15 minutos**.
- **RN04 (meia-entrada)** — vale o texto da ERS: o sistema apenas registra a
  intenção de meia-entrada e **não exige** o número do documento de estudante na
  emissão. O backend, que hoje exige, será alinhado — débito técnico em
  [tech-debt](../team/tech-debt.md).
- **RF04 (pagamento)** — pagamento por **Pix** através do gateway **AbacatePay**.
- **RNF02–RNF04** — as metas numéricas de desempenho (p95 de 1–2 s),
  disponibilidade (99% ao mês) e usabilidade (fluxo em até 5 passos, WCAG 2.1
  AA) valem como critério de Definition of Ready.
