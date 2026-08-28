# ADR 002 — Stack: FastAPI + React + PostgreSQL + Docker

> **Status:** aceito
> **Data:** 2026-07-31 · **Responsável:** Desenvolvedor Backend / DevOps · **Decisor:** PO
> Formaliza a stack definida no [Acordo de Manutenibilidade §5.1](../../team/acordo-de-manutenibilidade.md).

## Contexto

O Processo 18 exige a construção de uma plataforma web com backend, frontend e
banco de dados, executável de forma reproduzível pela equipe e pela pipeline. A
equipe tem um Desenvolvedor Backend (Python) e um Desenvolvedor Frontend (React).

## Decisão

| Camada | Tecnologia | Motivo |
| --- | --- | --- |
| Backend | Python 3.12 + **FastAPI** | Tipagem via Pydantic, documentação automática (`/docs`), afinidade da equipe com Python; adequado a DDD |
| Frontend | **React** (Vite) | Afinidade da equipe; Vite dá dev server rápido e build simples |
| Banco de dados | **PostgreSQL** | Relacional, transacional — necessário para o controle atômico de disponibilidade ([RN05](../../requirements/business-rules.md#rn05--consistência-de-disponibilidade)) |
| Execução | **Docker / Docker Compose** | Ambiente reproduzível, igual em dev e na pipeline |

O **ambiente conteinerizado é o padrão** de execução. A aplicação deve construir
e subir via Docker sem erros — é um portão de merge (ver
[CI/CD](../../engineering/ci-cd.md)).

## Alternativas consideradas

- **Django** no backend — descartado: mais opinativo e pesado do que o
  necessário; FastAPI casa melhor com a separação de camadas do DDD.
- **SQLite** para desenvolvimento — descartado: diverge do Postgres em
  concorrência e tipos; o controle de disponibilidade precisa do comportamento
  transacional real.

## Consequências

- Um `docker-compose.yml` sobe backend + banco; o frontend entra no compose
  quando for scaffoldado.
- A pipeline roda os linters e testes de cada stack (Ruff + Pytest;
  ESLint + Prettier) e valida o build Docker.
- Versões fixadas: Python 3.12, Node 20 (ver configuração de cada repo de
  código).

## Ações decorrentes

- [x] `docker-compose.yml` com serviços `db` e `backend` (`api.ludens`)
- [ ] Adicionar serviço `frontend` ao compose quando o app React for scaffoldado
