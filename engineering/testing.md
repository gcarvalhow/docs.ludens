# Estratégia de Testes

> **Responsável:** QA · **Aprovação:** PO · **Última revisão:** 2026-08-28 · **Status:** vigente
> Versão operacional do [Acordo de Manutenibilidade §4](../team/acordo-de-manutenibilidade.md).

## Foco primário — camada de domínio do backend

Os testes automatizados focam nas **regras de negócio da camada de domínio do
backend** (DDD):

- reserva e compra de ingressos;
- controle de disponibilidade de assentos ([RN05](../requirements/business-rules.md#rn05--consistência-de-disponibilidade));
- cálculo de valores (inteira/meia-entrada);
- processamento de pedidos.

**Regra crítica coberta hoje:** meia-entrada exige o número do documento de
estudante — `api.ludens`, `app/domain/tickets/models.py::issue_ticket`, com
testes em `tests/test_tickets.py` (inteira usa preço cheio; meia com documento
aplica metade; meia sem documento é recusada; meia com documento em branco é
recusada). Ver [RN04](../requirements/business-rules.md#rn04--meia-entrada) e
[ADR 004](../architecture/design/004-escopo-da-regra-de-meia-entrada.md).

## Roteiro de testes antes de cada entrega

Antes de cada entrega oficial, o QA executa um roteiro cobrindo o **fluxo
principal** e as regras críticas:

```text
busca do espetáculo → seleção da sessão e do ingresso → reserva → pagamento → confirmação
```

Casos obrigatórios no roteiro:
- Compra de inteira e de meia-entrada (com e sem documento).
- Reserva que expira e devolve os ingressos à disponibilidade.
- Duas compras concorrentes que, juntas, não podem exceder a capacidade.
- Falha de pagamento libera a reserva.

## Registro de bugs

Todo bug encontrado é documentado com:

| Campo | Conteúdo |
| --- | --- |
| Passos para reproduzir | Sequência numerada |
| Resultado esperado | O que deveria acontecer |
| Resultado obtido | O que aconteceu |
| Severidade | Alta / Média / Baixa |
| Evidência | Print, log ou stack trace |

**Bugs de alta severidade** (bloqueiam compra/pagamento ou expõem dados) impedem
o fechamento das tarefas e do ciclo.

## Ferramentas

- Backend: **Pytest** (`pytest -q`), rodando na pipeline.
- Frontend: lint como portão mínimo hoje; testes de componente a definir.
