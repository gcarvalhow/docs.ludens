# Visão Geral do Produto

## Contexto e problema

Um grupo de teatro comunitário e independente produz peças e apresenta sessões
para o público.
Hoje a venda de ingressos acontece em dois canais desconectados: a bilheteria
física do teatro e vendas informais pela internet.
Não existe um sistema que seja a fonte única da disponibilidade de assentos de
cada sessão.

Como os dois canais não compartilham o estado de disponibilidade, **o mesmo
assento pode ser vendido duas vezes**: uma na bilheteria física e outra pela
internet, para a mesma sessão.
O resultado é overbooking, público sem lugar na porta do evento, retrabalho
manual de conferência e perda de confiança na venda online.

Sem uma plataforma dedicada, o teatro também não tem:

* **Disponibilidade confiável em tempo real**: não há como saber, num dado
  instante, quantos ingressos de uma sessão ainda podem ser vendidos.
* **Reserva com garantia**: o comprador não consegue segurar os ingressos pelo
  tempo necessário para concluir o pagamento sem risco de perdê-los.
* **Registro estruturado das vendas**: pedidos, cancelamentos e reembolsos
  ficam espalhados e não são consultáveis de forma consistente.
* **Base para relatórios de ocupação**: medir a taxa de ocupação de sala por
  sessão depende de contagem manual.

## Critério de sucesso

* **Fonte única de disponibilidade**: a plataforma é o único lugar que define
  quantos ingressos de uma sessão estão disponíveis; a bilheteria física passa a
  operar sobre ela.
* **Sem venda em duplicidade**: duas compras concorrentes nunca conseguem,
  juntas, exceder a capacidade da sessão, mesmo sob acesso simultâneo.
* **Compra completa pela internet**: um comprador encontra o espetáculo,
  seleciona a sessão e o tipo de ingresso (inteira/meia), reserva, paga e recebe
  a confirmação sem intervenção manual do teatro.
* **Reserva temporária confiável**: os ingressos ficam bloqueados durante o
  checkout e voltam automaticamente à disponibilidade se o pagamento não for
  concluído no prazo.
* **Rastreabilidade**: todo pedido tem status consultável (confirmado, cancelado,
  reembolsado) e um ingresso com identificador único validável na entrada.
* **Ocupação mensurável**: a taxa de ocupação de sala por sessão pode ser
  extraída da plataforma, sem contagem manual (evolução N2/N3).

## O que é o Ludens

Ludens é uma **plataforma web de venda de ingressos** para um teatro comunitário.
Ela cobre a jornada do comprador, da busca do espetáculo à confirmação do
ingresso, e a operação administrativa de cadastro de espetáculos e sessões.

O escopo é dimensionado para fins didáticos (disciplina de Manutenção e Melhoria
de Software, 6º semestre de Engenharia de Software, Processo 18).

## Dentro do escopo

**Catálogo**

* Buscar e filtrar espetáculos em cartaz (por data, por categoria/gênero).
* Visualizar detalhes de uma sessão: data, horário, local, tipos de ingresso e
  valores, disponibilidade em tempo real.
* Cadastro, edição e encerramento de espetáculos e sessões pelo administrador.

**Bilheteria / Reserva**

* Selecionar quantidade e tipo de ingresso (inteira/meia-entrada).
* Reservar ingressos temporariamente durante o checkout.
* Controle atômico de disponibilidade: duas reservas concorrentes nunca excedem
  a capacidade da sessão.
* Expiração automática da reserva não paga, devolvendo os ingressos à
  disponibilidade.

**Pagamento**

* Pagamento dos ingressos reservados por Pix, via gateway AbacatePay.
* Falha ou cancelamento do pagamento libera a reserva imediatamente.
* O sucesso do pagamento gera um pedido (order) vinculado ao comprador.

**Confirmação e histórico**

* E-mail de confirmação com o ingresso (identificador único / QR) após aprovação
  do pagamento.
* Consulta ao histórico de compras e pedidos, com status.
* Solicitação de cancelamento e reembolso conforme política vigente.

**Conta**

* Cadastro e autenticação do comprador (CPF, e-mail, senha).
* Recuperação de senha por e-mail.

