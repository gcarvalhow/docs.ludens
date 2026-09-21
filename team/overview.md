# Equipe e Papéis (Matriz RACI)

Papéis conforme a Matriz RACI do Processo 18 (elaboração 31/07/2026).

## Integrantes e papéis

* Product Owner (PO): Gabriel Carvalho ([@gabrielcarvallho](https://github.com/gabrielcarvallho))
* DevOps: Gabriel Carvalho ([@gabrielcarvallho](https://github.com/gabrielcarvallho))
* Engenheiro de Requisitos: Renato Colin Neto ([@RenatoColin](https://github.com/RenatoColin))
* Quality Assurance (QA): Adrian Cesar Gonçalves ([@adrian-cesar](https://github.com/adrian-cesar))
* Desenvolvedor Frontend: Diego Nessler ([@Diegonessler](https://github.com/Diegonessler))
* Desenvolvedor Backend: Igor Thiago Seberino ([@igorSeberino](https://github.com/igorSeberino))

## Matriz RACI

Legenda: R é Responsável (executa), A é Autoridade / Aprovador, C é
Consultado, I é Informado. Só entram na lista abaixo os papéis efetivamente
envolvidos em cada atividade.

* Definir o plano inicial de testes e os critérios de qualidade: Adrian (QA)
  é R; Gabriel (PO) é A; Renato (Eng. Requisitos) é C.
* Configurar o projeto Next.js (App Router) e prototipar as telas
  principais: Diego (Frontend) é R; Adrian (QA) é C; Gabriel (PO) é A.
* Organizar e priorizar o backlog do produto: Gabriel (PO) é R e A; Renato
  (Eng. Requisitos) é C.
* Configurar o ambiente Docker e a pipeline inicial de CI/CD: Gabriel
  (DevOps) é R; Igor (Backend) é A; Diego (Frontend) é I.
* Estruturar o backend (FastAPI) e modelar os módulos de domínio: Igor
  (Backend) é R; Renato (Eng. Requisitos) é A; Gabriel (DevOps) é C; Diego
  (Frontend) é I.
* Levantar e documentar os requisitos e regras de negócio: Renato (Eng.
  Requisitos) é R; Gabriel (PO) é A; Adrian (QA) é C.

## Backlog

O backlog do produto e as histórias de usuário vivem no **GitHub Project
[`@ludens`](https://github.com/orgs/gcarvalhow/projects/2)** da organização
`gcarvalhow`, que agrega issues dos repositórios `api.ludens`, `web.ludens` e
`docs.ludens`. Este repositório (`docs.ludens`) guarda os requisitos e os
padrões; os itens de backlog ficam só no Project.

Cada feature é fatiada em issues por superfície (Backend / Frontend), seguindo
[`ISSUE_TEMPLATE.md`](templates/ISSUE_TEMPLATE.md). Campos do Project: `Area`
(backend/frontend/infra/docs), `Priority` (3 é o mais importante, 0 é o
menos importante), `Issue Type` (feature/task/refactor/bug), `Status`
(Backlog/In Progress/Done). Labels: `module: *`, `N1`/`N2`/`N3`,
`débito técnico`.

## Qualidade: DoR e DoD

Versão operacional do [Acordo de Manutenibilidade §3](maintainability.md).

### Definition of Ready (DoR): pronto para desenvolver

Validado pelo QA em conjunto com o Engenheiro de Requisitos. Um card só
entra em desenvolvimento quando:

* [ ] A história está no formato *"Como [papel], eu quero [funcionalidade] para
      que [benefício]"*.
* [ ] Os critérios de aceitação são objetivos e verificáveis.
* [ ] Regras de negócio essenciais e exceções especificadas (ex.: limite de
      ingressos por CPF, política de reembolso, expiração da reserva; ver
      [regras de negócio](../product/overview.md#regras-de-negócio-rn01rn05)).
* [ ] Dependências técnicas mapeadas (ex.: gateway de pagamento, esquema do
      banco, e-mail de confirmação).
* [ ] Layout/protótipo da interface aprovado, quando aplicável.

### Definition of Done (DoD): pronto para entrega

Um card só é *Done* quando:

* [ ] O código segue o [guia de estilo](../backend/code-style.md).
* [ ] Passou por Code Review: **PR aprovado por, no mínimo, outro
      desenvolvedor**.
* [ ] A funcionalidade foi validada conforme a
      [estratégia de teste](../backend/testing.md), sem erros críticos.
* [ ] Os testes automatizados relevantes foram criados/atualizados e estão
      passando na pipeline.
* [ ] Código integrado em `master` sem quebrar o build.

### Relação com os templates de issue

Os repositórios de código (`api.ludens`, `web.ludens`) trazem o checklist de
DoR no template de história de usuário e o checklist de DoD no template de
Pull Request.

## Gestão de débito técnico

Versão operacional do [Acordo de Manutenibilidade §2](maintainability.md).

### Política de registro

Todo atalho técnico, pendência de refatoração ou *workaround* é registrado
**imediatamente** no backlog (GitHub Project da organização) como uma issue do
tipo **Débito Técnico**, contendo:

* descrição do problema;
* motivo do atalho;
* impacto estimado;
* proposta de solução.

Os repositórios `api.ludens` e `web.ludens` trazem o template de issue "Débito
Técnico" com esses campos, e a label `débito técnico` existe nos quatro repos.

### Orçamento de ciclo

A equipe reserva **cerca de 15% do esforço de cada ciclo** para liquidar débitos
técnicos registrados.

### Priorização

Têm **prioridade máxima** e são tratados no ciclo seguinte (validados com o PO no
planejamento) os débitos que afetam:

* **Segurança:** dados de compradores ou de pagamento;
* **Desempenho:** por exemplo, consulta de disponibilidade de ingressos;
* **O trabalho de outro membro** do time.

### Débitos conhecidos hoje

* **`web.ludens` desalinhado com o contrato de `identity-auth`.**
  Origem: correções de 2026-09-17 no `api.ludens`, que migraram as rotas do
  módulo `identity` de `/auth/...` e `/users/...` para `/identity/...` e
  `/identity/users/...`.
  Impacto: alto. Login, cadastro, refresh e demais chamadas de auth do
  frontend quebram contra o backend atual, pois `web.ludens` ainda chama os
  caminhos antigos.
  Proposta: revisar `web.ludens` contra o contrato canônico atualizado e
  ajustar `src/routes/endpoints.ts` (e o cookie `Path` do refresh token)
  para os novos caminhos antes do próximo deploy conjunto.
* **`web.ludens` sem páginas para os links de confirmação por e-mail de
  `identity-user-management`.**
  Origem: `api.ludens#34`. O backend já envia e-mail de verdade (ACS/Mailpit)
  com links para `/confirmar-troca-de-email?token=...` e
  `/confirmar-exclusao-de-conta?token=...`, mas essas rotas de frontend não
  existem.
  Impacto: alto. Sem a página, quem clica no link não consegue confirmar a
  troca de e-mail nem a exclusão de conta; os fluxos ficam inacessíveis na
  prática.
  Proposta: criar as duas páginas em `web.ludens`. Elas recebem `token` via
  query string no `GET` e chamam, via JS, `PATCH`/`DELETE` no backend (mesmo
  padrão de `/redefinir-senha`).

Itens já resolvidos nesta preparação:

* **CODEOWNERS / handles desatualizados na separação de repos.** Origem:
  migração do monorepo. Resolução: `CODEOWNERS` recriado em `api.ludens`
  (Igor) e `web.ludens` (Diego) com a org `gcarvalhow` (2026-09-01).
* **Meia-entrada exigindo documento de estudante contra a RN04.** Origem:
  ERS original. Resolução: sem código, virou critério de aceite da
  funcionalidade `booking-ticket-issuance`; ver
  [RN04](../product/overview.md#rn04-meia-entrada).

## Fluxo Trunk-Based Development

Versão operacional do [Acordo de Manutenibilidade §6](maintainability.md#6-fluxo-de-versionamento-e-pipeline-de-cicd).
`master` é a única branch de longa duração: sempre estável, integrável e apta
a implantação. Todo trabalho passa pelos passos abaixo, cada um com seu papel
da Matriz RACI acima. Os templates citados vivem em
[`team/templates/`](templates/).

### 1. Abrir a issue

Toda tarefa nasce como issue no GitHub Project
[`@ludens`](https://github.com/orgs/gcarvalhow/projects/2), escrita a partir de
[`ISSUE_TEMPLATE.md`](templates/ISSUE_TEMPLATE.md), que identifica o escopo
(história de usuário, critérios de aceitação, área, dependências técnicas). Só
entra em desenvolvimento depois de bater o
[Definition of Ready](#definition-of-ready-dor-pronto-para-desenvolver).

* R: quem vai implementar (Frontend ou Backend, conforme a área).
* A: PO, que também é quem organiza e prioriza o backlog.
* C: QA e Engenheiro de Requisitos, que validam o DoR em conjunto.

### 2. Criar a branch

Branch de curta duração a partir de `master`, no padrão
`feature/nome-da-funcionalidade` ou `fix/descricao-do-bug`, reintegrada em
poucos dias. Funcionalidade incompleta é protegida por feature flag quando
necessário.

* R: quem implementa (Frontend ou Backend).

### 3. Commitar

Commits pequenos e frequentes, seguindo
[`COMMIT_TEMPLATE.md`](templates/COMMIT_TEMPLATE.md) (padrão
[Conventional Commits](https://www.conventionalcommits.org/pt-br/): `feat`,
`fix`, `refactor`, `docs`, `test`, `chore`), mensagem em português no
imperativo.

* R: quem implementa.

### 4. Abrir o Pull Request

PR pequeno, escrito a partir de
[`PULL_REQUEST_TEMPLATE.md`](templates/PULL_REQUEST_TEMPLATE.md), referenciando
a issue e trazendo o checklist de
[Definition of Done](#definition-of-done-dod-pronto-para-entrega).

* R: autor do PR (quem implementou).
* C: QA, quando o PR mexe em fluxo coberto pelo roteiro de teste manual.

### 5. Code review e pipeline

Nenhum PR é aprovado sem pelo menos **1 aprovação de outro desenvolvedor**, e
sem a pipeline verde: lint do frontend sem erro crítico e testes automatizados
do backend passando, além do build Docker subindo sem erro.

* R: outro desenvolvedor (Frontend revisa Backend ou vice-versa, conforme o
  que o PR toca).
* A: mesmo revisor, que aprova ou pede mudança.

### 6. Merge em `master`

Só depois da aprovação e da pipeline verde. `master` nunca fica quebrada.

* R: autor do PR, depois de aprovado.

### 7. Débito técnico, quando houver atalho

Todo atalho assumido durante a implementação vira issue de **Débito Técnico**
imediatamente, seguindo a
[política de registro](#política-de-registro).

* R: quem assumiu o atalho.
* A: PO, que prioriza o pagamento do débito no planejamento do ciclo
  seguinte.
