# ADR 004 — Escopo da regra de meia-entrada

> **Status:** proposto — pendente de aprovação do PO
> **Data:** 2026-08-28 · **Responsável:** Engenheiro de Requisitos · **Decisor:** Product Owner

## Contexto

Há uma divergência direta entre a Especificação de Requisitos e o código do
backend sobre o que o sistema faz quando o comprador escolhe meia-entrada:

| Fonte | O que diz |
| --- | --- |
| [ERS — RN04](../../requirements/business-rules.md#rn04--meia-entrada) (12/08/2026) | O sistema **apenas registra a intenção** de compra de meia-entrada. A validação do documento comprobatório é **presencial**, na entrada do evento, e está **fora do escopo** do sistema. |
| README do Processo 18 (§1.1) e [Acordo de Manutenibilidade](../../team/acordo-de-manutenibilidade.md) | "Ao selecionar 'Meia-Entrada', o sistema **deve exigir a comprovação/número do documento de estudante**." O Acordo chama isso de "a regra de domínio prioritária para os testes automatizados do backend". |
| Código do `api.ludens` (`app/domain/tickets/models.py::issue_ticket`) | Recusa a emissão de um ingresso `HALF` **sem** o número do documento de estudante (`DomainError`), e calcula metade do preço quando o documento é informado. Há 4 testes cobrindo (`tests/test_tickets.py`). |

Ou seja: o requisito mais recente (ERS) diz "fora de escopo", mas o requisito
mais antigo (README/Acordo) **e** o código implementado dizem "exigir o número".

## Decisão

**Manter a exigência do número do documento de estudante no checkout de
meia-entrada**, alinhando a ERS ao código e ao Acordo. Concretamente:

- Ao emitir um ingresso de meia-entrada, informar o **número do documento de
  estudante é obrigatório**; sem ele, a emissão é recusada (comportamento atual
  do `issue_ticket`).
- A **validação da autenticidade** do documento (conferir se o documento é
  verdadeiro e pertence ao comprador) permanece **presencial**, na entrada do
  evento — isso sim está fora do escopo do sistema.
- A ERS RN04 é **reescrita** para refletir essa distinção (número obrigatório no
  checkout; autenticidade conferida no local).

## Alternativas consideradas

- **Seguir a ERS literal** (só registrar a intenção, sem exigir número) —
  rejeitado: exigiria remover uma regra de domínio já implementada e testada,
  contrariando o Acordo assinado, que a define como regra crítica para os testes
  do backend.
- **Deixar como está, com a divergência documentada** — rejeitado: a ERS é a
  fonte de requisitos; manter uma contradição aberta prejudica o Definition of
  Ready.

## Consequências

- Nenhuma mudança de código no `api.ludens` — o comportamento atual passa a ser
  o oficialmente especificado.
- [`requirements/business-rules.md`](../../requirements/business-rules.md)
  atualiza o texto da RN04 e muda seu status de "em revisão" para "vigente"
  após a aprovação do PO.
- O critério de aceitação de RF03/RF05 relacionado a meia-entrada passa a citar
  a obrigatoriedade do número.

## Ações decorrentes

- [ ] PO aprova ou ajusta esta decisão
- [ ] Reescrever RN04 em `requirements/business-rules.md` e atualizar o status
- [ ] Revisar critérios de aceitação de RF03 e RF05 quanto à meia-entrada
