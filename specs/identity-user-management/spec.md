---
status: approved
domain: identity
created_at: 2026-09-11
approved_at: 2026-09-11
updated_at: 2026-09-17
---

# Gestão de conta de usuário

> **Nota de reescopo (2026-09-17):** o PO decidiu inverter duas decisões desta
> spec e trazer uma terceira de `identity-auth`:
>
> 1. **A exclusão de conta passa a ser exclusivamente autosserviço**, confirmada
>    por link enviado por e-mail. O administrador **deixa de poder excluir a
>    conta de terceiros** — passa só a listar (antes: só o administrador
>    excluía, e nunca a própria).
> 2. **A troca de e-mail muda de dono e de mecanismo** e passa a viver aqui, não
>    mais em [`identity-auth`](../identity-auth/spec.md): a confirmação é
>    enviada ao **e-mail atual** da conta (antes: ao novo), e confirmar derruba
>    todas as sessões.
>
> A edição de CPF foi considerada no mesmo alinhamento e descartada por
> simplicidade — a decisão original de 2026-09-11 (CPF **imutável** após o
> cadastro) se mantém; só o **nome** passa a ser editável.
>
> Origem: decisão do PO em 2026-09-17, ao alinhar esta spec com o plano técnico
> da Workstream B do módulo `identity` já aprovado por ele. As duas perguntas
> que essas mudanças abriam — revogação do papel de administrador que sai, e o
> caminho de quem perde acesso ao e-mail cadastrado — foram fechadas pelo PO no
> mesmo alinhamento (ver § 8; não há mais pendência em § 9).
>
> `logic.md` foi reescrito junto e voltou para `draft`: precisa de nova revisão
> conjunta de FE/BE antes de valer como `reviewed` outra vez.

## 1. Visão da feature

Toda pessoa com conta na plataforma — comprador ou administrador — nasce aqui:
é aqui que a conta é criada. Depois de criada, é aqui também que a pessoa cuida
da própria conta: vê e corrige seus dados, troca o endereço de e-mail e, se
quiser, encerra a conta. E é aqui que o administrador enxerga quem tem conta na
plataforma.

É a "ficha" de cada conta e tudo que a própria pessoa faz com ela — separada de
como ela entra (isso é `identity-auth`) e de como ela vira administrador (isso é
`identity-admin-invite`).

O fio que costura as três ações sensíveis desta feature é o mesmo: **mudanças
que afetam o acesso à conta só valem depois que a pessoa confirma por um link
enviado ao e-mail que já está na conta.** Trocar de e-mail e encerrar a conta
seguem exatamente esse ritual. Quem estiver de posse de uma sessão roubada não
consegue nenhuma das duas coisas sem ter também a caixa de entrada da pessoa.

## 2. Problema que resolve

**Criar conta:** sem uma conta, não há como comprar (RN01, RF06) nem como
administrar o catálogo (RF08). É o ponto de entrada de qualquer pessoa na
plataforma.

**Corrigir o próprio nome:** se o nome cadastrado está errado (erro de
digitação, mudança), não existe hoje como a própria pessoa corrigir —
precisaria de intervenção manual do teatro. (CPF continua imutável após o
cadastro — ver § 6.)

**Trocar o endereço de e-mail:** a pessoa muda de provedor, sai da conta do
trabalho, quer separar a vida pessoal. O e-mail é por onde o ingresso chega
(RF05) e por onde a senha é recuperada — ficar preso a um endereço que não se
usa mais é ficar a um passo de perder a conta.

**Encerrar a conta:** hoje não existe caminho nenhum. Quem quer sair da
plataforma precisa pedir a alguém do teatro, que precisa pedir a alguém
técnico. Para uma plataforma que guarda CPF e histórico de compras, "eu quero
sair" precisa ser uma ação do próprio dono do dado, não um favor.

**Enxergar quem tem conta (administrador):** o administrador não tem hoje
nenhuma visão de quem tem conta na plataforma — nem de quantas pessoas
administram junto com ele, o que importa desde que convidar administrador virou
uma ação de produto (`identity-admin-invite`).

## 3. Para quem é

- **Beneficiário direto:** toda pessoa com conta — criação, correção dos
  próprios dados, troca de e-mail e encerramento da conta; o administrador
  (visão de quem tem conta na plataforma).
- **Beneficiário indireto:** o teatro, que deixa de ser acionado manualmente
  para corrigir um cadastro, trocar um e-mail ou apagar uma conta.

Criar conta acontece no início da jornada de qualquer pessoa. Corrigir dados,
trocar e-mail e encerrar a conta são ações pontuais, fora do fluxo de compra —
ninguém faz isso com pressa, e nenhuma delas pode acontecer por acidente.

## 4. Como melhora a experiência atual

**Antes:** criar conta já existia (como parte do que hoje é `identity-auth`),
mas sem edição de dados próprios, sem troca de e-mail com um mecanismo que
resista a uma sessão roubada, sem forma de encerrar a conta, e sem nenhuma
visão de contas para o administrador.

