# Regras de Negócio (RN01–RN05)

> **Responsável:** Engenheiro de Requisitos (R) · **Aprovação:** Product Owner (A) · **Consulta:** QA (C)
> **Última revisão:** 2026-08-28 · **Status:** vigente (com itens pendentes de aprovação — ver abaixo)

Estas regras formalizam os exemplos citados no
[Acordo de Manutenibilidade](../team/acordo-de-manutenibilidade.md) como condição
de [Definition of Ready](../engineering/quality.md). Valores numéricos marcados
como *proposta* precisam da aprovação do Product Owner antes de entrarem em
vigor.

| Regra | Assunto | Status |
| --- | --- | --- |
| RN01 | Limite de ingressos por CPF | ⚠️ Proposta — pendente de aprovação do PO |
| RN02 | Política de reembolso | ⚠️ Proposta — pendente de aprovação do PO |
| RN03 | Expiração da reserva | ⚠️ Proposta — pendente de aprovação do PO |
| RN04 | Meia-entrada | ⚠️ Em revisão — divergência ERS × código ([ADR 004](../architecture/design/004-escopo-da-regra-de-meia-entrada.md)) |
| RN05 | Consistência de disponibilidade | ✅ Vigente |

## RN01 — Limite de ingressos por CPF

Cada CPF pode adquirir no máximo **6 ingressos por sessão**. Tentativas de
exceder o limite são bloqueadas na reserva (RF03).

**Status:** proposta do Engenheiro de Requisitos, pendente de aprovação do PO
(conforme RACI). O valor exato (6) é o item a validar.

## RN02 — Política de reembolso

- Cancelamento solicitado **até 48h** antes da sessão: reembolso **integral**.
- Entre **48h e 24h** antes: reembolso de **50%**.
- **Menos de 24h** antes: **sem reembolso**.

**Status:** proposta do Engenheiro de Requisitos, pendente de aprovação do PO
(conforme RACI). Prazos e percentuais são os itens a validar.

## RN03 — Expiração da reserva

Uma reserva não paga expira em **15 minutos**. Ao expirar, os ingressos retornam
automaticamente à disponibilidade da sessão.

**Status:** proposta do Engenheiro de Requisitos, pendente de aprovação do PO
(conforme RACI). O tempo (15 min) é o item a validar.

## RN04 — Meia-entrada

**Texto da ERS original:** o sistema apenas registra a intenção de compra de
meia-entrada no checkout. A validação do documento comprobatório é feita
presencialmente na entrada do evento e está fora do escopo do sistema.

**Divergência conhecida.** O backend (`api.ludens`) implementa hoje uma regra
mais forte: ao emitir um ingresso de meia-entrada, o número do documento de
estudante é **obrigatório** e a emissão é recusada sem ele
(`app/domain/tickets/models.py::issue_ticket`, com testes cobrindo). O
[Acordo de Manutenibilidade](../team/acordo-de-manutenibilidade.md) também trata
essa exigência como "a regra de domínio prioritária para os testes automatizados
do backend".

**Status:** em revisão. A decisão está registrada e recomendada em
[ADR 004 — Escopo da regra de meia-entrada](../architecture/design/004-escopo-da-regra-de-meia-entrada.md),
pendente de aprovação do PO.

## RN05 — Consistência de disponibilidade

O controle de disponibilidade de ingressos deve ser **atômico**: duas reservas
concorrentes nunca podem, juntas, exceder a capacidade da sessão, mesmo sob
acesso simultâneo.

**Status:** vigente. É a regra que sustenta o critério de sucesso "sem venda em
duplicidade" ([problema](../product/problem.md)).
