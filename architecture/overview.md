# Arquitetura — Visão Geral

> **Responsável:** Desenvolvedor Backend (R) · **Aprovação:** Engenheiro de Requisitos (A) · **Consulta:** DevOps (C)
> **Última revisão:** 2026-08-28 · **Status:** vigente

## 1. Mapa dos repositórios

| Repositório | Responsabilidade |
| --- | --- |
| [`gcarvalhow/api.ludens`](https://github.com/gcarvalhow/api.ludens) | API REST em FastAPI. Catálogo, reserva/emissão de ingressos, pagamento, contas, notificação. Monólito modular + DDD. |
| [`gcarvalhow/web.ludens`](https://github.com/gcarvalhow/web.ludens) | Aplicação React (Vite). Consome a API. |
| [`gcarvalhow/docs.ludens`](https://github.com/gcarvalhow/docs.ludens) | Este repositório — fonte de entrada da documentação. |

## 2. Stack

| Camada | Tecnologia |
| --- | --- |
| Frontend | React (Vite) |
| Backend | Python 3.12 + FastAPI |
| Banco de dados | PostgreSQL |
| Execução | Docker / Docker Compose |
| Arquitetura | Monólito modular + DDD |

O ambiente **conteinerizado é o padrão** de execução local e de pipeline. A
aplicação deve construir e subir via Docker sem erros. Ver
[CI/CD](../engineering/ci-cd.md).

Decisão registrada em [ADR 002](design/002-stack-fastapi-react-postgresql-docker.md).

## 3. Monólito modular + DDD

A API é um **único serviço deployável**, organizado internamente em **módulos
isolados**. Cada módulo encapsula seu próprio domínio — roteamento, lógica de
aplicação, acesso a dados, schemas — sem dependências cruzadas diretas entre
módulos. As regras de negócio vivem na **camada de domínio**, isoladas da API e
da infraestrutura.

Decisão e consequências em [ADR 001](design/001-monolito-modular-com-ddd.md).

### Boas práticas que sustentam a arquitetura

- Respeitar as fronteiras entre módulos e isolar as regras de domínio, evitando
  acoplamento direto entre módulos.
- As rotas apenas orquestram; as regras de negócio ficam no domínio.
- Ver [guia de estilo e código](../engineering/code-style.md).

## 4. Bounded contexts

Os contextos de domínio abaixo orientam a divisão dos módulos (ver
[ERS §1.3](../requirements/overview.md#13-contextos-de-domínio-ddd)):

| Contexto | Responsabilidade | Exemplos de regra |
| --- | --- | --- |
| **Catálogo** | Espetáculos e sessões | Sessão sem sessões futuras não aparece na busca (RF01) |
| **Bilheteria / Reserva** | Disponibilidade, reserva e emissão de ingressos | Meia-entrada exige documento ([RN04](../requirements/business-rules.md#rn04--meia-entrada)); reserva expira ([RN03](../requirements/business-rules.md#rn03--expiração-da-reserva)); disponibilidade atômica ([RN05](../requirements/business-rules.md#rn05--consistência-de-disponibilidade)) |
| **Pagamento** | Integração com gateway e processamento de pedidos | Falha no pagamento libera a reserva (RF04) |
| **Conta** | Cadastro, autenticação e histórico do comprador | Senha com hash; CPF válido (RF09) |
| **Notificação** | E-mails transacionais | Falha no e-mail não invalida a compra (RF05) |

## 5. Exemplo de domínio já implementado — emissão de ingresso

O módulo `tickets` do `api.ludens` já implementa a regra crítica da
meia-entrada na camada de domínio (`app/domain/tickets/models.py`):
`issue_ticket(...)` recusa a emissão de um ingresso `HALF` sem o número do
documento de estudante e calcula metade do preço quando o documento é
informado. A rota `POST /tickets` apenas orquestra e traduz a violação de regra
em `HTTP 422`. Ver a divergência com a ERS em
[ADR 004](design/004-escopo-da-regra-de-meia-entrada.md).

## 6. Diagramas

As fontes dos diagramas (mermaid / drawio) e os PNG exportados ficam em
[`assets/diagramas/`](../assets/diagramas/). *A fazer:* diagrama de contexto
(C4 nível 1) e diagrama de containers.
