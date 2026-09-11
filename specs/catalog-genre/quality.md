---
status: draft
spec: catalog-genre
surface: quality
created_at: 2026-09-11
---

# Gêneros do catálogo — Quality

**Resumo:** gênero vira entidade própria no módulo `catalog`, criação exclusiva pelo admin com nome normalizado (identidade única) e ícone obrigatório; formulário de espetáculo passa a selecionar gênero de uma lista.
**RF:** fortalece RF01 · ajusta RF08 · **RN:** nenhuma RN01–RN05 diretamente
**Contrato:** `docs.ludens/specs/catalog-genre/integration.md`

---

## 1. Definition of Ready — checagem

| Item DoR | Situação |
|---|---|
| Critérios de aceite testáveis | ✅ — `spec.md` §8 + `logic.md` §3 dão regras verificáveis (duplicidade, ícone obrigatório, migração, alcance do filtro) |
| Contrato (`integration.md`) definido | ✅ — reconciliado entre `backend.md`/`frontend.md` desta pasta (rotas `/admin/genres`, shape `{id,name,icon}` único) |
| RN citadas com valores aprovados | N/A — feature não introduz nem depende de RN01–RN05 numeradas; a regra própria ("nome normalizado é identidade única") está fechada em `logic.md` §3 |
| Dependências de outras features resolvidas | ⚠️ parcial — depende de RF08 (`catalog-admin-management`) já implementado (está); mas **quebra o contrato de `catalog-show-search`** (RF01) sem esse documento estar atualizado ainda — ver bloqueio |

## 2. Casos de teste de domínio (pytest)

| Caso | Cenário | RN/RF |
|---|---|---|
| `test_cria_genero_com_nome_e_icone` | cria gênero válido → campos corretos | RF08 (ajuste) |
| `test_evento_genrecreated_e_acumulado_com_dados_corretos` | cria gênero → evento `GenreCreated` acumulado | padrão de domínio |
| `test_normalizacao_remove_acento_case_e_espaco_nas_bordas` | grafias variadas → mesma forma normalizada | `logic.md` §3 |
| `test_grafias_diferentes_do_mesmo_genero_colidem_na_mesma_identidade` | "Comédia"/"comedia" → mesma identidade | `logic.md` §3 |
| `test_nomes_de_generos_diferentes_nao_colidem` | nomes diferentes → identidades diferentes | — |
| `test_icone_vazio_e_recusado` / `..._so_com_espacos_e_recusado` / `..._none_e_recusado` | ícone ausente/vazio → `DomainError` | `logic.md` §3 |
| `test_icone_repetido_entre_generos_diferentes_e_permitido` | dois gêneros, mesmo ícone → aceito | `logic.md` §5 (caso de borda) |
| `test_nao_expoe_metodo_de_edicao_ou_renomeacao` / `..._exclusao_ou_desativacao` | aggregate não tem `update`/`delete`/`deactivate` | spec §6 |