## Fora do escopo

* **Bilheteria física presencial**: a operação de caixa no local permanece
  manual; a plataforma é a fonte de disponibilidade que a bilheteria consulta,
  não o sistema de ponto de venda dela.
* **Validação presencial de meia-entrada**: a conferência do documento de
  estudante na entrada do evento é manual; o sistema apenas registra a intenção
  de compra de meia-entrada. Ver [RN04](#rn04-meia-entrada) abaixo.
* **Gestão do gateway de pagamento**: a plataforma integra com a AbacatePay; não
  implementa processamento de pagamento nem antifraude próprios.
* **Disponibilidade dos serviços externos**: indisponibilidade do gateway de
  pagamento ou do serviço de e-mail está fora do controle da plataforma.

## Regras de negócio (RN01–RN05)

Formalizam os exemplos citados no
[Acordo de Manutenibilidade](../team/maintainability.md) como condição de
[Definition of Ready](../team/quality.md). Todos os valores numéricos foram
aprovados pelo Product Owner em 2026-08-28.

* RN01, limite de ingressos por CPF: aprovada.
* RN02, política de reembolso: aprovada.
* RN03, expiração da reserva: aprovada.
* RN04, meia-entrada: aprovada, vale o texto da ERS (ver [RN04](#rn04-meia-entrada)).
* RN05, consistência de disponibilidade: vigente.

### RN01: Limite de ingressos por CPF

Cada CPF pode adquirir no máximo **6 ingressos por sessão**. Tentativas de
exceder o limite são bloqueadas na reserva (RF03).

**Status:** aprovada pelo PO em 2026-08-28 (limite de 6 ingressos por sessão).

### RN02: Política de reembolso

* Cancelamento solicitado **até 48h** antes da sessão: reembolso **integral**.
* Entre **48h e 24h** antes: reembolso de **50%**.
* **Menos de 24h** antes: **sem reembolso**.

**Status:** aprovada pelo PO em 2026-08-28 (prazos de 48h/24h e percentuais de
100%/50%/0%).

### RN03: Expiração da reserva

Uma reserva não paga expira em **15 minutos**. Ao expirar, os ingressos retornam
automaticamente à disponibilidade da sessão.

**Status:** aprovada pelo PO em 2026-08-28 (expiração em 15 minutos).

### RN04: Meia-entrada

O sistema apenas **registra a intenção** de compra de meia-entrada no checkout.
A validação do documento comprobatório de estudante é feita **presencialmente**
na entrada do evento e está **fora do escopo do sistema**: a emissão do ingresso
não exige o número do documento.

**Decisão do PO (2026-08-28).** A ERS original chegou a descrever a
meia-entrada exigindo o número do documento de estudante. O PO decidiu **alinhar
à ERS acima**: a emissão do ingresso **não** exige o documento. Como o backend
ainda não tem código, isso não é débito técnico, é **critério de aceite** da
funcionalidade `booking-ticket-issuance`: a emissão de meia-entrada não pode
exigir o número do documento.

**Status:** aprovada pelo PO em 2026-08-28. Ajuste no backend pendente.

### RN05: Consistência de disponibilidade

O controle de disponibilidade de ingressos deve ser **atômico**: duas reservas
concorrentes nunca podem, juntas, exceder a capacidade da sessão, mesmo sob
acesso simultâneo.

**Status:** vigente. É a regra que sustenta o critério de sucesso "sem venda em
duplicidade" (ver [Critério de sucesso](#critério-de-sucesso) acima).

## Premissas

* A conta na AbacatePay está disponível para integração (pagamento por Pix).
* Existe um serviço de envio de e-mail transacional para as confirmações.
* O teatro adota a plataforma como fonte única de disponibilidade: a bilheteria
  física passa a registrar suas vendas nela.
* O ambiente conteinerizado (Docker) é o padrão de execução local e de pipeline.
* Os valores numéricos das regras de negócio (limite por CPF, prazos de
  reembolso, tempo de expiração) foram aprovados pelo PO em 2026-08-28 e valem
  como critério de Definition of Ready. Ver
  [regras de negócio](#regras-de-negócio-rn01rn05) acima.
