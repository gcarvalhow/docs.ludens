# Requisitos Funcionais e Não Funcionais

> **Status:** vigente — RF01–RF09 e RNF01–RNF06 aprovados pelo PO em 2026-08-28 · **Última revisão:** 2026-08-28
>
> **Papéis (RACI)**
>
> - **Responsável (R):** Renato Colin Neto — Engenheiro de Requisitos
> - **Aprovação (A):** Gabriel Carvalho — Product Owner
> - **Consulta (C):** Adrian Cesar Gonçalves — QA

Este documento cobre os **requisitos funcionais** (RF01–RF09), os **requisitos
não funcionais** (RNF01–RNF06) e o **quadro consolidado de dependências técnicas
por requisito**. Ver [visão geral da ERS](overview.md) e
[regras de negócio](business-rules.md).

## Requisitos funcionais (RF01–RF09)

### RF01 — Buscar e filtrar espetáculos

**História de usuário:** Como visitante, eu quero buscar e filtrar espetáculos em
cartaz para encontrar um evento do meu interesse.

**Critérios de aceitação:**

- A listagem exibe título, imagem, sinopse curta, datas de sessões futuras e
  faixa de preço.
- É possível filtrar por data e por categoria/gênero.
- Espetáculos sem sessões futuras não aparecem na busca padrão.

### RF02 — Visualizar detalhes da sessão

**História de usuário:** Como visitante, eu quero ver os detalhes de uma sessão
específica para decidir a compra.

**Critérios de aceitação:**

- Exibe data, horário, local, tipos de ingresso (inteira/meia-entrada) e valores.
- Exibe a quantidade de ingressos disponíveis em tempo real.
- Sessões esgotadas são sinalizadas como "esgotado" e impedem nova reserva.

**Dependências técnicas:** esquema de banco de dados (disponibilidade em tempo
real).

### RF03 — Selecionar e reservar ingressos

**História de usuário:** Como comprador, eu quero selecionar a quantidade e o
tipo de ingresso e reservá-los temporariamente para concluir o pagamento sem
risco de perdê-los.

**Critérios de aceitação:**

