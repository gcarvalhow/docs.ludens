# Convenções de arquitetura e código: Backend

> Ver também [`code-style.md`](code-style.md) para formatação, idioma e regras
> de manutenibilidade; este documento cobre padrão de arquitetura e de
> código que não é formatação.

Estas convenções foram corrigidas contra o código real (`identity-auth`,
mergeado) e centralizadas aqui pra qualquer novo trabalho linkar em vez de
redescobrir.

## Ordenação de imports

Blocos por camada, cada bloco separado por uma linha em branco:

1. `from __future__ import annotations` (só quando o arquivo precisa de
   forward reference; não é obrigatório em todo arquivo; ver nota abaixo).
2. Biblioteca padrão (`uuid`, `datetime`, `re`, ...).
3. Bibliotecas de terceiros (`sqlalchemy`, `fastapi`, `pydantic`, `jwt`, ...).
4. `app.core.*`.
5. `app.modules.<próprio módulo>.*`, agrupado por camada interna
   (`domain` → `application` → `infrastructure` → `api`).

Dentro de cada bloco, ordem crescente por tamanho de linha. Exemplo real
(`identity/application/usecases/user_usecase.py`):

```python
from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession

from app.core.domain import ConflictError, NotFoundError

from app.modules.identity.domain.aggregates import User
from app.modules.identity.domain.value_objects import CPF, Email
from app.modules.identity.application.schemas.request import RegisterRequest
from app.modules.identity.application.schemas.response import TokenResponse, UserResponse
from app.modules.identity.application.usecases.utils.session import issue_session

from app.modules.identity.infrastructure.services import PasswordService, TokenService

from app.modules.identity.infrastructure.repositories import (
    RefreshTokenRepository,
    UserRepository
)
```

Sempre importar do **pacote** (`__init__.py` agregador), nunca do submódulo
direto: `from app.core.domain import DomainError`, não
`from app.core.domain.errors import DomainError`; `from
app.modules.identity.domain.aggregates import User`, não
`....domain.aggregates.user import User`.

`from __future__ import annotations` não é uma regra absoluta e uniforme no
código real: aparece em `domain/aggregates/user.py` e em
`core/domain/errors.py`, mas não em usecases, schemas, entidades ou value
objects. Use-a quando o arquivo precisar (forward reference de tipo que ainda
não foi declarado); não a adicione por hábito em todo arquivo novo.

## Proibição de linha em branco dupla

Nunca duas linhas em branco seguidas em código de aplicação; nem entre
imports, nem antes de uma classe/função, nem dentro de um método. Uma exceção
aceita: migrations do Alembic (geradas por ferramenta, seguem o estilo padrão
do `script.py.mako`).

## Padrão `__init__.py` agregador

Cada subpacote de domínio (`aggregates`, `entities`, `value_objects`,
`enumerations`, `events`) e de infraestrutura (`repositories`, `services`)
reexporta suas classes com `__all__` explícito; um só import cobre o
subpacote inteiro, em vez de um import por arquivo. Exemplo real
(`identity/domain/value_objects/__init__.py`):

```python
from .cpf import CPF
from .email import Email

__all__ = ["CPF", "Email"]
```

`__init__.py` de pacote que não agrega nada (`domain/__init__.py`,
`application/__init__.py`, `api/__init__.py`, etc.) fica vazio: só marcador
de pacote.

## Sem módulo `views.py`

A resposta é montada dentro do próprio usecase: uma função privada de módulo
(`_show_response`, `_session_response`) ou, quando simples, direto no
`return` do método. Não existe camada de apresentação separada
(`application/views.py`); o usecase é quem sabe transformar aggregate em
schema de resposta.

## Usecases sem sufixo de papel

`UserUseCase`, não `UserAdminUseCase`; `ShowUseCase`, não `ShowAdminUseCase`.
Quem restringe uma rota por papel é o **router**, via `Depends(require_admin)`
ou uma checagem inline antes de chamar o usecase, nunca o nome ou a lógica
interna do usecase. Exemplo real (`identity/api/routers/user_router.py`):

```python
@router.get("/{user_id}", response_model=UserResponse)
async def get(user_id: UUID, session: AsyncSession = Depends(get_db), current_user: User = Depends(get_current_user)) -> UserResponse:
    if not current_user.is_admin and current_user.id != user_id:
        raise ForbiddenError("Acesso restrito ao próprio usuário.")

    return await UserUseCase(session).get_by_id(user_id)
```

