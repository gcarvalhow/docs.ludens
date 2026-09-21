# Configuração: Variáveis de Ambiente

> Proposto: não há código implementado ainda. Fixa o **mecanismo** de configuração (igual a um backend privado anterior do
> mesmo autor) e as
> variáveis **base**. Variáveis específicas de uma funcionalidade (gateway de
> pagamento, SMTP, prazos de regra de negócio) são adicionadas pela spec da
> funcionalidade correspondente.

## Mecanismo

Todas as variáveis são lidas pelo `pydantic-settings` a partir de `.env.local`
(desenvolvimento) ou `.env.production` (produção), via `src/app/config.py`. A
aplicação **falha na inicialização** se qualquer variável obrigatória estiver
ausente. Copie `.env.example` para `.env.local` para começar.

### Classificação de segurança

Cada variável carrega uma classificação:

* `SECRET`: nunca commitar no git; nunca logar; rotacionar periodicamente.
* `SENSITIVE`: contém credenciais; nunca logar; commitar apenas `.env.example`
  com valores em branco.
* `CONFIG`: pode ser versionado em `.env.example` com valores reais de
  desenvolvimento.

## Variáveis base

* **`ENVIRONMENT`** (`development | staging | production`, default
  `development`; `CONFIG`): afeta o flag `Secure` do cookie, o CORS e as
  mensagens de erro expostas.
* **`DATABASE_URL`** (`str`, obrigatória; `SENSITIVE`):
  `postgresql+asyncpg://<user>:<pass>@<host>:<port>/<db>`. O host é o nome do
  contêiner na rede Docker, nunca `localhost`.
* **`JWT_SECRET_KEY`** (`str`, obrigatória; `SECRET`): gerar com
  `openssl rand -hex 32`. Rotacionar invalida todos os access tokens ativos.
* **`ACCESS_TOKEN_EXPIRE_MINUTES`** (`int`, default `30`; `CONFIG`): TTL do
  access token.
* **`REFRESH_TOKEN_EXPIRE_DAYS`** (`int`, default `7`; `CONFIG`): TTL do
  refresh token.
* **`ALLOWED_ORIGINS`** (`list[str]`, default `["http://localhost:3000"]`;
  `CONFIG`): origens permitidas no CORS. Em produção, o domínio real do
  `web.ludens` (Next.js roda em :3000 em dev).
* **`OUTBOX_RELAY_INTERVAL_SECONDS`** (`int`, default `2`; `CONFIG`): cadência
  do polling do relay.

## Variáveis adicionadas por funcionalidade

Cada spec que precisar de configuração externa registra as suas variáveis aqui.

**Notificação** (`notification-transactional-email`,
[RF05](../../product/functional.md#rf05-confirmar-compra-e-emitir-ingresso)/
RF09), decidido em 2026-09-11: **AWS SES**, não SMTP.

* **`EMAIL_BACKEND`** (`ses | console`, default `console`; `CONFIG`):
  `console` só loga (dev, sem conta AWS); trocar pra `ses` em produção.
* **`EMAIL_FROM_ADDRESS`** (`str`; `CONFIG`): precisa ser um
  endereço/domínio verificado na conta SES.
* **`EMAIL_FROM_NAME`** (`str`, default `Ludens`; `CONFIG`): nome de exibição
  do remetente.
* **`AWS_REGION`** (`str`, default `us-east-1`; `CONFIG`): região onde o
  domínio remetente foi verificado no SES.
* **`AWS_ACCESS_KEY_ID`** / **`AWS_SECRET_ACCESS_KEY`** (`str`; `SECRET`):
  **não** entram no `.env` da aplicação nem em `Settings`; o `boto3` resolve
  por IAM role em produção. Só definir manualmente pra testar o adapter SES
  localmente, nunca commitar.
* **`FRONTEND_BASE_URL`** (`str`, default `http://localhost:3000`; `CONFIG`):
  base pra montar links de e-mail (ex.: `/redefinir-senha?token=...`).

**Pagamento** ([RF04](../../product/functional.md#rf04-efetuar-pagamento)):
chave de API e URL base da AbacatePay, segredo de webhook. Pendente de spec.

**Reserva** ([RN01](../../product/overview.md#rn01-limite-de-ingressos-por-cpf),
[RN03](../../product/overview.md#rn03-expiração-da-reserva)): limite de
ingressos por CPF e tempo de expiração da reserva. Pendente de spec.

## Docker Compose (produção)

Variáveis lidas só pelo `docker-compose.Production.yml`, não pela aplicação:
credenciais do contêiner Postgres (`POSTGRES_USER`, `POSTGRES_PASSWORD`,
`POSTGRES_DB`) e o domínio público da API para o proxy reverso e TLS.
