---
status: done
spec: notification-transactional-email
surface: frontend
created_at: 2026-09-11
---

# E-mails transacionais — Frontend

**Resumo:** esta feature não tem superfície de frontend própria.

**RF:** RF09 (parte) · **RN:** — · **Feature frontend:** nenhuma

## Por que não há arquivo nenhum aqui

Por `logic.md` §4 ("Pontos de integração"), o único ponto que tocaria o
frontend é a ação "reenviar ingresso" em Minhas Compras — e essa tela, seu
botão e sua chamada de API já pertencem a `identity-order-history` /
`booking-ticket-issuance` (é lá que o `POST /orders/{id}/resend-ticket` é
consumido). Este módulo (`notification`) só reage a eventos no backend; não
existe tela, componente, hook nem rota que o frontend precise construir
especificamente para ele.

A tela de "esqueci minha senha" que dispara o e-mail cujo handler este spec
implementa já existe em `identity-auth/frontend.md` — nenhuma mudança nela é
necessária por causa desta feature (o contrato `POST /auth/forgot-password`
não muda).

## Ordem entre as superfícies

Sem dependência de frontend — QA e Backend são as únicas superfícies desta
feature.
