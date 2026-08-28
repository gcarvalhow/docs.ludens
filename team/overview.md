# Equipe e Papéis (Matriz RACI)

> **Responsável:** Product Owner · **Última revisão:** 2026-08-28 · **Status:** vigente
> Papéis conforme a Matriz RACI do Processo 18 (elaboração 31/07/2026). Planilha original em [`assets/originais/`](../assets/originais/).

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
| Configurar o projeto React (Vite) e prototipar as telas principais | **C** | **R** | **A** | — | — | — |
| Organizar e priorizar o backlog do produto | — | — | **R/A** | — | — | **C** |
| Configurar o ambiente Docker e a pipeline inicial de CI/CD | — | **I** | — | **R** | **A** | — |
| Estruturar o backend (FastAPI) e modelar os módulos de domínio | — | **I** | — | **C** | **R** | **A** |
| Levantar e documentar os requisitos e regras de negócio | **C** | — | **A** | — | — | **R** |

## Responsáveis por área da documentação

| Área | Responsável (R) | Aprovador (A) |
| --- | --- | --- |
| [Produto](../product/) e [backlog](https://github.com/orgs/gcarvalhow/projects) | PO | PO |
| [Requisitos](../requirements/) e regras de negócio | Eng. de Requisitos | PO |
| [Arquitetura](../architecture/) e módulos de domínio | Dev. Backend | Eng. de Requisitos |
| [Estratégia de testes](../engineering/testing.md) e [DoR/DoD](../engineering/quality.md) | QA | PO |
| [Fluxo Git](../engineering/git-flow.md) e [CI/CD](../engineering/ci-cd.md) | DevOps | PO |
| [Gestão de débito técnico](../engineering/tech-debt.md) | Dev. Backend | PO |

## Backlog

O backlog do produto e as histórias de usuário vivem num **GitHub Project da
organização `gcarvalhow`**, que agrega issues dos repositórios `api.ludens` e
`web.ludens`. Este repositório (`docs.ludens`) guarda apenas os requisitos e os
padrões — não os itens de backlog.
