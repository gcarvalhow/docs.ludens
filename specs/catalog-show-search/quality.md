---
status: draft
spec: catalog-show-search
surface: quality
created_at: 2026-09-10
updated_at: 2026-09-10
---

# Busca e filtro de espetáculos — Quality

**Resumo:** cobre a leitura pública de `catalog` (RF01) — filtro "só
publicado com sessão futura à venda", faixa de preço e datas derivadas só de
sessões futuras, ordenação, paginação e a lista de gêneros. Sem aggregate
novo (reusa `Show`/`Session` de `catalog-admin-management`), então quase todo
o comportamento relevante é do **repositório** (query), não do domínio — os
únicos testes sem banco são de uma função pura de formatação.
**RF:** RF01 · **RN:** — · **Módulo backend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-show-search/integration.md`

---

## 1. Definition of Ready — checagem

Contra `docs.ludens/team/quality.md`.

| Item DoR | Situação |
| --- | --- |
| História no formato "Como [papel], eu quero [func.] para [benefício]" | OK — RF01 + spec.md §1/§3 |
| Critérios de aceite objetivos e verificáveis | OK — `integration.md` + `logic.md` §2/§3 |
| Regras de negócio e exceções especificadas | OK — `logic.md` §3, §5; sem RN numerada nova |
| Dependências técnicas mapeadas | Parcial — ver bullets abaixo |
| Layout/protótipo aprovado | N/A — sem protótipo formal; UI conforme `frontend.md` |

Detalhe "Dependências técnicas" (parcial):

- `catalog-admin-management` (aggregates `Show`/`Session`, migration, índice
  `ix_sessions_starts_at_status`) ainda não mergeado — ver a nota de
  consistência no topo de `backend.md`/`frontend.md` desta spec.
- **Não existe fixture de banco de teste no repo** (`tests/conftest.py` não
  existe) e o job `test` do CI (`.github/workflows/ci.yml`) declara
  `DATABASE_URL` mas **não sobe um serviço Postgres** — hoje nenhum teste
  toca banco de verdade. Esta é a primeira fatia cujos testes de maior valor
  (a query de busca) só fazem sentido contra Postgres real
  (`docs.ludens/backend/testing.md`: "Casos de uso e repositórios usam banco
  de teste... sem mockar SQLAlchemy"). Ver §5.

Conclusão: **pronta para desenvolver** com a ressalva de infraestrutura de
teste acima — não é uma pendência de produto, é uma peça de CI que falta ser
criada (por esta fatia ou por quem chegar primeiro).

---

## 2. Casos de teste de domínio (pytest, sem banco)

Sem `Show`/`Session` novos, o único comportamento isolável sem banco é a
truncagem da sinopse curta (`show_search_usecase.py`).

| Caso | Cenário | RF |
| --- | --- | --- |
| `test_synopsis_short_mantem_texto_curto` | sinopse ≤ 160 chars → devolvida sem alteração | RF01 |
| `test_synopsis_short_trunca_e_adiciona_reticencias` | sinopse > 160 chars → corta em 160 + `…` | RF01 |

```python
# tests/modules/catalog/test_show_search_usecase.py  — novo
"""Função pura de show_search_usecase — sem banco, sem HTTP."""

from app.modules.catalog.application.usecases.show_search_usecase import _synopsis_short

def test_synopsis_short_mantem_texto_curto():
    assert _synopsis_short("Uma peça curta.") == "Uma peça curta."

def test_synopsis_short_trunca_e_adiciona_reticencias():
    curta = _synopsis_short("a" * 200)

    assert curta.endswith("…")
    assert len(curta) == 161  # 160 caracteres + reticências
