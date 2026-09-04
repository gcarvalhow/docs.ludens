---
status: draft
spec: catalog-admin-management
surface: quality
created_at: 2026-09-03
updated_at: 2026-09-04
---

# Gestão de espetáculos e sessões (admin) — Quality

**Resumo:** cobre o domínio de `catalog` para a gestão admin (RF08) — criar
`Show`/`Session`, a regra "sessão vendida não se apaga, só se cancela"
(`ConflictError`), o evento `SessionCancelled` que aciona o reembolso (RF07 /
RN02), a barreira de capacidade abaixo do comprometido, a data futura na
criação e a meia = 50% da inteira (RN04). Mais os testes cross-surface (admin
cria → aparece na vitrine; rota admin exige `ADMIN`) e o roteiro manual.
**RF:** RF08 · **RN:** — (aciona RN02 via RF07; RN04) · **Módulo backend:**
`catalog`
**Contrato:** `docs.ludens/specs/catalog-admin-management/integration.md`

---

## 1. Definition of Ready — checagem

Contra `docs.ludens/team/quality.md`.

| Item DoR | Situação |
| --- | --- |
| História no formato "Como [papel], eu quero [func.] para [benefício]" | OK — RF08 + spec.md §1/§3 |
| Critérios de aceite objetivos e verificáveis | OK — integration.md + logic.md §2/§3 |
| Regras de negócio e exceções especificadas | OK — logic.md §3; valores RN02/RN04 aprovados (PO 2026-08-28) |
| Dependências técnicas mapeadas | Parcial — ver bullets abaixo |
| Layout/protótipo aprovado | N/A — área interna, sem protótipo; UI conforme `frontend.md` |

Detalhe do item "Regras de negócio": cancela ≠ exclui, capacidade nunca abaixo
do comprometido, data futura na criação, meia derivada — todos em `logic.md` §3;
RN02 (política de reembolso) e RN04 (meia) com valores aprovados em
`docs.ludens/requirements/business-rules.md`.

Detalhe do item "Dependências técnicas" (parcial):

- `identity-auth` (`require_admin`, `Role`) ainda não mergeado.
- Tabelas de `booking` (`tickets` / `reservations`, lidas por
  `SeatCountsRepository`) ainda não existem.
- O domínio e seus testes não dependem de nenhum dos dois; só a contagem real
  de vendas depende de `booking` (ver §5).

Conclusão: **pronta para desenvolver** com a ressalva de dependência acima
registrada — as fatias `identity-auth` e `booking` já estão no backlog.

---

## 2. Casos de teste de domínio (pytest)

Sem DB e sem HTTP — instancia o aggregate, chama o método, verifica estado e
eventos acumulados (`docs.ludens/backend/testing.md`).

| Caso                                                    | Cenário (estado → ação → asserção)                                                                  | RN/RF       |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | ----------- |
| `test_cria_show_como_rascunho`                          | `Show.create` → `status=DRAFT`, evento `ShowCreated`, `version=1`                                   | RF08        |
| `test_publish_unpublish_alterna_status`                 | `create` → `publish` → `PUBLISHED`+`ShowPublished`; `unpublish` → `DRAFT`+`ShowUnpublished`         | RF08        |
| `test_publish_e_idempotente`                            | `publish` de show já publicado → nenhum evento novo                                                 | RF08        |
| `test_update_show_troca_campos`                         | `update(title=…)` → campos aplicados, evento `ShowUpdated`                                          | RF08        |
| `test_deactivate_show_marca_inativo`                    | `deactivate()` → `is_active=False`, evento `ShowDeactivated`                                        | RF08        |
| `test_cria_sessao_futura_ok`                            | `Session.create(starts_at=amanhã)` → `ON_SALE`, evento `SessionCreated`, `full_price_cents` correto | RF08        |
| `test_cria_sessao_no_passado_recusa`                    | `starts_at=ontem` → `DomainError`                                                                   | RF08        |
| `test_meia_e_metade_da_inteira`                         | `full=R$120` → `half_price.reais == 60.0`; ímpar `1201c` → `600c` (trunca)                          | RN04        |
| `test_reduzir_capacidade_abaixo_do_comprometido_recusa` | cap 100, `committed=8`, `update(capacity=5)` → `ConflictError`                                      | RF08        |
| `test_atualizar_capacidade_acima_do_comprometido_ok`    | `committed=8`, `update(capacity=50)` → evento `SessionUpdated`                                      | RF08        |
| `test_cancelar_sessao_emite_evento`                     | `cancel()` → `status=CANCELLED`, evento `SessionCancelled` com `show_id`/`starts_at`/`cancelled_at` | RF08 / RF07 |
| `test_cancelar_sessao_ja_cancelada_recusa`              | `cancel()` duas vezes → `ConflictError`                                                             | RF08        |
| `test_excluir_sessao_com_venda_recusa`                  | `deactivate(tickets_sold=1)` → `ConflictError("Cancele a sessão em vez de excluir.")`               | RF08        |
| `test_excluir_sessao_sem_venda_ok`                      | `deactivate(tickets_sold=0)` → `is_active=False`, evento `SessionDeactivated`                       | RF08        |
| `test_editar_sessao_cancelada_recusa`                   | `cancel()` → `update(...)` → `ConflictError`                                                        | RF08        |
| `test_money_from_reais_e_negativo`                      | `from_reais("12.34")==1234c`; `Money(cents=-1)` → `DomainError`                                     | RN04        |