```python
# tests/test_catalog_genre.py — novo
"""
Testes de domínio do aggregate Genre (catalog-genre).

Cobre: nome normalizado é a identidade de unicidade do gênero (remove
acento, ignora maiúscula/minúscula), ícone obrigatório na criação, e ausência
de ciclo de vida (sem editar/excluir — spec §6, logic.md §1 e §3).

Domínio testado sem banco nem HTTP, conforme docs.ludens/backend/testing.md.
"""

import pytest

from app.core.domain.errors import DomainError
from app.modules.catalog.domain.aggregates.genre import Genre
from app.modules.catalog.domain.events.domain_events import GenreCreated


class TestGenreCreate:
    def test_cria_genero_com_nome_e_icone(self):
        genre = Genre.create(name="Comédia", icon="laugh")

        assert genre.icon == "laugh"

    def test_evento_genrecreated_e_acumulado_com_dados_corretos(self):
        genre = Genre.create(name="Drama", icon="drama")

        events = genre.dequeue_events()

        assert len(events) == 1
        assert isinstance(events[0], GenreCreated)
        assert events[0].version == 1
        assert events[0].id == genre.id
        assert events[0].icon == "drama"

    @pytest.mark.parametrize(
        "bruto,esperado",
        [
            ("Comédia", "comedia"),
            ("comedia", "comedia"),
            ("COMÉDIA", "comedia"),
            ("  Comédia  ", "comedia"),
            ("Ação", "acao"),
        ],
    )
    def test_normalizacao_remove_acento_case_e_espaco_nas_bordas(self, bruto, esperado):
        genre = Genre.create(name=bruto, icon="drama")

        assert genre.name == esperado

    def test_grafias_diferentes_do_mesmo_genero_colidem_na_mesma_identidade(self):
        a = Genre.create(name="Comédia", icon="laugh")
        b = Genre.create(name="comedia", icon="popcorn")

        assert a.name == b.name

    def test_nomes_de_generos_diferentes_nao_colidem(self):
        a = Genre.create(name="Comédia", icon="laugh")
        b = Genre.create(name="Drama", icon="drama")

        assert a.name != b.name

    def test_icone_vazio_e_recusado(self):
        with pytest.raises(DomainError):
            Genre.create(name="Comédia", icon="")

    def test_icone_so_com_espacos_e_recusado(self):
        with pytest.raises(DomainError):
            Genre.create(name="Comédia", icon="   ")

    def test_icone_none_e_recusado(self):
        with pytest.raises(DomainError):
            Genre.create(name="Comédia", icon=None)  # type: ignore[arg-type]

    def test_icone_repetido_entre_generos_diferentes_e_permitido(self):
        a = Genre.create(name="Comédia", icon="laugh")
        b = Genre.create(name="Comédia Musical", icon="laugh")

        assert a.icon == b.icon == "laugh"
        assert a.name != b.name

    def test_nao_expoe_metodo_de_edicao_ou_renomeacao(self):
        genre = Genre.create(name="Dança", icon="disc-3")

        assert not hasattr(genre, "update")
        assert not hasattr(genre, "rename")

    def test_nao_expoe_metodo_de_exclusao_ou_desativacao(self):
        genre = Genre.create(name="Dança", icon="disc-3")

        assert not hasattr(genre, "delete")
        assert not hasattr(genre, "deactivate")
```

> Nota: `test_icone_vazio_e_recusado` etc. dependem de `Genre.create` validar
> ícone vazio (`DomainError`) — reconciliado com `backend.md` (`Genre.create`
> valida ícone além do nome, não só a forma via `GenreRequest.icon`).

## 3. Testes de integração cross-surface

