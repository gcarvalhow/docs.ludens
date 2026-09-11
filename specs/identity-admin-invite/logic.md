---
status: draft
spec: identity-admin-invite
created_at: 2026-09-11
---

# Convite de administrador por link — Lógica de Negócio

## 1. Fluxo por perfil

### Administrador (quem convida)

1. Na área administrativa, escolhe convidar um novo administrador e informa o
   e-mail da pessoa.
2. O sistema valida: formato do e-mail; e que nenhuma conta (comprador ou
   administrador) já existe com esse e-mail.
3. Confirma. Um convite é gerado e o sistema envia automaticamente um e-mail
   para o endereço informado, com o link de conclusão do cadastro. O
   administrador não vê o link — só a confirmação de que foi enviado e a
   validade (48h).
4. Se já havia um convite pendente (de qualquer e-mail, gerado antes e ainda
   não usado nem expirado), ele é invalidado automaticamente e substituído
   pelo novo — o sistema avisa disso antes de confirmar, mas não bloqueia.
5. A tela de administração mostra, quando existe, o convite pendente atual: a
   quem foi enviado e quando expira. Não existe histórico de convites
   passados nesta versão.
6. Se o e-mail informado já tem conta: recusado com mensagem específica
   ("Este e-mail já possui cadastro").

### Pessoa convidada (com o link, ainda sem conta)

1. Abre o link recebido por e-mail.
2. O sistema valida o token do convite. Se válido, mostra o formulário de
   conclusão de cadastro com o e-mail já preenchido e fixo (o mesmo que o
   administrador informou) — a pessoa define apenas nome e senha.
3. Confirma. A conta é criada com acesso administrativo completo — mesmo
   mecanismo de criação de conta de `identity-user-management` — o convite é
   marcado como usado, e a pessoa é autenticada imediatamente (mesmo
   mecanismo de sessão de `identity-auth`), caindo na área administrativa.
4. Se o link já foi usado, expirou, ou nunca existiu: mensagem única e não
   técnica ("Este convite não é mais válido, peça um novo ao administrador").

### Visitante e Comprador autenticado

Não interagem com esta feature. Gerar convite exige estar autenticado como
administrador; abrir e concluir um convite não exige login, mas só é possível
com o token específico recebido por e-mail — não há nenhuma tela ou ação
alcançável por um visitante ou comprador comum.

## 2. Estados e transições

### Convite de administrador

**Estados:** pendente · usado · expirado · substituído.

- **inexistente → pendente:** administrador gera o convite.
- **pendente → usado:** pessoa convidada completa o cadastro com sucesso.
- **pendente → expirado:** passadas 48h da emissão sem uso.
- **pendente → substituído:** administrador gera um novo convite antes que o
  anterior seja usado ou expire.
- Usado, expirado e substituído são estados finais — nenhum convite volta a
  ficar pendente; um novo convite sempre nasce do zero.

### Conta administrativa criada por convite

**Estados:** inexistente · ativa.

