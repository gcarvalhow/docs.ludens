# docs.ludens — orientação para agentes

Este repositório é a **fonte de entrada** do projeto Ludens: produto, requisitos,
arquitetura, acordo de manutenibilidade, RACI e padrões de engenharia. Os
repositórios de código (`gcarvalhow/api.ludens`, `gcarvalhow/web.ludens`) têm
READMEs curtos que apontam para cá.

## Regras ao editar

- Conteúdo em **português (pt-BR)**. Nomes de arquivo em `kebab-case` minúsculo;
  ADRs como `NNN-slug.md`.
- Specs de feature ficam em `specs/[domínio]-[conceito]/`; convenção de pastas,
  estado de cada uma e o que existe hoje (o pipeline que as gerava foi
  descontinuado) vivem em `specs/README.md`.
- **Uma casa canônica por assunto.** Para relacionar documentos, use link — não
  copie o texto. Se dois documentos divergirem, o código real e o
  comportamento observado têm prioridade: corrija o documento, não repita a
  informação desatualizada.
- Sem linha de metadados de cabeçalho (`Responsável · Última revisão ·
  Status`) nem blocos RACI por seção: é ruído que ninguém lê. Responsável e
  aprovador de cada área vivem só na Matriz RACI (`team/overview.md`).
- Requisitos e regras de negócio carregam **Status** explícito
  (`aprovada` | `proposta, pendente PO`).
- Não versione binários como fonte de verdade. Os `.docx`/`.xlsx` originais das
  entregas da disciplina ficam em `archive/`, local mas fora do git
  (`.gitignore`).
- Em `product/`, `backend/` e `team/`: sem tabela markdown (vira lista
  aninhada) e sem travessão `—` em frase (vira `,`/`;`/`.` ou parênteses).
  Marcador de lista é `*`, não `-`. Não vale para hífen em palavra composta
  (`meia-entrada`), nome de arquivo/pasta (`kebab-case`) ou conteúdo dentro de
  bloco de código.
- O backlog do projeto vive num **GitHub Project da organização**, não aqui;
  não crie item de backlog neste repositório. Os templates de
  issue/PR/commit do fluxo TBD (`team/templates/`) são exceção: descrevem o
  processo, não são backlog.

## Antes de commitar

- Rode `npx markdownlint-cli2 "**/*.md"` e um verificador de links.
- Commits em [Conventional Commits](https://www.conventionalcommits.org/pt-br/),
  mensagem em português no imperativo.
