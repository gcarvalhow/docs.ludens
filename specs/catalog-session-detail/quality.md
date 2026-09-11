---
status: draft
spec: catalog-session-detail
surface: quality
created_at: 2026-09-10
updated_at: 2026-09-10
---

# Detalhe da sessão — Quality

**Resumo:** cobre a derivação de status público (`on_sale`/`sold_out`/
`closed`/`cancelled`), a leitura de disponibilidade (reuso de
`SeatCountsRepository`) e os 404 de espetáculo/sessão inexistente ou
despublicado. `_public_status` é pura (testável sem banco); o resto depende
de Postgres real, igual a `catalog-show-search`.
**RF:** RF02 · **RN:** RN05 (leitura) · **Módulo backend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-session-detail/integration.md`

---

## 1. Definition of Ready — checagem

Contra `docs.ludens/team/quality.md`.

| Item DoR | Situação |
| --- | --- |
| História no formato "Como [papel], eu quero [func.] para [benefício]" | OK — RF02 + spec.md §1/§3 |
| Critérios de aceite objetivos e verificáveis | OK — `integration.md` + `logic.md` §2/§3 |
| Regras de negócio e exceções especificadas | OK — `logic.md` §3, §5 |
| Dependências técnicas mapeadas | Parcial — ver bullets |
| Layout/protótipo aprovado | N/A — sem protótipo formal |

Detalhe "Dependências técnicas" (parcial):

- `catalog-admin-management` e, de preferência, `catalog-show-search`
  mergeados primeiro (mesmos arquivos editados).
- Mesma lacuna de infraestrutura de teste registrada em
  `catalog-show-search/quality.md` §1/§6 (`tests/conftest.py`, CI sem
  serviço Postgres no job `test`) — não repetir aqui, só herdar.
- `tickets`/`reservations` (tabelas de `booking`) ainda não existem —
  `available_count` só é testável como "igual à capacidade" até `booking`
  mergear (mesma ressalva de `catalog-admin-management/quality.md` §5).

Conclusão: **pronta para desenvolver** com as mesmas ressalvas de
infraestrutura já registradas nas fatias anteriores do módulo `catalog`.

---

## 2. Casos de teste de domínio (pytest, sem banco)

`_public_status` é função pura — instancia `Session` em memória (sem
`ShowRepository`/DB), sem precisar de sessão realmente passada (`Session.create`
recusa `starts_at` no passado) porque `now` é passado separado da criação.

| Caso | Cenário | RF |
| --- | --- | --- |
| `test_status_on_sale_quando_disponivel_positivo` | sessão futura, à venda, `available=5` → `"on_sale"` | RF02 |
| `test_status_sold_out_quando_disponivel_zero` | sessão futura, à venda, `available=0` → `"sold_out"` | RF02 |
| `test_status_closed_quando_now_passa_do_horario` | `now` passado como depois de `starts_at` → `"closed"`, mesmo com `available > 0` | RF02 |
| `test_status_cancelled_tem_prioridade_sobre_closed` | sessão cancelada **e** já teria passado do horário → `"cancelled"` (checado primeiro) | RF02 |

```python
# tests/modules/catalog/test_session_query_usecase.py  — novo
"""_public_status é pura — sem banco, sem HTTP."""

from datetime import datetime, timedelta, timezone
from uuid import uuid4

from app.modules.catalog.application.usecases.session_query_usecase import _public_status
from app.modules.catalog.domain.aggregates.session import Session

NOW = datetime.now(timezone.utc)
STARTS_AT = NOW + timedelta(hours=2)

def _new_session() -> Session:
    return Session.create(
        show_id=uuid4(), starts_at=STARTS_AT, venue="Sala 1", capacity=50,
        full_price_cents=8_000, now=NOW,
    )

def test_status_on_sale_quando_disponivel_positivo():
    session = _new_session()
    assert _public_status(session, available=5, now=NOW) == "on_sale"

