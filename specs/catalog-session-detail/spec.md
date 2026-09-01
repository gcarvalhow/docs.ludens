---
status: approved
domain: catalog
created_at: 2026-09-01
approved_at: 2026-09-01
---

# Detalhe da sessão com disponibilidade em tempo real

## 1. Visão da feature

Ao escolher um espetáculo, a pessoa vê a página de uma sessão específica: data,
horário, local, os tipos de ingresso (inteira e meia) com seus preços, e —
o mais importante — **quantos ingressos ainda estão disponíveis, atualizado em
tempo quase real**. Se a sessão está esgotada, isso fica claro e o botão de
reservar some.

## 2. Problema que resolve

Sem disponibilidade confiável e visível, a pessoa não sabe se vale a pena tentar
comprar, e o teatro não tem como mostrar "última chance" ou "esgotado" de forma
honesta. É a informação que falta hoje na venda informal.

## 3. Para quem é

- **Beneficiário direto:** o visitante decidindo a compra.
- **Beneficiário indireto:** o teatro (menos frustração na porta; base para
  RN05).

## 4. Como melhora a experiência atual

**Antes:** comprar às cegas, sem saber se ainda há lugar.

**Depois:** número de disponíveis visível e atualizado; esgotado sinalizado;
decisão informada.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** `catalog-admin-management` (a sessão existe);
`catalog-show-search` (chega-se aqui a partir da vitrine).

**O que habilita:** `booking-reservation` (RF03) — a página tem o botão de
reservar; e o módulo `catalog` passa a **expor para o `booking`** a trava de
linha da sessão e a contagem de ingressos confirmados, que o RN05 exige.

**Posição:** core, N1.

**RF/RN cobertos:** RF02 · RN05 (leitura da disponibilidade). Meta:
disponibilidade ≤ 1 s p95 (RNF02).

## 6. O que não é (escopo negativo)

- **Não inclui** mapa de assentos com poltronas numeradas — só o número de
  disponíveis por sessão (setores A/B/C sem numeração no N1).
- **Não inclui** atualização por WebSocket/push — o frontend faz *polling* curto.
- **Não inclui** a reserva em si (feature separada).
- **Não inclui** avaliações, comentários ou compartilhamento social.

## 7. Custos adicionais

Nenhum custo externo identificado.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Disponibilidade | `capacidade − ingressos confirmados − reservas abertas não vencidas`. |
| Atualização | *Polling* do frontend a cada ~15 s enquanto a página está aberta. |
| Esgotado | Quando disponível = 0: sinaliza "esgotado" e não permite reservar. |
| Precisão | O número exibido é uma leitura; a verdade sob concorrência é decidida na criação da reserva, sob trava (RN05). |
| Exports para o `booking` | O `catalog` expõe `lock_session_for_update` e `count_confirmed_tickets_for_session` em `dependencies.py`. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
