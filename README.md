# Ludens — Documentação

**Ludens** é a plataforma web de venda de ingressos de um teatro comunitário —
busca de espetáculos, reserva, compra e confirmação de ingressos. É o projeto
acadêmico do **Processo/Grupo 18** da disciplina de Engenharia de Software do
Centro Universitário Católica de Santa Catarina. Arquitetura de **monólito
modular** orientada a **Domain-Driven Design (DDD)**, backend em FastAPI,
frontend em React, execução em Docker.

Este repositório é a **fonte de entrada do projeto**: a documentação de produto,
requisitos, arquitetura e padrões de engenharia vive centralizada aqui. Os
repositórios de código têm READMEs curtos que apontam para cá.

## Repositórios

| Repositório | Papel | Conteúdo |
| --- | --- | --- |
| [`gcarvalhow/docs.ludens`](https://github.com/gcarvalhow/docs.ludens) | Fonte de entrada | Este repositório — produto, requisitos, arquitetura, RACI, padrões |
| [`gcarvalhow/api.ludens`](https://github.com/gcarvalhow/api.ludens) | Backend | API FastAPI (monólito modular + DDD), Postgres, Docker |
| [`gcarvalhow/web.ludens`](https://github.com/gcarvalhow/web.ludens) | Frontend | Aplicação React (Vite) |

## Produto

- [Problema](product/problem.md) — por que a plataforma existe (ingresso vendido em duplicidade na bilheteria e no site) e o que define sucesso
- [Escopo](product/scope.md) — o que está dentro e fora; níveis de entrega N1 (MVP) / N2 / N3; premissas

## Requisitos

- [Visão geral da ERS](requirements/overview.md) — objetivo, escopo, contextos de domínio (DDD); lacunas em aberto (RNF)
- [Requisitos funcionais](requirements/functional.md) — RF01–RF09, com histórias de usuário e critérios de aceitação
- [Regras de negócio](requirements/business-rules.md) — RN01–RN05, cada uma com status de aprovação
- [Glossário](requirements/glossary.md) — linguagem ubíqua do domínio (DDD)

## Arquitetura

- [Visão geral](architecture/overview.md) — monólito modular + DDD, bounded contexts, stack, mapa dos repositórios
- [Decisões de design (ADRs)](architecture/design/) — decisões registradas, cada uma com status

## Time

- [Equipe e RACI](team/overview.md) — papéis, integrantes, Matriz RACI, contatos GitHub
- [Acordo de Manutenibilidade](team/acordo-de-manutenibilidade.md) — o acordo assinado do Processo 18 (registro da entrega; a versão operacional vive em `engineering/`)

## Engenharia

- [Guia de estilo e código](engineering/code-style.md) — PEP 8/Ruff, ESLint/Prettier, idioma, boas práticas de manutenibilidade
- [Fluxo Git](engineering/git-flow.md) — estratégia de branches, Conventional Commits, fluxo de Pull Request e code review
- [CI/CD](engineering/ci-cd.md) — pipeline de lint e testes, Docker como padrão de execução, portões de merge
- [Qualidade — DoR e DoD](engineering/quality.md) — Definition of Ready e Definition of Done
- [Estratégia de testes](engineering/testing.md) — foco no domínio do backend, roteiro do fluxo principal, registro de bugs
- [Gestão de débito técnico](engineering/tech-debt.md) — política de registro, orçamento de ciclo, priorização

---

**Convenção.** Este repositório descreve o projeto; o código real e o
comportamento observado têm prioridade sobre o que está escrito aqui se
divergirem. Ao encontrar uma divergência, corrija o documento — não repita a
informação desatualizada. Toda mudança em documento vigente entra por Pull
Request com revisão, igual a código. Os arquivos `.docx`/`.xlsx` originais das
entregas da disciplina estão preservados em [`assets/originais/`](assets/originais/)
apenas como registro histórico.
