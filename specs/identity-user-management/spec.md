---
status: approved
domain: identity
created_at: 2026-09-11
approved_at: 2026-09-11
---

# Gestão de conta de usuário

## 1. Visão da feature

Toda pessoa com conta na plataforma — comprador ou administrador — nasce aqui:
é aqui que a conta é criada. Depois de criada, é aqui também que a pessoa vê e
corrige seus próprios dados básicos, e que um administrador enxerga quem tem
conta na plataforma e pode remover o acesso de alguém quando necessário.

É a "ficha" de cada conta — separada de como ela entra (isso é
`identity-auth`) e de como ela vira administrador (isso é
`identity-admin-invite`).

## 2. Problema que resolve

**Criar conta:** sem uma conta, não há como comprar (RN01, RF06) nem como
administrar o catálogo (RF08). É o ponto de entrada de qualquer pessoa na
plataforma.

**Ver e corrigir dados próprios:** hoje, se o nome cadastrado está errado
(erro de digitação, mudança), não existe como a própria pessoa corrigir —
precisaria de intervenção manual do teatro.

**Enxergar e remover contas (administrador):** conforme a equipe do teatro
muda — alguém sai, uma conta de comprador some do radar depois de anos — o
administrador não tem hoje nenhuma visão de quem tem conta na plataforma, nem
como encerrar o acesso de alguém. Isso já apareceu como lacuna real ao
desenhar `identity-admin-invite`: dá pra convidar um administrador novo, mas
não dá pra tirar o acesso de um que saiu.

## 3. Para quem é

- **Beneficiário direto:** toda pessoa com conta (criação e edição dos
  próprios dados); o administrador (visão de quem tem conta e remoção de
  acesso).
- **Beneficiário indireto:** o teatro, que passa a ter controle sobre quem
  acessa a plataforma sem depender de intervenção técnica.

Criar conta acontece no início da jornada de qualquer pessoa. Ver/editar dados
próprios e a gestão pelo administrador são ações pontuais, fora do fluxo de
compra.

## 4. Como melhora a experiência atual

**Antes:** criar conta já existia (como parte do que hoje é `identity-auth`),
mas sem visão de perfil consultável, sem edição de dados próprios, e sem
nenhuma forma de o administrador ver ou remover contas.

**Depois:** a pessoa vê e corrige seu nome sozinha; o administrador tem uma
lista de quem tem conta na plataforma e pode encerrar o acesso de alguém —
comprador ou outro administrador — sem depender de ninguém de fora do
produto.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** nenhuma — junto com `identity-auth`, é uma das
bases do módulo de identidade.

**O que habilita:** é pré-requisito de `identity-auth` (a conta precisa
existir antes de logar), de `identity-admin-invite` (o convite aceito termina
numa conta criada por esta feature) e de tudo que exige um `Comprador` ou
`Admin` identificado (RF03, RF04, RF06, RF08).

**Posição no produto:** core, N1 — a criação de conta já era N1 dentro do
antigo escopo de `identity-auth`; a visão e a remoção de conta pelo
administrador entram junto, como extensão direta da mesma necessidade que
motivou `identity-admin-invite`.

**RF/RN cobertos:** RF09 (parte de cadastro). Toca RF08 (controle de acesso
administrador × comprador, ao listar e remover contas). Nenhum RF aprovado em
2026-08-28 cobre listagem ou remoção de conta diretamente — é extensão de
escopo na mesma linha de `identity-admin-invite`.

## 6. O que não é (escopo negativo)

- **Não inclui** login, sessão, redefinição de senha ou alteração de e-mail —
  isso é `identity-auth`. Esta feature cria a conta; não autentica ninguém.
- **Não inclui** convite ou promoção de comprador a administrador — a única
  forma de uma conta virar `Admin` é aceitar um convite
  (`identity-admin-invite`). Não existe aqui um botão "tornar administrador".
- **Não inclui** edição de e-mail ou senha pela própria pessoa — ambos ficam
  em `identity-auth`, por exigirem confirmação de posse/identidade que este
  fluxo não cobre. Aqui a pessoa edita só dados básicos de perfil (nome).
- **Não inclui** edição de CPF — imutável após o cadastro, mesma decisão já
  tomada em `identity-auth`.
- **Não inclui** exclusão da própria conta pela própria pessoa (autoexclusão)
  — só um administrador remove uma conta, e nunca a própria.
- **Não inclui** histórico de quem foi removido, motivo da remoção, ou
  reativação de conta removida — a remoção é definitiva do ponto de vista de
  produto (mesmo que preserve dado internamente para não quebrar histórico de
  pedidos).
- **Não inclui** edição de dados de outra pessoa pelo administrador — o
  administrador vê e remove contas, mas não edita nome/dados de terceiros.

## 7. Custos adicionais

Nenhum custo externo identificado.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Nível de escopo | N1 — criação de conta já era N1; visão e remoção pelo administrador entram junto, no mesmo corte de `identity-admin-invite`. |
| Quem edita o quê | Qualquer pessoa edita só os próprios dados básicos (nome). Ninguém edita dados de terceiros. |
| Quem remove conta | Só administrador. Pode remover qualquer conta — comprador ou outro administrador — exceto a própria. |
| Administrador remove a própria conta | Não permitido — evita perder acesso administrativo sem ter como convidar um substituto (mesmo risco já registrado em `identity-admin-invite`). |
| Efeito da remoção | A conta para de autenticar e some das listagens ativas; o histórico de pedidos e convites ligados a ela permanece íntegro (não é apagado de verdade — ver `identity-admin-invite` e `identity-order-history`). |
| Verificação de e-mail no cadastro | Continua fora de escopo — decisão herdada de `identity-auth`: o MVP aceita o e-mail informado sem verificar posse na criação da conta. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
