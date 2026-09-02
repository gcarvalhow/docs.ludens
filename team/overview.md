# Equipe e Papéis (Matriz RACI)

> **Responsável:** Product Owner (Gabriel Carvalho) · **Última revisão:** 2026-08-28 · **Status:** vigente
> Papéis conforme a Matriz RACI do Processo 18 (elaboração 31/07/2026). Planilha original em [`archive/`](../archive/).

## Integrantes e papéis

| Papel | Integrante | GitHub |
| --- | --- | --- |
| Product Owner (PO) | Gabriel Carvalho | [@gabrielcarvallho](https://github.com/gabrielcarvallho) |
| DevOps | Gabriel Carvalho | [@gabrielcarvallho](https://github.com/gabrielcarvallho) |
| Engenheiro de Requisitos | Renato Colin Neto | [@RenatoColin](https://github.com/RenatoColin) |
| Quality Assurance (QA) | Adrian Cesar Gonçalves | [@adrian-cesar](https://github.com/adrian-cesar) |
| Desenvolvedor Frontend | Diego Nessler | [@Diegonessler](https://github.com/Diegonessler) |
| Desenvolvedor Backend | Igor Thiago Seberino | [@igorSeberino](https://github.com/igorSeberino) |

> Gabriel Carvalho acumula os papéis de **Product Owner** e **DevOps**.

A organização GitHub do projeto é **[`gcarvalhow`](https://github.com/gcarvalhow)**.
(Nota: o handle pessoal do mantenedor é `@gabrielcarvallho`; a organização é
`gcarvalhow` — grafias parecidas, entidades diferentes.)

## Matriz RACI

Legenda: **R** = Responsável (executa) · **A** = Autoridade / Aprovador · **C** =
Consultado · **I** = Informado.

| Atividade | Adrian (QA) | Diego (Frontend) | Gabriel (PO) | Gabriel (DevOps) | Igor (Backend) | Renato (Eng. Requisitos) |
| --- | :---: | :---: | :---: | :---: | :---: | :---: |
| Definir o plano inicial de testes e os critérios de qualidade | **R** | — | **A** | — | — | **C** |
| Configurar o projeto Next.js (App Router) e prototipar as telas principais | **C** | **R** | **A** | — | — | — |
| Organizar e priorizar o backlog do produto | — | — | **R/A** | — | — | **C** |
| Configurar o ambiente Docker e a pipeline inicial de CI/CD | — | **I** | — | **R** | **A** | — |
| Estruturar o backend (FastAPI) e modelar os módulos de domínio | — | **I** | — | **C** | **R** | **A** |
| Levantar e documentar os requisitos e regras de negócio | **C** | — | **A** | — | — | **R** |

## Responsáveis por área da documentação

| Área | Responsável (R) | Aprovador (A) |
| --- | --- | --- |
| [Produto](../product/) e [backlog](https://github.com/orgs/gcarvalhow/projects) | PO (Gabriel Carvalho) | PO (Gabriel Carvalho) |
| [Requisitos](../requirements/) e regras de negócio | Eng. de Requisitos (Renato Colin Neto) | PO (Gabriel Carvalho) |
| [Arquitetura do backend](../backend/) e [ADRs](../backend/design/) | Dev. Backend (Igor Thiago Seberino) | Eng. de Requisitos (Renato Colin Neto) |
| [Testes e CI](../backend/testing.md) e [DoR/DoD](quality.md) | QA (Adrian Cesar Gonçalves) | PO (Gabriel Carvalho) |
| [Ambiente de desenvolvimento](development.md) | DevOps (Gabriel Carvalho) | PO (Gabriel Carvalho) |
| [Gestão de débito técnico](tech-debt.md) | Dev. Backend (Igor Thiago Seberino) | PO (Gabriel Carvalho) |
| [Ferramentaria do time (`team.ludens`)](https://github.com/gcarvalhow/team.ludens) e [pipeline de specs](../specs/README.md) | DevOps / PO (Gabriel Carvalho) | PO (Gabriel Carvalho) |

## Backlog

O backlog do produto e as histórias de usuário vivem no **GitHub Project
[`@ludens`](https://github.com/orgs/gcarvalhow/projects/2)** da organização
`gcarvalhow`, que agrega issues dos repositórios `api.ludens`, `web.ludens`,
`docs.ludens` e `team.ludens`. Este repositório (`docs.ludens`) guarda os
requisitos, os padrões e as **specs de feature** (`specs/`) — não os itens de
backlog.

Cada feature é fatiada em issues por responsável (Backend / Frontend / QA) a
partir da sua `implementation-spec.md`, pelo fluxo `/team-ludens:tbd-start` do
plugin. Campos do Project: `Area` (backend/frontend/infra/docs), `Priority`
(3 = mais importante … 0), `Issue Type` (feature/task/refactor/bug), `Status`
(Backlog/In Progress/Done). Labels: `module: *`, `N1`/`N2`/`N3`,
`débito técnico`.