### `tests/modules/__init__.py`

```python
# tests/modules/__init__.py  — novo
# (arquivo vazio — marca o pacote de testes de módulo)
```

### `tests/modules/catalog/__init__.py`

```python
# tests/modules/catalog/__init__.py  — novo
# (arquivo vazio)
```

### `tests/modules/catalog/test_show.py`

```python
# tests/modules/catalog/test_show.py  — novo
"""Domínio de catalog — aggregate Show. Sem DB, sem HTTP."""

from app.modules.catalog.domain.aggregates.show import Show
from app.modules.catalog.domain.enumerations.show_status import ShowStatus


def _new_show() -> Show:
    return Show.create(
        title="Hamlet",
        synopsis="Príncipe da Dinamarca.",
        image_url="https://exemplo.test/hamlet.jpg",
        genre="drama",
    )


def test_cria_show_como_rascunho():
    show = _new_show()

    assert show.status is ShowStatus.DRAFT
    assert show.is_active is True
    assert show.version == 1
    assert [type(e).__name__ for e in show.dequeue_events()] == ["ShowCreated"]


def test_publish_unpublish_alterna_status():
    show = _new_show()
    show.dequeue_events()

    show.publish()
    assert show.status is ShowStatus.PUBLISHED
    assert [type(e).__name__ for e in show.dequeue_events()] == ["ShowPublished"]

    show.unpublish()
    assert show.status is ShowStatus.DRAFT
    assert [type(e).__name__ for e in show.dequeue_events()] == ["ShowUnpublished"]


def test_publish_e_idempotente():
    show = _new_show()
    show.publish()
    show.dequeue_events()

    show.publish()

    assert show.status is ShowStatus.PUBLISHED
    assert show.dequeue_events() == []


def test_update_show_troca_campos():
    show = _new_show()
    show.dequeue_events()

    show.update(
        title="Macbeth",
        synopsis="Ambição e profecia.",
        image_url="https://exemplo.test/macbeth.jpg",
        genre="tragédia",
    )

    assert show.title == "Macbeth"
    assert show.genre == "tragédia"
    assert [type(e).__name__ for e in show.dequeue_events()] == ["ShowUpdated"]


def test_deactivate_show_marca_inativo():
    show = _new_show()
    show.dequeue_events()

    show.deactivate()

    assert show.is_active is False
    assert [type(e).__name__ for e in show.dequeue_events()] == ["ShowDeactivated"]
```

### `tests/modules/catalog/test_session.py`

