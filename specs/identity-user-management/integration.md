---
status: canônico
spec: identity-user-management
updated_at: 2026-09-21
responsavel: (auditoria retroativa — ver nota abaixo)
---

# Integration Contract — Gestão de conta de usuário

> **Nota de auditoria (2026-09-21):** contrato escrito depois do código já
> estar mergeado (mesma situação de `backend.md` desta pasta — ver a nota lá).
> `status: canônico` porque reflete o código real hoje, não uma proposta —
> mas ver "Lacunas / decisões em aberto" no fim: há pontos do contrato que
> **deveriam** mudar pra bater com `logic.md` e ainda não mudaram.

**Status:** canônico (reflete o código real já mergeado — ver `backend.md`).
**Módulo backend:** `identity` (rotas) + `notification` (e-mails, assíncrono
via outbox).

## Rotas

| Método | Caminho | Auth | Sucesso | Descrição |
| --- | --- | --- | --- | --- |
| PATCH | `/identity/users` | Bearer | 200 | Atualiza o próprio nome |
| POST | `/identity/users/email/change` | Bearer | 202 | Pede troca de e-mail; link de confirmação vai pro **e-mail atual** |
| PATCH | `/identity/users/email/change?token=...` | pública (token na query) | 204 | Confirma a troca; e-mail muda, todas as sessões caem |
| POST | `/identity/users/deletion` | Bearer | 202 | Pede o encerramento da própria conta |
| DELETE | `/identity/users/deletion?token=...` | pública (token na query) | 204 | Confirma o encerramento; conta encerrada, todas as sessões caem |
| GET | `/identity/users` | Bearer + admin (`require_admin`) | 200 | Lista contas ativas, paginada |
| GET | `/identity/users/{id}` | Bearer | 200 | Detalhe de uma conta — próprio usuário só o próprio id, admin qualquer id |

`POST /identity/users` (cadastro) continua documentado em
[`identity-auth/integration.md`](../identity-auth/integration.md) — não
repetido aqui, mesmo módulo/arquivo de router.

## Request / Response (shapes reais)

Contrato inteiro em **snake_case**, sem exceção — mesma convenção de
`identity-auth`.

* `PATCH /identity/users` → Bearer, body `{ name }` (1–120 caracteres) →
  200 `{ id, name, email, cpf, is_admin }`. `cpf` não é aceito no corpo,
  campo inexistente no schema, não é "ignorado", é rejeitado como campo
  desconhecido só se o cliente Pydantic estiver em modo estrito (por padrão
  o FastAPI ignora campo extra), na prática o front simplesmente não deve
  mandar `cpf`.
* `POST /identity/users/email/change` → Bearer, body `{ new_email }`
  (≤254 caracteres, formato validado no backend) → 202
  `{ "message": "Enviamos um link de confirmação para <e-mail ATUAL da conta>." }`.
  **O e-mail no corpo do pedido não aparece na mensagem de sucesso, quem
  aparece é o endereço atual**, exatamente o endereço pra onde o link foi.
