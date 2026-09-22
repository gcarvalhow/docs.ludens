# Template de Commit

Formato [Conventional Commits](https://www.conventionalcommits.org/pt-br/),
mensagem em português no imperativo:

```text
<tipo>(<escopo opcional>): <descrição curta no imperativo>

<corpo opcional: por que a mudança foi feita, não o quê>

<rodapé opcional: issue relacionada, breaking change>
```

## Tipos

* `feat`: nova funcionalidade visível para o usuário ou para outro módulo.
* `fix`: correção de bug.
* `refactor`: mudança de estrutura interna sem alterar comportamento.
* `docs`: mudança só de documentação.
* `test`: adição ou ajuste de teste, sem mudar código de produção.
* `chore`: manutenção que não se encaixa nos anteriores (dependências, configuração, build).

## Exemplos

```text
feat(booking): adiciona expiração automática de reserva não paga

fix(identity): corrige validação de CPF com dígito verificador zero

docs(team): documenta o fluxo Trunk-Based Development com RACI
```

## Regras

* Descrição curta com até ~72 caracteres, sem ponto final.
* Um commit, uma mudança lógica: prefira vários commits pequenos a um grande.
* Nunca commitar segredo ou credencial (ver [guia de estilo § Boas práticas de manutenibilidade](../../backend/code-style.md#boas-práticas-de-manutenibilidade)).