```

---

## 3. Casos de teste de repositório (precisam de Postgres real)

Sem mock de SQLAlchemy, por convenção do projeto — seed via os próprios
aggregates/repositórios, consulta via `ShowRepository`.

| Caso | Cenário (estado → ação → asserção) | RF |
| --- | --- | --- |
| `test_so_espetaculo_publicado_com_sessao_futura_aparece` | 1 show publicado + 1 rascunho, ambos com sessão futura → só o publicado no resultado | RF01 |
| `test_espetaculo_sem_sessao_futura_nao_aparece` | show publicado só com sessão passada → `total == 0` | RF01 |
| `test_faixa_de_preco_considera_so_sessao_futura` | sessões passada (barata) + futuras → `price_min`/`price_max` ignoram a passada | RF01 |
| `test_filtro_por_genero` | dois shows, gêneros diferentes, filtro por um → só esse aparece | RF01 |
| `test_filtro_data_no_passado_vira_hoje` | `from_date` 30 dias atrás → sessão de amanhã ainda aparece (o filtro não reabre o passado) | RF01 |
| `test_ordenacao_por_proxima_sessao_ascendente` | show com sessão daqui 7 dias + show com sessão amanhã → ordem: amanhã primeiro | RF01 |
| `test_paginacao_retorna_total_correto` | 3 shows visíveis, `size=2` → `total == 3`, `len(rows) == 2` | RF01 |
| `test_list_genres_so_com_espetaculo_visivel` | show publicado (drama) + rascunho (terror) → `list_genres_in_catalog() == ["drama"]` | RF01 |

### `tests/conftest.py`

**Compartilhado** — se outra fatia (`catalog-admin-management` ou
`catalog-session-detail`) já criou este arquivo quando esta for implementada,
reusar em vez de recriar; só ajustar se faltar algo.

```python
"""Banco de teste real (Postgres em contêiner), per docs.ludens/backend/testing.md.

Usa as mesmas credenciais de docker/docker-compose.Development.yml. Em CI,
falta subir um serviço Postgres no job `test` de .github/workflows/ci.yml —
ver quality.md de catalog-show-search, seção "Riscos".
"""
from collections.abc import AsyncGenerator

import pytest_asyncio
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.config import settings
from app.core.domain.model import Model

_engine = create_async_engine(settings.database_url)
_session_factory = async_sessionmaker(_engine, expire_on_commit=False)

@pytest_asyncio.fixture
async def db_session() -> AsyncGenerator[AsyncSession, None]:
    # `Model.metadata` só conhece as tabelas dos aggregates importados até
    # aqui — os testes que usam esta fixture precisam importar Show/Session
    # (diretamente ou via repositories) antes deste ponto.
    async with _engine.begin() as conn:
        await conn.run_sync(Model.metadata.create_all)

    async with _session_factory() as session, session.begin():
        yield session

    async with _engine.begin() as conn:
        await conn.run_sync(Model.metadata.drop_all)
```

### `tests/modules/catalog/test_show_search_repository.py`

```python
"""ShowRepository.search_with_upcoming / list_genres_in_catalog — precisa de
Postgres real (sem mock de SQLAlchemy, per docs.ludens/backend/testing.md).
"""

from datetime import date, datetime, timedelta, timezone

from app.modules.catalog.domain.aggregates.session import Session
from app.modules.catalog.domain.aggregates.show import Show
from app.modules.catalog.infrastructure.repositories import SessionRepository, ShowRepository

NOW = datetime.now(timezone.utc)
TOMORROW = NOW + timedelta(days=1)
NEXT_WEEK = NOW + timedelta(days=7)
YESTERDAY = NOW - timedelta(days=1)

async def _seed_show(
    db_session, *, published: bool, sessions: list[tuple[datetime, int]], genre: str = "drama"
) -> Show:
    show = Show.create(title="Peça", synopsis="Sinopse.", image_url="https://x.test/i.jpg", genre=genre)
    if published:
        show.publish()
    await ShowRepository(db_session).save(show)

    for starts_at, price_cents in sessions:
        session = Session.create(
            show_id=show.id,
            starts_at=starts_at,
            venue="Sala 1",
            capacity=50,
            full_price_cents=price_cents,
            now=NOW,
        )
        await SessionRepository(db_session).save(session)

    await db_session.flush()
    return show

async def test_so_espetaculo_publicado_com_sessao_futura_aparece(db_session):
    await _seed_show(db_session, published=True, sessions=[(TOMORROW, 10_000)])
    await _seed_show(db_session, published=False, sessions=[(TOMORROW, 10_000)])

    rows, total = await ShowRepository(db_session).search_with_upcoming(
        from_date=date.today(), genre=None, page=1, size=10
    )

    assert total == 1
    assert len(rows) == 1

