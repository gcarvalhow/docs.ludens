# Template de Pull Request

PR pequeno, de uma branch de curta duração (`feature/...` ou `fix/...`) contra
`master`. Um PR só é aprovado quando bate o
[Definition of Done](../overview.md#definition-of-done-dod-pronto-para-entrega)
completo.

## Issue relacionada

`Closes #<número>` ou `Refs #<número>`.

## Escopo da mudança

O que este PR muda e por quê, em 2 a 3 frases. Não repita o diff, explique a
motivação.

## Como testar

Passo a passo pra quem for revisar reproduzir o comportamento localmente.

## Checklist de Definition of Done

* [ ] O código segue o [guia de estilo](../../backend/code-style.md).
* [ ] A funcionalidade foi validada conforme a
      [estratégia de teste](../../backend/testing.md), sem erros críticos.
* [ ] Os testes automatizados relevantes foram criados/atualizados e estão
      passando na pipeline.
* [ ] Lint do frontend sem erro crítico, quando aplicável.
* [ ] Nenhum atalho técnico assumido sem a issue de
      [débito técnico](../overview.md#política-de-registro) correspondente.

## Screenshots

Quando a mudança altera interface visível.