**Depois:** a pessoa corrige o próprio nome sozinha, troca o e-mail confirmando
no endereço que ela comprovadamente ainda acessa, e encerra a própria conta
quando quiser — com a mesma confirmação por e-mail nos dois casos sensíveis. O
administrador tem a lista de quem tem conta na plataforma.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:**
[`notification-transactional-email`](../notification-transactional-email/spec.md)
— o link de confirmação da troca de e-mail e o link de confirmação do
encerramento da conta usam o mesmo serviço de e-mail transacional já previsto
para a recuperação de senha. **Atenção:** essa spec está aprovada mas ainda não
implementada no backend — as duas ações confirmadas por link não podem ir ao ar
antes dela.
[`identity-auth`](../identity-auth/spec.md) — a criação de conta já autentica a
pessoa, e tanto confirmar a troca de e-mail quanto encerrar a conta derrubam
todas as sessões pelo mesmo mecanismo de invalidação que `identity-auth` define
e possui.

**O que habilita:** é pré-requisito de `identity-auth` (a conta precisa existir
antes de logar), de `identity-admin-invite` (o convite aceito termina numa conta
criada por esta feature) e de tudo que exige um `Comprador` ou `Admin`
identificado (RF03, RF04, RF06, RF08).

**Posição no produto:** core, N1.

**RF/RN cobertos:** RF09 (parte de cadastro). Toca RF08 (controle de acesso
administrador × comprador, na listagem de contas). CPF continuar imutável
preserva **RN01** (limite de 6 ingressos por sessão, contado por CPF) sem
mudança de regra. Nenhum RF aprovado em 2026-08-28 cobre listagem de contas,
encerramento de conta ou troca de e-mail diretamente — é extensão de escopo na
mesma linha de `identity-admin-invite`.

## 6. O que não é (escopo negativo)

- **Não inclui** login, sessão, troca de senha ou redefinição de senha esquecida
  — isso é [`identity-auth`](../identity-auth/spec.md). Esta feature cria a
  conta e cuida dos dados dela; não autentica ninguém.
- **Não inclui** convite ou promoção de comprador a administrador — a única
  forma de uma conta virar `Admin` é aceitar um convite
  (`identity-admin-invite`). Não existe aqui um botão "tornar administrador".
- **Não inclui** exclusão de conta pelo administrador. **A partir de
  2026-09-17 nenhum perfil exclui a conta de outra pessoa** — só o próprio dono
  encerra a própria conta. Isso reabre, de forma consciente, a lacuna que esta
  spec citava em 2026-09-11: um administrador que sai do teatro e não encerra a
  própria conta continua com acesso administrativo completo e ninguém consegue
  tirá-lo de dentro do produto. Decisão do PO (2026-09-17): aceitar esse risco
  por agora — ver bullet seguinte.
- **Não inclui** revogação do papel de administrador sem apagar a conta
  ("rebaixar para comprador") — seria o caminho natural para a lacuna acima.
  Decisão do PO (2026-09-17): fica para depois, não entra neste N1. Candidata a
  melhoria futura.
- **Não inclui** edição de CPF — imutável após o cadastro, mesma decisão já
  tomada em `identity-auth` e mantida em 2026-09-17 (a edição foi considerada e
  descartada por simplicidade). Só o nome é editável pela própria pessoa.
- **Não inclui** recuperação de conta de quem perdeu o acesso ao e-mail
  cadastrado. Com a confirmação indo para o e-mail atual, quem não acessa mais a
  própria caixa de entrada não consegue trocar o e-mail — nem recuperar a senha,
  que já ia para lá. Esse caso sai do produto e vira contato manual com o
  teatro. Decisão do PO (2026-09-17): custo consciente da escolha anti-sequestro
  (§ 8), aceito sem compromisso de produto para este caso no N1.
- **Não inclui** edição de dados de outra pessoa pelo administrador — o
  administrador vê contas, mas não edita nome, CPF nem e-mail de terceiros.
- **Não inclui** CPF na listagem administrativa — a lista existe para o
  administrador saber quem tem conta e quem administra junto com ele; nenhuma
  ação nasce dela, então o CPF não acrescenta nada e só amplia a exposição de
  dado pessoal (RNF01).
- **Não inclui** histórico de contas encerradas, motivo do encerramento ou
  reativação — encerrar é definitivo do ponto de vista de produto (mesmo que o
  dado permaneça internamente para não quebrar histórico de pedidos).
- **Não inclui** verificação do e-mail informado no **cadastro** — decisão
  mantida: o MVP aceita o e-mail do cadastro sem comprovar posse. Só a *troca*
  de e-mail e o *encerramento* exigem confirmação.

## 7. Custos adicionais

Nenhum custo externo novo. Os dois e-mails novos (confirmação de troca de
e-mail e confirmação de encerramento de conta) usam o mesmo serviço de e-mail
transacional já previsto para a recuperação de senha — não é provedor novo.

