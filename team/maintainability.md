# Acordo de Manutenibilidade e Engenharia de Software

> **Registro da entrega do Processo 18.** Conversão fiel do documento assinado
> (elaboração 31/07/2026). O `.docx` original está em
> [`archive/`](../archive/).
>
> Este é o documento fundacional. A **versão operacional e viva** de cada seção
> está em [`backend/`](../backend/), [`team/`](.) e
> [`team/overview.md`](overview.md) — se houver divergência, a versão viva
> (alinhada ao código) prevalece; ver a
> [convenção do repositório](../README.md).

- **Projeto:** Plataforma de Venda de Ingressos para Teatro Comunitário (Processo 18) — marca **Ludens**
- **Equipe:** Grupo 18
- **Arquitetura:** Monólito modular com Domain-Driven Design (DDD)
- **Data da elaboração:** 31/07/2026
- **Disciplina:** Manutenção e Melhoria de Software — 6º semestre do curso de Engenharia de Software
- **Instituição:** Centro Universitário Católica de Santa Catarina

---

## 1. Identificação da equipe e papéis

Distribuição de papéis conforme a Matriz RACI do Processo 18. Gabriel Carvalho
acumula os papéis de **Product Owner** e **DevOps**.

Ver a tabela completa de integrantes e a Matriz RACI em
[`team/overview.md`](overview.md).

## 2. Processo e gestão de débito técnico

**Responsável:** Desenvolvedor Backend (Igor Thiago Seberino).

### 2.1. Política de registro de débito técnico

Todo atalho técnico, pendência de refatoração ou solução temporária (*workaround*)
assumida durante o desenvolvimento deve ser registrada **imediatamente** no
backlog do projeto como uma issue/card do tipo **Débito Técnico**. O registro
deve conter: descrição do problema, motivo do atalho, impacto estimado e proposta
de solução.

### 2.2. Priorização e pagamento da dívida

A equipe reserva **aproximadamente 15% do esforço de cada ciclo (sprint)** para a
liquidação dos débitos técnicos registrados. Débitos que afetem **segurança**
(ex.: dados de compradores e de pagamento), **desempenho** (ex.: consulta de
disponibilidade de ingressos) ou o **trabalho de outro membro** do time têm
**prioridade máxima** e são tratados no ciclo seguinte. A priorização é validada
com o Product Owner no planejamento do ciclo.

Versão viva: [`team/tech-debt.md`](tech-debt.md).

## 3. Critérios de qualidade: DoR e DoD

**Responsável:** Quality Assurance (QA) — Adrian Cesar Gonçalves — o DoR é
validado em conjunto com o Engenheiro de Requisitos (Renato Colin Neto).

### 3.1. Definition of Ready — DoR (pronto para desenvolver)

Uma tarefa só entra em desenvolvimento quando atende a todos os itens:

- A história de usuário está clara no formato: "Como [papel], eu quero
  [funcionalidade] para que [benefício]".
- Os critérios de aceitação foram descritos de forma objetiva e verificável.
- As regras de negócio essenciais e exceções estão especificadas (ex.: limite de
  ingressos por CPF, política de reembolso, expiração da reserva).
- As dependências técnicas foram mapeadas (ex.: gateway de pagamento, esquema do
  banco de dados, envio de e-mail de confirmação).
- O layout/protótipo da interface foi aprovado, quando aplicável.

### 3.2. Definition of Done — DoD (pronto para entrega)

Uma tarefa só é considerada "Concluída" se atende rigorosamente a todos os itens:

- O código segue o Guia de Estilo (Seção 5).
- O código passou por Code Review (Pull Request aprovado por, no mínimo, outro
  desenvolvedor).
- A funcionalidade foi validada e testada conforme a estratégia de QA (Seção 4),
  sem erros críticos.
- Os testes automatizados relevantes foram criados/atualizados e estão passando
  na pipeline.
- O código foi integrado à branch principal (`master` / trunk) sem quebrar o build.

Versão viva: [`team/quality.md`](quality.md).

## 4. Estratégia de testabilidade

**Responsável:** Quality Assurance (QA) — Adrian Cesar Gonçalves.

