---
status: reviewed
spec: booking-reservation
created_at: 2026-09-01
reviewed_at: 2026-09-01
---

# Reserva temporária de ingressos — Lógica de Negócio

## 1. Fluxo por perfil

### Comprador

1. Na página da sessão, escolhe quantidade (1 a 6) e tipo (inteira/meia).
2. Clica em reservar. O sistema, numa única operação atômica:
   - confere que a sessão existe e está à venda;
   - confere o limite por CPF (RN01): ingressos já confirmados do CPF nessa
     sessão + reservas abertas do CPF nessa sessão + a quantidade pedida ≤ 6;
   - confere a disponibilidade (RN05): confirmados + reservas abertas não
     vencidas + quantidade pedida ≤ capacidade;
   - se tudo passa, cria a reserva com prazo de 15 minutos e leva o comprador ao
     checkout (`/checkout/:reservationId`), mostrando o contador.
3. Se falhar: mensagem específica ("Não há ingressos suficientes para esta
   sessão", "Você já atingiu o limite de 6 ingressos por sessão", "Esta sessão
   não está mais à venda").
4. No checkout, enquanto a reserva está aberta, o comprador pode **cancelar** —
   os ingressos voltam na hora e ele volta à página da sessão.
5. Se o contador zerar sem pagamento, a reserva expira: a tela informa "a
   reserva expirou, os ingressos voltaram para a sessão" e oferece reservar de
   novo.
6. Uma reserva já paga (confirmada) não pode ser cancelada por aqui — vira o
   fluxo de `payment-cancellation-refund` (RF07).

### Visitante (não autenticado)

Não reserva. Ao tentar, é levado ao login guardando a intenção; ao voltar
autenticado, retoma a seleção.

### Admin

Não reserva pela interface de compra. (Cancelar uma sessão com reservas é
`catalog-admin-management` + RF07.)

## 2. Estados e transições

### Reserva

**Estados:** aberta · confirmada · expirada · cancelada.

- **inexistente → aberta:** reserva criada com sucesso (prazo = agora + 15 min).
- **aberta → confirmada:** pagamento aprovado (evento vindo de `payment`).
- **aberta → expirada:** o prazo passou sem confirmação; o processo de
  expiração devolve os ingressos.
- **aberta → cancelada:** o comprador cancela, ou o pagamento falha/é cancelado
  (RF04), ou a sessão é cancelada pelo admin.
- `confirmada`, `expirada` e `cancelada` são terminais. Só reservas `aberta`
  seguram disponibilidade.

## 3. Regras de negócio

- Quantidade entre 1 e 6 por reserva (forma) → recusa fora disso.
- Limite por CPF: confirmados(CPF, sessão) + abertos(CPF, sessão) + pedido ≤ 6
  → senão recusa (RN01).
- Disponibilidade: confirmados(sessão) + abertos_não_vencidos(sessão) + pedido
  ≤ capacidade → senão recusa (RN05).
- A verificação de disponibilidade acontece com a linha da sessão travada — duas
  reservas concorrentes são serializadas, nunca somam além da capacidade (RN05).
- Prazo da reserva = 15 minutos (RN03), a partir da criação.
- Reserva `aberta` vencida deixa de contar para disponibilidade **e** é
  transicionada para `expirada` pelo processo de expiração (não fica "aberta
  vencida" para sempre).
- Só o dono da reserva (ou o sistema) muda o estado dela. Um comprador não
  cancela a reserva de outro.
- Sessão não "à venda" (não publicada, cancelada, ou já começou) → nenhuma
  reserva nova.

## 4. Pontos de integração

```text
Frontend precisa saber:
  - O reservationId e o expiresAt (ISO) retornados na criação — para o contador
  - Que disponibilidade pode mudar entre a página da sessão e o clique de
    reservar: tratar a recusa por indisponibilidade como resultado normal, não erro
  - Que o contador zerando não é erro de rede — é expiração; ao zerar, revalidar
    o estado da reserva com a API antes de mostrar a mensagem final
  - Que cancelar a reserva é uma ação explícita disponível só enquanto "aberta"
  - As mensagens de recusa por caso (indisponível / limite CPF / sessão fechada)

Backend precisa garantir:
  - A criação da reserva trava a linha da sessão (find_by_id_for_update) antes
    de qualquer contagem — RN05
  - As contagens de "confirmados" e "abertos não vencidos" são feitas na mesma
    transação travada
  - O limite por CPF usa o CPF do comprador autenticado, nunca um CPF do payload
  - Um processo de background transiciona reservas abertas vencidas para expirada
    e devolve disponibilidade, de forma idempotente e resistente a restart
  - Ao confirmar/cancelar/expirar, emitir o evento correspondente
    (ReservationConfirmed / ReservationCancelled / ReservationExpired) na mesma
    transação
```

## 5. Casos de borda

**Duas reservas concorrentes pela última poltrona.** A trava da linha da sessão
serializa: a primeira transação cria a reserva e libera a trava; a segunda, ao
travar, reconta e vê capacidade esgotada → recusa. Nunca as duas passam. `[fechada]`

**Comprador com reserva aberta tenta abrir uma segunda para a mesma sessão.**
Permitido, desde que a soma respeite RN01 (6 por CPF). As duas reservas abertas
contam juntas para o limite e para a disponibilidade. `[fechada]`

**Reserva expira exatamente enquanto o webhook de pagamento chega.** O webhook
tenta confirmar; se a reserva já não está "aberta", a confirmação é recusada e o
pagamento entra no fluxo de estorno automático (RF04/RF07). `[fechada]`

**Aplicação reinicia com reservas abertas.** Ao subir, o processo de expiração
retoma a varredura; reservas cujo prazo passou durante o downtime são expiradas
normalmente. Nenhuma reserva paga se perde (o pagamento é durável no banco).
`[fechada]`

**Admin cancela a sessão com reservas abertas.** Todas as reservas abertas da
sessão são canceladas e a disponibilidade deixa de importar (sessão cancelada).
Reservas já confirmadas viram reembolso (RF07). `[fechada]`
