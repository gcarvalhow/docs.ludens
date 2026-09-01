---
status: approved
domain: identity
created_at: 2026-09-01
approved_at: 2026-09-01
---

# Cadastro e autenticação do comprador

## 1. Visão da feature

Antes de comprar um ingresso, a pessoa cria uma conta com CPF, e-mail e senha, e
depois entra com e-mail e senha. Uma vez dentro, ela continua conectada entre
visitas sem precisar digitar a senha de novo a cada vez, e pode sair quando
quiser. Se esquecer a senha, pede uma redefinição por e-mail e volta a acessar a
conta em poucos minutos.

É o que separa "navegar o catálogo" de "ter ingressos, um histórico e um CPF
associado a cada compra".

## 2. Problema que resolve

Sem conta, não há como amarrar uma compra a uma pessoa, aplicar o limite de 6
ingressos por CPF por sessão (RN01), nem oferecer um histórico consultável
(RF06). E sem um login que se mantém, a pessoa teria que se autenticar a cada
passo do checkout — fricção que faz desistir da compra.

Hoje isso não existe: a venda informal pela internet não identifica o comprador,
o que impede qualquer controle e qualquer histórico.

## 3. Para quem é

- **Beneficiário direto:** o visitante que quer comprar (vira `Comprador` ao se
  cadastrar/autenticar).
- **Beneficiário indireto:** o teatro, que passa a ter cada compra ligada a um
  CPF — base para RN01, para o histórico e para o contato em caso de
  cancelamento de sessão.

Entra logo no início da jornada: a pessoa encontra a sessão, decide comprar, e o
sistema pede cadastro/login antes da reserva.

## 4. Como melhora a experiência atual

**Antes:** compra informal, sem identidade, sem histórico, sem como recuperar um
ingresso perdido nem provar que comprou.

**Depois:** conta própria, login que se mantém entre visitas, "Minhas compras"
com todos os pedidos e ingressos, e recuperação de senha autônoma sem falar com
o teatro.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** nenhuma — é a base. É pré-requisito de
`booking-reservation` (RF03), `payment-pix-checkout` (RF04),
`identity-order-history` (RF06) e das rotas de administração (RF08, que exigem
papel `ADMIN`).

**O que habilita:** todo o resto do fluxo autenticado.

**Posição no produto:** core, N1.

**RF/RN cobertos:** RF09. Reforça RNF01 (hash de senha, nenhum dado pessoal em
log/URL/erro).

## 6. O que não é (escopo negativo)

- **Não inclui** login social (Google, etc.) — pode ser um método adicional no
  futuro, nunca o único.
- **Não inclui** verificação de e-mail por link de confirmação no cadastro — o
  MVP aceita o e-mail informado; a recuperação de senha já valida a posse do
  e-mail quando necessário.
- **Não inclui** autenticação em dois fatores.
- **Não inclui** gestão de múltiplos endereços, dados de perfil além de
  nome/CPF/e-mail, ou edição de CPF (o CPF é imutável após o cadastro).
- **Não inclui** papéis além de `BUYER` e `ADMIN` — sem permissão granular.
- **Não inclui** criação de administrador pela interface — o admin é semeado por
  script (`seed_admin.py`), fora do fluxo público.

## 7. Custos adicionais

Nenhum custo externo próprio. O envio do e-mail de recuperação de senha usa o
mesmo serviço de e-mail transacional de `notification-transactional-email` — não
é um provedor novo.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Identificador de login | E-mail + senha. O CPF é obrigatório no cadastro (RN01) mas não é usado para autenticar. |
| Sessão que se mantém | Dual-token JWT: access token curto (30 min) + refresh token opaco (7 dias) em cookie `HttpOnly; Secure; SameSite=Strict`. Ver `docs.ludens/backend/security/authentication.md`. |
| Logout | Regenera o `security_stamp` do comprador — invalida todos os tokens em qualquer dispositivo. |
| Troca / recuperação de senha | Também regenera o `security_stamp` — desconecta todos os dispositivos. |
| Link de recuperação | Expira em 1 hora. Uso único. |
| Resposta a "esqueci a senha" com e-mail inexistente | Mensagem idêntica à de e-mail existente (não revela se o e-mail está cadastrado). |
| Hash de senha | bcrypt. |
| CPF | Validado (dígitos verificadores) no cadastro; imutável depois. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
