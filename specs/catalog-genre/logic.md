---
status: draft
spec: catalog-genre
created_at: 2026-09-11
---

# Gêneros do catálogo — Lógica de Negócio

## 1. Fluxo por perfil

### Admin

Cadastra um gênero (nome + um ícone escolhido de uma paleta fixa) → o gênero
passa a existir e fica disponível para ser usado em qualquer espetáculo, seu
ou não. Não há edição nem exclusão nesta entrega — uma vez criado, o gênero
existe permanentemente (ver §6 da spec e casos de borda abaixo).

Ao cadastrar ou editar um espetáculo, em vez de digitar o gênero, o admin
escolhe um entre os gêneros já cadastrados. A lista que ele vê aqui é a lista
completa — inclui gêneros que nenhum espetáculo publicado usa ainda.

Ações proibidas e o que acontece:

- Cadastrar um gênero com nome igual a um já existente (ignorando maiúscula e
  acento) → recusado, com aviso de que o gênero já existe.
- Cadastrar um gênero sem escolher um ícone → recusado, o ícone é obrigatório.
- Cadastrar ou editar um espetáculo sem escolher um gênero → recusado, gênero
  continua obrigatório, só que agora por seleção em vez de texto.
- Editar ou remover um gênero já cadastrado → não é uma ação disponível nesta
  entrega, independente de o gênero estar em uso ou não.

### Comprador / Visitante

Não cadastra nem altera gênero — só usa a lista de gêneros como filtro na
busca de espetáculos (RF01), exatamente como já acontece hoje. A diferença que
o visitante percebe é indireta: os nomes de gênero no filtro deixam de ter
variações (grafias diferentes do mesmo gênero) e passam a vir acompanhados de
um ícone reconhecível.

## 2. Estados e transições

### Gênero

Um gênero não tem ciclo de vida — não existe estado "rascunho" nem
"inativo": no momento em que é criado, já existe e já pode ser usado por
qualquer espetáculo, para sempre (dentro do que esta entrega cobre).

O que muda de estado é a **visibilidade do gênero no filtro que o visitante
vê**:

- **Não aparece no filtro** — nenhum espetáculo publicado com sessão futura à
  venda usa esse gênero no momento (inclui o instante em que o gênero acaba de
  ser criado e ainda não foi usado em nenhum espetáculo publicado).
- **Aparece no filtro** — pelo menos um espetáculo publicado com sessão futura
  à venda usa esse gênero.

A transição entre os dois é automática, decorrente do estado dos espetáculos
que usam o gênero — o admin nunca ativa/desativa isso diretamente. Mesmo
comportamento que já existe hoje para o filtro de gênero (texto livre), agora
aplicado sobre a lista curada.

## 3. Regras de negócio

- Nome de gênero é sempre normalizado antes de virar a identidade do gênero
  (sem acento, sem diferenciar maiúscula/minúscula) → "Comédia", "comedia" e
  "COMÉDIA" são a mesma coisa para o sistema; a normalização é o que garante
  que nunca existam dois gêneros repetidos, mesmo sob duas criações
  simultâneas com o mesmo nome.
- Todo gênero precisa de um ícone escolhido no momento da criação → sem ícone,
  a criação é recusada.
- Só o admin cria gênero → qualquer tentativa sem essa permissão é recusada,
  mesmo padrão já aplicado hoje ao resto da gestão de espetáculos (RF08).
- Espetáculo sempre referencia exatamente um gênero, escolhido de um já
  cadastrado → nunca um espetáculo fica sem gênero, e nunca aceita um valor
  que não esteja na lista de gêneros cadastrados.
- O filtro público de gênero só mostra gêneros usados por pelo menos um
  espetáculo publicado com sessão futura à venda → gênero cadastrado sem uso
  não aparece nesse filtro (mesma regra de hoje, agora sobre a lista curada).
