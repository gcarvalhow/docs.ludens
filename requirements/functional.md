# Requisitos Funcionais (RF01–RF09)

> **Responsável:** Engenheiro de Requisitos (R) · **Aprovação:** Product Owner (A) · **Consulta:** QA (C)
> **Última revisão:** 2026-08-28 · **Status:** vigente
> Ver [visão geral da ERS](overview.md) e [regras de negócio](business-rules.md).

## RF01 — Buscar e filtrar espetáculos

**História de usuário:** Como visitante, eu quero buscar e filtrar espetáculos em
cartaz para encontrar um evento do meu interesse.

**Critérios de aceitação:**

- A listagem exibe título, imagem, sinopse curta, datas de sessões futuras e
  faixa de preço.
- É possível filtrar por data e por categoria/gênero.
- Espetáculos sem sessões futuras não aparecem na busca padrão.

## RF02 — Visualizar detalhes da sessão

**História de usuário:** Como visitante, eu quero ver os detalhes de uma sessão
específica para decidir a compra.

**Critérios de aceitação:**

- Exibe data, horário, local, tipos de ingresso (inteira/meia-entrada) e valores.
- Exibe a quantidade de ingressos disponíveis em tempo real.
- Sessões esgotadas são sinalizadas como "esgotado" e impedem nova reserva.

**Dependências técnicas:** esquema de banco de dados (disponibilidade em tempo
real).

## RF03 — Selecionar e reservar ingressos

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

## RF04 — Efetuar pagamento

**História de usuário:** Como comprador, eu quero pagar os ingressos reservados
por meio de um gateway de pagamento para concluir minha compra.

**Critérios de aceitação:**

- Suporta ao menos um meio de pagamento eletrônico (cartão e/ou Pix — **a
  definir com o PO**).
- Falha ou cancelamento do pagamento libera a reserva imediatamente.
- O sucesso do pagamento gera um pedido (order) vinculado ao comprador.

**Dependências técnicas:** gateway de pagamento.

## RF05 — Confirmar compra e emitir ingresso

**História de usuário:** Como comprador, eu quero receber a confirmação da compra
e o ingresso por e-mail para apresentar na entrada do evento.

**Critérios de aceitação:**

- O e-mail de confirmação é enviado em até 5 minutos após a aprovação do
  pagamento.
- O ingresso traz um identificador único (código/QR) validável na entrada.
- Falha no envio do e-mail não invalida a compra; o ingresso permanece
  disponível na área do usuário.

**Dependências técnicas:** envio de e-mail de confirmação.

## RF06 — Consultar histórico de compras

**História de usuário:** Como comprador cadastrado, eu quero consultar meus
ingressos e pedidos anteriores para acompanhar minhas compras.

**Critérios de aceitação:**

- A lista mostra cada pedido com status (confirmado, cancelado, reembolsado).
- É possível reenviar o ingresso por e-mail a partir do histórico.

## RF07 — Solicitar cancelamento e reembolso

**História de usuário:** Como comprador, eu quero solicitar o cancelamento e
reembolso de um ingresso dentro da política vigente.

**Critérios de aceitação:**

- O sistema aplica a política de reembolso
  ([RN02](business-rules.md#rn02--política-de-reembolso)) automaticamente
  conforme o prazo até a sessão.
- Solicitações fora do prazo são bloqueadas com mensagem explicativa.
- Ingressos cancelados retornam à disponibilidade da sessão.

**Dependências técnicas:** gateway de pagamento (estorno).

## RF08 — Gerenciar espetáculos e sessões

**História de usuário:** Como Product Owner/administrador do teatro, eu quero
cadastrar, editar e encerrar espetáculos e sessões para manter o catálogo
atualizado.

**Critérios de aceitação:**

- CRUD completo de espetáculo: título, sinopse, imagem e categoria.
- CRUD de sessão vinculada a um espetáculo: data, horário, capacidade, tipos e
  valores de ingresso.
- Não é possível excluir uma sessão com ingressos já vendidos — apenas
  cancelá-la, disparando o fluxo de reembolso (RF07).

## RF09 — Cadastro e autenticação de usuário

**História de usuário:** Como visitante, eu quero me cadastrar e autenticar para
comprar ingressos e acompanhar meu histórico.

**Critérios de aceitação:**

- O cadastro exige CPF válido, e-mail e senha.
- Há recuperação de senha por e-mail.
- A senha é armazenada com hash; nenhum dado sensível é exposto em log.
