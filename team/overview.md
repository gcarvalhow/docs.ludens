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
[Definition of Ready](quality.md#definition-of-ready-dor-pronto-para-desenvolver).

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
[Definition of Done](quality.md#definition-of-done-dod-pronto-para-entrega).

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
[política de registro](tech-debt.md#política-de-registro).

* R: quem assumiu o atalho.
* A: PO, que prioriza o pagamento do débito no planejamento do ciclo
  seguinte.