- O sistema valida o limite de ingressos por CPF ([RN01](business-rules.md#rn01--limite-de-ingressos-por-cpf))
  antes de confirmar a reserva.
- A reserva expira conforme [RN03](business-rules.md#rn03--expiração-da-reserva);
  ao expirar, os ingressos retornam à disponibilidade automaticamente.
- Duas reservas simultâneas não podem exceder a capacidade da sessão (controle
  de concorrência — ver [RN05](business-rules.md#rn05--consistência-de-disponibilidade)).

**Dependências técnicas:** esquema de banco de dados; controle de disponibilidade
de assentos.

### RF04 — Efetuar pagamento

**História de usuário:** Como comprador, eu quero pagar os ingressos reservados
por meio de um gateway de pagamento para concluir minha compra.

**Critérios de aceitação:**

- O pagamento é feito por **Pix**, através do gateway **AbacatePay**.
- Falha ou cancelamento do pagamento libera a reserva imediatamente.
- O sucesso do pagamento gera um pedido (order) vinculado ao comprador.

**Dependências técnicas:** integração com o gateway AbacatePay (Pix).

### RF05 — Confirmar compra e emitir ingresso

**História de usuário:** Como comprador, eu quero receber a confirmação da compra
e o ingresso por e-mail para apresentar na entrada do evento.

**Critérios de aceitação:**

- O e-mail de confirmação é enviado em até 5 minutos após a aprovação do
  pagamento.
- O ingresso traz um identificador único (código/QR) validável na entrada.
- Falha no envio do e-mail não invalida a compra; o ingresso permanece
  disponível na área do usuário.

**Dependências técnicas:** envio de e-mail de confirmação.

### RF06 — Consultar histórico de compras

**História de usuário:** Como comprador cadastrado, eu quero consultar meus
ingressos e pedidos anteriores para acompanhar minhas compras.

**Critérios de aceitação:**

- A lista mostra cada pedido com status (confirmado, cancelado, reembolsado).
- É possível reenviar o ingresso por e-mail a partir do histórico.

### RF07 — Solicitar cancelamento e reembolso

**História de usuário:** Como comprador, eu quero solicitar o cancelamento e
reembolso de um ingresso dentro da política vigente.

**Critérios de aceitação:**

- O sistema aplica a política de reembolso
  ([RN02](business-rules.md#rn02--política-de-reembolso)) automaticamente
  conforme o prazo até a sessão.
- Solicitações fora do prazo são bloqueadas com mensagem explicativa.
- Ingressos cancelados retornam à disponibilidade da sessão.

**Dependências técnicas:** gateway de pagamento (estorno).

### RF08 — Gerenciar espetáculos e sessões

**História de usuário:** Como Product Owner/administrador do teatro, eu quero
cadastrar, editar e encerrar espetáculos e sessões para manter o catálogo
atualizado.

**Critérios de aceitação:**

- CRUD completo de espetáculo: título, sinopse, imagem e categoria.
- CRUD de sessão vinculada a um espetáculo: data, horário, capacidade, tipos e
  valores de ingresso.
- Não é possível excluir uma sessão com ingressos já vendidos — apenas
  cancelá-la, disparando o fluxo de reembolso (RF07).

### RF09 — Cadastro e autenticação de usuário

**História de usuário:** Como visitante, eu quero me cadastrar e autenticar para
comprar ingressos e acompanhar meu histórico.

**Critérios de aceitação:**

- O cadastro exige CPF válido, e-mail e senha.
- Há recuperação de senha por e-mail.
- A senha é armazenada com hash; nenhum dado sensível é exposto em log.

## Requisitos não funcionais

Seis requisitos não funcionais (RNF01–RNF06), **todos aprovados pelo PO em
2026-08-28**. As metas numéricas de RNF02–RNF04 (percentis de tempo de resposta,
disponibilidade mensal, nível WCAG e limite de passos) valem como critério de
[Definition of Ready](../team/quality.md) a partir dessa data. O registro
da decisão está em
[visão geral da ERS § 6](overview.md#6-decisões-do-product-owner).

### RNF01 — Segurança e proteção de dados

- Senhas armazenadas apenas como *hash* com *salt* (algoritmo forte, ex.: bcrypt
  ou Argon2) — nunca em texto puro nem de forma reversível.
- Nenhum dado de pagamento (dados bancários, identificadores do pagador) é
  persistido pela plataforma: o pagamento é delegado ao gateway AbacatePay
  (RF04, [escopo do produto](../product/scope.md)).
- CPF, e-mail e histórico de compras são dados pessoais — acesso restrito ao
  próprio comprador autenticado e ao administrador; nunca expostos em logs, URLs
  ou mensagens de erro (reforça RF09).
- Segredos (chaves do gateway, credenciais de banco e de SMTP) ficam em
  variáveis de ambiente, fora do controle de versão
  ([guia de estilo](../backend/code-style.md)).
- Todo o tráfego entre cliente e servidor sobre HTTPS.

**Métrica / verificação:** teste automatizado garantindo o *hash* de senha e a
recusa de emissão ou log com dado sensível; item de segurança no roteiro de QA
antes de cada entrega.

**Status:** aprovada pelo PO em 2026-08-28 — consolida compromissos já firmados
no Acordo de Manutenibilidade e nos critérios de RF09.

### RNF02 — Desempenho

- A consulta de disponibilidade de uma sessão (RF02) responde em até **1 s** no
  percentil 95, sob carga de navegação normal.
- A busca e listagem de espetáculos (RF01) devolve resultado em até **2 s** (p95).
- A confirmação de reserva (RF03) — com verificação de limite por CPF e controle
  de concorrência — conclui em até **2 s** (p95).
- O e-mail de confirmação (RF05) é enviado em até **5 min** após a aprovação do
  pagamento (já em RF05).

**Métrica / verificação:** medição em ambiente de homologação com volume de dados
representativo; o roteiro de QA antes de cada entrega inclui a medição do tempo
de resposta do fluxo principal.

**Status:** aprovada pelo PO em 2026-08-28.

### RNF03 — Disponibilidade e confiabilidade

- Disponibilidade-alvo do serviço: **99% ao mês**, excluída janela de manutenção
  comunicada — meta dimensionada ao contexto didático.
- Degradação graciosa: a indisponibilidade do gateway de pagamento ou do serviço
  de e-mail não derruba o catálogo nem a navegação; a compra falha de forma
  controlada e libera a reserva (RF04) e a falha de e-mail não invalida a compra
  (RF05).
- Recuperação de estado: após reinício da aplicação, reservas expiradas
  continuam sendo devolvidas à disponibilidade (RN03) e nenhuma compra paga é
  perdida — a fonte de verdade é o banco, não a memória do processo.
- Consistência sob concorrência conforme
  [RN05](business-rules.md#rn05--consistência-de-disponibilidade) (regra
  vigente).

**Métrica / verificação:** teste de resiliência no roteiro de QA (derrubar
gateway/e-mail e confirmar catálogo no ar e reserva liberada) e teste de reinício
da aplicação com reservas em aberto.

**Status:** aprovada pelo PO em 2026-08-28. O comportamento de degradação e
recuperação já é exigido pelos RF/RN citados.

### RNF04 — Usabilidade e acessibilidade

- O caminho feliz da compra (descoberta → confirmação) se conclui em no máximo
  **5 passos**.
- Mensagens de erro específicas e acionáveis (ex.: limite por CPF atingido,
  reserva expirada, reembolso fora do prazo — RF03, RF07), sem expor detalhe
  técnico.
- Interface responsiva, utilizável em tela de celular — pré-requisito para a
  leitura de ingressos na portaria via dispositivo móvel (nível N2 do
  [escopo do produto](../product/scope.md)).
- Acessibilidade: navegação por teclado e contraste seguindo **WCAG 2.1 nível
  AA** como referência.
- Idioma da interface: português (pt-BR).

**Métrica / verificação:** teste de usabilidade com o roteiro do fluxo principal
antes de cada entrega; verificação de contraste e navegação por teclado nas telas
de checkout.

**Status:** aprovada pelo PO em 2026-08-28.

### RNF05 — Manutenibilidade

- Testes automatizados cobrem as regras de negócio da camada de domínio do
  backend ([estratégia de testes](../backend/testing.md)).
- Padrão de código, responsabilidade única com funções de até ~30 linhas,
  proibição de `except`/`catch` vazio e lint obrigatório na pipeline
  ([guia de estilo](../backend/code-style.md)).
- Monólito modular com fronteiras explícitas entre módulos de domínio (DDD);
  comunicação entre módulos apenas por contrato público.
- Cerca de **15% do esforço de cada ciclo** reservado para débito técnico
  ([gestão de débito técnico](../team/tech-debt.md)).

**Métrica / verificação:** pipeline verde (lint + testes) como portão de merge; o
[Definition of Done](../team/quality.md) exige testes relevantes passando.

**Status:** aprovada pelo PO em 2026-08-28 — consolida os compromissos do Acordo
de Manutenibilidade.

### RNF06 — Portabilidade e operação

- A aplicação sobe via **Docker** sem passos manuais além da definição das
  variáveis de ambiente; o ambiente conteinerizado é o padrão local e de
  pipeline ([escopo do produto](../product/scope.md),
  [CI/CD](../backend/testing.md)).
- Dependências externas (gateway de pagamento, SMTP) são acessadas por
  configuração, permitindo trocar de provedor sem alterar o código de domínio.
- Banco de dados PostgreSQL, com migrações versionadas no repositório do backend.

**Métrica / verificação:** o job de build da pipeline levanta a aplicação em
contêiner e executa o roteiro mínimo; falha no build Docker bloqueia o merge.

**Status:** aprovada pelo PO em 2026-08-28 — deriva de premissa do escopo e do
CI/CD vigente.

## Dependências técnicas por requisito

Quadro consolidado a partir das dependências listadas em cada RF, para uso na
priorização do backlog junto ao Backend e ao DevOps.

| Requisito(s) | Dependência técnica | Viabilização |
| --- | --- | --- |
| RF01, RF02, RF08 | Modelo de dados de catálogo (espetáculo, sessão, capacidade, valores) no PostgreSQL | Backend |
| RF02, RF03, RF06 | Consulta de disponibilidade em tempo real (leitura consistente do estoque da sessão) | Backend |
| RF03 | Controle de concorrência atômico do estoque de ingressos ([RN05](business-rules.md#rn05--consistência-de-disponibilidade)) e expiração agendada da reserva ([RN03](business-rules.md#rn03--expiração-da-reserva)) | Backend |
| RF03 | Validação do limite de ingressos por CPF ([RN01](business-rules.md#rn01--limite-de-ingressos-por-cpf)) | Backend |
| RF04, RF07 | Integração com o gateway AbacatePay — cobrança e estorno via Pix | Backend + DevOps (secret da AbacatePay) |
| RF05, RF06, RF09 | Serviço de e-mail transacional (confirmação, reenvio de ingresso, recuperação de senha) | Backend + DevOps (credenciais SMTP) |
| RF05 | Geração de identificador único / QR do ingresso | Backend |
| RF09 | Autenticação, *hash* de senha e sessão do usuário | Backend |
| RF08, RF09 | Controle de acesso: perfil administrador × comprador | Backend |
| Todos | Ambiente Docker + PostgreSQL provisionado e pipeline com secrets configurados | DevOps |