`UserUseCase.get_by_id` não sabe quem está pedindo nem o quê: só busca e
traduz `None` em `NotFoundError`. A checagem de "próprio ou admin" mora inteira
no router.

## Repositório base mínimo

`BaseRepository`/`AggregateRepository` (`app.core.infrastructure.repositories`)
só têm cinco métodos genéricos:

```text
find_by(field, value)
find_all(*, order_by=None)
find_all_by(*, order_by=None, **filters)
exists_by(field, value)
save(entity)
```

**Não existe** `find_by_id` nem `find_by_id_for_update` na base; mesmo sendo
um lookup comum, não vira método genérico. Cada repositório especializado
implementa o que seu próprio agregado precisar e a base não cobre, mas só
quando a operação realmente não se reduz a um `find_by`/`find_all_by` de campo
único:

* `SessionRepository.find_by_id_for_update`: precisa de `.with_for_update()`,
  algo que `find_by` genérico não expressa.
* `RefreshTokenRepository.deactivate_all_for_user`: muda várias linhas numa
  operação, não é um lookup.

Um método que só embrulha `find_by("campo", valor)` sem fazer nada a mais
**não** vira método próprio; chame `find_by` direto no usecase.

## Naming de atributo de repositório/serviço

Por extenso, não abreviado: `self._user_repository`, não `self._user_repo`.
Quando o nome completo da entidade tornaria o atributo redundante dado o
contexto do usecase, usa-se a forma curta natural, não uma abreviação
arbitrária: `self._refresh_repository` (para `RefreshTokenRepository`, dentro
de um usecase que já lida só com sessão), não `self._refresh_token_repo` nem
`self._refresh_token_repository`.

## Porta com múltiplos adapters reais (Protocol + factory)

> **Nota (2026-09-17):** primeiro precedente real deste padrão:
> `notification/infrastructure/services/email_service.py`.
>
> **Atualização (2026-09-18):** a exceção própria do módulo (`EmailServiceError`)
> deixou de morar no mesmo arquivo do `Protocol`; ela foi movida para
> `notification/shared/errors.py`, com a introdução da camada `shared/` por
> módulo (ver seção "Camada `shared/` por módulo" abaixo). O restante do
> padrão descrito aqui (Protocol sozinho, adapters em arquivos próprios,
> factory com `@lru_cache`) continua válido sem mudança.

O caso comum em `infrastructure/services/` é o de `references/05` da skill
`backend-architecture` (`PaymentGateway`, `TokenService`, `PasswordService`):
uma classe concreta única, a interface e a implementação são a mesma
classe, porque só existe um adapter real. Isso muda quando o serviço
**precisa mesmo** de mais de um adapter real (não hipotético); o caso
canônico é dev vs. produção de uma dependência externa trocável por
configuração (RNF06), como `EmailService`.

Quando isso acontece:

