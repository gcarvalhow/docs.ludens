# Modelo de Autenticação e Segurança

> Implementado no módulo `identity` (PR #8); revisado em 2026-09-17 para
> refletir o código real.
> Fixa o **modelo de segurança** que a autenticação segue, herdado da
> autenticação humana de um backend privado anterior do mesmo autor. O
> Ludens tem **apenas autenticação humana**; não existe autenticação de
> serviço/worker.

Atende [RF09](../../product/functional.md#rf09-cadastro-e-autenticação-de-usuário)
e o RNF01 (segurança e proteção de dados).

## Dual-token

* **Access token:** JWT HS256, TTL curto (`ACCESS_TOKEN_EXPIRE_MINUTES`, default
  30 min). Header `Authorization: Bearer <token>`. Payload: `sub` (user id),
  `is_admin`, `security_stamp`, `type: "access"`, `exp`, `iat`. O claim `type`
  é validado; um refresh token usado no lugar é rejeitado com `401`.
* **Refresh token:** JWT HS256 usado de forma **opaca**: a API só compara o hash
  SHA-256 do token recebido contra o hash salvo no banco. TTL longo
  (`REFRESH_TOKEN_EXPIRE_DAYS`, default 7 dias). Trafega **só** por cookie
  `HttpOnly; Secure; SameSite=None` em produção (revisado 2026-09-21; ver
  seção "CORS e cookie"), `Path` restrito à rota de autenticação. `HttpOnly`
  bloqueia acesso via JavaScript (proteção contra XSS).

TTLs separados: o access token tem janela curta de exploração se vazar, sem
forçar login a cada 30 minutos.

## Security stamp

Cada comprador tem um `security_stamp` (UUID) incluído no payload do access token
no login. Em cada request autenticado, a API compara o stamp do token com o do
banco; se divergirem, `401`, mesmo o JWT sendo válido e não expirado.

Regenerado no **logout** e na **troca/recuperação de senha**
([RF09](../../product/functional.md#rf09-cadastro-e-autenticação-de-usuário)).
Invalida imediatamente todos os tokens emitidos antes, em qualquer dispositivo.

## Senhas

Armazenadas com **bcrypt** (salt automático, custo configurável). Nunca logadas
nem retornadas por nenhum endpoint (RNF01).

## CPF

O cadastro exige **CPF válido** (dígitos verificadores conferidos), e-mail e
senha. O **e-mail** é o identificador de login. O CPF é dado pessoal: guardado
sem máscara, nunca exposto em log, URL ou mensagem de erro, e usado para o limite
por CPF ([RN01](../../product/overview.md#rn01-limite-de-ingressos-por-cpf)).

## Papéis

Booleano `is_admin`, sem enum/campo `role`. `is_admin = true` é o teatro/PO e
habilita o gerenciamento de catálogo
([RF08](../../product/functional.md#rf08-gerenciar-espetáculos-e-sessões)).
Sem permissão granular. O módulo `identity` exporta as dependências de injeção
(`get_current_user`, `require_admin`) para os demais módulos.

## CORS e cookie

* `ALLOWED_ORIGINS` define as origens permitidas no CORS.
* Cookie do refresh token, fora de dev (`ENVIRONMENT != development`):
  `HttpOnly; Secure; SameSite=None`. Em dev: `HttpOnly; SameSite=Lax`, sem
  `Secure` (permite dev local sem TLS).
* **Revisado 2026-09-21**: `api.ludens` (`*.azurewebsites.net`) e
  `web.ludens` (`*.vercel.app`) são domínios (eTLD+1) diferentes: é tráfego
  **cross-site**, não só cross-origin-mesmo-site. `SameSite=Strict`/`Lax`
  nunca envia o cookie num request cross-site; o CORS corrigido (issue #66)
  não é suficiente por si só, o refresh ficava quebrado silenciosamente até
  esta mudança. `SameSite=None` exige `Secure` (por isso o dev não pode usar
  `None` sem TLS) e abre mão de parte da proteção contra CSRF que
  `SameSite=Strict` dava; trade-off aceito enquanto não existir um domínio
  próprio que unifique front/back no mesmo site (issue de domínio, ver
  `terraform/README.md` do `api.ludens`). O `Path` restrito continua fazendo
  o cookie não acompanhar as rotas de negócio, e é a mitigação de CSRF que
  resta pós-mudança: nenhuma rota fora de `/api/identity/authentication` lê
  esse cookie.
* **Risco residual aceito (achado do `/code-review` em 2026-09-21, não
  corrigido de propósito):** com `SameSite=None`, uma página maliciosa que a
  vítima logada visite pode disparar um `POST /api/identity/authentication/refresh`
  cross-site com o cookie anexado. Isso rotaciona o refresh token da vítima
  sem ela pedir; se colidir com uma rotação legítima do próprio app, o
  mecanismo de detecção de reuso (`AuthUseCase.refresh` → revoga todas as
  sessões quando um token já consumido é reapresentado) pode ser acionado,
  deslogando a vítima de todas as sessões via CSRF. Não é *account takeover*
  (o atacante não ganha um token válido); é disrupção/DoS de sessão.
  Proporcional à escala do projeto (teatro comunitário, sem dado financeiro
  sensível na sessão em si); revisitar com CSRF token (double-submit) ou com
  o domínio próprio (ver acima) se o produto crescer ou esse incômodo virar
  reclamação real de usuário.