Há um custo de suporte, não de dinheiro: como a recuperação de conta por e-mail
perdido sai do produto (§ 6), todo caso desses vira atendimento manual do
teatro. Em um teatro comunitário o volume esperado é baixo, mas não é zero.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Nível de escopo | N1 — criação de conta, perfil, troca de e-mail, encerramento e listagem administrativa entram juntos no MVP. |
| Quem edita o quê | Cada pessoa edita só os próprios dados. Ninguém — nem administrador — edita dados de terceiros. |
| Nome | Editável livremente pela própria pessoa, a qualquer momento. |
| CPF | **Imutável** após o cadastro — decisão de 2026-09-11 mantida em 2026-09-17 (edição foi considerada e descartada por simplicidade). Nunca editável, nem pela própria pessoa nem por administrador. |
| Troca de e-mail — onde vive | Aqui, não em `identity-auth`. O e-mail é dado da conta, e a troca usa exatamente o mesmo ritual do encerramento (link de confirmação para o e-mail atual). `identity-auth` fica com o que é sessão e credencial: entrar, sair, continuar conectado, trocar e recuperar senha. |
| Troca de e-mail — mecanismo | A pessoa autenticada informa o novo endereço; o link de confirmação vai para o **e-mail atual** da conta. Só quando esse link é aberto o e-mail muda. Substitui a decisão de 2026-09-11 (link para o endereço novo, senha atual no pedido). |
| Troca de e-mail — por que no e-mail atual | Anti-sequestro: quem tomou uma sessão emprestada não consegue trocar o e-mail sem ter também a caixa de entrada da pessoa. O dono da conta fica sabendo da tentativa, porque o aviso chega onde ele lê. Escolha explícita do PO em 2026-09-17, com o custo registrado em § 6 e § 9. |
| Troca de e-mail — efeito na sessão | Confirmar a troca **derruba todas as sessões**, em todos os dispositivos, e exige novo login. Substitui a decisão de 2026-09-11 ("não derruba a sessão atual"). É o que fecha o cerco: mesmo que o intruso tenha disparado a troca, ele perde o acesso no momento em que o dono confirma. |
| Digitação do novo e-mail | A pessoa informa o novo endereço **duas vezes**, e a plataforma manda um aviso de cortesia para o endereço novo assim que a troca é confirmada. Como nada mais comprova que o endereço novo existe, um erro de digitação confirmado deixa a conta com um e-mail inalcançável — sem ingresso, sem recuperação de senha, sem volta. Decisão do PO (2026-09-17): mitigação confirmada, entra no N1. |
| Encerramento de conta — quem | **Só o próprio dono**, para a própria conta. Nenhum administrador encerra a conta de ninguém. Substitui integralmente a decisão de 2026-09-11. |
| Encerramento de conta — mecanismo | A pessoa autenticada pede o encerramento; um link de confirmação vai para o e-mail da conta; a conta só é encerrada quando o link é aberto. Mesmo ritual e mesma validade do link de recuperação de senha. |
| Encerramento — último administrador | O **único administrador restante da plataforma não consegue encerrar a própria conta**. Sem isso, o teatro pode ficar sem nenhum administrador e sem caminho de volta — a única forma de criar um administrador é o convite de outro administrador (`identity-admin-invite`). Trava de segurança aceita, sem necessidade de decisão de produto separada. |
| Encerramento — o que a pessoa perde | O aviso antes de confirmar diz, com todas as letras: as sessões futuras com ingresso já emitido continuam valendo na porta (o ingresso é validado pelo código, não pela conta), mas o histórico de pedidos e o pedido de cancelamento/reembolso (RF07) deixam de estar ao alcance dela. |
| Encerramento — compra em andamento | Bloqueado enquanto houver reserva aberta ou pagamento em processamento, com mensagem explicando que é preciso concluir ou deixar expirar antes. Evita encerrar uma conta no meio de um checkout com assento bloqueado. |
| Efeito do encerramento | A conta para de autenticar e some das listagens; o histórico de pedidos, ingressos e convites ligados a ela permanece íntegro (não é apagado de verdade — ver `identity-admin-invite` e `identity-order-history`). |
| Listagem administrativa | Só administrador. Mostra nome, e-mail, papel e data de criação. **Sem CPF** (RNF01). É uma tela de leitura: nenhuma ação sobre a conta de terceiros nasce dela. |
| Verificação de e-mail no cadastro | Continua fora de escopo — o MVP aceita o e-mail informado no cadastro sem verificar posse. |

## 9. Perguntas abertas

Nenhuma. As três pendências abertas pelo reescopo de 2026-09-17 foram fechadas
pelo PO no mesmo alinhamento:

1. **CPF editável × RN01** — descartado: CPF continua imutável após o
   cadastro (decisão de 2026-09-11 mantida). RN01 não é afetada.
2. **Ninguém remove o administrador que saiu** — revogação do papel de
   administrador fica para depois, fora deste N1; o teatro aceita conviver com
   o risco por agora (ver § 6).
3. **Quem perdeu o e-mail perdeu a conta** — confirmado: o caminho é contato
   manual com o teatro, sem compromisso de produto no N1; a mitigação de
   digitação (endereço novo informado duas vezes + aviso de cortesia) entra no
   N1 (ver § 8).
