# Gestão de Débito Técnico

> **Responsável:** Desenvolvedor Backend · **Aprovação:** PO · **Última revisão:** 2026-08-28 · **Status:** vigente
> Versão operacional do [Acordo de Manutenibilidade §2](../team/acordo-de-manutenibilidade.md).

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
| RNF da ERS não redigidos (desempenho, disponibilidade, usabilidade) | [ERS §4](../requirements/overview.md#4-requisitos-não-funcionais-rnf) | Sem metas objetivas para DoR/DoD | Eng. de Requisitos redige e submete ao PO |
| Quadro consolidado de dependências técnicas por requisito ausente | [ERS §5](../requirements/overview.md#5-dependências-técnicas-por-requisito) | Priorização do backlog menos precisa | Consolidar a partir das dependências já listadas em cada RF |
| Divergência ERS RN04 × código da meia-entrada | [ADR 004](../architecture/design/004-escopo-da-regra-de-meia-entrada.md) | Contradição em requisito, afeta DoR | PO aprova o ADR 004; reescrever RN04 |
| Divergência TBD × `main`+`develop` | [ADR 003](../architecture/design/003-estrategia-de-branches.md) | Fluxo de trabalho ambíguo | PO aprova o ADR 003; remover `develop` |
| CODEOWNERS / handle desatualizados na separação de repos | Migração do monorepo | Revisão automática pode não acionar o dono certo | Recriar CODEOWNERS por repo com a org `gcarvalhow` |