- **inexistente → ativa:** cadastro concluído com sucesso a partir do convite.
- Segue, a partir daí, o mesmo ciclo de vida de conta já definido em
  [`identity-user-management`](../identity-user-management/logic.md#2-estados-e-transições)
  (ativa/removida) e o mesmo mecanismo de sessão de
  [`identity-auth`](../identity-auth/logic.md#2-estados-e-transições) — não
  há nada específico do convite depois que a conta existe.

## 3. Regras de negócio

- Apenas administrador autenticado pode gerar um convite (mesmo controle de
  acesso administrativo de RF08/RF09).
- Convite recusado na criação se o e-mail informado já pertence a uma conta
  existente, comprador ou administrador.
- No máximo um convite pendente por vez: gerar um novo invalida
  automaticamente qualquer convite pendente anterior, mesmo que endereçado a
  outro e-mail.
- Convite não usado em 48h da emissão expira; a validade é checada no momento
  em que o link é aberto, não depende de nenhuma rotina de limpeza periódica
  para valer.
- Link de convite é de uso único — depois que a conta é criada, o mesmo link
  não serve mais, mesmo que ainda dentro das 48h.
- O e-mail da conta criada é sempre o e-mail que o administrador informou ao
  gerar o convite; não pode ser alterado no formulário de conclusão.
- A senha definida pela pessoa convidada segue as mesmas regras de
  `identity-user-management` (mínimo 8 caracteres, guardada como hash, nunca
  logada — RNF01).
- A conta criada por convite tem acesso administrativo completo, idêntico ao
  de qualquer outro administrador — não existe nível reduzido.
- O token do convite nunca é exposto na tela do administrador nem em log —
  só viaja dentro do e-mail enviado à pessoa convidada (RNF01).

## 4. Pontos de integração

```text
Frontend precisa saber:
  - Que convidar pede só o e-mail da pessoa; a resposta é a confirmação de
    envio e a validade (48h) — nunca o link ou o token em si
  - Que existe no máximo um convite pendente por vez; se já houver um, avisar
    antes de confirmar que ele será substituído
  - O estado do convite pendente atual (e-mail de destino, quando expira) pra
    exibir na tela de administração — sem histórico de convites passados
  - Que a rota de conclusão do convite é pública (sem login), igual à de
    redefinição de senha, e que o e-mail vem fixo — só nome e senha são
    formulário
  - Mensagens de erro por caso: token inválido/expirado/já usado → mensagem
    única não técnica; e-mail já cadastrado ao convidar → mensagem específica

Backend precisa garantir:
  - Token de convite único, guardado como hash, nunca devolvido nem logado em
    texto puro (mesmo padrão do token de redefinição de senha)
  - Validade de 48h checada no momento do uso, não em rotina periódica
  - Ao gerar um novo convite, qualquer convite pendente anterior é invalidado
    junto da criação do novo — nunca dois convites pendentes ao mesmo tempo
  - Envio do e-mail de convite por evento assíncrono (outbox), mesmo padrão
    de RF05/recuperação de senha — falha no envio não invalida o convite
  - Consumo do convite (criação da conta) e marcação do convite como usado
    são atômicos — impossível o mesmo link criar duas contas
  - Conta criada por convite recebe acesso administrativo completo, sem
    diferença de permissão frente a um administrador semeado por script
```

## 5. Casos de borda

**Administrador tenta convidar um e-mail que já tem conta (comprador ou
administrador).** Recusado na criação do convite, mensagem específica ("Este
e-mail já possui cadastro"). Não promove uma conta de comprador existente a
administrador — esta feature só cria contas novas. `[decisão fechada]`

**Duas pessoas abrem o mesmo link e tentam concluir o cadastro ao mesmo
tempo.** Consumo do convite e criação da conta são atômicos — só a primeira
conclusão vence; a segunda recebe a mensagem de link não mais válido.
`[decisão fechada]`

**Administrador gera um novo convite enquanto o anterior ainda não expirou e
ninguém o usou.** O anterior é invalidado automaticamente. Se alguém tentar
usar o link antigo depois, recebe a mesma mensagem de convite inválido — não
há aviso adicional a essa pessoa, já que ela nunca chegou a ter conta.
`[decisão fechada]`

**E-mail do convite não chega (serviço de e-mail fora do ar).** Mesmo
tratamento dado à recuperação de senha em `identity-auth`: envio por outbox
idempotente com retry; o convite continua válido pelas 48h independente de o
e-mail ter chegado; o administrador pode gerar um novo convite se preferir.
`[decisão fechada]`

**O único administrador da plataforma perde acesso à própria conta (esqueceu
a senha) e não há outro administrador para convidar um substituto.** Fora do
escopo desta feature — depende da recuperação de senha já coberta em
`identity-auth`, ou de intervenção operacional fora do produto. Risco aceito,
mesmo tratamento dado hoje a qualquer conta que perde acesso sem alternativa.
`[decisão fechada]`

**O serviço de e-mail transacional (`notification-transactional-email`) ainda
não existe no backend quando esta feature for implementada.** Verificado no
código do `api.ludens`: só os módulos `catalog` e `identity` existem hoje.
Como o envio do convite depende desse serviço (§4), esta feature **não pode
ser implementada nem entregue antes dele** — é uma dependência de
sequenciamento, não uma decisão de lógica de negócio. Registrar como bloqueio
na priorização do backlog. `[decisão fechada]`
