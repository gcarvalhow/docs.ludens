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

Os repositórios `api.ludens` e `web.ludens` trazem o template de issue "Débito
Técnico" com esses campos, e a label `débito técnico` existe nos quatro repos.

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

Nenhum débito técnico em aberto — não há código implementado ainda.

Itens já resolvidos nesta preparação:

| Item | Origem | Resolução |
| --- | --- | --- |
| CODEOWNERS / handles desatualizados na separação de repos | Migração do monorepo | `CODEOWNERS` recriado em `api.ludens` (Igor) e `web.ludens` (Diego) com a org `gcarvalhow` (2026-09-01) |
| Meia-entrada exigindo documento de estudante contra a RN04 | ERS original | Sem código: virou critério de aceite da spec `booking-ticket-issuance` — ver [RN04](../requirements/business-rules.md#rn04--meia-entrada) |