def test_status_sold_out_quando_disponivel_zero():
    session = _new_session()
    assert _public_status(session, available=0, now=NOW) == "sold_out"

def test_status_closed_quando_now_passa_do_horario():
    session = _new_session()
    depois_do_horario = STARTS_AT + timedelta(minutes=1)
    assert _public_status(session, available=10, now=depois_do_horario) == "closed"

def test_status_cancelled_tem_prioridade_sobre_closed():
    session = _new_session()
    session.cancel()
    depois_do_horario = STARTS_AT + timedelta(minutes=1)
    assert _public_status(session, available=10, now=depois_do_horario) == "cancelled"
```

---

## 3. Casos de teste de repositório/usecase (precisam de Postgres real)

Reusa a fixture `db_session` de `tests/conftest.py` (criada por
`catalog-show-search/quality.md` — não recriar).

| Caso | Cenário | RF |
| --- | --- | --- |
| `test_get_session_detail_sessao_inexistente_404` | id aleatório → `NotFoundError` | RF02 |
| `test_get_session_detail_espetaculo_despublicado_404` | show em rascunho, sessão futura à venda → `NotFoundError` (mesma inferência de `catalog-show-search`) | RF02 |
| `test_get_session_detail_disponivel_igual_capacidade_sem_booking` | sem tabelas `tickets`/`reservations` → `available_count == capacity` | RN05 |
| `test_get_show_detail_so_lista_sessoes_futuras_a_venda` | 1 sessão futura à venda + 1 sessão futura cancelada → `sessions` só tem a à venda | RF02 |
| `test_get_show_detail_espetaculo_nao_encontrado_404` | show inexistente ou em rascunho → `NotFoundError` | RF02 |

```python
# tests/modules/catalog/test_session_query_repository.py  — novo
"""SessionQueryUseCase — precisa de Postgres real (sem mock)."""

from datetime import datetime, timedelta, timezone
from uuid import uuid4

import pytest

from app.core.domain import NotFoundError
from app.modules.catalog.application.usecases.session_query_usecase import SessionQueryUseCase
from app.modules.catalog.domain.aggregates.session import Session
from app.modules.catalog.domain.aggregates.show import Show
from app.modules.catalog.infrastructure.repositories import SessionRepository, ShowRepository

NOW = datetime.now(timezone.utc)
TOMORROW = NOW + timedelta(days=1)

async def _seed(db_session, *, published: bool, cancel: bool = False) -> tuple[Show, Session]:
    show = Show.create(title="Peça", synopsis="Sinopse.", image_url="https://x.test/i.jpg", genre="drama")
    if published:
        show.publish()
    await ShowRepository(db_session).save(show)

    session = Session.create(
        show_id=show.id, starts_at=TOMORROW, venue="Sala 1", capacity=50,
        full_price_cents=8_000, now=NOW,
    )
    if cancel:
        session.cancel()
    await SessionRepository(db_session).save(session)
    await db_session.flush()
    return show, session

async def test_get_session_detail_sessao_inexistente_404(db_session):
    with pytest.raises(NotFoundError):
        await SessionQueryUseCase(db_session).get_session_detail(uuid4())

async def test_get_session_detail_espetaculo_despublicado_404(db_session):
    _, session = await _seed(db_session, published=False)

    with pytest.raises(NotFoundError):
        await SessionQueryUseCase(db_session).get_session_detail(session.id)

async def test_get_session_detail_disponivel_igual_capacidade_sem_booking(db_session):
    _, session = await _seed(db_session, published=True)

    detail = await SessionQueryUseCase(db_session).get_session_detail(session.id)

    assert detail.available_count == session.capacity
    assert detail.status == "on_sale"
    assert [t.type for t in detail.ticket_types] == ["full", "half"]

