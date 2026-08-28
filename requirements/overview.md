# Especificação de Requisitos (ERS) — Visão Geral

> **Responsável:** Engenheiro de Requisitos (R) · **Aprovação:** Product Owner (A) · **Consulta:** QA (C)
> **Última revisão:** 2026-08-28 · **Status:** vigente
> Conversão da ERS do Processo 18 (elaboração original 12/08/2026 por Renato Colin Neto). Original em [`assets/originais/`](../assets/originais/).

Este é o índice da Especificação de Requisitos. O conteúdo está dividido em:

- [Requisitos funcionais](functional.md) — RF01–RF09
- [Regras de negócio](business-rules.md) — RN01–RN05
- [Glossário (linguagem ubíqua)](glossary.md)

## 1. Introdução

### 1.1. Objetivo

Especificar os requisitos funcionais, as regras de negócio e os requisitos não
funcionais da plataforma de venda de ingressos para um teatro comunitário,
servindo de base para a construção do backlog, a validação do
[Definition of Ready](../engineering/quality.md) e a modelagem dos módulos de
domínio (DDD) pelo time de backend.

### 1.2. Escopo

O sistema cobre o fluxo completo do comprador — busca de espetáculos, seleção de
sessão e ingresso, reserva, pagamento, confirmação e gestão de compras — e o
fluxo administrativo de cadastro de espetáculos e sessões pelo Product Owner.
Estão fora do escopo a bilheteria física presencial e a validação de
meia-entrada no local, que permanecem processos manuais do teatro. Ver
[escopo do produto](../product/scope.md).

### 1.3. Contextos de domínio (DDD)

Os requisitos estão organizados considerando os contextos de domínio abaixo, que
servem de referência para a modelagem dos módulos do monólito
([arquitetura](../architecture/overview.md)):

| Contexto | Responsabilidade |
| --- | --- |
| **Catálogo** | Espetáculos e sessões |
| **Bilheteria / Reserva** | Disponibilidade, reserva e emissão de ingressos |
| **Pagamento** | Integração com gateway e processamento de pedidos |
| **Conta** | Cadastro, autenticação e histórico do comprador |
| **Notificação** | Envio de e-mails transacionais |

## 2. Requisitos funcionais

Ver [functional.md](functional.md) — RF01 a RF09, cada um com história de
usuário, critérios de aceitação e dependências técnicas.

## 3. Regras de negócio

Ver [business-rules.md](business-rules.md) — RN01 a RN05. As regras RN01, RN02 e
RN03 têm valores numéricos que são **propostas do Engenheiro de Requisitos,
pendentes de aprovação do PO** antes de entrarem em vigor como critério de DoR.

## 4. Requisitos não funcionais (RNF)

> **Lacuna conhecida.** A seção de requisitos não funcionais da ERS original
> está **vazia**. As referências a `RNF02`, `RNF03` e `RNF04` (metas de
> desempenho, disponibilidade e usabilidade) aparecem na lista de
> [itens pendentes de validação com o PO](#7-itens-pendentes-de-validação-com-o-product-owner),
> mas os requisitos em si ainda não foram redigidos. **A fazer:** o Engenheiro
> de Requisitos redige os RNF e submete ao PO.

## 5. Dependências técnicas por requisito

> **Lacuna conhecida.** A ERS original prevê uma consolidação das dependências
> técnicas mapeadas requisito a requisito (para uso na priorização do backlog
> junto ao Backend e DevOps), mas a seção está **vazia**. As dependências já
> aparecem em cada RF individual em [functional.md](functional.md); falta o
> quadro consolidado.

## 6. Glossário (linguagem ubíqua)

Ver [glossary.md](glossary.md) — termos alinhados com o time de Backend para uso
consistente no código (DDD) e na documentação.

## 7. Itens pendentes de validação com o Product Owner

Conforme a [Matriz RACI](../team/overview.md), o Product Owner é o Aprovador (A)
dos requisitos e regras de negócio levantados pelo Engenheiro de Requisitos. Os
itens a seguir precisam de decisão do PO antes de entrarem no backlog como
critério de DoR:

- **RN01** — valor exato do limite de ingressos por CPF
- **RN02** — prazos e percentuais da política de reembolso
- **RN03** — tempo de expiração da reserva
- **RF04** — meios de pagamento suportados e gateway escolhido
- **RNF02, RNF03, RNF04** — metas de desempenho, disponibilidade e usabilidade
- **RN04 (escopo da meia-entrada)** — divergência entre a ERS e o código do
  backend, ver [ADR 004](../architecture/design/004-escopo-da-regra-de-meia-entrada.md)
