# Modelo de Autenticação e Segurança

> **Status:** proposto — **não há código implementado** · **Última revisão:** 2026-08-28
> Fixa o **modelo de segurança** que a autenticação vai seguir, herdado da
> autenticação humana do backend da DOM Med. O contrato de rotas
> (`/identity/...`) entra com a spec do módulo `identity`, não aqui. O Ludens tem
> **apenas autenticação humana** — não existe autenticação de serviço/worker.

Atende [RF09](../../requirements/functional.md#rf09--cadastro-e-autenticação-de-usuário)
e o RNF01 (segurança e proteção de dados).

## Dual-token

- **Access token** — JWT HS256, TTL curto (`ACCESS_TOKEN_EXPIRE_MINUTES`, default
  30 min). Header `Authorization: Bearer <token>`. Payload: `sub` (buyer id),
  `role`, `security_stamp`, `type: "access"`, `exp`, `iat`. O claim `type` é
  validado — um refresh token usado no lugar é rejeitado com `401`.
- **Refresh token** — JWT HS256 usado de forma **opaca**: a API só compara o hash
  SHA-256 do token recebido contra o hash salvo no banco. TTL longo
  (`REFRESH_TOKEN_EXPIRE_DAYS`, default 7 dias). Trafega **só** por cookie
  `HttpOnly; Secure; SameSite=Strict`, com `Path` restrito à rota de
  autenticação. `HttpOnly` bloqueia acesso via JavaScript (proteção contra XSS).

TTLs separados: o access token tem janela curta de exploração se vazar, sem
forçar login a cada 30 minutos.

## Security stamp

Cada comprador tem um `security_stamp` (UUID) incluído no payload do access token
no login. Em cada request autenticado, a API compara o stamp do token com o do
banco — se divergirem, `401`, mesmo o JWT sendo válido e não expirado.

Regenerado no **logout** e na **troca/recuperação de senha**
([RF09](../../requirements/functional.md#rf09--cadastro-e-autenticação-de-usuário)) —
invalida imediatamente todos os tokens emitidos antes, em qualquer dispositivo.

## Senhas

Armazenadas com **bcrypt** (salt automático, custo configurável). Nunca logadas
nem retornadas por nenhum endpoint (RNF01).

## CPF

O cadastro exige **CPF válido** (dígitos verificadores conferidos), e-mail e
senha. O **e-mail** é o identificador de login. O CPF é dado pessoal: guardado
sem máscara, nunca exposto em log, URL ou mensagem de erro, e usado para o limite
por CPF ([RN01](../../requirements/business-rules.md#rn01--limite-de-ingressos-por-cpf)).

## Papéis

`role` binário: `BUYER | ADMIN`. `ADMIN` é o teatro/PO e habilita o
gerenciamento de catálogo
([RF08](../../requirements/functional.md#rf08--gerenciar-espetáculos-e-sessões)).
Sem permissão granular. O módulo `identity` exporta as dependências de injeção
(`get_current_buyer`, `require_admin`) para os demais módulos.

## CORS e cookie

- `ALLOWED_ORIGINS` define as origens permitidas no CORS.
- Cookie do refresh token: `HttpOnly; Secure; SameSite=Strict`. `Secure` é
  desligado automaticamente quando `ENVIRONMENT=development` para permitir dev
  local sem TLS. `SameSite=Strict` protege contra CSRF; o `Path` restrito faz o
  cookie não acompanhar as rotas de negócio.
