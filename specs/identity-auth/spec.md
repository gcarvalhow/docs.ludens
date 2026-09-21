---
status: approved
domain: identity
created_at: 2026-09-01
approved_at: 2026-09-01
updated_at: 2026-09-17
---

# Autenticação e sessão de usuário

> **Nota de reescopo (2026-09-17):** a **alteração de e-mail sai desta spec** e
> passa a viver em
> [`identity-user-management`](../identity-user-management/spec.md), junto com o
> encerramento de conta — as duas são mudanças no dado da conta confirmadas por
> link enviado ao e-mail atual, e formam uma família só. O mecanismo também
> mudou: a confirmação passa a ir para o **e-mail atual** (era para o novo), não
> se pede mais a senha atual no pedido, e confirmar **derruba todas as sessões**
> (antes: não derrubava nenhuma). O motivo da escolha (anti-sequestro de sessão)
> e o que ela custa estão registrados em
> [`identity-user-management` § 8 e § 9](../identity-user-management/spec.md#8-decisões-tomadas).
>
> Esta spec fica com o que é **sessão e credencial**: entrar, sair, continuar
> conectado, trocar a senha e redefinir a senha esquecida. Ela continua dona do
> mecanismo de invalidação de sessão que `identity-user-management` aciona ao
> confirmar uma troca de e-mail ou um encerramento de conta.
>
> Origem: decisão do PO em 2026-09-17, ao alinhar as specs de conta com o plano
> técnico da Workstream B do módulo `identity`.
>
> **Nota de reescopo (2026-09-11):** esta spec cobria originalmente cadastro +
> autenticação do comprador. O cadastro (criação de conta) e a consulta de
> perfil (próprio ou por um administrador) saíram daqui e agora vivem em
> [`identity-user-management`](../identity-user-management/spec.md). Esta spec
> passa a cobrir só os mecanismos de **sessão e credencial** — o que é comum a
> qualquer conta já existente, seja `Comprador` ou `Admin`: entrar, sair,
> continuar conectado e redefinir senha esquecida (a alteração de e-mail, que
> esta nota de 2026-09-11 trazia para cá, saiu em 2026-09-17 — ver acima).
> `backend.md`
> / `frontend.md` / `quality.md` ainda descrevem o código sob o escopo antigo
> (que inclui cadastro) e precisam de rework antes de bater com este documento
> — ver aviso no topo de cada um.

## 1. Visão da feature

Depois que a pessoa tem uma conta, ela entra com e-mail e senha e continua
conectada entre visitas, sem precisar digitar a senha de novo a cada vez, até
decidir sair. Se esquecer a senha, pede uma redefinição por e-mail e volta a
acessar a conta em poucos minutos. Se quiser trocar a senha que já sabe, troca —
e todos os dispositivos conectados caem junto, para que trocar a senha seja de
fato uma forma de retomar o controle da conta.

Vale tanto para quem compra ingresso (`Comprador`) quanto para quem administra
o catálogo (`Admin`) — é o mesmo mecanismo de sessão para as duas contas, só
muda o que cada uma pode fazer depois de entrar.

## 2. Problema que resolve

Sem um login que se mantém, a pessoa teria que se autenticar a cada passo do
checkout — fricção que faz desistir da compra. Sem redefinição de senha
autônoma, uma senha esquecida vira um chamado pro teatro resolver na mão. E sem
uma troca de senha que derrube as outras sessões, quem desconfia que deixou a
conta aberta em algum lugar não tem como fechar essa porta sozinho.

## 3. Para quem é

- **Beneficiário direto:** qualquer pessoa com conta na plataforma —
  `Comprador` ou `Admin`.
- **Beneficiário indireto:** o teatro, que deixa de precisar intervir
  manualmente em senha esquecida.

Entra toda vez que uma pessoa com conta volta à plataforma, ou perde acesso a
ela.

## 4. Como melhora a experiência atual

**Antes:** login que se mantém entre visitas, recuperação de senha autônoma —
já cobertos. Trocar o e-mail cadastrado não tinha caminho nenhum: a pessoa
ficava presa ao e-mail do cadastro original.

**Depois:** além de login persistente e recuperação de senha, a pessoa também
consegue atualizar o e-mail da própria conta, com a mesma segurança
(confirmação por link) usada na redefinição de senha.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:**
[`identity-user-management`](../identity-user-management/spec.md) — a conta
precisa existir (criada por lá) antes de qualquer login; esta spec assume que
o cadastro já aconteceu.
[`notification-transactional-email`](../notification-transactional-email/spec.md)
— envio do e-mail de redefinição de senha e do e-mail de confirmação de troca
de e-mail.

**O que habilita:** é pré-requisito de `booking-reservation` (RF03),
`payment-pix-checkout` (RF04), `identity-order-history` (RF06) e de qualquer
rota que exija sessão — comprador ou administrador.

**Posição no produto:** core, N1.

**RF/RN cobertos:** RF09 (parte de autenticação). Reforça RNF01 (hash de
senha, nenhum dado pessoal em log/URL/erro).

## 6. O que não é (escopo negativo)

- **Não inclui** criação de conta (cadastro) — ver
  [`identity-user-management`](../identity-user-management/spec.md).
- **Não inclui** consulta de perfil, próprio ou de terceiros — ver
  [`identity-user-management`](../identity-user-management/spec.md).
- **Não inclui** login social (Google, etc.) — pode ser um método adicional no
  futuro, nunca o único.
- **Não inclui** autenticação em dois fatores.
- **Não inclui** verificação do e-mail original no cadastro — só a *troca* de
  e-mail exige confirmação de posse; o e-mail informado no cadastro inicial
  continua aceito sem verificação (decisão que pertence a
  `identity-user-management`).

## 7. Custos adicionais

Nenhum custo externo próprio. O envio dos e-mails de redefinição de senha e de
confirmação de troca de e-mail usa o mesmo serviço de e-mail transacional de
`notification-transactional-email` — não é um provedor novo.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Identificador de login | E-mail + senha. |
| Sessão que se mantém | Dual-token JWT: access token curto (30 min) + refresh token opaco (7 dias) em cookie `HttpOnly; Secure; SameSite=Strict`. Ver `docs.ludens/backend/security/authentication.md`. |
| Logout | Regenera o `security_stamp` do usuário — invalida todos os tokens em qualquer dispositivo. |
| Troca / recuperação de senha | Também regenera o `security_stamp` — desconecta todos os dispositivos. |
| Link de recuperação de senha | Expira em 1 hora. Uso único. |
| Resposta a "esqueci a senha" com e-mail inexistente | Mensagem idêntica à de e-mail existente (não revela se o e-mail está cadastrado). |
| Alteração de e-mail | Exige confirmação de posse do **novo** endereço por link antes de valer — mesmo padrão da redefinição de senha. Enquanto não confirmado, o e-mail antigo continua sendo o de login. |
| Link de confirmação de novo e-mail | Expira em 1 hora. Uso único. Mesma janela do link de senha, por consistência. |
| Hash de senha | bcrypt. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