* A porta vira um `typing.Protocol` sozinho num arquivo (`email_service.py`),
  sem implementação nenhuma, só o contrato (com um docstring curto
  explicando que é interface estrutural, já que um arquivo só com um
  `Protocol` pode parecer "vazio" ou incompleto sem essa explicação; não é).
  A exceção própria do módulo (`EmailServiceError`) mora em `shared/errors.py`
  do módulo, não no mesmo arquivo do `Protocol` (ver seção "Camada `shared/`
  por módulo" abaixo).
* Cada adapter vira um arquivo próprio (`acs_email_service.py`,
  `smtp_email_service.py`), implementando o `Protocol` estruturalmente
  (sem herança explícita; é assim que `Protocol` funciona).
* Um `factory.py` com uma função `@lru_cache` (`get_email_service`,
  `get_<serviço>_service`) escolhe o adapter certo por
  `settings.<nome>_backend`, igual a qualquer outro ponto de configuração
  do projeto.
* A porta **não** vive em `domain/`: o projeto não usa hexagonal clássico
  com porta em `domain/`/adapter em `infrastructure/`. A porta fica em
  `infrastructure/services/` mesmo, e quem consome (usecase, handler de
  outbox) importa direto do pacote de `services/`, sem camada de
  abstração intermediária em `domain/`.
* A exceção própria do módulo **nunca** estende `DomainError`
  (`core/domain/errors.py`): ela representa falha de infraestrutura
  (rede, serviço externo fora do ar), não violação de regra de negócio, e
  não deve ser mapeada pelo `exception_handler(DomainError)` genérico da
  API. Também não vive em `core/shared/` (que não tem nenhuma classe de
  erro, só utilitário de formatação); vive em `shared/errors.py` do
  próprio módulo (ver seção seguinte).

## Camada `shared/` por módulo

> **Nota (2026-09-18):** convenção nova, introduzida nesta data em
> `notification` (`api.ludens`#38) e `identity` (`api.ludens`#40). Antes
> desta data, `shared/` só existia em `core/` (compartilhado entre módulos);
> a versão anterior desta seção dizia explicitamente que uma camada
> `shared/` por módulo "não existe como convenção neste projeto". Essa frase
> ficou obsoleta e foi substituída pelo texto abaixo. `catalog` foi
> considerado no mesmo ciclo (`api.ludens`#42) mas **não** ganhou `shared/`;
> ver a ressalva no fim desta seção sobre por quê.

Cada módulo pode ter uma pasta `shared/`, paralela a `api/`, `application/`,
`domain/` e `infrastructure/`, para o que é reaproveitado **entre camadas**
dentro do mesmo módulo (ex.: usecase + router, ou infra + handler de outbox),
o equivalente, em escopo de módulo, ao que `core/shared/` já é em escopo de
projeto inteiro (que mistura `format_validation_errors`, `Page`/
`PaginationParams`, `cents_from_reais`, `IdentifierResponse` sob o mesmo
critério: "reaproveitado por mais de uma camada").

Vai para `modules/<módulo>/shared/`:

* Exceção própria de infraestrutura do módulo que não é `DomainError`
  (`notification/shared/errors.py::EmailServiceError`).
* Conteúdo de apresentação puro sem I/O, usado por mais de uma camada do
  módulo (`notification/shared/templates.py`, usado pela infra de e-mail e
  por `handlers.py`, camadas diferentes).
* Helper reaproveitado por mais de uma camada do mesmo módulo
  (`identity/shared/session.py::issue_session`, chamado pela camada de
  `application/` a partir de dois usecases; `identity/shared/cookies.py`,
  usado pela camada `api/` em dois routers).

**Não** vai para `shared/` de módulo:

* Regra de negócio pura sobre um agregado: isso é método do próprio agregado
  em `domain/aggregates/` (ex.: `Session.available_count()`, `Session.status_at()`,
  `Show.is_published`), não uma função solta em `shared/` nem um guard
  separado que só chama esse método e levanta `NotFoundError`; o usecase
  chama o predicado do agregado direto e faz o `raise` inline (visto em
  `catalog`, `api.ludens`#42: um `guards.py` só para isso foi descartado por
  não agregar nada além do que o `if` já diz sozinho).
* Código que toca banco de dados (SQL/SQLAlchemy), mesmo quando não é uma
  operação de agregado (ex.: uma leitura agregada/read-model tipo
  `catalog/infrastructure/queries/show_search_query.py::ShowSearchQuery`),
  continua em `infrastructure/`, nunca em `shared/`.
* Mapper que só traduz agregado/read-model → DTO de resposta (estilo
  AutoMapper), mesmo quando reaproveitado por mais de um usecase: isso é
  `application/mappers/` (ver seção seguinte), não `shared/`, porque nunca
  cruza pra fora da camada `application/`.
* Utilitário puro usado por um único usecase: fica em
  `application/usecases/utils/` (ou privado no próprio usecase), não em
  `shared/`. `shared/` de módulo é para reuso **entre camadas**, não faz
  sentido pra algo que um único usecase consome.
* Algo usado por uma única camada com um único consumidor: não crie
  `shared/` preventivamente (mesmo espírito do ADR 003, contrato mínimo sem
  abstração antecipada). Foi exatamente por isso que `catalog` (`api.ludens`#42)
  não ganhou `shared/`: seus mappers (`show_mapper.py`, `session_mapper.py`)
  ficam inteiros dentro de `application/` (mappers reaproveitados por
  usecases, mas nunca por um router ou pela infra), e seus dois utilitários
  restantes (`genre_slug.py`, `search_floor.py`) têm um único usecase
  consumidor; nenhum dos dois casos cruza camada.

## Mappers de apresentação (`application/mappers/`)

> **Nota (2026-09-18):** primeiro precedente: `catalog/application/mappers/`
> (`api.ludens`#42): `show_mapper.py` (`card_response`, a partir do read-model
> `ShowCardRow`) e `session_mapper.py` (`session_response`, a partir do
> agregado `Session`).

Quando uma função só traduz um agregado ou um read-model de
`infrastructure/queries/` para um DTO de resposta (`application/schemas/
response.py`), o equivalente a um `Profile`/mapper do AutoMapper (.NET), ela
vive em `application/mappers/`, um arquivo por família de DTO
(`<entidade>_mapper.py`). Continua sendo `application/`, não `shared/`,
mesmo quando mais de um usecase do módulo chama o mesmo mapper: o critério
de `shared/` de módulo é cruzar camada (`application` ↔ `api`, ou
`infrastructure` ↔ handler de outbox), não múltiplos consumidores dentro da
mesma camada.

## Política de comentário

Comentário só para referenciar regra de negócio (`# RF08: ...`, `# RN04 —
...`), colocado inline acima da linha relevante. Nunca um comentário
explicativo solto que só parafraseia o que o código já diz.

## Contrato REST

`pydantic.BaseModel` puro, snake_case, sem `CamelModel`, sem
`alias_generator`, sem camelCase em corpo de request/response. Exceção
documentada caso a caso, nunca como padrão: `catalog-show-search` usa
`fromDate` como nome de query param porque é URL pública, não JSON body, e o
`integration.md` já fixava essa grafia; decisão registrada na spec da
feature, não uma regra geral.

Erros de domínio: `DomainError` e subclasses (`ConflictError`, `AuthError`,
`ForbiddenError`, `GoneError`, `NotFoundError`) carregam só `message`, sem
`status_code`/`field` na classe. O mapeamento pra HTTP é uma lista central em
`main.py` (`_DOMAIN_ERROR_STATUS`), por subclasse, com fallback `422`. O
envelope de erro de domínio é `{"detail": "mensagem"}` (string), **diferente**
do envelope de validação Pydantic, `{"detail": [{"field", "message"}]}` (lista,
via `format_validation_errors`). Não confundir os dois formatos ao documentar
contrato de erro numa spec nova.

## `PUT` para várias propriedades, `PATCH` para uma só

> **Nota de revisão (2026-09-17):** regra original ("só `PUT`, nunca `PATCH`")
> revista ao desenhar os endpoints de perfil de `identity-user-management`
> (Workstream B do plano técnico aprovado em `/plan`, `api.ludens`). A decisão
> de 2026-09-11 continua valendo para o caso que ela cobria (edição de recurso
> completo, como `Show`/`Session`); a mudança é granular, não uma reversão.

* **`PUT`** quando o endpoint atualiza **múltiplas propriedades** do recurso
  de uma vez, com corpo completo: um único schema serve criação e edição
  (`ShowRequest`, não `CreateShowRequest`/`UpdateShowRequest` com campos
  opcionais). Sem edição parcial de um recurso multi-campo.
* **`PATCH`** quando o endpoint atualiza **uma única propriedade** do recurso
  (ex.: `PATCH /identity/users` só edita `name`). Não é edição parcial genérica
  com campos todos opcionais; o schema de corpo tem exatamente o campo que
  aquela rota edita, obrigatório.

Nenhum endpoint usa `/me` ou `/admin` como segmento de URL; a permissão
(self, admin, self-or-admin) é resolvida por `Depends()`/checagem inline
dentro do handler, nunca por namespace de rota.

### Confirmação por link de e-mail: token via query param

Endpoints que confirmam uma ação a partir de um link de e-mail (troca de
e-mail, exclusão de conta) recebem o token pela **query string**
(`?token=...`), não no corpo; quem abre o link é o navegador seguindo uma URL,
não um cliente montando um JSON. Esses endpoints não são autenticados: a
posse do token (opaco, hasheado no banco, TTL de 1h) é a própria credencial da
ação. Decisão de 2026-09-17; `POST /identity/password/reset`
(`identity-auth`, anterior a esta decisão) continua recebendo o token no
corpo; não foi alterado por esta revisão, e não é o padrão a copiar para
endpoints novos.