async def test_get_show_detail_so_lista_sessoes_futuras_a_venda(db_session):
    show, _ = await _seed(db_session, published=True)
    cancelled_session = Session.create(
        show_id=show.id, starts_at=TOMORROW, venue="Sala 2", capacity=20,
        full_price_cents=6_000, now=NOW,
    )
    cancelled_session.cancel()
    await SessionRepository(db_session).save(cancelled_session)
    await db_session.flush()

    detail = await SessionQueryUseCase(db_session).get_show_detail(show.id)

    assert len(detail.sessions) == 1

async def test_get_show_detail_espetaculo_nao_encontrado_404(db_session):
    with pytest.raises(NotFoundError):
        await SessionQueryUseCase(db_session).get_show_detail(uuid4())
```

---

## 4. Testes de integração cross-surface

| Fluxo | Verifica |
| --- | --- |
| `catalog-show-search`: clicar num card da vitrine → `GET /shows/{id}` → clicar numa sessão → `GET /sessions/{id}` | Navegação ponta a ponta funciona; o `showId`/`sessionId` da vitrine batem com os das rotas de detalhe. |
| Admin cancela uma sessão (`catalog-admin-management`) → `GET /sessions/{id}` | `status == "cancelled"`; botão "Reservar" desabilitado no frontend. |
| Sessão com `starts_at` no passado (via ajuste manual de dado em teste, já que `Session.create` recusa) → `GET /sessions/{id}` | `status == "closed"`. |
| Frontend: abrir `/sessoes/{id}`, esperar ~15 s | Nova chamada de rede a `GET /sessions/{id}` (*polling*); sem *flicker* perceptível (React Query mantém os dados anteriores durante o refetch). |

---

## 5. Roteiro de teste manual pré-entrega

Ambiente: API no contêiner, `catalog-admin-management` e
`catalog-show-search` já implementados.

1. **Navegar da vitrine.** Na home, clicar num espetáculo. **Esperado:** vai
   para `/espetaculos/{id}`, mostra sinopse e lista de sessões futuras.
2. **Abrir uma sessão.** Clicar numa sessão. **Esperado:** `/sessoes/{id}`
   mostra data, local, preços (inteira/meia) e disponibilidade.
3. **Esgotar e ver refletido.** Vender/reservar ingressos até
   `available_count == 0` (quando `booking` existir). **Esperado:** rótulo
   "Esgotado", botão desabilitado, sem precisar recarregar a página
   (*polling*).
4. **Cancelar a sessão pelo admin** enquanto a página de detalhe está aberta
   em outra aba. **Esperado:** em até ~15 s a página reflete "Cancelada".
5. **Sessão inexistente.** Acessar `/sessoes/{uuid aleatório}`. **Esperado:**
   "Sessão não encontrada.", sem erro não tratado na tela.
6. **Medir tempo de resposta** de `GET /sessions/{id}` (meta ≤ 1 s p95,
   RNF02).

---

## 6. Riscos e pontos de atenção

- **`available_count` só testável como "= capacidade" até `booking`
  mergear** — mesma ressalva de `catalog-admin-management/quality.md` §5;
  os testes de `§3` documentam esse limite, não o escondem.
- **CI sem serviço Postgres no job `test`** — mesma pendência já registrada
  em `catalog-show-search/quality.md` §6; não duplicar a issue, só confirmar
  que segue aberta.
- **`find_by_id_for_update` é criado e testado em `catalog-admin-management`**
  (`session_usecase`, lá) — esta spec não reintroduz teste para o mesmo
  método, só o reusa via `dependencies.py`.

---

## 7. Passo a passo TBD (QA)

```text
git checkout master && git pull && git checkout -b test/<NN>-catalog-session-detail

git add tests/modules/catalog/test_session_query_usecase.py \
        tests/modules/catalog/test_session_query_repository.py
git commit -m "test(catalog): cobrir status publico e leitura de detalhe da sessao"

pytest -q
```

Depois: `/team-ludens:tbd-pr`.

---

## 8. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 fechada). Pendências de
infraestrutura já registradas em `catalog-show-search/quality.md` (CI sem
Postgres) valem igual aqui — não bloqueiam abrir a issue.