```python
# tests/modules/catalog/test_session.py  — novo
"""Domínio de catalog — aggregate Session. Sem DB, sem HTTP.

Importa Show para o SQLAlchemy resolver a FK sessions.show_id -> shows.id ao
configurar o mapper na primeira instância de Session.
"""

from datetime import datetime, timedelta, timezone
from uuid import uuid4

import pytest

from app.core.domain.errors import ConflictError, DomainError
from app.modules.catalog.domain.aggregates.session import Session
from app.modules.catalog.domain.aggregates.show import Show  # noqa: F401
from app.modules.catalog.domain.enumerations.session_status import SessionStatus
from app.modules.catalog.domain.value_objects.money import Money

NOW = datetime.now(timezone.utc)
TOMORROW = NOW + timedelta(days=1)
YESTERDAY = NOW - timedelta(days=1)


def _new_session(*, capacity: int = 100, price_reais: float = 120.0) -> Session:
    return Session.create(
        show_id=uuid4(),
        starts_at=TOMORROW,
        venue="Sala Principal",
        capacity=capacity,
        full_price=Money.from_reais(price_reais),
        now=NOW,
    )


def test_cria_sessao_futura_ok():
    session = _new_session(price_reais=120.0)

    assert session.status is SessionStatus.ON_SALE
    assert session.full_price_cents == 12_000
    assert session.is_on_sale(NOW) is True
    assert [type(e).__name__ for e in session.dequeue_events()] == ["SessionCreated"]


def test_cria_sessao_no_passado_recusa():
    with pytest.raises(DomainError):
        Session.create(
            show_id=uuid4(),
            starts_at=YESTERDAY,
            venue="Sala Principal",
            capacity=50,
            full_price=Money.from_reais(90.0),
            now=NOW,
        )


def test_meia_e_metade_da_inteira():
    session = _new_session(price_reais=120.0)
    assert session.half_price.reais == 60.0

    impar = _new_session(price_reais=12.01)  # 1201 centavos
    assert impar.half_price.cents == 600  # trunca ao centavo (RN04)


def test_reduzir_capacidade_abaixo_do_comprometido_recusa():
    session = _new_session(capacity=100)
    session.dequeue_events()

    with pytest.raises(ConflictError):
        session.update(
            starts_at=TOMORROW,
            venue="Sala Principal",
            capacity=5,
            full_price=session.full_price,
            committed=8,
            now=NOW,
        )


def test_atualizar_capacidade_acima_do_comprometido_ok():
    session = _new_session(capacity=100)
    session.dequeue_events()

    session.update(
        starts_at=TOMORROW,
        venue="Sala Anexa",
        capacity=50,
        full_price=session.full_price,
        committed=8,
        now=NOW,
    )

    assert session.capacity == 50
    assert session.venue == "Sala Anexa"
    assert [type(e).__name__ for e in session.dequeue_events()] == ["SessionUpdated"]


def test_cancelar_sessao_emite_evento():
    session = _new_session()
    session.dequeue_events()

    session.cancel()

    assert session.status is SessionStatus.CANCELLED
    assert session.is_on_sale(NOW) is False
    events = session.dequeue_events()
    assert [type(e).__name__ for e in events] == ["SessionCancelled"]
    cancelled = events[0]
    assert cancelled.show_id == session.show_id
    assert cancelled.starts_at == session.starts_at
    assert cancelled.cancelled_at is not None


def test_cancelar_sessao_ja_cancelada_recusa():
    session = _new_session()
    session.cancel()
    session.dequeue_events()

    with pytest.raises(ConflictError):
        session.cancel()


def test_excluir_sessao_com_venda_recusa():
    session = _new_session()
    session.dequeue_events()

    with pytest.raises(ConflictError):
        session.deactivate(tickets_sold=1)


def test_excluir_sessao_sem_venda_ok():
    session = _new_session()
    session.dequeue_events()

    session.deactivate(tickets_sold=0)

    assert session.is_active is False
    assert [type(e).__name__ for e in session.dequeue_events()] == ["SessionDeactivated"]


def test_editar_sessao_cancelada_recusa():
    session = _new_session()
    session.cancel()
    session.dequeue_events()

    with pytest.raises(ConflictError):
        session.update(
            starts_at=TOMORROW,
            venue="Sala Principal",
            capacity=100,
            full_price=session.full_price,
            committed=0,
            now=NOW,
        )
```

### `tests/modules/catalog/test_money.py`

```python
# tests/modules/catalog/test_money.py  — novo
"""Value object Money — evita float em preço; meia = 50% (RN04)."""

import pytest

from app.core.domain.errors import DomainError
from app.modules.catalog.domain.value_objects.money import Money


def test_from_reais_converte_para_centavos():
    assert Money.from_reais("12.34").cents == 1234
    assert Money.from_reais(120).cents == 12_000


def test_half_trunca_ao_centavo():
    assert Money(cents=1201).half().cents == 600
    assert Money(cents=12_000).half().cents == 6_000


def test_zero():
    assert Money.zero().cents == 0


def test_valor_negativo_recusa():
    with pytest.raises(DomainError):
        Money(cents=-1)
```

---

## 3. Testes de integração cross-surface

Precisam do backend com Postgres em contêiner; alguns só ficam observáveis
ponta a ponta quando `identity-auth` e `booking` entrarem (anotado).

