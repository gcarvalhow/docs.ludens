# Glossário — Linguagem Ubíqua (DDD)

> **Responsável:** Engenheiro de Requisitos (R) · **Consulta:** Desenvolvedor Backend (C)
> **Última revisão:** 2026-08-28 · **Status:** vigente

Termos alinhados com o time de Backend para uso consistente no código (DDD) e na
documentação. Os identificadores em código usam o termo em **inglês** entre
parênteses (ver [guia de estilo](../engineering/code-style.md)).

| Termo (PT) | Termo em código (EN) | Definição |
| --- | --- | --- |
| **Espetáculo** | `Show` / `Production` | Produção teatral que pode ter uma ou mais sessões. |
| **Sessão** | `Session` | Apresentação específica de um espetáculo, com data, horário e local definidos. Tem capacidade e mapa de assentos (Setor A, B, C no MVP). |
| **Ingresso** | `Ticket` | Direito de acesso a uma sessão, de um tipo específico (inteira / meia-entrada). Emitido com identificador único validável na entrada. |
| **Tipo de ingresso** | `TicketType` | `inteira` (`FULL`) ou `meia` (`HALF`). Meia-entrada tem regra própria — ver [RN04](business-rules.md#rn04--meia-entrada). |
| **Reserva** | `Reservation` | Bloqueio temporário de um ou mais ingressos até a confirmação do pagamento. Expira conforme [RN03](business-rules.md#rn03--expiração-da-reserva). |
| **Pedido** | `Order` | Registro de uma compra confirmada, podendo conter um ou mais ingressos. Tem status: confirmado, cancelado, reembolsado. |
| **Comprador** | `Buyer` / `Customer` | Usuário cadastrado (CPF, e-mail, senha) que realiza a compra de ingressos. |
| **Disponibilidade** | `Availability` | Quantidade de ingressos de uma sessão que ainda pode ser vendida, em tempo real. Controlada de forma atômica ([RN05](business-rules.md#rn05--consistência-de-disponibilidade)). |
| **Gateway de pagamento** | `PaymentGateway` | Serviço externo que processa o pagamento dos ingressos reservados e o estorno em reembolsos. |
