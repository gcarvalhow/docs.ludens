---
status: reviewed
spec: identity-auth
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# Cadastro e autenticação do comprador — Lógica de Negócio

## 1. Fluxo por perfil

### Visitante

1. Abre `/registro`, informa nome, CPF, e-mail e senha.
2. O sistema valida: CPF com dígitos verificadores corretos e ainda não
   cadastrado; e-mail com formato válido e ainda não cadastrado; senha com
   mínimo de 8 caracteres.
3. Confirma. A conta é criada, o comprador é autenticado imediatamente (recebe
   access token + cookie de refresh) e cai na tela de onde veio (catálogo ou
   detalhe da sessão).
4. Se o CPF ou o e-mail já existem: a criação é recusada com mensagem
   específica ("Este CPF já possui cadastro" / "Este e-mail já está em uso").

### Visitante que já tem conta

1. Abre `/login`, informa e-mail e senha.
2. Credenciais corretas → autenticado, redirecionado.
3. Credenciais incorretas → mensagem genérica ("E-mail ou senha inválidos"),
   sem dizer qual dos dois está errado.

### Comprador autenticado

- **Continuar conectado:** quando o access token expira (30 min), o cliente usa
  o cookie de refresh para obter um novo par sem pedir a senha. O refresh token
  é rotacionado a cada uso.
- **Sair:** o `security_stamp` do comprador é regenerado; todos os tokens
  (access e refresh) emitidos antes deixam de valer, em qualquer dispositivo.
- **Trocar a senha (logado):** informa a senha atual + a nova. Sucesso regenera
  o `security_stamp` — desconecta todos os dispositivos, inclusive o atual, que
  precisa logar de novo.

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

### Conta do comprador

**Estados:** ativa · inativa (soft delete — fora do fluxo do MVP, existe só como
mecanismo).

- **inexistente → ativa:** cadastro bem-sucedido.
- Não há autoexclusão de conta no MVP.

### Sessão de autenticação (do ponto de vista da pessoa)

**Estados:** anônima · autenticada · expirada-renovável.

- **anônima → autenticada:** login ou cadastro.
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

## 3. Regras de negócio

- CPF sem dígitos verificadores válidos → cadastro recusado (RF09).
- CPF ou e-mail já cadastrado → cadastro recusado, mensagem específica por campo.
- Senha com menos de 8 caracteres → recusada na validação de forma.
- Senha é sempre guardada como hash bcrypt; nunca retornada, nunca logada (RNF01).
- Login com e-mail inexistente e login com senha errada → **mesma** mensagem
  genérica (não revela se o e-mail existe).
- "Esqueci a senha" para e-mail inexistente → **mesma** resposta que para e-mail
  existente (não revela cadastro).
- Logout, troca de senha e redefinição de senha → regeneram o `security_stamp`
  → invalidam todos os tokens anteriores.
- Token de redefinição: uso único, validade 1 hora.
- Nenhuma rota devolve CPF, e-mail ou hash de senha de outro usuário. O comprador
  só acessa os próprios dados; o admin acessa por necessidade operacional (RF08).

## 4. Pontos de integração

```text
Frontend precisa saber:
  - Que o access token vai no header Authorization: Bearer e expira em ~30 min
  - Que a renovação é automática via cookie HttpOnly de refresh — o frontend não
    lê nem guarda o refresh token; só chama a rota de refresh quando recebe 401
  - Que logout / troca de senha invalidam a sessão em todos os dispositivos —
    após qualquer um, tratar como anônimo e redirecionar ao login
  - As mensagens de erro por caso (cadastro: campo específico; login: genérica;
    recuperação: sempre a mesma)
  - Que o link de recuperação leva a uma rota do próprio frontend
    (/redefinir-senha?token=...) que chama a API para efetivar

Backend precisa garantir:
  - Hash bcrypt da senha; nunca devolver nem logar senha, CPF ou e-mail em erro
  - Validação de CPF (dígitos verificadores) e unicidade de CPF e e-mail
  - Emissão do par access+refresh no cadastro e no login; rotação do refresh a
    cada uso; cookie HttpOnly; Secure; SameSite=Strict; Path restrito
  - security_stamp no claim do access token; regeneração no logout, troca e
    redefinição de senha; recusa de qualquer token cujo security_stamp não bata
  - Token de redefinição uso único, validade 1h; resposta neutra em "esqueci a
    senha"; envio do e-mail via evento de outbox (não bloqueia a resposta)
```

## 5. Casos de borda

**Refresh token roubado e usado em paralelo com o legítimo.** A rotação a cada
uso faz o segundo uso do mesmo refresh token falhar; a detecção de reuso
regenera o `security_stamp` e derruba as duas sessões — a pessoa loga de novo.
`[decisão fechada]`

**Pessoa solicita vários links de recuperação seguidos.** Cada solicitação emite
um token novo e **invalida os anteriores** — só o último link funciona.
`[decisão fechada]`

**E-mail de recuperação não chega (serviço de e-mail fora do ar).** A resposta da
API já foi dada (neutra); o envio é um handler de outbox idempotente que o relay
retenta. A pessoa pode solicitar de novo. `[decisão fechada]`

**Cadastro com e-mail de outra pessoa (sem verificação).** Aceito no MVP — o
escopo negativo da spec exclui verificação de e-mail no cadastro. A recuperação
de senha, essa sim, comprova posse do e-mail. `[decisão fechada]`
