---
status: approved
domain: notification
created_at: 2026-09-01
approved_at: 2026-09-01
---

# E-mails transacionais

## 1. Visão da feature

O sistema envia e-mails automáticos nos momentos que importam: confirmação da
compra com os ingressos, redefinição de senha, aviso de reembolso, e aviso
quando uma sessão comprada é cancelada ou tem o horário alterado. Nenhum desses
e-mails é enviado à mão — eles saem sozinhos quando o evento acontece, em poucos
minutos, e uma falha de envio nunca desfaz o que já aconteceu no sistema.

## 2. Problema que resolve

Sem e-mails automáticos, o comprador fica sem prova da compra fora da tela, sem
recuperação de senha, e o teatro precisa avisar cada pessoa manualmente quando
algo muda. É a peça que fecha a comunicação da plataforma com quem compra.

## 3. Para quem é

- **Beneficiário direto:** o `Comprador`, que recebe confirmações e avisos.
- **Beneficiário indireto:** o teatro, que não precisa avisar ninguém à mão.

## 4. Como melhora a experiência atual

**Antes:** comunicação informal, caso a caso, sujeita a esquecimento.

**Depois:** todo evento relevante gera um e-mail consistente, automático, com o
conteúdo certo.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** os eventos de domínio de `payment`
(`OrderPaid`, `OrderRefunded`), `identity` (`PasswordResetRequested`),
`catalog` (`SessionCancelled`, sessão alterada) e `booking`
(reenvio de ingresso).

**O que habilita:** RF05 (confirmação), RF09 (recuperação de senha), e o aviso de
mudança/cancelamento de sessão (RF08 → comprador).

**Posição:** módulo de suporte, N1. **Não tem aggregate próprio** — é só um
conjunto de handlers de evento (outbox) + um serviço de envio.

**RF/RN cobertos:** RF05, RF09 (parcial — a parte de e-mail).

## 6. O que não é (escopo negativo)

- **Não inclui** SMS, WhatsApp ou push — só e-mail no N1.
- **Não inclui** central de preferências de notificação do usuário (opt-out por
  tipo) — os e-mails transacionais são sempre enviados.
- **Não inclui** e-mail de marketing / newsletter.
- **Não inclui** editor de templates pela interface — os templates são código.
- **Não inclui** um dashboard de entregabilidade — só log de sucesso/falha.

## 7. Custos adicionais

**Depende de um serviço de e-mail transacional** (SMTP) — já previsto nas
premissas do produto. É uma dependência de configuração (host, credenciais,
remetente), trocável sem mexer no domínio. Sem provedor novo além desse.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Arquitetura | Só handlers de outbox + um `EmailService`. Sem aggregate, sem tabela própria (além de, opcionalmente, um log de envios). |
| Gatilhos no N1 | `OrderPaid` → confirmação + ingressos; `PasswordResetRequested` → link; `OrderRefunded` → aviso de reembolso; `SessionCancelled` → aviso ao comprador; alteração de horário de sessão vendida → aviso; reenvio de ingresso (ação do comprador) → reenvia a confirmação. |
| Idempotência | Todo handler é idempotente por (destinatário, tipo de e-mail, referência) — o relay entrega at-least-once. |
| Falha de envio | Logada; o relay retenta. **Nunca** desfaz a transação de origem (RF05). |
| Prazo | Confirmação em ≤ 5 min após a aprovação (RF05) — folgado para a latência do relay (~2 s) + o SMTP. |
| Conteúdo | Texto claro em pt-BR, com os dados da sessão e o código/QR do ingresso quando aplicável. Sem dado sensível além do necessário (RNF01). |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
