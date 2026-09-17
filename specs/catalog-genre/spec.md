---
status: approved
domain: catalog
created_at: 2026-09-11
approved_at: 2026-09-11
---

# Gêneros do catálogo

## 1. Visão da feature

O administrador passa a manter uma lista própria de gêneros de espetáculo, em
vez de digitar um texto livre toda vez que cadastra uma peça nova. Cada gênero
cadastrado carrega um ícone que o representa visualmente — por exemplo, uma
máscara para "drama", uma risada para "comédia" — e essa mesma lista, com os
mesmos ícones, é o que a pessoa que visita o site usa para filtrar o que está
em cartaz. Gênero deixa de ser uma palavra qualquer digitada às pressas no
formulário do espetáculo e passa a ser uma categoria reconhecível, visual e
consistente em toda a plataforma.

## 2. Problema que resolve

Hoje o gênero de um espetáculo é texto livre: o admin digita o que quiser toda
vez que cadastra uma peça. Isso cria duas dores reais. Primeira, inconsistência
silenciosa — "Comédia", "comedia" e "Comédia " (com espaço) viram três
categorias diferentes aos olhos do sistema, e o filtro que o visitante usa para
achar o que quer (RF01) fica poluído com variações do mesmo gênero em vez de
ajudar a pessoa a decidir rápido. Segunda, falta de identidade visual — na área
de gestão, o admin vê uma lista de espetáculos onde o gênero é só uma palavra
solta no meio do texto, sem nenhuma pista visual rápida de "isso é uma comédia,
isso é um espetáculo infantil, isso é dança".

## 3. Para quem é

- **Beneficiário direto:** o administrador do teatro, que cadastra os
  espetáculos e hoje não tem controle sobre a consistência dos gêneros que ele
  mesmo digita.
- **Beneficiário indireto:** o visitante que usa o filtro por gênero (RF01) —
  uma lista de gêneros curada e visualmente reconhecível ajuda a encontrar o
  que interessa mais rápido do que uma lista de texto ambígua.

## 4. Como melhora a experiência atual

**Antes:** o admin digita o gênero à mão em cada espetáculo; pequenas
variações de grafia criam categorias fantasmas no filtro; a lista de gêneros
que o visitante vê na vitrine é imprevisível — aparece e desaparece conforme os
espetáculos em cartaz mudam, sem nenhuma identidade visual.

**Depois:** o admin escolhe o gênero de uma lista que ele mesmo cadastrou
antes; a mesma lista, com o mesmo ícone por gênero, aparece tanto na área de
gestão quanto no filtro da vitrine — reforçando o mesmo modelo mental dos dois
lados.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:** a gestão de espetáculos pelo admin (RF08) —
sem ela não existe onde aplicar o gênero.

**O que habilita/fortalece:** o filtro por gênero da busca de espetáculos
(RF01) — hoje esse filtro já existe, mas opera sobre texto livre; esta feature
o torna confiável.

**Posição:** complementar — refina a qualidade de um dado que já existe no
catálogo (o gênero do espetáculo), não faz parte do caminho crítico do
problema que o produto resolve (nunca vender o mesmo assento duas vezes,
ver `product/problem.md`).

**RF/RN cobertos:** fortalece RF01. Ajusta um detalhe do critério de aceite de
RF08 — "categoria" deixa de ser um texto digitado livremente e passa a ser
escolhida de uma lista cadastrada previamente pelo próprio admin.

## 6. O que não é (escopo negativo)

- **Não inclui** editar ou excluir um gênero depois de criado — nesta entrega
  o admin só cria. Corrigir um gênero cadastrado errado, ou remover um que não
  faz mais sentido, fica para uma evolução futura.
- **Não inclui** mais de um gênero por espetáculo (hierarquia de gêneros,
  subgêneros, múltiplas categorias na mesma peça) — cada espetáculo continua
  tendo exatamente um gênero, igual a hoje.
- **Não inclui** relatório de uso de gênero (quantos espetáculos usam cada
  categoria, quão popular é cada uma na busca) — é uma pergunta de
  relatório/ocupação, mesma linha de evolução N2/N3 já prevista no produto.
- **Não inclui** filtrar a busca por mais de um gênero ao mesmo tempo — o
  filtro continua sendo um gênero por vez, igual ao comportamento atual.
- **Não inclui** upload de ícone personalizado pelo admin — o ícone é
  escolhido de uma paleta fixa já existente na interface, nunca um arquivo
  enviado por quem cadastra o gênero.

## 7. Custos adicionais

Nenhum custo externo identificado — os ícones vêm de um conjunto que já faz
parte da própria interface da plataforma, sem serviço externo nem
armazenamento adicional.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Quem cadastra gênero | Só o administrador. |
| Editar ou excluir gênero | Não faz parte desta entrega — só criação (ver §6). |
| Gênero deixa de ser texto livre | Sim — o espetáculo passa a ter um gênero escolhido de uma lista cadastrada previamente pelo admin, não mais digitado à mão. |
| Origem do ícone | O admin escolhe o ícone de uma paleta fixa no momento de criar o gênero — não é atribuído automaticamente, porque gênero é nome livre e um mapeamento fixo não cobriria um gênero novo que o admin inventar. |
| Gêneros já em uso hoje | Migram automaticamente: os valores distintos já usados nos espetáculos cadastrados (sem diferenciar maiúscula/acento) viram gêneros na lista curada — nenhum espetáculo existente fica sem gênero reconhecido. |
| Duplicidade | O sistema recusa cadastrar um gênero cujo nome já exista (comparação sem diferenciar maiúscula/acento — "Comédia" bloqueia "comedia"). |
| Alcance do filtro público | Mantém o comportamento atual — a vitrine só lista, como opção de filtro, os gêneros que têm pelo menos um espetáculo disponível para compra no momento, não a lista completa cadastrada pelo admin. |
| Prioridade e momento | Entra na entrega atual, junto da gestão de espetáculos (RF08) — a janela está aberta porque o PR do frontend ainda está em revisão. |

## 9. Perguntas abertas

Nenhuma. Todas as decisões de produto estão fechadas.
