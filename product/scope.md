# Escopo do Produto

> **Responsável:** Product Owner · **Última revisão:** 2026-08-28 · **Status:** vigente

Ludens é uma **plataforma web de venda de ingressos** para um teatro comunitário.
Ela cobre a jornada do comprador — da busca do espetáculo à confirmação do
ingresso — e a operação administrativa de cadastro de espetáculos e sessões.

O escopo é dimensionado para fins didáticos (disciplina de Engenharia de
Software, Processo 18) e organizado em níveis de entrega.

## Níveis de entrega

| Nível | Escopo |
| --- | --- |
| **N1 (MVP)** | Cadastro de espetáculos e sessões; mapa simplificado de assentos (Setor A, B, C); compra de ingressos (inteira/meia-entrada). |
| **N2 / N3 (Evolução)** | Leitura de ingressos na portaria via dispositivo móvel; relatórios de ocupação de sala por sessão. |

## Dentro do escopo

**Catálogo**
- Buscar e filtrar espetáculos em cartaz (por data, por categoria/gênero)
- Visualizar detalhes de uma sessão: data, horário, local, tipos de ingresso e
  valores, disponibilidade em tempo real
- Cadastro, edição e encerramento de espetáculos e sessões pelo administrador

**Bilheteria / Reserva**
- Selecionar quantidade e tipo de ingresso (inteira/meia-entrada)
- Reservar ingressos temporariamente durante o checkout
- Controle atômico de disponibilidade: duas reservas concorrentes nunca excedem
  a capacidade da sessão
- Expiração automática da reserva não paga, devolvendo os ingressos à
  disponibilidade

**Pagamento**
- Pagamento dos ingressos reservados via gateway de pagamento
- Falha ou cancelamento do pagamento libera a reserva imediatamente
- O sucesso do pagamento gera um pedido (order) vinculado ao comprador

**Confirmação e histórico**
- E-mail de confirmação com o ingresso (identificador único / QR) após aprovação
  do pagamento
- Consulta ao histórico de compras e pedidos, com status
- Solicitação de cancelamento e reembolso conforme política vigente

**Conta**
- Cadastro e autenticação do comprador (CPF, e-mail, senha)
- Recuperação de senha por e-mail

## Fora do escopo

- **Bilheteria física presencial**: a operação de caixa no local permanece
  manual; a plataforma é a fonte de disponibilidade que a bilheteria consulta,
  não o sistema de ponto de venda dela
- **Validação presencial de meia-entrada**: a conferência do documento de
  estudante na entrada do evento é manual; o sistema apenas registra a intenção
  de compra de meia-entrada — ver [ADR 004](../architecture/design/004-escopo-da-regra-de-meia-entrada.md)
- **Gestão do gateway de pagamento**: a plataforma integra com um gateway
  externo; não implementa processamento de cartão nem antifraude próprios
- **Disponibilidade dos serviços externos**: indisponibilidade do gateway de
  pagamento ou do serviço de e-mail está fora do controle da plataforma

## Premissas

- Existe um gateway de pagamento disponível para integração (cartão e/ou Pix — a
  definir com o PO, ver [regras de negócio](../requirements/business-rules.md))
- Existe um serviço de envio de e-mail transacional para as confirmações
- O teatro adota a plataforma como fonte única de disponibilidade — a bilheteria
  física passa a registrar suas vendas nela
- O ambiente conteinerizado (Docker) é o padrão de execução local e de pipeline
- O escopo dos valores numéricos das regras de negócio (limite por CPF, prazos
  de reembolso, tempo de expiração) depende de aprovação do PO antes de virar
  critério de Definition of Ready
