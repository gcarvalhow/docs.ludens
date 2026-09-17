---
status: draft
spec: identity-auth
created_at: 2026-09-01
updated_at: 2026-09-11
---

# Autenticação e sessão de usuário — Lógica de Negócio

> **Nota de reescopo (2026-09-11):** o fluxo de cadastro (seção "Visitante" da
> versão anterior) saiu daqui — ver
> [`identity-user-management/logic.md`](../identity-user-management/logic.md).
> Status voltou a `draft`: precisa de nova revisão conjunta de FE/BE antes de
> valer como `reviewed` outra vez.

## 1. Fluxo por perfil

### Pessoa com conta (Comprador ou Admin), não autenticada

1. Abre `/login`, informa e-mail e senha.
2. Credenciais corretas → autenticado, redirecionado.
3. Credenciais incorretas → mensagem genérica ("E-mail ou senha inválidos"),
   sem dizer qual dos dois está errado.

### Pessoa autenticada (Comprador ou Admin)

- **Continuar conectado:** quando o access token expira (30 min), o cliente usa
  o cookie de refresh para obter um novo par sem pedir a senha. O refresh token
  é rotacionado a cada uso.
- **Sair:** o `security_stamp` do usuário é regenerado; todos os tokens
  (access e refresh) emitidos antes deixam de valer, em qualquer dispositivo.
- **Trocar a senha (logado):** informa a senha atual + a nova. Sucesso regenera
  o `security_stamp` — desconecta todos os dispositivos, inclusive o atual, que
  precisa logar de novo.
- **Alterar e-mail (logado):** informa o novo e-mail (e a senha atual, pra
  confirmar que é a própria pessoa pedindo). O sistema envia um link de
  confirmação para o **novo** endereço; o e-mail da conta só muda quando esse
  link é aberto. Até lá, a pessoa continua entrando com o e-mail antigo.

### Recuperação de senha (não autenticado)

