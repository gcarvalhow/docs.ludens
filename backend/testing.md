# Estratégia de testes automatizados: Backend

> Versão operacional do [Acordo de Manutenibilidade §4](../team/maintainability.md#4-estratégia-de-testabilidade)
> e do gate de testes do [DoD](../team/quality.md#definition-of-done-dod-pronto-para-entrega).
> Ver também [`code-style.md`](code-style.md) para idioma/formatação e
> [`conventions.md`](conventions.md) para padrão de arquitetura.

Este documento existia só como referência dentro da skill de arquitetura
(`references/09-testing-and-quality.md`); este é o documento que ela cita.
Escrito junto com a primeira suíte real de testes do `api.ludens`
(`test/60-test-suite`).

## Diferente do backend anterior que inspirou este

O `api.ludens` foi arquiteturalmente inspirado num backend privado anterior
(.NET/DDD), mas **tem testes automatizados obrigatórios no pipeline desde o
início**; o anterior não tinha esse gate. A abordagem de teste em si também
diverge num ponto deliberado: o backend anterior mocka a camada de
persistência em todo teste de handler (`Moq` sobre a interface de projeção).
Aqui, a camada de usecase/repositório é testada contra um **Postgres real**,
não mockado. Ver por quê na Seção 2.

## 1. Camada de domínio: sem DB, sem HTTP

Alvo: os aggregates (`app/modules/*/domain/aggregates/`) e os métodos de
regra de negócio pura que não dependem de infraestrutura (ex.
`Session.available_count`, `Session.half_price_cents`).

* Instancia o aggregate diretamente (`Show.create(...)`, `User.register(...)`),
  chama o método sob teste, e afirma **estado final + eventos de domínio
  enfileirados** (`aggregate.dequeue_events()`).
* Cobre também os *guards* silenciosos (ex. `Show.publish()` chamado duas
  vezes seguidas não levanta um segundo `ShowPublished`); regra de negócio
  sutil que só aparece testando o caminho repetido, não só o caminho feliz.
* Não sobe nenhum container, não abre sessão de banco. Roda em milissegundos.

```python
def test_meia_entrada_nao_exige_documento():
    session = Session.create(show_id=..., starts_at=..., venue=..., capacity=10, full_price_cents=4999)
    assert session.half_price_cents == 2499  # RN04 — 50%, truncado ao centavo
```

Localização: `tests/unit/modules/<módulo>/domain/aggregates/test_*.py`.

## 2. Camada de usecase/repositório: Postgres real, sem mock de SQLAlchemy

Alvo: os usecases (`app/modules/*/application/usecases/`) e os repositórios
que eles usam.

**SQLAlchemy não é mockado.** Mockar a sessão/repositório escondia exatamente
os bugs mais caros de detectar num monólito com Postgres real por trás: SQL
mal formado, filtro `is_active` esquecido, paginação com off-by-one, FK
faltando. O teste sobe um Postgres descartável via
`docker/docker-compose.Staging.yml` (porta `5433`, banco `ludens_test`, separado
do Postgres de desenvolvimento) e exercita o usecase de ponta a ponta contra
ele.

* Fixtures em `tests/conftest.py`: `engine` (session-scoped, cria o schema
  uma vez via `Model.metadata.create_all`, sem depender do Alembic,
  proporcional ao estágio do projeto) e `session` (function-scoped, abre uma
  transação por teste e faz `rollback` no fim, isolamento sem recriar o
  schema a cada teste).
* Cada teste constrói o usecase direto com a `session` da fixture
  (`UserUseCase(session)`) e chama os métodos públicos normalmente.
* Quando o fluxo depende de um token opaco que só existe em texto puro no
  evento de domínio (ex. confirmação de troca de e-mail), o teste lê o evento
  gravado no outbox (`app.outbox.models.Event`) em vez de reimplementar o hash;
  é o mesmo dado que o handler de e-mail real consumiria.
* Marcados com `@pytest.mark.integration` (registrado em `pyproject.toml`);
  permite rodar só os testes de domínio sem Docker: `pytest -m "not integration"`.

Localização: `tests/integration/modules/<módulo>/application/test_*.py`.

## 3. O que não é testado por esta suíte (ainda)

* **Camada HTTP** (routers FastAPI): não há teste de endpoint por reflection
  (o `api.societiza`, referência arquitetural, tem um `EndpointConventionsTests.cs`
  cobrindo isso via reflection; não existe equivalente aqui ainda).
* **Concorrência** (RN05, duas reservas simultâneas na última poltrona): só
  se aplica quando o módulo de reserva/ingresso existir; não há ainda.
* **Provedores externos reais** (ACS, gateway de pagamento): a suíte
  automatizada não dispara e-mail/pagamento real; isso é coberto pelo roteiro
  de QA manual pré-entrega (Seção 4.1 do Acordo de Manutenibilidade) contra um
  Mailpit/sandbox local.

Nenhum desses é bloqueio para o DoD hoje; não há threshold numérico de
cobertura, o critério é qualitativo ("testes relevantes criados/atualizados e
passando na pipeline"). Ficam registrados aqui como próximos passos naturais
quando os módulos correspondentes existirem.

## 4. Rodando localmente

```bash
docker compose -f docker/docker-compose.Staging.yml up -d
pip install ".[dev]"
pytest -q                      # suíte completa
pytest -q -m "not integration" # só domínio, sem precisar do Postgres de teste
```

## 5. Na pipeline

`pipelines.ludens/_test.yaml` (workflow reutilizável, `workflow_call`) já
aceita `compose-file`; `api.ludens/.github/workflows/ci.yml` aponta pro
`docker/docker-compose.Staging.yml` deste repositório.
