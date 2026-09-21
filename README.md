# docs.ludens

<p align="center">
  <img src=".github/assets/logo.png" alt="ludens" width="180">

  <h3 align="center">ludens</h3>

  <p align="center">
    Plataforma de Venda de Ingressos para Teatro Comunitário
  </p>
</p>

**Ludens** é a plataforma web de venda de ingressos de um teatro comunitário:
busca de espetáculos, reserva, compra e confirmação de ingressos. É o projeto
acadêmico do **Processo/Grupo 18** da disciplina de **Manutenção e Melhoria de
Software** (6º semestre do curso de Engenharia de Software) do Centro
Universitário Católica de Santa Catarina.

Este repositório é a **fonte de entrada do projeto**: a documentação de produto,
requisitos e padrões de engenharia vive centralizada aqui. Os repositórios de
código têm READMEs curtos que apontam para cá.

## Repositórios

* [`gcarvalhow/docs.ludens`](https://github.com/gcarvalhow/docs.ludens): fonte de entrada. Este repositório, produto, requisitos, RACI, padrões de engenharia.
* [`gcarvalhow/api.ludens`](https://github.com/gcarvalhow/api.ludens): backend. API FastAPI (monólito modular + DDD), Postgres, Docker.
* [`gcarvalhow/web.ludens`](https://github.com/gcarvalhow/web.ludens): frontend. Aplicação Next.js (App Router, TypeScript).

## Publicado vs. contexto de repo

`product/` e `backend/` são a **documentação publicada** (site Mintlify).
`team/` é **contexto de repo**: markdown normal, não navegável no site, usado
por quem trabalha no projeto.

## Produto (publicado)

* [Visão geral do produto](product/overview.md): por que a plataforma existe,
  critério de sucesso, escopo, regras de negócio (RN01–RN05) e premissas.
* [Requisitos funcionais e não funcionais](product/functional.md): RF01–RF09
  (histórias de usuário e critérios de aceitação), RNF01–RNF06 e o quadro
  consolidado de dependências técnicas.

## Arquitetura, backend `api.ludens` (publicado)

* [Visão geral da arquitetura](backend/overview.md): módulos, fluxos e schema (desenho, ainda não implementado).
* [Segurança](backend/security/): autenticação (JWT + refresh) e variáveis de ambiente.
* [Guia de estilo e código](backend/code-style.md): PEP 8, idioma do código, boas práticas de manutenibilidade.
* [Convenções de arquitetura](backend/conventions.md): padrão de código que não é formatação.
* [Estratégia de testes](backend/testing.md): o que e como testamos no backend.
* [Template de contrato de integração](backend/integration/_template.md): modelo backend → frontend.

## Time (contexto de repo)

* [Equipe e RACI](team/overview.md): papéis, integrantes, Matriz RACI,
  qualidade (DoR/DoD), gestão de débito técnico e o fluxo Trunk-Based
  Development mapeado a cada papel.
* [Acordo de Manutenibilidade](team/maintainability.md): registro histórico do acordo assinado do Processo 18 (a versão operacional vive nos links acima).
* [Templates de issue, PR e commit](team/templates/): usados no fluxo TBD.
* [Specs de feature](specs/): produto, lógica de negócio e (quando implementada)
  contrato e código de cada feature, uma pasta por `[domínio]-[conceito]`.
  Convenção em [`specs/README.md`](specs/README.md).

**Convenção.** Este repositório descreve o projeto; o código real e o
comportamento observado têm prioridade sobre o que está escrito aqui se
divergirem. Ao encontrar uma divergência, corrija o documento, não repita a
informação desatualizada. Toda mudança em documento vigente entra por Pull
Request com revisão, igual a código. Os arquivos `.docx`/`.xlsx` originais das
entregas da disciplina estão preservados em [`archive/`](archive/)
apenas como registro histórico.