* `PATCH /identity/users/email/change?token=<token>` → sem body, token na
  **query string** → 204. Depois disso, qualquer chamada autenticada com o
  access token antigo passa a devolver 401 (ver "Semântica de estado de
  negócio").
* `POST /identity/users/deletion` → Bearer, sem body → 202
  `{ "message": "Enviamos um link de confirmação para <e-mail da conta>." }`.
* `DELETE /identity/users/deletion?token=<token>` → sem body, token na
  **query string** → 204.
* `GET /identity/users?page=1&size=20` → Bearer (admin) → 200
  `{ items: [ { id, name, email, cpf, is_admin } ], page, size, total }`.
  `page` ≥ 1 (padrão 1); `size` 1–50 (padrão 20). **Ver "Lacunas", `cpf`
  não deveria estar aqui, `created_at` deveria.**
* `GET /identity/users/{id}` → Bearer → 200
  `{ id, name, email, cpf, is_admin }`. Mesma observação de `cpf` quando
  quem pede é admin olhando a conta de **outra** pessoa.

## Erros esperados (linguagem de negócio)

| Rota | Status | Quando | Mensagem |
| --- | --- | --- | --- |
| `PATCH /identity/users` | 422 | `name` vazio ou > 120 caracteres | validação de forma padrão (`detail: [{field, message}]`) |
| `POST /identity/users/email/change` | 409 | novo e-mail já em uso | "Este e-mail já está em uso." |
| `POST /identity/users/email/change` | 422 | e-mail em formato inválido | "E-mail inválido." |
| `PATCH /identity/users/email/change` | 410 | token inexistente, expirado ou já usado | "Este link não é mais válido, solicite um novo." |
| `PATCH /identity/users/email/change` | 409 | o e-mail pretendido passou a pertencer a outra conta entre o pedido e a confirmação | "Este e-mail já está em uso." |
| `POST /identity/users/deletion` | 409 | quem pede é o único administrador ativo | "Você é a única pessoa administradora da plataforma. Convide outra pessoa administradora antes de encerrar sua conta." |
| `DELETE /identity/users/deletion` | 410 | token inexistente, expirado ou já usado | "Este link não é mais válido, solicite um novo." |
| `DELETE /identity/users/deletion` | 409 | quem confirma virou o único administrador ativo entre o pedido e a confirmação | mesma mensagem acima |
| `GET /identity/users` | 403 | quem pede não é admin | "Acesso restrito a administradores." |
| `GET /identity/users/{id}` | 403 | usuário comum pedindo id de outra pessoa | "Acesso restrito ao próprio usuário." |
| qualquer rota Bearer | 401 | sem token, token inválido, ou `security_stamp` divergente (sessão derrubada por troca de e-mail/exclusão/troca de senha em outra aba) | "Sessão inválida ou expirada." / "Não autenticado." |

Envelope de erro: os mesmos dois formatos já documentados em
`identity-auth/integration.md`, lista `{field, message}` para 422 de
validação Pydantic, `{"detail": "mensagem"}` para erro de domínio
(409/401/403/404/410). Nenhuma rota nova desta feature introduz um terceiro
formato.

## Semântica de estado de negócio

* **Conta:** `ativa` (implícito: `is_active = true` na tabela `users`) ou
  `encerrada` (`is_active = false`). Não existe estado intermediário, o
  pedido de exclusão **não** muda o estado da conta, só cria um token
  pendente numa tabela separada (`account_deletion_tokens`); a conta segue
  `ativa` até o link ser aberto.
* **Pedido de troca de e-mail / pedido de exclusão de conta:** modelados
  implicitamente por uma linha em `email_change_tokens`/
  `account_deletion_tokens` com `used_at`. Não há coluna de status
  (`pending`/`confirmed`/`expired`), "pendente" é `used_at IS NULL AND
  expires_at > now()`, "expirado" é `used_at IS NULL AND expires_at <= now()`,
  "confirmado" é `used_at IS NOT NULL`. Um novo pedido marca `used_at = now()`
  em qualquer token anterior ainda pendente do mesmo usuário
  (`invalidate_all_for_user`), efetivamente "confirmado" e "invalidado por
  um pedido novo" ficam indistinguíveis olhando só a linha (os dois têm
  `used_at` preenchido). Isso é suficiente pro comportamento de produto
  (o link velho sempre para de valer), mas não dá pra reconstruir depois,
  via banco, "esse pedido foi cancelado por um novo pedido ou foi de fato
  confirmado" sem cruzar com a mudança de `email`/`is_active` do `User`.
* **Sessão:** inalterada em relação a `identity-auth`, access token JWT 30
  min, refresh opaco 7 dias, `security_stamp` como mecanismo de invalidação
  em massa. Confirmar troca de e-mail e confirmar exclusão de conta são,
  junto com logout/troca de senha/redefinição de senha, os cinco pontos do
  sistema que regeneram o `security_stamp` e desativam todo `refresh_token`
  do usuário.

## Impacto de UX

* **As duas mensagens de sucesso (202) já vêm com o e-mail de destino
  interpolado** (`"Enviamos um link de confirmação para fulano@x.com."`),
  o frontend não precisa (nem deve) montar essa frase sozinho; basta exibir
  `message` direto. Isso cobre a exigência de `logic.md` de a tela dizer
  claramente que o link foi para o e-mail **atual**, não o novo, sem o
  frontend precisar saber qual é o e-mail atual por fora.
