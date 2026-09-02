# Ludens — Documentação

**Ludens** é a plataforma web de venda de ingressos de um teatro comunitário —
busca de espetáculos, reserva, compra e confirmação de ingressos. É o projeto
acadêmico do **Processo/Grupo 18** da disciplina de **Manutenção e Melhoria de
Software** (6º semestre do curso de Engenharia de Software) do Centro
Universitário Católica de Santa Catarina.

Este repositório é a **fonte de entrada do projeto**: a documentação de produto,
requisitos e padrões de engenharia vive centralizada aqui. Os repositórios de
código têm READMEs curtos que apontam para cá.

## Repositórios

| Repositório | Papel | Conteúdo |
| --- | --- | --- |
| [`gcarvalhow/docs.ludens`](https://github.com/gcarvalhow/docs.ludens) | Fonte de entrada | Este repositório — produto, requisitos, RACI, padrões de engenharia |
| [`gcarvalhow/api.ludens`](https://github.com/gcarvalhow/api.ludens) | Backend | API FastAPI (monólito modular + DDD), Postgres, Docker |
| [`gcarvalhow/web.ludens`](https://github.com/gcarvalhow/web.ludens) | Frontend | Aplicação Next.js (App Router, TypeScript) |
| [`gcarvalhow/team.ludens`](https://github.com/gcarvalhow/team.ludens) | Ferramentaria | Plugin Claude Code — papéis do time, pipeline de spec, fluxo TBD |

## Produto

- [Problema](product/problem.md) — por que a plataforma existe (ingresso vendido em duplicidade na bilheteria e no site) e o que define sucesso
- [Escopo](product/scope.md) — o que está dentro e fora; níveis de entrega N1 (MVP) / N2 / N3; premissas

## Requisitos

- [Visão geral da ERS](requirements/overview.md) — objetivo, escopo e índice das seções da especificação
- [Requisitos funcionais e não funcionais](requirements/functional.md) — RF01–RF09 (histórias de usuário e critérios de aceitação), RNF01–RNF06 e o quadro consolidado de dependências técnicas
- [Regras de negócio](requirements/business-rules.md) — RN01–RN05, cada uma com status de aprovação

## Specs de feature

- [Pipeline de specs](specs/README.md) — como cada feature vai de ideia a issues
  para a equipe: `spec.md` → `logic.md` → `integration.md` →
  `implementation-spec.md`, operado pelo plugin
  [`gcarvalhow/team.ludens`](https://github.com/gcarvalhow/team.ludens)

## Backend (`api.ludens`)

- [Visão geral da arquitetura](backend/overview.md) — módulos, fluxos e schema (desenho, ainda não implementado)
- [Decisões de design (ADRs)](backend/design/) — outbox in-process, monólito modular
- [Segurança](backend/security/) — autenticação (JWT + refresh) e variáveis de ambiente
- [Testes e CI](backend/testing.md) — estratégia de testes do domínio e pipeline de integração contínua
- [Guia de estilo e código](backend/code-style.md) — PEP 8/Ruff, idioma do código, boas práticas de manutenibilidade
- [Template de contrato de integração](backend/integration/_template.md) — modelo backend → frontend

## Time

- [Equipe e RACI](team/overview.md) — papéis, integrantes, Matriz RACI, contatos GitHub
- [Acordo de Manutenibilidade](team/maintainability.md) — o acordo assinado do Processo 18 (registro da entrega; a versão operacional vive em `backend/` e `team/`)
- [Qualidade — DoR e DoD](team/quality.md) — Definition of Ready e Definition of Done
- [Gestão de débito técnico](team/tech-debt.md) — política de registro, orçamento de ciclo, priorização
- [Ambiente de desenvolvimento](team/development.md) — como subir o projeto localmente

---

**Convenção.** Este repositório descreve o projeto; o código real e o
comportamento observado têm prioridade sobre o que está escrito aqui se
divergirem. Ao encontrar uma divergência, corrija o documento — não repita a
informação desatualizada. Toda mudança em documento vigente entra por Pull
Request com revisão, igual a código. Os arquivos `.docx`/`.xlsx` originais das
entregas da disciplina estão preservados em [`archive/`](archive/)
apenas como registro histórico.
