# Acordo de Manutenibilidade e Engenharia de Software

> **Registro da entrega do Processo 18** (elaboração 31/07/2026). O texto
> integral assinado está preservado, sem edição, no `.docx` original em
> [`archive/`](../archive/). Esta página é um índice histórico das decisões
> tomadas ali, não uma transcrição. Cada seção aponta para a versão
> **operacional e viva** correspondente; em caso de divergência, a versão viva
> (alinhada ao código) prevalece. Ver a
> [convenção do repositório](../README.md). O conteúdo normativo vive nos
> links abaixo.

* **Projeto:** Plataforma de Venda de Ingressos para Teatro Comunitário (Processo 18), marca **Ludens**
* **Equipe:** Grupo 18
* **Arquitetura:** Monólito modular com Domain-Driven Design (DDD)
* **Data da elaboração:** 31/07/2026
* **Disciplina:** Manutenção e Melhoria de Software, 6º semestre do curso de Engenharia de Software
* **Instituição:** Centro Universitário Católica de Santa Catarina

## 1. Identificação da equipe e papéis

Ver [`team/overview.md`](overview.md): integrantes e Matriz RACI.

## 2. Processo e gestão de débito técnico

Ver [`team/overview.md`](overview.md#gestão-de-débito-técnico): política de
registro, orçamento de ciclo (cerca de 15%) e critérios de priorização.

## 3. Critérios de qualidade: DoR e DoD

Ver [`team/overview.md`](overview.md#qualidade-dor-e-dod): Definition of
Ready e Definition of Done.

## 4. Estratégia de testabilidade

Ver [`backend/testing.md`](../backend/testing.md): estratégia de testes
automatizados do backend.

## 5. Guia de estilo e padrões de código

Ver [`backend/code-style.md`](../backend/code-style.md): convenções de
código do backend (o guia do frontend virá em `frontend/`).

## 6. Fluxo de versionamento e pipeline de CI/CD

Ver [`team/overview.md`](overview.md#fluxo-trunk-based-development), que
cobre Trunk-Based Development, Conventional Commits e o gate de PR mapeados à
Matriz RACI, e [`backend/code-style.md`](../backend/code-style.md) para o
estado do lint/formatter no backend.

## 7. Compromisso da equipe

Todos os membros da equipe leram, concordam e se comprometem a seguir as
diretrizes estabelecidas neste documento para garantir a qualidade, a
manutenibilidade e a entrega sustentável do software.