async def test_espetaculo_sem_sessao_futura_nao_aparece(db_session):
    await _seed_show(db_session, published=True, sessions=[(YESTERDAY, 10_000)])

    rows, total = await ShowRepository(db_session).search_with_upcoming(
        from_date=date.today(), genre=None, page=1, size=10
    )

    assert total == 0
    assert rows == []

async def test_faixa_de_preco_considera_so_sessao_futura(db_session):
    await _seed_show(
        db_session,
        published=True,
        sessions=[(YESTERDAY, 1_000), (TOMORROW, 8_000), (NEXT_WEEK, 12_000)],
    )

    rows, _ = await ShowRepository(db_session).search_with_upcoming(
        from_date=date.today(), genre=None, page=1, size=10
    )

    assert rows[0].price_min_cents == 8_000
    assert rows[0].price_max_cents == 12_000

async def test_filtro_por_genero(db_session):
    await _seed_show(db_session, published=True, sessions=[(TOMORROW, 5_000)], genre="drama")
    await _seed_show(db_session, published=True, sessions=[(TOMORROW, 5_000)], genre="comédia")

    rows, total = await ShowRepository(db_session).search_with_upcoming(
        from_date=date.today(), genre="comédia", page=1, size=10
    )

    assert total == 1
    assert rows[0].genre == "comédia"

async def test_filtro_data_no_passado_vira_hoje(db_session):
    await _seed_show(db_session, published=True, sessions=[(TOMORROW, 5_000)])

    rows, total = await ShowRepository(db_session).search_with_upcoming(
        from_date=date.today() - timedelta(days=30), genre=None, page=1, size=10
    )

    assert total == 1  # a sessão de amanhã aparece; o passado não "reabre" nada

async def test_ordenacao_por_proxima_sessao_ascendente(db_session):
    later = await _seed_show(db_session, published=True, sessions=[(NEXT_WEEK, 5_000)])
    sooner = await _seed_show(db_session, published=True, sessions=[(TOMORROW, 5_000)])

    rows, _ = await ShowRepository(db_session).search_with_upcoming(
        from_date=date.today(), genre=None, page=1, size=10
    )

    assert [r.id for r in rows] == [sooner.id, later.id]

async def test_paginacao_retorna_total_correto(db_session):
    for i in range(3):
        await _seed_show(
            db_session, published=True, sessions=[(TOMORROW + timedelta(days=i), 5_000)]
        )

    rows, total = await ShowRepository(db_session).search_with_upcoming(
        from_date=date.today(), genre=None, page=1, size=2
    )

    assert total == 3
    assert len(rows) == 2

async def test_list_genres_so_com_espetaculo_visivel(db_session):
    await _seed_show(db_session, published=True, sessions=[(TOMORROW, 5_000)], genre="drama")
    await _seed_show(db_session, published=False, sessions=[(TOMORROW, 5_000)], genre="terror")

    genres = await ShowRepository(db_session).list_genres_in_catalog()

    assert genres == ["drama"]
