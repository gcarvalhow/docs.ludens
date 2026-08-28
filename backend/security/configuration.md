# Configuração — Variáveis de Ambiente

> **Status:** proposto — **não há código implementado** · **Última revisão:** 2026-08-28
> Fixa o **mecanismo** de configuração (igual ao backend da DOM Med) e as
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

- `SECRET` — nunca commitar no git; nunca logar; rotacionar periodicamente.
- `SENSITIVE` — contém credenciais; nunca logar; commitar apenas `.env.example`
  com valores em branco.
- `CONFIG` — pode ser versionado em `.env.example` com valores reais de
  desenvolvimento.

## Variáveis base

| Variável | Tipo | Classificação | Notas |
| --- | --- | --- | --- |
| `ENVIRONMENT` | `development \| staging \| production` (default `development`) | `CONFIG` | Afeta o flag `Secure` do cookie, o CORS e as mensagens de erro expostas. |
| `DATABASE_URL` | `str` — obrigatória | `SENSITIVE` | `postgresql+asyncpg://<user>:<pass>@<host>:<port>/<db>`. O host é o nome do contêiner na rede Docker, nunca `localhost`. |
| `JWT_SECRET_KEY` | `str` — obrigatória | `SECRET` | Gerar com `openssl rand -hex 32`. Rotacionar invalida todos os access tokens ativos. |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `int` (default `30`) | `CONFIG` | TTL do access token. |
| `REFRESH_TOKEN_EXPIRE_DAYS` | `int` (default `7`) | `CONFIG` | TTL do refresh token. |
| `ALLOWED_ORIGINS` | `list[str]` (default `["http://localhost:5173"]`) | `CONFIG` | Origens permitidas no CORS. Em produção, o domínio real do `web.ludens`. |
| `OUTBOX_RELAY_INTERVAL_SECONDS` | `int` (default `2`) | `CONFIG` | Cadência do polling do relay ([ADR 001](../design/001-outbox-in-process.md)). |

## Variáveis adicionadas por funcionalidade

Cada spec que precisar de configuração externa registra as suas variáveis aqui.
Previstas pelos requisitos já aprovados (a formalizar na spec correspondente):

- **Pagamento** ([RF04](../../requirements/functional.md#rf04--efetuar-pagamento)) —
  chave de API e URL base da AbacatePay, segredo de webhook.
- **Notificação** ([RF05](../../requirements/functional.md#rf05--confirmar-compra-e-emitir-ingresso)) —
  conexão SMTP e endereço remetente.
- **Reserva** ([RN01](../../requirements/business-rules.md#rn01--limite-de-ingressos-por-cpf),
  [RN03](../../requirements/business-rules.md#rn03--expiração-da-reserva)) —
  limite de ingressos por CPF e tempo de expiração da reserva.

## Docker Compose (produção)

Variáveis lidas só pelo `docker-compose.Production.yml`, não pela aplicação:
credenciais do contêiner Postgres (`POSTGRES_USER`, `POSTGRES_PASSWORD`,
`POSTGRES_DB`) e o domínio público da API para o proxy reverso e TLS.