### 4.1. Cobertura e níveis de teste

O foco primário dos testes automatizados são as **regras de negócio da camada de
domínio do backend** (conforme o DDD): reserva e compra de ingressos, controle de
disponibilidade de assentos, cálculo de valores e processamento de pedidos.

Antes de cada entrega oficial, o QA executa um **roteiro de testes** cobrindo o
fluxo principal (busca do evento → seleção do ingresso → pagamento → confirmação)
e as regras de negócio críticas.

### 4.2. Registro e resolução de erros (bugs)

Todo bug identificado durante os testes deve ser documentado com: passos para
reproduzir, resultado esperado, resultado obtido, severidade e evidência
(print/log). Bugs de **alta severidade** (que bloqueiam compra ou pagamento, ou
que expõem dados) impedem o encerramento das tarefas e o fechamento do ciclo.

Versão viva: [`backend/testing.md`](../backend/testing.md).

## 5. Guia de estilo e padrões de código

**Responsável:** Desenvolvedores Frontend (Diego Nessler) e Backend (Igor Thiago Seberino).

### 5.1. Convenções de código

Stack: Frontend em Next.js (App Router) com TypeScript; Backend em Python com FastAPI; banco de dados
PostgreSQL; ambiente em contêineres Docker.

- **Backend (Python/FastAPI):** `snake_case` para variáveis e funções,
  `PascalCase` para classes, `snake_case` para módulos/arquivos (ex.:
  `ticket_service.py`), seguindo a PEP 8.
- **Frontend (React):** `camelCase` para variáveis e funções, `PascalCase` para
  componentes e arquivos de componente (ex.: `CheckoutPage.tsx`).
- **Idioma do código:** identificadores (classes, métodos, variáveis) em
  **Inglês**; comentários em **Português**.

### 5.2. Boas práticas de manutenibilidade

- Respeitar as fronteiras entre os módulos (monólito modular) e isolar as regras
  de domínio (DDD), evitando acoplamento direto entre módulos.
- Evitar duplicação de código criando funções e módulos reutilizáveis (princípio
  DRY).
- Funções/métodos com responsabilidade única e **até 30 linhas** sem
  justificativa técnica.
- Exceções devem ser capturadas e tratadas/registradas adequadamente. **Proibido
  bloco `catch`/`except` vazio.**
- Nenhuma credencial ou segredo versionado no código; usar variáveis de ambiente
  (`.env` fora do controle de versão).

Versão viva: [`backend/code-style.md`](../backend/code-style.md).

## 6. Fluxo de versionamento e pipeline de CI/CD

**Responsável:** DevOps (Gabriel Carvalho).

### 6.1. Estratégia de branches — Trunk Based Development (TBD)

- `master` (trunk): única branch de longa duração; sempre estável, integrável e
  apta a implantação.
- Branches de curta duração: criadas a partir de `master` no padrão
  `feature/nome-da-funcionalidade` ou `fix/descricao-do-bug`, reintegradas em
  poucos dias.
- Integração contínua: commits pequenos e frequentes na `master` via Pull Request;
  funcionalidades incompletas protegidas por feature flags quando necessário.

### 6.2. Padronização de commits

Commits seguem a especificação [Conventional Commits](https://www.conventionalcommits.org/pt-br/):
`feat`, `fix`, `refactor`, `docs`, `test`, `chore`.

### 6.3. Verificação automática, linters e contêineres

- **Frontend (React):** ESLint e Prettier, na pipeline.
- **Backend (Python):** Ruff.
- **Contêineres Docker:** a aplicação deve construir e subir via Docker sem
  erros; o ambiente conteinerizado é o padrão de execução e da pipeline.
- **Regra de integração:** nenhum Pull Request é aprovado se o linter acusar
  erros críticos ou se os testes automatizados falharem.

Versão viva: [`backend/testing.md`](../backend/testing.md).

## 7. Compromisso da equipe

Todos os membros da equipe leram, concordam e se comprometem a seguir as
diretrizes estabelecidas neste documento para garantir a qualidade, a
manutenibilidade e a entrega sustentável do software.