| Fluxo                                                                                                                                                                             | Verifica                                                                                                                                                                                                                                                                                            |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Admin: `POST /admin/shows` → `POST /admin/shows/{id}/sessions` (data futura) → `POST /admin/shows/{id}/publish` → `GET /sessions/{sessionId}` (contrato `catalog-session-detail`) | A sessão aparece no detalhe público com `status="on_sale"` e `availableCount = capacity` (0 vendas). Alimenta a vitrine.                                                                                                                                                                            |
| Admin publica espetáculo **sem** sessão futura → `GET /shows` (contrato `catalog-show-search`)                                                                                    | O espetáculo **não** aparece na vitrine (regra "só com sessão futura à venda").                                                                                                                                                                                                                     |
| Admin cria 2ª sessão futura no mesmo show → `GET /shows/{showId}`                                                                                                                 | Ambas as sessões futuras retornam em `sessions`; sessão no passado não.                                                                                                                                                                                                                             |
| `POST /admin/sessions/{id}/cancel` numa sessão à venda → 202 + tabela `events`                                                                                                    | Uma linha `event_type="SessionCancelled"`, `dispatched_at` NULL → preenchido pelo relay (~2 s). Payload traz `id`, `show_id`, `starts_at`, `cancelled_at`. Quando `payment` existir: cada pedido confirmado da sessão entra na fila de estorno (RF07 / RN02 a partir do cancelamento), idempotente. |
| `DELETE /admin/sessions/{id}` com `ticketsSold > 0` (após `booking` permitir vender)                                                                                              | 409, corpo `{ "detail": "Cancele a sessão em vez de excluir." }`. Nenhuma linha apagada.                                                                                                                                                                                                            |
| `DELETE /admin/sessions/{id}` sem vendas                                                                                                                                          | 204; a sessão some das leituras (`is_active=False`), `events` intacta.                                                                                                                                                                                                                              |
| `PATCH /admin/sessions/{id}` com `capacity` < `ticketsSold + reservedOpen`                                                                                                        | 409 `{ "detail": "Já há ingressos comprometidos nesta sessão." }`.                                                                                                                                                                                                                                  |
| `PATCH /admin/sessions/{id}` com `startsAt` no passado                                                                                                                            | 422 `{ "detail": "A data da sessão deve ser futura." }`.                                                                                                                                                                                                                                            |
| Qualquer rota `/admin/*` com token de `role=BUYER` (ou sem token)                                                                                                                 | 403 (`require_admin`). Depende de `identity-auth` mergeado; até lá, testar com um stub de `require_admin`.                                                                                                                                                                                          |
| Frontend (Playwright, quando a suíte existir): `/admin/espetaculos` como não-admin                                                                                                | `RequireAuth` redireciona para `/` (vitrine); a tela de gestão não monta.                                                                                                                                                                                                                           |
| Frontend: criar show + sessão → publicar → item some/aparece na lista; tentar "Excluir" numa sessão com `canDelete=false`                                                         | Botão "Excluir" desabilitado; "Cancelar sessão" abre o `ConfirmCancelSessionDialog` com aviso de reembolso; confirmar → toast e lista atualizada.                                                                                                                                                   |

---

## 4. Roteiro de teste manual pré-entrega

Ambiente: API no contêiner, `alembic upgrade head`, admin semeado
(`scripts/seed_admin.py`), frontend apontando para a API.

1. **Login admin.** Entrar em `/login` com a conta admin semeada. Ir a
   `/admin/espetaculos`. **Esperado:** a tela de gestão carrega; lista vazia
   mostra "Nenhum espetáculo cadastrado ainda.".
2. **Criar espetáculo.** "Novo espetáculo" → preencher título, sinopse, URL de
   imagem, gênero → Salvar. **Esperado:** toast "Espetáculo criado."; o card
   aparece com selo "Rascunho".
3. **Criar sessão futura.** No card, "Nova sessão" → data/hora amanhã, local,
   capacidade 50, inteira R$ 80 → Salvar. **Esperado:** toast "Sessão criada.";
   a linha mostra "À venda", "Vendidos: 0", "Inteira R$ 80,00 · Meia R$ 40,00".
4. **Data no passado é recusada.** Editar a sessão, pôr data de ontem → Salvar.
   **Esperado:** erro no campo/toast "A data da sessão deve ser futura."; nada
   é salvo.
5. **Publicar.** No card, "Publicar". **Esperado:** toast; selo vira
   "Publicado".
6. **Ver na vitrine.** Abrir `/` (anônimo, outra aba). **Esperado:** o
   espetáculo aparece; abrir o detalhe da sessão mostra disponibilidade =
   capacidade.
7. **Espetáculo sem sessão não aparece.** Criar um 2º espetáculo, publicar sem
   sessão. **Esperado:** não aparece na vitrine; aparece na lista admin como
   "Publicado".
