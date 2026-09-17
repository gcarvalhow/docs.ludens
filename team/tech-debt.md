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

| Item | Origem | Impacto | Proposta |
| --- | --- | --- | --- |
| `web.ludens` desalinhado com o contrato de `identity-auth` | Correções de 2026-09-17 no `api.ludens`: rotas do módulo `identity` migradas de `/auth/...` e `/users/...` para `/identity/...` e `/identity/users/...` (ver [`identity-auth/integration.md`](../specs/identity-auth/integration.md) e [`identity-auth/backend.md`](../specs/identity-auth/backend.md)) | Alto — login, cadastro, refresh e demais chamadas de auth do frontend quebram contra o backend atual, pois `web.ludens` ainda chama os caminhos antigos | Revisar `web.ludens` contra o contrato canônico atualizado e ajustar `src/routes/endpoints.ts` (e o cookie `Path` do refresh token) para os novos caminhos antes do próximo deploy conjunto |
| `web.ludens` sem páginas para os links de confirmação por e-mail de `identity-user-management` | `api.ludens#34`: o backend já envia e-mail de verdade (ACS/Mailpit) com links para `/confirmar-troca-de-email?token=...` e `/confirmar-exclusao-de-conta?token=...`, mas essas rotas de frontend não existem (ver [`notification-transactional-email/backend.md`](../specs/notification-transactional-email/backend.md)) | Alto — sem a página, quem clica no link não consegue confirmar a troca de e-mail nem a exclusão de conta; os fluxos ficam inacessíveis na prática | Criar as duas páginas em `web.ludens`: recebem `token` via query string no `GET` e chamam, via JS, `PATCH`/`DELETE` no backend (mesmo padrão de `/redefinir-senha`) |

Itens já resolvidos nesta preparação:

| Item | Origem | Resolução |
| --- | --- | --- |
| CODEOWNERS / handles desatualizados na separação de repos | Migração do monorepo | `CODEOWNERS` recriado em `api.ludens` (Igor) e `web.ludens` (Diego) com a org `gcarvalhow` (2026-09-01) |
| Meia-entrada exigindo documento de estudante contra a RN04 | ERS original | Sem código: virou critério de aceite da spec `booking-ticket-issuance` — ver [RN04](../requirements/business-rules.md#rn04--meia-entrada) |
