# Regras de Negócio (RN01–RN05)

> **Status:** vigente — RN01–RN05 aprovadas pelo PO em 2026-08-28 · **Última revisão:** 2026-08-28
>
> **Papéis (RACI)**
>
> - **Responsável (R):** Renato Colin Neto — Engenheiro de Requisitos
> - **Aprovação (A):** Gabriel Carvalho — Product Owner
> - **Consulta (C):** Adrian Cesar Gonçalves — QA

Estas regras formalizam os exemplos citados no
[Acordo de Manutenibilidade](../team/maintainability.md) como condição
de [Definition of Ready](../team/quality.md). Todos os valores numéricos
foram aprovados pelo Product Owner em 2026-08-28.

| Regra | Assunto | Status |
| --- | --- | --- |
| RN01 | Limite de ingressos por CPF | ✅ Aprovada |
| RN02 | Política de reembolso | ✅ Aprovada |
| RN03 | Expiração da reserva | ✅ Aprovada |
| RN04 | Meia-entrada | ✅ Aprovada — vale o texto da ERS (ver [RN04](#rn04--meia-entrada)) |
| RN05 | Consistência de disponibilidade | ✅ Vigente |

## RN01 — Limite de ingressos por CPF

Cada CPF pode adquirir no máximo **6 ingressos por sessão**. Tentativas de
exceder o limite são bloqueadas na reserva (RF03).

**Status:** aprovada pelo PO em 2026-08-28 (limite de 6 ingressos por sessão).

## RN02 — Política de reembolso

- Cancelamento solicitado **até 48h** antes da sessão: reembolso **integral**.
- Entre **48h e 24h** antes: reembolso de **50%**.
- **Menos de 24h** antes: **sem reembolso**.

**Status:** aprovada pelo PO em 2026-08-28 (prazos de 48h/24h e percentuais de
100%/50%/0%).

## RN03 — Expiração da reserva

Uma reserva não paga expira em **15 minutos**. Ao expirar, os ingressos retornam
automaticamente à disponibilidade da sessão.

**Status:** aprovada pelo PO em 2026-08-28 (expiração em 15 minutos).

## RN04 — Meia-entrada

O sistema apenas **registra a intenção** de compra de meia-entrada no checkout.
A validação do documento comprobatório de estudante é feita **presencialmente**
na entrada do evento e está **fora do escopo do sistema** — a emissão do ingresso
não exige o número do documento.

**Decisão do PO (2026-08-28).** A ERS original chegou a descrever a
meia-entrada exigindo o número do documento de estudante. O PO decidiu **alinhar
à ERS acima**: a emissão do ingresso **não** exige o documento. Como o backend
ainda não tem código, isso não é débito técnico — é **critério de aceite** da
spec [`booking-ticket-issuance`](../specs/): a emissão de meia-entrada e seus
testes de domínio não podem exigir o número do documento.

**Status:** aprovada pelo PO em 2026-08-28. Ajuste no backend pendente.

## RN05 — Consistência de disponibilidade

O controle de disponibilidade de ingressos deve ser **atômico**: duas reservas
concorrentes nunca podem, juntas, exceder a capacidade da sessão, mesmo sob
acesso simultâneo.

**Status:** vigente. É a regra que sustenta o critério de sucesso "sem venda em
duplicidade" ([problema](../product/problem.md)).
