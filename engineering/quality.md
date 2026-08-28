# Qualidade — DoR e DoD

> **Responsável:** QA (o DoR é validado com o Engenheiro de Requisitos) · **Aprovação:** PO
> **Última revisão:** 2026-08-28 · **Status:** vigente
> Versão operacional do [Acordo de Manutenibilidade §3](../team/acordo-de-manutenibilidade.md).

## Definition of Ready (DoR) — pronto para desenvolver

Validado pelo **QA** em conjunto com o **Engenheiro de Requisitos**. Um card só
entra em desenvolvimento quando:

- [ ] A história está no formato *"Como [papel], eu quero [funcionalidade] para
      que [benefício]"*.
- [ ] Os critérios de aceitação são objetivos e verificáveis.
- [ ] Regras de negócio essenciais e exceções especificadas (ex.: limite de
      ingressos por CPF, política de reembolso, expiração da reserva — ver
      [regras de negócio](../requirements/business-rules.md)).
- [ ] Dependências técnicas mapeadas (ex.: gateway de pagamento, esquema do
      banco, e-mail de confirmação).
- [ ] Layout/protótipo da interface aprovado, quando aplicável.

## Definition of Done (DoD) — pronto para entrega

Responsável: **QA**. Um card só é *Done* quando:

- [ ] O código segue o [guia de estilo](code-style.md).
- [ ] Passou por Code Review — **PR aprovado por, no mínimo, outro
      desenvolvedor**.
- [ ] Validado e testado conforme a [estratégia de testes](testing.md), sem erros
      críticos.
- [ ] Testes automatizados relevantes criados/atualizados e **passando na
      pipeline**.
- [ ] Código integrado em `main` sem quebrar o build.

## Relação com os templates de issue

Os repositórios de código (`api.ludens`, `web.ludens`) trazem o checklist de DoR
no template de história de usuário e o checklist de DoD no template de Pull
Request. Ver também
[`requirements/templates/requisito-funcional.template.md`](../requirements/templates/requisito-funcional.template.md).
