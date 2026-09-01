---
status: alvo
spec: payment-pix-checkout
updated_at: 2026-09-01
responsavel: Igor (Backend)
---

# Integration Contract — Pagamento Pix e criação do pedido

**Status:** alvo. **Módulo backend:** `payment`.

## Rotas

| Método | Caminho | Auth | Sucesso |
| --- | --- | --- | --- |
| POST | `/checkout` | Bearer (comprador) | 201 |
| GET | `/orders/{id}` | Bearer (dono) | 200 |
| POST | `/payments/webhook` | assinatura do gateway | 200 |

## Request / Response

- `POST /checkout` → `{ reservationId }` → 201
  `{ orderId, status: "pending", amount, pixQrCode, pixCopyPaste, expiresAt }`.
- `GET /orders/{id}` → `{ id, sessionId, quantity, ticketType, amount, status,
  createdAt, tickets: [{ id, type, code }] (quando pago) }`.
- `POST /payments/webhook` → corpo do AbacatePay; valida assinatura
  (`ABACATEPAY_WEBHOOK_SECRET`); 200 sempre que processado (mesmo se duplicado).

## Erros esperados

| Rota | Status | Quando | Mensagem |
| --- | --- | --- | --- |
| checkout | 409 | reserva não está "aberta" | "Sua reserva não está mais ativa." |
| checkout | 409 | reserva de outro comprador | "Reserva não encontrada." |
| checkout | 502 | falha ao criar cobrança no gateway | "Não foi possível iniciar o pagamento, tente de novo." |
| orders | 404 | pedido de outro comprador | "Pedido não encontrado." |

## Estados

`pending → paid` (webhook) · `pending → failed` (webhook recusa, ou reserva
expira) · `paid → refunded` (RF07).

## Impacto de UX

Após `POST /checkout`, exibir QR + copia-e-cola + valor; fazer polling de
`GET /orders/{id}` até `status != pending`. `paid` → confirmação com ingressos
(RF05). `failed` → voltar à sessão. 502 → botão "tentar de novo".

## Semântica de dados

Nenhum dado do pagador é retornado ou persistido. `pixQrCode` pode ser
imagem base64 ou URL — o frontend renderiza o que vier. Segredos do AbacatePay
só em variáveis de ambiente.

## Lacunas / decisões em aberto

- Formato exato do payload do webhook do AbacatePay: confirmar na doc do gateway
  ao implementar; mapear campos de status para `paid`/`failed`.
- Prefixo/base path, versionamento, envelope de erro — `<a definir globalmente>`.
