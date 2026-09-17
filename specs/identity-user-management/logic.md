---
status: draft
spec: identity-user-management
created_at: 2026-09-11
reviewed_at: 2026-09-11
updated_at: 2026-09-17
---

# Gestão de conta de usuário — Lógica de Negócio

> **Nota de reescopo (2026-09-17):** reescrito para acompanhar o reescopo de
> [`spec.md`](spec.md) — encerramento de conta virando autosserviço confirmado
> por e-mail (o administrador deixa de excluir contas e passa só a listar) e a
> troca de e-mail chegando aqui, vinda de
> [`identity-auth`](../identity-auth/logic.md), com confirmação no **e-mail
> atual**. A edição de CPF foi descartada no mesmo alinhamento — CPF continua
> imutável, só o nome passa a ser editável. **Status voltou para `draft`:** o
> fluxo do administrador e as ações sensíveis mudaram de comportamento, não só
> de lugar — precisa de nova revisão conjunta de FE/BE antes de valer como
> `reviewed` outra vez.
>
> Todas as pendências que motivaram esse rascunho já foram fechadas pelo PO —
> ver [`spec.md` § 9](spec.md#9-perguntas-abertas). As regras abaixo estão
> escritas como decisão fechada, não como proposta.

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

### Comprador autenticado — dados do perfil

1. Acessa "meu perfil": vê nome, CPF e e-mail.
2. **Edita o nome.** Confirma; atualizado na hora, sem confirmação por e-mail
   — não muda o acesso à conta. CPF aparece como leitura — imutável após o
   cadastro (mesma decisão de `identity-auth`, mantida).
3. Não vê nem edita dados de nenhuma outra conta.

### Comprador autenticado — troca o e-mail

1. Em "meu perfil", pede a troca e informa o novo endereço **duas vezes**.
   Nenhuma senha é pedida.
2. O sistema envia um link de confirmação para o **e-mail atual** da conta — não
   para o novo — e mostra na tela "enviamos um link para [e-mail atual]; a troca
   só vale depois que você abrir esse link".
3. Enquanto o link não é aberto, nada muda: o e-mail da conta continua o antigo,
   e é com ele que a pessoa loga.
4. A pessoa abre o link. O e-mail da conta passa a ser o novo, **todas as
   sessões caem** (em todos os dispositivos, inclusive o atual) e ela é levada
   ao login para entrar de novo — agora com o e-mail novo.
5. Um aviso de cortesia é enviado ao endereço novo depois da troca confirmada,
   só para a pessoa perceber cedo se digitou errado.
6. Link expirado, já usado, ou novo e-mail que passou a pertencer a outra conta
   nesse meio-tempo → a troca é recusada, o e-mail da conta continua o antigo e
   a mensagem diz o que aconteceu.

### Comprador autenticado — encerra a própria conta

1. Em "meu perfil", pede o encerramento da conta.
2. O sistema mostra, antes de qualquer confirmação, o que ela perde: o histórico
   de pedidos e a possibilidade de pedir cancelamento/reembolso (RF07) saem do
   alcance dela; os ingressos já emitidos para sessões futuras continuam valendo
   na porta, porque o que é validado na entrada é o código do ingresso.
3. Confirma o pedido. O sistema envia um link de confirmação para o e-mail da
   conta e avisa na tela que a conta **ainda não foi encerrada**.
4. A pessoa abre o link. A conta é encerrada, todas as sessões caem, e ela volta
   a ser uma visitante.
5. Enquanto o link não é aberto, a conta segue normal — dá para comprar, entrar,
   sair. Pedir o encerramento não limita nada.
6. Se houver reserva aberta ou pagamento em processamento, o pedido nem chega a
   ser aceito: mensagem explicando que é preciso concluir a compra ou esperar a
   reserva expirar.

### Administrador autenticado

1. Faz tudo que qualquer pessoa faz com a própria conta: edita o nome, troca o
   e-mail e encerra a própria conta — mesmos fluxos, mesmas confirmações.
2. Acessa a lista de contas da plataforma: nome, e-mail, papel
   (comprador/administrador) e data de criação de cada uma. **Sem CPF.**
3. Abre uma conta específica para ver esses mesmos dados com mais calma.
4. **Não encerra, não edita e não altera a conta de ninguém.** A lista é de
   leitura — não existe ação sobre conta de terceiro em lugar nenhum do produto.
5. Se for o único administrador restante, o encerramento da própria conta é
   recusado, com mensagem explicando que é preciso convidar outro administrador
   antes.

## 2. Estados e transições

### Conta

**Estados:** ativa · encerrada.

- **inexistente → ativa:** cadastro bem-sucedido (próprio ou via convite de
  administrador).
- **ativa → encerrada:** a própria pessoa abre o link de confirmação de
  encerramento. Nenhum outro caminho leva a este estado — nenhum perfil encerra
  a conta de outra pessoa.
- Não há reativação: encerrar é definitivo do ponto de vista de produto, mesmo
  que o dado permaneça internamente para não quebrar histórico de pedidos,
  ingressos e convites.

### Pedido de encerramento de conta

**Estados:** pendente · confirmado · expirado.

- **inexistente → pendente:** a pessoa autenticada pede o encerramento e não há
  reserva aberta nem pagamento em processamento.
- **pendente → confirmado:** a pessoa abre o link enviado ao e-mail da conta; a
  conta passa a encerrada e todas as sessões caem.
- **pendente → expirado:** passada 1 hora sem confirmação. A conta não muda; se
  ainda quiser sair, a pessoa pede de novo.
- Um novo pedido invalida qualquer pedido anterior ainda pendente — só o último
  link vale (mesmo padrão do link de recuperação de senha).

### Pedido de troca de e-mail

**Estados:** pendente · confirmado · expirado.

- **inexistente → pendente:** a pessoa autenticada pede a troca informando o
  novo endereço, que precisa ser válido e não pertencer a outra conta.
- **pendente → confirmado:** a pessoa abre o link enviado ao **e-mail atual**; o
  e-mail da conta passa a ser o novo e todas as sessões caem.
- **pendente → expirado:** passada 1 hora sem confirmação. O e-mail da conta não
  muda.
- Um novo pedido invalida qualquer pedido anterior ainda pendente — só o último
  link vale.

## 3. Regras de negócio

**Cadastro**

- CPF sem dígitos verificadores válidos → cadastro recusado.
- CPF ou e-mail já cadastrado → cadastro recusado, mensagem específica por
  campo.
- Senha com menos de 8 caracteres → recusada na validação de forma (mesma regra
  de `identity-auth`).
- Cadastro bem-sucedido autentica a pessoa imediatamente — mesmo mecanismo de
  sessão emitido no login (`identity-auth`).
- E-mail informado no cadastro não tem posse verificada → aceito assim mesmo no
  N1.

**Dados do perfil**

- Nome é editado pela própria pessoa, sem confirmação por e-mail → não muda o
  acesso à conta.
- CPF é imutável após o cadastro (mesma decisão de `identity-auth`, mantida) —
  não há edição, nem própria nem por administrador. Preserva o limite de 6
  ingressos por sessão (RN01), contado pelo CPF do comprador autenticado (ver
  [`booking-reservation`](../booking-reservation/logic.md)).
- Ninguém edita dados de conta de terceiros, em nenhum perfil — inclusive
  administrador.

**Troca de e-mail**

- Pedido de troca não exige senha → exige só estar autenticado; a garantia vem
  da confirmação no e-mail atual, não da senha.
- Link de confirmação vai sempre para o **e-mail atual** da conta → uma sessão
  sequestrada não consegue completar a troca sem acesso à caixa de entrada da
  pessoa, e o dono fica sabendo da tentativa.
- Novo e-mail já pertencente a outra conta → recusado no pedido **e** de novo na
  confirmação (a checagem se repete, porque outra conta pode ter tomado o
  endereço enquanto o link estava pendente).
- Troca confirmada → o e-mail da conta muda e **todas as sessões caem**, em
  todos os dispositivos, exigindo novo login com o e-mail novo.
- Token de confirmação de troca: uso único, validade 1 hora, nunca exposto em
  tela nem em log (RNF01). Um novo pedido invalida o anterior.
- Enquanto a troca não é confirmada → o e-mail antigo continua sendo o de login
  e o de recuperação de senha.
- Novo endereço informado duas vezes no pedido, e aviso de cortesia enviado a
  ele depois da troca → nada mais comprova que o endereço novo é alcançável; um
  erro de digitação confirmado deixa a conta sem caminho de recuperação.

**Encerramento de conta**

- Só o próprio dono encerra a própria conta → nenhum perfil, inclusive
  administrador, encerra a conta de outra pessoa.
- Encerramento só vale depois que a pessoa abre o link enviado ao e-mail da
  conta → pedir não encerra nada.
- Reserva aberta ou pagamento em processamento → o pedido de encerramento é
  recusado com mensagem explicando o motivo (evita encerrar conta no meio de um
  checkout com assento bloqueado — RN03, RN05).
- Único administrador restante → encerramento da própria conta recusado; sem
  isso a plataforma pode ficar sem administrador e sem caminho de volta, já que
  administrador só nasce de convite de outro administrador
  (`identity-admin-invite`).
- Token de confirmação de encerramento: uso único, validade 1 hora. Um novo
  pedido invalida o anterior.
- Conta encerrada → para de autenticar, some das listagens, e todas as sessões
  dela caem no momento do encerramento.
- Conta encerrada tentando logar depois → mesma mensagem genérica de credenciais
  inválidas de `identity-auth`; não se revela que a conta existiu.
- Histórico de pedidos, ingressos e convites ligados à conta encerrada permanece
  íntegro — o dado não é apagado, só deixa de ser acessível pela pessoa.

**Listagem administrativa**

- Só administrador lista contas → comprador tentando acessar recebe o mesmo
  tratamento de qualquer rota administrativa (RF08).
- A listagem expõe nome, e-mail, papel e data de criação. **Nunca CPF**, nunca
  senha (RNF01).
- Contas encerradas não aparecem na listagem.
- A listagem não habilita nenhuma ação sobre a conta listada.

## 4. Pontos de integração

```text
Frontend precisa saber:
  - Que o cadastro pede nome, CPF, e-mail e senha, e que sucesso já vem com
    sessão iniciada — mesmo contrato de sessão usado no login
  - Que "meu perfil" edita só o nome — CPF aparece na tela como leitura, sem
    campo editável
  - Que a troca de e-mail não pede senha, pede o novo endereço duas vezes, e que
    a tela de sucesso precisa dizer que o link foi para o e-mail ATUAL — a
    pessoa vai procurar na caixa errada se a tela não disser isso
  - Que confirmar a troca de e-mail derruba a sessão: depois de abrir o link, o
    frontend trata a pessoa como anônima e manda para o login
  - Que pedir o encerramento não encerra nada — a tela precisa deixar claro que
    falta abrir o link, e mostrar antes o que se perde (histórico e reembolso)
    e o que continua valendo (ingresso já emitido, validado pelo código)
  - Que a lista de contas só aparece para administrador, não traz CPF, e não
    tem nenhuma ação por linha — não existe botão de remover em lugar nenhum
  - Mensagens de erro por caso: CPF/e-mail já cadastrado (específica por campo);
    novo e-mail já em uso; link expirado ou já usado; encerramento bloqueado por
    compra em andamento; encerramento bloqueado por ser o último administrador;
    login após encerramento (mensagem genérica, igual à de credenciais
    inválidas)

Backend precisa garantir:
  - Validação de CPF (dígitos verificadores) e unicidade de CPF e e-mail no
    cadastro; CPF não é aceito em nenhum payload de edição de perfil
  - Emissão do par de sessão imediatamente após o cadastro bem-sucedido — mesmo
    mecanismo de `identity-auth`
  - Edição de perfil e as duas ações confirmadas por e-mail sempre restritas ao
    próprio usuário autenticado — nunca aceitam o identificador de outra pessoa
  - Envio do link de troca de e-mail para o endereço ATUAL da conta, nunca para
    o novo; e do aviso de cortesia para o novo, depois da confirmação
  - Tokens de troca de e-mail e de encerramento: uso único, validade 1h, novo
    pedido invalida o anterior, nunca devolvidos nem logados
  - Recusa da confirmação de troca se o novo e-mail passou a pertencer a outra
    conta durante a pendência
  - Invalidação de todas as sessões (o mesmo mecanismo que `identity-auth` usa
    no logout e na troca de senha) ao confirmar a troca de e-mail e ao encerrar
    a conta
  - Recusa do pedido de encerramento quando houver reserva aberta ou pagamento
    em processamento, e quando o solicitante for o único administrador restante
  - Encerramento como exclusão lógica: a conta para de autenticar e some das
    listagens, sem apagar o registro (preserva histórico de pedidos, ingressos
    e convites)
  - Listagem restrita a administrador, sem CPF, sem contas encerradas, e sem
    nenhuma operação de escrita sobre conta de terceiro em lugar nenhum da API
```

## 5. Casos de borda

**Pessoa digitou o CPF errado no cadastro e quer corrigir.** Não há caminho no
produto — CPF é imutável após o cadastro (§ 6 de `spec.md`). Caminho: contato
manual com o teatro. `[decisão fechada]`

**Pessoa digita errado o novo e-mail e confirma a troca pelo e-mail antigo.** A
conta fica com um endereço inalcançável: sem ingresso, sem recuperação de senha
e sem como trocar o e-mail de novo (o link iria para o endereço errado). A
sessão já caiu, então nem entrar ela consegue, a menos que lembre a senha — e,
mesmo lembrando, precisa logar com o e-mail errado, que ela sabe qual é.
Mitigação: dupla digitação no pedido e aviso de cortesia no endereço novo.
`[decisão fechada]`

**Pessoa perdeu o acesso ao e-mail cadastrado e quer trocá-lo.** Não consegue —
o link vai para o endereço que ela não acessa mais. É o custo consciente de
confirmar no e-mail atual. Caminho: contato manual com o teatro.
`[decisão fechada — ver spec.md § 6]`

**Sessão sequestrada tenta trocar o e-mail da conta.** O pedido é aceito, mas o
link chega na caixa de entrada do dono legítimo, que não pediu nada. O intruso
não completa a troca; o dono descobre que sua sessão está comprometida e pode
trocar a senha (`identity-auth`), o que derruba todas as sessões — inclusive a
do intruso. É o cenário que motivou o mecanismo. `[decisão fechada]`

**Administrador sai do teatro e não encerra a própria conta.** Continua com
acesso administrativo completo e ninguém consegue tirá-lo de dentro do produto —
não há remoção por terceiro nem revogação de papel no corte atual. Decisão do
PO (2026-09-17): risco aceito por agora, revogação de papel fica para depois.
`[decisão fechada — ver spec.md § 6]`

**Último administrador tenta encerrar a própria conta.** Recusado, com mensagem
dizendo para convidar outro administrador antes. Se fosse permitido, a
plataforma ficaria sem administrador e sem caminho de volta pelo produto.
`[decisão fechada]`

**Pessoa pede o encerramento e, antes de abrir o link, compra um ingresso.** A
conta segue normal até a confirmação, então a compra acontece. Se depois ela
abrir o link, a conta é encerrada com um ingresso emitido para uma sessão
futura — que continua valendo na porta, porque é o código do ingresso que é
validado. O aviso da etapa 2 já dizia isso. `[decisão fechada]`

**Pessoa pede a troca de e-mail e, antes de confirmar, pede o encerramento da
conta (ou o contrário).** Os dois pedidos são independentes e podem coexistir;
o que for confirmado primeiro derruba as sessões. Se o encerramento for
confirmado primeiro, o link de troca de e-mail deixa de valer junto com a conta.
`[decisão fechada]`

**Comprador tenta acessar a listagem de contas.** Bloqueado — restrita a
administrador, mesmo tratamento de qualquer rota administrativa (RF08).
`[decisão fechada]`

**Conta encerrada tenta logar depois.** Tratada como credenciais inválidas —
mesma mensagem genérica de sempre, para não revelar que aquela conta existiu.
Consistente com a filosofia de "nunca confirmar existência de conta" já usada em
`identity-auth`. `[decisão fechada]`
