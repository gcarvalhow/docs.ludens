---
status: reviewed
spec: identity-user-management
created_at: 2026-09-11
reviewed_at: 2026-09-11
---

# Gestão de conta de usuário — Lógica de Negócio

## 1. Fluxo por perfil

### Visitante (cria conta)

1. Abre `/cadastro`, informa nome, CPF, e-mail e senha.
2. O sistema valida: CPF com dígitos verificadores corretos e ainda não
   cadastrado; e-mail com formato válido e ainda não cadastrado; senha com
   mínimo de 8 caracteres.
3. Confirma. A conta é criada como `Comprador`, a pessoa é autenticada
   imediatamente (mesmo contrato de sessão de `identity-auth`) e cai na tela
   de onde veio (catálogo ou detalhe da sessão).
4. Se o CPF ou o e-mail já existem: a criação é recusada com mensagem
   específica ("Este CPF já possui cadastro" / "Este e-mail já está em uso").

### Comprador autenticado

1. Acessa "meu perfil": vê nome, CPF e e-mail. CPF e e-mail são somente
   leitura aqui (trocar e-mail é uma tela separada, de `identity-auth`; CPF é
   imutável).
2. Edita o nome; confirma; atualizado.
3. Não vê nem edita dados de nenhuma outra conta.

### Administrador autenticado

1. Vê e edita o próprio nome, igual a qualquer outra pessoa.
2. Acessa a lista de contas da plataforma: nome, e-mail, papel
   (comprador/administrador) e data de criação de cada uma.
3. Abre uma conta específica pra ver detalhes.
4. Remove uma conta — comprador ou outro administrador — com confirmação
   explícita antes de efetivar (ação sensível, sem desfazer).
5. Se tentar remover a própria conta: bloqueado, com mensagem explicando que
   não é permitido.

## 2. Estados e transições

### Conta

**Estados:** ativa · removida.

- **inexistente → ativa:** cadastro bem-sucedido.
- **ativa → removida:** um administrador remove a conta (nunca a própria).
- Não há reativação nesta versão — remoção é definitiva do ponto de vista de
  produto, mesmo que o dado permaneça internamente para não quebrar histórico
  de pedidos e convites já existentes.

## 3. Regras de negócio

- CPF sem dígitos verificadores válidos → cadastro recusado.
- CPF ou e-mail já cadastrado → cadastro recusado, mensagem específica por
  campo.
- Senha com menos de 8 caracteres → recusada na validação de forma (mesma
  regra de `identity-auth`).
- Cadastro bem-sucedido autentica a pessoa imediatamente — mesmo mecanismo de
  sessão emitido no login (`identity-auth`).
- Qualquer pessoa edita só o próprio nome. Tentar editar e-mail, senha ou CPF
  aqui é recusado — e-mail e senha têm fluxo próprio em `identity-auth`; CPF é
  imutável.
- Qualquer pessoa só vê e edita a própria conta — exceto o administrador, que
  também **vê** (nunca edita) contas de terceiros, pra listar e remover.
- Só administrador lista contas e remove contas.
- Administrador não remove a própria conta — bloqueado, mensagem explicando o
  motivo.
- Remoção é irreversível do ponto de vista de produto: a conta para de
  autenticar e some da listagem de contas ativas; os dados continuam
  associados a pedidos e convites já existentes, sem quebrar histórico.
- Conta removida tentando logar recebe a mesma mensagem genérica de
  credenciais inválidas de `identity-auth` — não revela que a conta existiu e
  foi removida.
- Nenhuma rota devolve CPF ou hash de senha de outra conta. Na listagem
  administrativa, só nome, e-mail, papel e data de criação são expostos
  (RNF01).

## 4. Pontos de integração

```text
Frontend precisa saber:
  - Que o cadastro pede nome, CPF, e-mail e senha, e que sucesso já vem com
    sessão iniciada — mesmo contrato de sessão usado no login
  - Que "meu perfil" mostra CPF e e-mail como somente leitura; só o nome é
    editável aqui (trocar e-mail leva pra tela de identity-auth)
  - Que a lista de contas e o botão de remover só aparecem para quem está
    autenticado como administrador
  - Que o botão de remover nunca aparece na própria linha do administrador
    logado, e a remoção pede confirmação explícita antes de efetivar
  - Mensagens de erro por caso: CPF/e-mail já cadastrado (específica por
    campo); tentativa de remover a própria conta (mensagem explicando o
    motivo); login após remoção (mensagem genérica, igual à de credenciais
    inválidas)

Backend precisa garantir:
  - Validação de CPF (dígitos verificadores) e unicidade de CPF e e-mail no
    cadastro
  - Emissão do par de sessão (access + refresh) imediatamente após o cadastro
    bem-sucedido — mesmo mecanismo de `identity-auth`
  - Qualquer edição de perfil restrita ao próprio usuário autenticado — nunca
    aceita um identificador de outra pessoa
  - Listagem e remoção de conta restritas a quem tem papel administrador
  - Remoção recusada quando o alvo é a própria conta do administrador
    autenticado
  - Remoção como soft delete: marca a conta como removida e impede
    autenticação futura, sem apagar o registro (preserva histórico de
    pedidos e convites)
  - Conta removida nunca aparece em listagem ou consulta padrão de contas
    ativas
```

## 5. Casos de borda

**Administrador tenta remover a própria conta.** Bloqueado, mensagem
explicando que não é permitido remover a própria conta administrativa.
`[decisão fechada]`

**Administrador remove a última outra conta de administrador, ficando ele
mesmo como único administrador da plataforma.** Permitido — o sistema não
impede. É o mesmo risco de "único administrador sem substituto" já registrado
como aceito em `identity-admin-invite`. `[decisão fechada]`

**Cadastro com e-mail de outra pessoa, sem verificação de posse.** Aceito no
MVP — decisão herdada de `identity-auth`: não há verificação de e-mail no
cadastro. `[decisão fechada]`

**Comprador tenta acessar a listagem de contas ou remover outra conta.**
Bloqueado — ação restrita a administrador, mesmo tratamento de qualquer rota
administrativa (RF08). `[decisão fechada]`

**Conta removida tenta logar depois.** Tratada como credenciais inválidas —
mesma mensagem genérica de sempre, para não revelar que aquela conta existiu
e foi removida. Consistente com a filosofia de "nunca confirmar existência de
conta" já usada em `identity-auth`. `[decisão fechada]`
