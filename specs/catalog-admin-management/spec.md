---
status: approved
domain: catalog
created_at: 2026-09-01
approved_at: 2026-09-01
---

# Gestão de espetáculos e sessões (admin)

## 1. Visão da feature

O administrador do teatro cadastra e edita os espetáculos (título, sinopse,
imagem, categoria) e, dentro de cada um, as sessões (data, horário, capacidade,
tipos e valores de ingresso). É o que alimenta toda a vitrine e todo o fluxo de
compra. Uma sessão que já vendeu ingressos não pode ser apagada — só cancelada,
o que dispara reembolso para quem comprou.

## 2. Problema que resolve

Sem uma área de gestão, o catálogo só existiria por inserção direta no banco.
E sem a regra "não apaga sessão vendida, cancela", um erro operacional apagaria o
registro de compras reais.

## 3. Para quem é

- **Beneficiário direto:** o `Admin` (PO / equipe do teatro).
- **Beneficiário indireto:** todos os visitantes e compradores, que veem o
  resultado na vitrine.

## 4. Como melhora a experiência atual

**Antes:** catálogo mantido de forma informal, sem estrutura, sujeito a
inconsistência entre canais.

**Depois:** uma fonte única, editável com segurança, que reflete imediatamente
na vitrine.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** `identity-auth` (papel `ADMIN`).

**O que habilita:** `catalog-show-search` (RF01) e `catalog-session-detail`
(RF02) leem esses dados; `booking-reservation` usa `capacity` e `is_on_sale`;
o cancelamento de sessão dispara `payment-cancellation-refund` (RF07).

**Posição:** core, N1.

**RF/RN cobertos:** RF08.

## 6. O que não é (escopo negativo)

- **Não inclui** upload/otimização de imagem — a imagem é uma URL informada pelo
  admin.
- **Não inclui** setores com layout visual — o "mapa" do N1 é a capacidade
  numérica (setores A/B/C são rótulos, não geometria).
- **Não inclui** múltiplos administradores com permissões diferentes — papel
  `ADMIN` é único e total.
- **Não inclui** agendamento de publicação (publicar automático numa data
  futura) — publicar/despublicar é ação manual.
- **Não inclui** relatórios de venda/ocupação (N2/N3).

## 7. Custos adicionais

Nenhum custo externo identificado. A hospedagem da imagem é externa ao sistema
(URL informada), sem custo assumido pela plataforma.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Imagem do espetáculo | URL informada pelo admin (sem upload no N1). |
| Publicar / despublicar espetáculo | Ação manual do admin. Só espetáculo publicado com sessão futura à venda aparece na vitrine. |
| Editar sessão com ingressos vendidos | Permitido para campos que não quebram compras (ex.: local, horário com aviso). **Reduzir capacidade abaixo do já vendido é bloqueado.** |
| Excluir sessão com ingressos vendidos | **Proibido.** Só cancelar, disparando o fluxo de reembolso (RF07). |
| Excluir sessão sem vendas | Permitido (soft delete). |
| Tipos de ingresso | Por sessão: inteira (preço) e meia (50% do inteira, derivado). |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
