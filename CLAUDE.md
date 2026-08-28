# docs.ludens — orientação para agentes

Este repositório é a **fonte de entrada** do projeto Ludens: produto, requisitos,
arquitetura, acordo de manutenibilidade, RACI e padrões de engenharia. Os
repositórios de código (`gcarvalhow/api.ludens`, `gcarvalhow/web.ludens`) têm
READMEs curtos que apontam para cá.

## Regras ao editar

- Conteúdo em **português (pt-BR)**. Nomes de arquivo em `kebab-case` minúsculo;
  ADRs como `NNN-slug.md`.
- **Uma casa canônica por assunto.** Para relacionar documentos, use link — não
  copie o texto. Se dois documentos divergirem, o código real e o
  comportamento observado têm prioridade: corrija o documento, não repita a
  informação desatualizada.
- Todo documento abre com uma linha de metadados
  (`Responsável · Última revisão · Status`). Ao revisar um documento contra a
  realidade, atualize a data e, se for uma reescrita, registre-a.
- Requisitos e regras de negócio carregam **Status** explícito
  (`aprovada` | `proposta — pendente PO`).
- Não versione binários como fonte de verdade. Os `.docx`/`.xlsx` originais das
  entregas da disciplina ficam só em `assets/originais/`.
- O backlog do projeto vive num **GitHub Project da organização**, não aqui.
  Não crie issue templates de produto neste repositório.

## Antes de commitar

- Rode `npx markdownlint-cli2 "**/*.md"` e um verificador de links.
- Commits em [Conventional Commits](https://www.conventionalcommits.org/pt-br/),
  mensagem em português no imperativo.