```

---

## 4. Testes de integração cross-surface

| Fluxo | Verifica |
| --- | --- |
| Admin publica espetáculo com sessão futura (`catalog-admin-management`) → `GET /shows` | O espetáculo aparece na vitrine — fecha o elo "admin alimenta a vitrine" citado nas duas specs. |
| Admin despublica espetáculo → `GET /shows` | O espetáculo some imediatamente (nenhum cache). |
| `GET /shows?genre=<inexistente>` | `items: []`, `total: 0` — sem erro 404/422 (filtro sem resultado não é erro). |
| `GET /shows?fromDate=<formato inválido>` | 422 (validação de query param pelo FastAPI/Pydantic). |
| Frontend: abrir `/` sem filtro, aplicar filtro de gênero, aplicar filtro de data | URL reflete os filtros (`?genre=...&fromDate=...`); grid atualiza; "Nenhum espetáculo em cartaz para esse filtro." quando vazio. |
| Frontend: paginar | Botões "Anterior"/"Próxima" habilitam/desabilitam nos limites; URL carrega `page`. |

---

## 5. Roteiro de teste manual pré-entrega

Ambiente: API no contêiner, `alembic upgrade head`, `catalog-admin-management`
já implementado (para cadastrar espetáculos), frontend apontando para a API.

1. **Vitrine vazia.** Banco limpo, abrir `/`. **Esperado:** "Nenhum espetáculo
   em cartaz para esse filtro." (sem filtro nenhum aplicado — é o estado vazio
   real, não um filtro sem match).
2. **Cadastrar e publicar** 3 espetáculos pelo admin, um deles só com sessão
   passada, um em rascunho (não publicado), um publicado com sessão futura.
   Abrir `/` (anônimo). **Esperado:** só o publicado com sessão futura
   aparece.
3. **Faixa de preço.** No espetáculo visível, criar 2 sessões futuras com
   preços diferentes. **Esperado:** o card mostra a faixa (menor–maior); se
   preços iguais, mostra um valor só.
4. **Filtrar por gênero.** Selecionar um gênero no filtro. **Esperado:** lista
   atualiza; URL ganha `?genre=...`.
5. **Filtrar por data.** Escolher uma data futura sem sessão nesse intervalo.
   **Esperado:** "Nenhum espetáculo em cartaz para esse filtro."
6. **Paginação.** Cadastrar espetáculos suficientes para passar de uma
   página. **Esperado:** botão "Próxima" habilita; navegar muda a URL
   (`?page=2`) e a lista.
7. **Medir tempo de resposta** de `GET /shows` com ~50 espetáculos cadastrados
   (meta ≤ 2 s p95, RNF02).

---

## 6. Riscos e pontos de atenção

- **CI não sobe Postgres no job `test`.** `.github/workflows/ci.yml` define
  `DATABASE_URL` apontando para `localhost:5432` mas o job `test` não tem um
  `services: postgres: ...` (o padrão do GitHub Actions para isso). Hoje isso
  não quebra porque nenhum teste toca banco; a partir desta fatia, os testes
  de `§3` falham em CI até esse serviço existir. Ajuste necessário em
  `api.ludens/.github/workflows/ci.yml`, job `test`:

  ```yaml
  services:
    postgres:
      image: postgres:16
      env:
        POSTGRES_USER: ludens
        POSTGRES_PASSWORD: ludens
        POSTGRES_DB: ludens
      ports: ["5432:5432"]
      options: >-
        --health-cmd pg_isready --health-interval 5s --health-timeout 5s --health-retries 10
  ```

  Isso não é específico desta feature — vale para qualquer teste de
  repositório futuro. Se outra fatia já resolveu isso quando esta for
  implementada, só confirmar.
- **`tests/conftest.py` é peça nova e compartilhada** — ver nota no arquivo em
  §3. Primeira fatia a precisar dele; as próximas (`catalog-session-detail`,
  `catalog-admin-management`) reusam.
- **`array_agg`/query com `GROUP BY` são específicos do Postgres** — coerente
  com o driver real (`asyncpg`), sem portabilidade a outro banco.
- **`catalog-admin-management` (backend/frontend) ainda não mergeado**, mas a
  spec já foi corrigida (2026-09-10) — ver as notas de consistência no topo
  de `backend.md`/`frontend.md` desta spec. Os testes de `§3` usam
  `Show.publish()`/`Session.create()` na mesma forma de
  `catalog-admin-management/quality.md`.

---

## 7. Passo a passo TBD (QA)

```text
git checkout master && git pull && git checkout -b test/<NN>-catalog-show-search

git add tests/conftest.py tests/modules/__init__.py tests/modules/catalog/__init__.py \
        tests/modules/catalog/test_show_search_usecase.py \
        tests/modules/catalog/test_show_search_repository.py
git commit -m "test(catalog): cobrir busca de espetaculos (visibilidade, preco, ordenacao, paginacao)"

pytest -q
```

(`tests/modules/__init__.py` e `tests/modules/catalog/__init__.py` só entram
neste commit se `catalog-admin-management` ainda não os tiver criado —
conferir antes de recriar.)

Depois: `/team-ludens:tbd-pr`.

---

## 8. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 fechada). Pendência real:
**CI sem serviço Postgres no job de testes** (§6) — não impede escrever o
código nem abrir a issue, mas os testes de `§3` não rodam em CI até isso ser
corrigido. Registrar como issue de DevOps separada se ainda não existir uma.