1. Em `/recuperar-senha`, informa o e-mail e pede o link.
2. O sistema responde **sempre** com a mesma mensagem ("Se houver uma conta com
   esse e-mail, enviamos um link de redefinição"), exista ou não a conta.
3. Se a conta existe, um e-mail com um link contendo um token de redefinição
   (uso único, validade 1 hora) é enviado.
4. A pessoa abre o link, define a nova senha. Sucesso: `security_stamp`
   regenerado, o token de redefinição é invalidado, a pessoa é levada ao login.
5. Link expirado ou já usado → mensagem "Este link não é mais válido, solicite
   um novo".

## 2. Estados e transições

### Sessão de autenticação (do ponto de vista da pessoa)

**Estados:** anônima · autenticada · expirada-renovável.

- **anônima → autenticada:** login (a conta já precisa existir — ver
  `identity-user-management`).
- **autenticada → expirada-renovável:** passados 30 min sem renovar.
- **expirada-renovável → autenticada:** renovação silenciosa via cookie de
  refresh.
- **qualquer → anônima:** logout, troca de senha, redefinição de senha, ou
  refresh token vencido (7 dias sem uso).

### Token de redefinição de senha

**Estados:** válido · usado · expirado.

- **inexistente → válido:** solicitação de recuperação para um e-mail cadastrado.
- **válido → usado:** a nova senha é definida com sucesso.
- **válido → expirado:** passada 1 hora da emissão.

### Solicitação de troca de e-mail

**Estados:** pendente · confirmada · expirada.

- **inexistente → pendente:** pessoa autenticada pede a troca, informando o
  novo endereço e a senha atual.
- **pendente → confirmada:** a pessoa abre o link enviado ao novo e-mail; a
  partir daí o e-mail da conta é o novo, e é o que passa a valer pra login.
- **pendente → expirada:** passada 1 hora da solicitação sem confirmação — o
  e-mail da conta não muda; a pessoa precisa solicitar de novo se ainda quiser
  trocar.
- Uma nova solicitação de troca invalida qualquer solicitação anterior ainda
  pendente (mesmo padrão do link de recuperação de senha: só o último vale).

## 3. Regras de negócio

- Login com e-mail inexistente e login com senha errada → **mesma** mensagem
  genérica (não revela se o e-mail existe).
- "Esqueci a senha" para e-mail inexistente → **mesma** resposta que para
  e-mail existente (não revela cadastro).
- Logout, troca de senha e redefinição de senha → regeneram o `security_stamp`
  → invalidam todos os tokens anteriores.
- Token de redefinição de senha: uso único, validade 1 hora.
- Alteração de e-mail exige a senha atual no pedido (evita que uma sessão
  sequestrada troque o e-mail sem reconfirmar a senha) e confirmação de posse
  do novo endereço por link antes de valer.
- Token de confirmação de novo e-mail: uso único, validade 1 hora. Uma nova
  solicitação invalida a anterior.
- Alteração de e-mail **não** exige nova senha nem regenera o
  `security_stamp` — a sessão atual continua válida; só o e-mail usado para
  logar da próxima vez muda.
- O novo e-mail não pode já pertencer a outra conta — recusado com mensagem
  específica se já estiver em uso.
- Senha é sempre guardada como hash bcrypt; nunca retornada, nunca logada
  (RNF01). O mesmo vale para o token de confirmação de e-mail.
- Nenhuma rota devolve senha (hash ou não) de nenhum usuário, nem o e-mail
  pendente de confirmação de terceiros.

## 4. Pontos de integração

```text
Frontend precisa saber:
  - Que o access token vai no header Authorization: Bearer e expira em ~30 min
  - Que a renovação é automática via cookie HttpOnly de refresh — o frontend não
    lê nem guarda o refresh token; só chama a rota de refresh quando recebe 401
  - Que logout / troca de senha invalidam a sessão em todos os dispositivos —
    após qualquer um, tratar como anônimo e redirecionar ao login
  - Que alterar e-mail NÃO invalida a sessão atual — a tela pode mostrar
    "verifique seu novo e-mail" sem deslogar a pessoa
  - As mensagens de erro por caso (login: genérica; recuperação: sempre a
    mesma; troca de e-mail: e-mail já em uso é mensagem específica)
  - Que os links de recuperação de senha e de confirmação de e-mail levam a
    rotas do próprio frontend (/redefinir-senha?token=... e
    /confirmar-email?token=...) que chamam a API para efetivar

Backend precisa garantir:
  - Hash bcrypt da senha; nunca devolver nem logar senha ou token de
    confirmação em erro
  - Emissão do par access+refresh no login; rotação do refresh a cada uso;
    cookie HttpOnly; Secure; SameSite=Strict; Path restrito
  - security_stamp no claim do access token; regeneração no logout, troca de
    senha e redefinição de senha (não na troca de e-mail); recusa de qualquer
    token cujo security_stamp não bata
  - Token de redefinição de senha uso único, validade 1h; resposta neutra em
    "esqueci a senha"
  - Token de confirmação de e-mail uso único, validade 1h; nova solicitação
    invalida a anterior; recusa se o novo e-mail já pertence a outra conta
  - Envio de ambos os e-mails via evento de outbox (não bloqueia a resposta)
```

## 5. Casos de borda

**Refresh token roubado e usado em paralelo com o legítimo.** A rotação a cada
uso faz o segundo uso do mesmo refresh token falhar; a detecção de reuso
regenera o `security_stamp` e derruba as duas sessões — a pessoa loga de novo.
`[decisão fechada]`

**Pessoa solicita vários links de recuperação de senha seguidos.** Cada
solicitação emite um token novo e **invalida os anteriores** — só o último
link funciona. `[decisão fechada]`

**Pessoa solicita várias trocas de e-mail seguidas, pra endereços
diferentes.** Mesmo padrão: só a solicitação mais recente tem link válido; as
anteriores, mesmo não confirmadas, deixam de valer. `[decisão fechada]`

**Pessoa confirma a troca de e-mail, mas o e-mail que ela quis trocar já foi
usado por outra conta enquanto o link estava pendente.** A confirmação é
recusada nesse momento (checa unicidade de novo), com a mesma mensagem de
"e-mail já em uso"; o e-mail da conta permanece o antigo. `[decisão fechada]`

**E-mail de recuperação ou de confirmação não chega (serviço de e-mail fora do
ar).** A resposta da API já foi dada; o envio é um handler de outbox
idempotente que o relay retenta. A pessoa pode solicitar de novo.
`[decisão fechada]`