- Os gêneros em texto livre já usados pelos espetáculos existentes no momento
  da mudança viram gêneros cadastrados automaticamente, sem intervenção do
  admin, agrupando variações que só diferem em maiúscula/acento → nenhum
  espetáculo existente fica sem um gênero reconhecido depois da mudança.

## 4. Pontos de integração

```text
Frontend precisa saber:
  - Cadastrar/editar espetáculo passa a exigir escolher um gênero de uma
    lista pré-existente, não mais digitar — o formulário de espetáculo
    precisa da lista completa de gêneros antes de permitir o envio.
  - Existem duas listas de gênero diferentes, com propósitos diferentes: a
    lista completa (todo gênero cadastrado, usada no formulário de
    espetáculo) e a lista de filtro público (só gêneros com espetáculo
    disponível pra compra no momento, usada na busca). Não são a mesma coisa.
  - Cada gênero carrega um ícone definido na criação — deve ser exibido junto
    do nome do gênero em qualquer lugar que hoje só mostra o texto (tela de
    gestão, formulário de espetáculo, filtro da vitrine).
  - Criar gênero com nome repetido (ignorando maiúscula/acento) é recusado —
    a tela deve mostrar uma mensagem específica, não um erro genérico.
  - Não existe editar nem excluir gênero nesta versão — a tela de gestão só
    oferece "criar novo gênero".
  - Se ainda não existe nenhum gênero cadastrado, o formulário de espetáculo
    não tem o que oferecer pra escolha — ver caso de borda "Nenhum gênero
    cadastrado ainda".

Backend precisa garantir:
  - A migração dos gêneros já usados em espetáculos existentes acontece de
    uma vez, antes de qualquer uso da nova lista — nenhum espetáculo antigo
    fica temporariamente sem gênero reconhecido.
  - Duplicidade de nome (ignorando maiúscula/acento) é bloqueada de forma
    atômica, mesmo sob duas criações concorrentes com o mesmo nome.
  - A lista completa de gêneros (pro formulário de espetáculo) e a lista de
    filtro público (pra busca) são consultas diferentes, com regras
    diferentes — a primeira devolve tudo que existe, a segunda só o que tem
    espetáculo disponível.
```

## 5. Casos de borda

**Nenhum gênero cadastrado ainda.** Num ambiente novo (sem nenhum espetáculo
cadastrado ainda) ou logo após a migração não encontrar nenhum valor de gênero
nos espetáculos existentes, a lista de gêneros começa vazia. Cadastrar um
espetáculo exige escolher um gênero — então, nessa situação, o admin precisa
cadastrar ao menos um gênero antes de conseguir cadastrar o primeiro
espetáculo. `[fechada]` — é consequência esperada de gênero virar obrigatório
por seleção; o formulário de espetáculo deve orientar o admin nesse sentido em
vez de mostrar um formulário vazio sem explicação.

**Grafias diferentes do mesmo gênero migrando ao mesmo tempo.** Se hoje
existem espetáculos com "Comédia" e outros com "comedia" (mesma palavra,
grafias diferentes), a normalização (regra §3) já resolve isso sozinha: as
duas colapsam no mesmo gênero migrado, sem precisar escolher entre uma
grafia e outra. `[fechada]` — decorre direto da regra de normalização, não é
uma decisão à parte.

**Ícone repetido entre gêneros diferentes.** Nada nesta feature impede dois
gêneros diferentes ("Comédia" e "Comédia Musical", por exemplo) de usar o
mesmo ícone da paleta — só o nome precisa ser único. `[fechada]` — o ícone é
só identidade visual de apoio, não é o identificador do gênero.

**Espetáculo publicado é o único a usar um gênero, e é despublicado.** O
gênero continua existindo (não há exclusão), só some do filtro público — volta
a aparecer automaticamente se outro espetáculo publicado passar a usá-lo.
`[fechada]` — mesmo comportamento de hoje, aplicado sobre a lista curada
(ver §2).