* **As rotas de confirmação não são `GET`.** O link do e-mail
  (`{frontend_base_url}/confirmar-troca-de-email?token=...` e
  `.../confirmar-exclusao-de-conta?token=...`) aponta pra uma página do
  **frontend**, não direto pra API, essa página precisa ler `token` da
  query string e disparar, ela mesma, um `PATCH`/`DELETE` pra
  `/identity/users/email/change`/`/identity/users/deletion`. Um simples link
  clicável direto pra API não funciona (métodos errados). Isso é diferente
  do padrão que `identity-auth` documentou pra redefinição de senha (que
  também precisa de página própria, mas por ser um formulário com senha
  nova, aqui a página não tem formulário nenhum, só dispara a chamada e
  mostra o resultado).
* **Confirmar troca de e-mail ou exclusão derruba a sessão atual também.**
  Depois de um 204 em qualquer uma das duas rotas de confirmação, qualquer
  chamada autenticada seguinte (mesmo na mesma aba que só pediu, não abriu o
  link) volta 401, o frontend deve tratar isso exatamente como já trata
  401 em `identity-auth` (tenta refresh uma vez, falha, limpa estado, manda
  pra `/login`).
* **A tela de "meu perfil" não deve renderizar um campo de edição para
  CPF**, o backend nunca aceita esse campo em `PATCH /identity/users`
  (`UpdateProfileRequest` não tem `cpf`); CPF só aparece como leitura, vindo
  do `GET /identity/users/{id}` do próprio usuário.
* **A tela de listagem administrativa não deve renderizar/expor a coluna
  `cpf`** que o backend devolve hoje em `GET /identity/users`, ver
  "Lacunas" abaixo. Esconder só no frontend não é a correção completa (o
  dado já trafegou na resposta), mas é a mitigação imediata possível sem
  mexer no backend.
* **A tela de listagem administrativa não tem hoje um campo `created_at`
  pra mostrar "data de criação da conta"** (exigido por `logic.md` §3),
  não existe no shape de resposta atual. Não há mitigação de frontend
  possível pra isso; depende de mudança no backend (ver "Lacunas").

## Lacunas / decisões em aberto

* **Bloqueante para publicar a listagem administrativa como o produto
  descreve:** `GET /identity/users` e `GET /identity/users/{id}` (quando
  admin olha conta de terceiro) devolvem `cpf` e não devolvem `created_at`,
  o oposto do que `logic.md` §3 fecha ("nome, e-mail, papel e data de
  criação... Nunca CPF"). Ver `backend.md` §4 item 1 para a correção
  sugerida (`AdminUserResponse`/`UserSummaryResponse` dedicado).
* **Mitigação de digitação do novo e-mail incompleta:** `POST
  /identity/users/email/change` aceita só `{ new_email }`; `logic.md`
  espera dois campos (`new_email` + confirmação) no pedido. O frontend pode
  compensar com um segundo campo de UI que só é enviado como um único
  `new_email` pro backend, mas isso não dá nenhuma garantia real, o
  backend nunca valida a igualdade dos dois. Ver `backend.md` §4 item 2.
* **Bloqueio de exclusão de conta por reserva aberta/pagamento em
  processamento não existe**, hoje `POST /identity/users/deletion` só
  bloqueia pelo critério de "último administrador". Não há reserva nem
  pagamento no backend ainda (`booking`/`payment` não implementados). Não é
  algo que o frontend deveria tentar contornar, é uma lacuna de produto
  que só fecha quando esses módulos existirem. Ver `backend.md` §4 item 3.
* **Token de confirmação na query string, não no corpo:** diferente do
  padrão de `identity-auth` (`reset_password`, token no corpo de um
  `POST`). Ver `backend.md` §4 item 4 para o racional e o risco
  (RNF01, token não deveria aparecer em URL).
* Se uma convenção global de prefixo/versionamento/erro/paginação for
  adotada depois (`docs.ludens/backend/integration/_template.md`), ela
  substitui o que está aqui, mesma ressalva já registrada em
  `identity-auth/integration.md`.
