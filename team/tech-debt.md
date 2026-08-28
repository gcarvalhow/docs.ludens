# Gestão de Débito Técnico

> **Responsável:** Desenvolvedor Backend (Igor Thiago Seberino) · **Aprovação:** PO (Gabriel Carvalho) · **Última revisão:** 2026-08-28 · **Status:** vigente
> Versão operacional do [Acordo de Manutenibilidade §2](maintainability.md).

## Política de registro

Todo atalho técnico, pendência de refatoração ou *workaround* é registrado
**imediatamente** no backlog (GitHub Project da organização) como uma issue do
tipo **Débito Técnico**, contendo:

- descrição do problema;
- motivo do atalho;
- impacto estimado;
- proposta de solução.

Os repositórios de código trazem um template de issue "Débito Técnico" com esses
campos.

## Orçamento de ciclo

A equipe reserva **~15% do esforço de cada ciclo** para liquidar débitos
técnicos registrados.

## Priorização

Têm **prioridade máxima** e são tratados no ciclo seguinte (validados com o PO no
planejamento) os débitos que afetam:

- **Segurança** — dados de compradores ou de pagamento;
- **Desempenho** — ex.: consulta de disponibilidade de ingressos;
- **O trabalho de outro membro** do time.

## Débitos conhecidos hoje

| Débito | Origem | Impacto | Proposta |
| --- | --- | --- | --- |
| Backend exige documento de estudante na meia-entrada, contra a RN04 aprovada | [RN04](../requirements/business-rules.md#rn04--meia-entrada) | Código diverge de requisito aprovado pelo PO | Remover a obrigatoriedade do documento em `issue_ticket` e nos testes de `tests/test_tickets.py` |
| CODEOWNERS / handle desatualizados na separação de repos | Migração do monorepo | Revisão automática pode não acionar o dono certo | Recriar CODEOWNERS por repo com a org `gcarvalhow` |