Sem `TestClient`/Postgres real em container no repo hoje (débito técnico
[api.ludens#22](https://github.com/gcarvalhow/api.ludens/issues/22)), estes
testes **não têm onde rodar automatizado ainda** — descrito para quando essa
infra existir.

| Fluxo | Verifica |
|---|---|
| `POST /admin/genres` duplo concorrente com nomes que normalizam igual | exatamente um 201, o outro 409 — prova que a constraint `uq_genres_name_active` (não um `exists_by` isolado) sustenta a atomicidade |
| `GET /admin/genres` (lista completa) vs. `GET /genres` (filtro público) | gênero recém-criado sem espetáculo publicado aparece na primeira, não na segunda |
| Migração `0003_catalog_genre` | espetáculos com grafias variantes ("Comédia"/"comedia"/"COMÉDIA ") terminam todos com o **mesmo** `genre_id`; nenhum fica com `genre_id` nulo |
| `create_show`/`update_show` com `genre_id` inexistente | recusado (404 — `NotFoundError` via `_require_genre`), nunca aceito silenciosamente |
| Frontend: `GET /admin/genres` retornando `[]` | `ShowForm` orienta "cadastre um gênero primeiro", não renderiza `<Select>` vazio sem explicação |

## 4. Roteiro de teste manual pré-entrega

**Fluxo principal:**

```
admin cadastra gênero (nome + ícone da paleta)
  → gênero aparece na lista completa do form de espetáculo
  → admin cadastra espetáculo escolhendo aquele gênero
  → espetáculo ainda não publicado: gênero NÃO aparece no filtro público
  → admin publica o espetáculo (com sessão futura à venda)
  → gênero passa a aparecer no filtro público, com o mesmo ícone
```

**Casos obrigatórios herdados (regressão — feature mexe no aggregate `Show`):**

1. Compra de inteira e meia-entrada segue funcionando sem exigir número de documento (RN04).
2. Expiração de reserva (RN03) e falha de pagamento seguem liberando ingressos.

**Casos de borda específicos (`logic.md` §5):**

3. **Nenhum gênero cadastrado ainda.** Formulário de novo espetáculo orienta a cadastrar gênero antes, não mostra `<Select>` vazio sem contexto.
4. **Nome duplicado com grafia diferente.** "Comédia" existe; tentar "comedia" ou "COMÉDIA " → recusado com mensagem específica, não erro genérico.
5. **Ícone repetido entre gêneros.** Dois gêneros, nomes diferentes, mesmo ícone → ambos aceitos.
6. **Espetáculo despublicado some do filtro.** Publicar → gênero aparece no filtro. Despublicar → some do filtro público, continua na lista completa do admin. Publicar de novo → reaparece.
7. **Ícone ausente no formulário de criação de gênero.** Submeter sem escolher ícone → recusado, campo aponta o erro.
8. **Migração de dados (ambiente com espetáculos pré-existentes).** Rodar contra cópia de dados com gêneros em texto livre variados — nenhum espetáculo fica sem gênero reconhecido; grafias equivalentes viram um único gênero.

**Teste de resiliência:** derrubar/simular erro no endpoint de listagem de gêneros durante o preenchimento do form de espetáculo → resto do catálogo (busca, detalhe, compra) continua no ar; form mostra erro claro, sem travar a tela de gestão inteira.

**Teste de restart:** iniciar criação de gênero, matar o processo da API no meio, subir de novo → nenhum gênero fica "meio criado"; estado pós-restart reflete só o que foi persistido no Postgres.

## 5. Riscos e pontos de atenção

- **Risco maior: duplicidade sob concorrência real não tem teste automatizado hoje.** A RN exige bloqueio atômico mesmo sob criação simultânea (`logic.md` §3/§4) — só se garante de verdade com constraint `UNIQUE` no Postgres (já prevista em `backend.md`, `uq_genres_name_active`) e só se **verifica** com Postgres real sob concorrência, não com mock. Sem a infra do débito #22, essa garantia fica sem rede de segurança automatizada — o usecase precisa capturar `IntegrityError` e traduzir para `ConflictError` (já especificado em `backend.md`), não confiar só no `exists_by` prévio (que tem janela de corrida).
- **`AggregateRepository` é concreto sobre SQLAlchemy** — não há porta/interface que permita um fake em memória para testar o usecase de criação sem tocar banco; não dá pra contornar o débito #22 com um teste "fake" equivalente ao real.
- **Nota de arquitetura:** a normalização de nome (`normalize_genre_name`) precisa existir num único lugar (domínio) e a migration/aplicação reaproveitarem dali — `backend.md` já isola isso em `domain/aggregates/genre.py`, mas a migration (`0003_catalog_genre.py`) **duplica** a função de propósito (migrations não importam `app.domain`, precisam ficar estáveis). Se a regra de normalização mudar no futuro, a migration antiga não deve ser alterada retroativamente — só o código novo.
- **Estado intermediário sem feedback claro no frontend.** Entre "gênero criado" e "lista do form de espetáculo atualizada" pode haver defasagem de cache se o admin tiver duas abas abertas — vale roteiro manual explícito (não está no `logic.md`, mas é o tipo de coisa que passa batido).
- **Mensagem de erro de duplicidade vazando detalhe técnico (RNF01).** Checar explicitamente no caso de borda 4 que a mensagem exibida é "Esse gênero já existe" — nunca nome de constraint/índice do Postgres.
- **Ícone da paleta sem fonte única de verdade compartilhada.** A paleta (`GENRE_ICON_OPTIONS`) vive só no frontend; o backend valida só forma (`^[a-z][a-z0-9-]*$`). Um ícone inventado/typo passa despercebido — não é bloqueio desta entrega, mas é uma lacuna de validação cross-surface a ter em mente.
- **Ícone de gênero migrado automaticamente** — ver bloqueio abaixo; sem essa decisão, a migration usa um placeholder que pode não fazer sentido visualmente pro catálogo real.

## 6. Passo a passo TBD (QA)

```
git checkout master && git pull && git checkout -b test/<NN>-catalog-genre
git add tests/test_catalog_genre.py && git commit -m "test(catalog): cobrir normalização, duplicidade e ícone obrigatório de gênero"
```

## 7. Bloqueios em aberto

- **Ícone padrão de gênero migrado automaticamente** — mesmo bloqueio de `backend.md` §7: precisa de decisão do PO antes de rodar a migration `0003_catalog_genre.py` contra dado real.
- **`catalog-show-search/integration.md` fica desatualizado** por esta feature (mesmo bloqueio de `backend.md` §7) — os testes de integração cross-surface do item 3 acima que tocam o filtro público (`GET /shows?genre_id=`) dependem dessa spec ser atualizada para não ficar testando contra um contrato documentado que diverge do real.