8. **Vender 1 ingresso** (quando `booking` estiver disponível): como comprador,
   reservar e pagar 1 ingresso da sessão do passo 3. **Esperado:** "Vendidos: 1"
   na lista admin; botão "Excluir" da sessão fica desabilitado (`canDelete`
   falso).
9. **Excluir sessão vendida é bloqueado.** Tentar "Excluir" a sessão vendida
   (via API direta, já que o botão está travado). **Esperado:** 409 "Cancele a
   sessão em vez de excluir.".
10. **Cancelar sessão.** "Cancelar sessão" → o diálogo avisa do reembolso →
    confirmar. **Esperado:** toast "Sessão cancelada..."; status vira
    "Cancelada"; a sessão some da vitrine; o comprador do passo 8 entra na fila
    de reembolso (quando `payment` existir) conforme RN02 a partir do
    cancelamento.
11. **Excluir sessão sem venda.** Numa sessão sem vendas, "Excluir".
    **Esperado:** 204; a sessão some da lista; nenhum erro.
12. **Reduzir capacidade abaixo do comprometido.** Editar uma sessão com N
    comprometidos, pôr capacidade < N. **Esperado:** 409 "Já há ingressos
    comprometidos nesta sessão.".
13. **Acesso restrito.** Deslogar / logar como comprador comum e abrir
    `/admin/espetaculos`. **Esperado:** redirecionado para a vitrine; chamada
    direta a `/admin/shows` → 403.
14. **Resiliência do relay.** Após o passo 10, derrubar e subir a API.
    **Esperado:** o `SessionCancelled` ainda `dispatched_at IS NULL` continua
    sendo processado no boot; nada se perde (RNF03).

---

## 5. Riscos e pontos de atenção

- **`booking` não mergeado → contagem de vendas = 0.** `SeatCountsRepository` lê
  `tickets`/`reservations` (tabelas de `booking`). Sem elas, `ticketsSold` e
  `reservedOpen` vêm 0 e a regra "não exclui sessão vendida" fica permissiva em
  runtime até `booking` entrar. Os testes de domínio (`test_session.py`) já
  fixam o invariante; o risco é só de integração. Ao mergear `booking`,
  revisar os literais de status no SQL (`status = 'valid'`, `status = 'open'`).
- **Ordem de merge com `identity-auth`.** `require_admin`/`Role` e
  `core/domain/errors.py` + `core/shared/schema.py` são compartilhados. Se
  `catalog` entrar primeiro, ajustar `down_revision` da migration para `None` e
  garantir que `identity-auth` reuse os arquivos de `core` em vez de recriar.
- **Envelope de erro ainda "global a definir".** Este backend adota
  `{ "detail": "<mensagem>" }` para erro de negócio (403/404/409/422-domínio) e
  `{ "detail": [{ "field", "message" }] }` para 422 de forma. O frontend
  (`apiErrorMessage`) já lê as duas formas. Registrado em `integration.md`.
- **Fuso horário.** `startsAt` trafega em ISO 8601 com offset; o `<input
  type="datetime-local">` é hora local — a conversão para UTC fica na service
  (`toISOString()`). Bug clássico: comparar naïve com aware. O domínio sempre
  usa `datetime.now(timezone.utc)` e o schema recusa `datetime` sem tzinfo.
- **Enum `native_enum=False`.** `show_status`/`session_status` são `VARCHAR +
  CHECK`, não tipo enum nativo do Postgres. A migration e o ORM precisam bater
  (ambos `native_enum=False`). Uma divergência aqui só aparece no
  `alembic upgrade` contra banco limpo — rodar de fato, não "mentalmente".
- **Sem suíte de teste de frontend** (`docs.ludens/backend/testing.md` §gaps):
  os cenários de UI da §3 são Playwright a rodar quando a suíte existir; hoje o
  portão é `npm run lint` + `npm run build`.

---

## 6. Passo a passo TBD (QA)

```text
git checkout master && git pull && git checkout -b test/<NN>-catalog-admin

git add tests/modules/__init__.py tests/modules/catalog
git commit -m "test(catalog): cobrir cancelar-nao-excluir, capacidade, data futura e meia"

pytest -q
```

Depois: `/team-ludens:tbd-pr`. Backend e QA sobem juntos a partir do `logic.md`;
os testes de domínio não dependem de `identity-auth` nem de `booking`.

---

## 7. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 e logic.md fechados). As
pendências de dependência (`identity-auth`, `booking`) estão detalhadas em §1 e
§5 e não impedem a fatia de virar issue — os testes de domínio cobrem o
invariante independentemente delas.
