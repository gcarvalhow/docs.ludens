# Design 001 — Padrão Outbox in-process para efeitos colaterais

> **Status:** proposto · **Última revisão:** 2026-08-28
> Adaptação do padrão Outbox do backend da DOM Med. **Diferença central:** o
> Ludens não usa broker de mensagens (RabbitMQ) — o relay chama handlers Python
> no próprio processo da API.

## Contexto

Exemplo motivador (a formalizar na spec de pagamento/notificação): quando um
pagamento é confirmado, duas coisas precisam acontecer — persistir o pedido e
emitir o ingresso, e disparar o e-mail de confirmação com o ingresso
([RF05](../../requirements/functional.md#rf05--confirmar-compra-e-emitir-ingresso)).
São responsabilidades distintas — o banco e o serviço de e-mail (SMTP). Se o
`commit` no banco passar e o envio de e-mail falhar logo depois (ou o contrário),
o sistema fica inconsistente.

Chamar o serviço de e-mail direto dentro do caso de uso também acopla o tempo de
resposta da compra à latência do SMTP e faz uma falha de e-mail derrubar a
transação — o que contraria RF05 ("falha no envio do e-mail não invalida a
compra").

A solução clássica para coordenar dois sistemas seria uma transação distribuída
(two-phase commit), mecanismo caro e difícil de operar.

## Decisão

O caso de uso **nunca** chama um efeito colateral externo (e-mail, estorno,
webhook) diretamente. Todo efeito que precisa acontecer "depois" é persistido
como uma linha na tabela `events`, com `dispatched_at = NULL`, **na mesma
transação** que muda o estado de domínio.

Um processo em background — o relay do outbox — roda como `asyncio.Task` no
`lifespan` da aplicação, faz *polling* da tabela `events` a cada
`OUTBOX_RELAY_INTERVAL_SECONDS` (default **2s**), em lotes, e para cada evento
pendente chama os **handlers registrados in-process** para aquele `event_type`.
Após o handler retornar com sucesso, marca `dispatched_at = now()`.

**Não há broker externo.** Os handlers são funções Python registradas num
dicionário `event_type → [handler]`, executadas no mesmo processo da API.

## Consequências

- **Consistência garantida**: o efeito nunca é perdido — está no banco antes de
  qualquer tentativa de execução.
- **Sem transação distribuída**: banco e efeito colateral são coordenados sem
  mecanismo complexo.
- **Execução at-least-once**: se o relay executar o handler mas cair antes de
  marcar `dispatched_at`, o handler roda de novo no próximo ciclo — **handlers
  precisam ser idempotentes** (ex.: registrar qual `event_id` já foi processado e
  não repetir o efeito).
- **Latência de até ~2s** entre a mudança de estado e o efeito — aceitável para
  e-mail de confirmação (RF05 pede "até 5 minutos").
- **Sem isolamento de falha entre relay e API**: se o processo da API cair, o
  relay para junto; os eventos ficam no banco e são processados quando a API
  voltar. Aceitável no escopo atual. Migrar o relay para um processo separado, ou
  plugar um broker, é uma evolução possível **sem mudar o contrato** — a tabela
  `events` já é o ponto de extensão.
- **Não é Event Sourcing**: o estado de verdade continua nas tabelas de domínio
  de cada módulo. A tabela `events` é apenas a fila de saída de efeitos; nenhum
  agregado é reconstruído a partir dela.
