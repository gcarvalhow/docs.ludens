---
status: reviewed
spec: catalog-show-search
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# Busca e filtro de espetáculos — Lógica de Negócio

## 1. Fluxo por perfil

### Visitante

1. Abre a home. Vê os espetáculos com sessão futura à venda, ordenados pela
   próxima sessão.
2. Cada card: título, imagem, sinopse curta, próximas datas, faixa de preço.
3. Aplica filtro por data ("a partir de") e/ou por gênero. A lista atualiza.
4. Se nenhum espetáculo casa os filtros, vê um estado vazio explicativo ("Nenhum
   espetáculo em cartaz para esse filtro").
5. Clica num card → vai para o espetáculo / a próxima sessão (RF02).

### Comprador / Admin

Mesma vitrine. O admin edita o catálogo por outra área (RF08).

## 2. Estados e transições

Não há estado de máquina aqui — é leitura. O que muda a lista é: o admin
publicar/despublicar espetáculo, criar/cancelar sessão, o tempo passar (sessões
viram passado).

## 3. Regras de negócio

- Um espetáculo só aparece se tem ≥ 1 sessão com data futura e status "à venda".
- Sessões passadas não contam para "próximas datas" nem para a faixa de preço.
- A faixa de preço é [menor preço, maior preço] entre os tipos de ingresso das
  sessões futuras.
- Ordenação padrão: data da próxima sessão futura, ascendente.
- Filtro de data: mostra espetáculos com sessão futura **na data informada ou
  depois**.
- Filtro de gênero: casa a categoria do espetáculo.

## 4. Pontos de integração

```text
Frontend precisa saber:
  - O shape do card (título, imagem, sinopse curta, lista de próximas datas,
    priceMin, priceMax) e a lista de gêneros disponíveis para o filtro
  - Que a resposta já vem filtrada por "tem sessão futura" — o frontend não
    reimplementa esse critério
  - Os parâmetros de filtro aceitos (fromDate, genre) e de paginação (page, size)

Backend precisa garantir:
  - A query exclui espetáculos sem sessão futura à venda
  - priceMin/priceMax e "próximas datas" consideram só sessões futuras
  - Resposta ≤ 2 s p95 (RNF02) — índice por data de sessão
```

## 5. Casos de borda

**Espetáculo com todas as sessões futuras esgotadas.** Ainda aparece na lista (há
sessão futura à venda), mas o detalhe da sessão sinaliza "esgotado" (RF02). O
card pode indicar "poucas unidades" / "esgotado" se o backend expuser isso —
decisão: no N1 o card **não** mostra disponibilidade, só o detalhe. `[fechada]`

**Duas sessões no mesmo dia.** As duas datas aparecem na lista de "próximas
datas" do card. `[fechada]`

**Filtro de data no passado.** Tratado como "a partir de hoje" (não faz sentido
buscar sessão passada). `[fechada]`
