---
status: draft
spec: catalog-session-detail
surface: frontend
created_at: 2026-09-10
updated_at: 2026-09-10
---

# Detalhe da sessão — Frontend

**Resumo:** rotas `/espetaculos/[showId]` (espetáculo + próximas sessões) e
`/sessoes/[sessionId]` (detalhe com disponibilidade, *polling* a cada 15 s).
Fecha o "O que habilita" de `catalog-show-search` — o card da vitrine passa a
linkar para cá.
**RF:** RF02 · **RN:** RN05 (leitura) · **Feature frontend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-session-detail/integration.md`
**Carregar antes:** skill `frontend-architecture`.

Stack: **Next.js (App Router) + TypeScript estrito**. Segmentos dinâmicos
`[showId]`/`[sessionId]`; `params` é `Promise` — `await params`. Aliases:
`@catalog/*`, `@web/*`.

> Mesma base de `catalog-show-search/frontend.md`: código real usa
> `services/` (não `server/services/`), schemas snake_case, `fetcher<T>`
> devolve `T` direto (com `ApiError` para status+corpo do erro, ver
> `catalog-admin-management/frontend.md` §2), registro `endpoints`
> (minúsculo). Este documento edita os arquivos que `catalog-show-search` já
> cria — não redefine nada, só acrescenta.
>
> **Revisão de 2026-09-21:** confirmado que todo arquivo listado em §1 existe
> de fato no código real (`session.schema.ts`, `session.types.ts`,
> `ShowSessions.tsx`, `SessionDetail.tsx`, `ShowSessionsView.tsx`,
> `SessionDetailView.tsx`, rotas `src/app/espetaculos/[showId]` e
> `src/app/sessoes/[sessionId]`) — não checado nesta revisão. `endpoints`
> real é plano (`endpoints.catalog.showById`, `sessionById`), sem grupo
> aninhado, com prefixo `/api` (ver `catalog-admin-management/frontend.md`
> §2) — os blocos de código abaixo não foram reconferidos campo a campo
> contra o real; tratar como não totalmente verificado.

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.ts` | editar — grupo `catalog` ganha `showById`/`sessionById` |
| 2 | schemas | `src/features/catalog/schemas/session.schema.ts` | novo |
| 3 | schemas | `src/features/catalog/schemas/index.ts` | editar — acrescentar export |
| 4 | server/types | `src/features/catalog/server/types/session.types.ts` | novo |
| 5 | server/types | `src/features/catalog/server/types/index.ts` | editar — acrescentar export |
| 6 | services | `src/features/catalog/services/show.service.ts` | editar — acrescentar `fetchShowById`/`fetchSessionById` a `catalogService` (mesmo objeto, mesmo padrão de `authService`: um serviço por feature) |
| 7 | queries | `src/features/catalog/hooks/queries/query-options.ts` | editar — acrescentar `showDetail`/`sessionDetail` |
| 8 | queries | `src/features/catalog/hooks/queries/useCatalogQueries.ts` | editar — acrescentar `useShowDetail`/`useSessionDetail` |
| 9 | components/ui | `src/features/catalog/components/ui/ShowSessionsView.tsx` | novo — puro |
| 10 | components/ui | `src/features/catalog/components/ui/SessionDetailView.tsx` | novo — puro |
| 11 | components/ui | `src/features/catalog/components/ui/index.ts` | editar — acrescentar exports |
| 12 | components | `src/features/catalog/components/ShowSessions.tsx` | novo — `'use client'` |
| 13 | components | `src/features/catalog/components/SessionDetail.tsx` | novo — `'use client'` |
| 14 | components | `src/features/catalog/components/index.ts` | editar — acrescentar exports |
| 15 | components/ui | `src/features/catalog/components/ui/ShowCard.tsx` | editar — o card da vitrine passa a linkar para `/espetaculos/{id}` |
| 16 | rota | `src/app/espetaculos/[showId]/page.tsx` | novo |
| 17 | rota | `src/app/sessoes/[sessionId]/page.tsx` | novo |

---

## 2. Código

### `src/routes/endpoints.ts`

Editar o grupo `catalog` que `catalog-show-search` já criou.

```ts
// src/routes/endpoints.ts  — editar (dentro do grupo catalog)
catalog: {
  shows: '/shows',
  showById: (id: string) => `/shows/${id}`,
  genres: '/genres',
  sessionById: (id: string) => `/sessions/${id}`,
},
```

### `src/features/catalog/schemas/session.schema.ts`

```ts
// src/features/catalog/schemas/session.schema.ts  — novo
import { z } from 'zod';

export const ticketTypeSchema = z.object({
  type: z.enum(['full', 'half']),
  price: z.number(),
});

export const sessionSummarySchema = z.object({
  id: z.string().uuid(),
  starts_at: z.coerce.date(),
  venue: z.string(),
});

export const showDetailSchema = z.object({
  id: z.string().uuid(),
  title: z.string(),
  synopsis: z.string(),
  image_url: z.string(),
  genre: z.string(),
  sessions: z.array(sessionSummarySchema),
});

export const sessionStatusEnum = z.enum(['on_sale', 'sold_out', 'closed', 'cancelled']);

export const sessionDetailSchema = z.object({
  id: z.string().uuid(),
  show: z.object({ id: z.string().uuid(), title: z.string() }),
  starts_at: z.coerce.date(),
  venue: z.string(),
  capacity: z.number().int(),
  available_count: z.number().int(),
  status: sessionStatusEnum,
  ticket_types: z.array(ticketTypeSchema),
});
```

### `src/features/catalog/schemas/index.ts`

```ts
// src/features/catalog/schemas/index.ts  — editar
export * from './session.schema';
export * from './show.schema';
```

### `src/features/catalog/server/types/session.types.ts`

```ts
// src/features/catalog/server/types/session.types.ts  — novo
import type { z } from 'zod';

import type {
  sessionDetailSchema,
  sessionSummarySchema,
  showDetailSchema,
  ticketTypeSchema,
} from '@catalog/schemas';

export type TicketType = z.infer<typeof ticketTypeSchema>;
export type SessionSummary = z.infer<typeof sessionSummarySchema>;
export type ShowDetail = z.infer<typeof showDetailSchema>;
export type SessionDetail = z.infer<typeof sessionDetailSchema>;
```

### `src/features/catalog/server/types/index.ts`

```ts
// src/features/catalog/server/types/index.ts  — editar
export * from './session.types';
export * from './show.types';
```

### `src/features/catalog/services/show.service.ts`

Acrescentar as duas funções ao objeto `catalogService` que
`catalog-show-search` já cria — um serviço por feature (mesmo padrão de
`authService`), não um arquivo/objeto por rota.

```ts
// src/features/catalog/services/show.service.ts  — editar (dentro de catalogService)
import type { SessionDetail, ShowDetail } from '@catalog/server/types';

export const catalogService = {
  // ...fetchShows, fetchGenres já existentes (catalog-show-search)...

  fetchShowById(showId: string) {
    return fetcher<ShowDetail>(endpoints.catalog.showById(showId), {
      method: 'GET',
      skipAuth: true,
    });
  },

  fetchSessionById(sessionId: string) {
    return fetcher<SessionDetail>(endpoints.catalog.sessionById(sessionId), {
      method: 'GET',
      skipAuth: true,
    });
  },
};
```

### `src/features/catalog/hooks/queries/query-options.ts`

```ts
// src/features/catalog/hooks/queries/query-options.ts  — editar
export const catalogQueryKeys = {
  all: ['catalog'] as const,
  shows: (filters: ShowFilters) => [...catalogQueryKeys.all, 'shows', filters] as const,
  genres: () => [...catalogQueryKeys.all, 'genres'] as const,
  showDetail: (showId: string) => [...catalogQueryKeys.all, 'show', showId] as const,
  sessionDetail: (sessionId: string) => [...catalogQueryKeys.all, 'session', sessionId] as const,
};

export const catalogQueryOptions = {
  // ...showList, genreList já existentes (catalog-show-search)...

  showDetail: (showId: string) =>
    queryOptions({
      queryKey: catalogQueryKeys.showDetail(showId),
      queryFn: () => catalogService.fetchShowById(showId),
    }),

  sessionDetail: (sessionId: string) =>
    queryOptions({
      queryKey: catalogQueryKeys.sessionDetail(sessionId),
      queryFn: () => catalogService.fetchSessionById(sessionId),
      // logic.md §4 — disponibilidade re-consultada por polling ~15s
      // enquanto a página está aberta.
      refetchInterval: 15_000,
      staleTime: 5_000,
    }),
};
```

### `src/features/catalog/hooks/queries/useCatalogQueries.ts`

```ts
// src/features/catalog/hooks/queries/useCatalogQueries.ts  — editar
export function useShowDetail(showId: string) {
  return useQuery(catalogQueryOptions.showDetail(showId));
}

export function useSessionDetail(sessionId: string) {
  return useQuery(catalogQueryOptions.sessionDetail(sessionId));
}
```

### `src/features/catalog/components/ui/ShowSessionsView.tsx`

```tsx
// src/features/catalog/components/ui/ShowSessionsView.tsx  — novo
import Link from 'next/link';

import type { ShowDetail } from '@catalog/server/types';

interface ShowSessionsViewProps {
  show: ShowDetail;
}

const DATE_TIME = new Intl.DateTimeFormat('pt-BR', { dateStyle: 'full', timeStyle: 'short' });

export function ShowSessionsView({ show }: ShowSessionsViewProps) {
  return (
    <article className="mx-auto max-w-3xl space-y-6 p-6">
      <img src={show.image_url} alt={show.title} className="w-full rounded-lg object-cover" />
      <h1 className="text-2xl font-bold">{show.title}</h1>
      <p className="text-sm text-gray-500">{show.genre}</p>
      <p className="text-sm text-gray-700">{show.synopsis}</p>

      <div>
        <h2 className="text-lg font-semibold">Sessões</h2>
        {show.sessions.length === 0 ? (
          <p className="mt-2 text-sm text-gray-500">
            Nenhuma sessão futura à venda no momento.
          </p>
        ) : (
          <ul className="mt-2 space-y-2">
            {show.sessions.map((session) => (
              <li key={session.id}>
                <Link
                  href={`/sessoes/${session.id}`}
                  className="flex min-h-11 items-center justify-between rounded-md border border-gray-200 px-4 text-sm hover:border-gray-400"
                >
                  <span>{DATE_TIME.format(session.starts_at)}</span>
                  <span className="text-gray-500">{session.venue}</span>
                </Link>
              </li>
            ))}
          </ul>
        )}
      </div>
    </article>
  );
}
```

### `src/features/catalog/components/ui/SessionDetailView.tsx`

Slot explícito para o `TicketPicker` de `booking-reservation` — esta fatia só
mostra disponibilidade, não reserva.

```tsx
// src/features/catalog/components/ui/SessionDetailView.tsx  — novo
import type { SessionDetail } from '@catalog/server/types';

interface SessionDetailViewProps {
  session: SessionDetail;
}

const DATE_TIME = new Intl.DateTimeFormat('pt-BR', { dateStyle: 'full', timeStyle: 'short' });
const BRL = new Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' });

const STATUS_LABEL: Record<SessionDetail['status'], string> = {
  on_sale: 'À venda',
  sold_out: 'Esgotado',
  closed: 'Encerrada',
  cancelled: 'Cancelada',
};

export function SessionDetailView({ session }: SessionDetailViewProps) {
  const canReserve = session.status === 'on_sale' && session.available_count > 0;

  return (
    <article className="mx-auto max-w-2xl space-y-6 p-6">
      <header>
        <p className="text-sm text-gray-500">{session.show.title}</p>
        <h1 className="text-2xl font-bold">{DATE_TIME.format(session.starts_at)}</h1>
        <p className="text-sm text-gray-600">{session.venue}</p>
      </header>

      <div className="flex items-center gap-3">
        <span className="rounded bg-gray-100 px-2 py-0.5 text-xs">
          {STATUS_LABEL[session.status]}
        </span>
        {session.status === 'on_sale' ? (
          <span className="text-sm text-gray-600">
            {session.available_count} ingresso(s) disponível(is)
          </span>
        ) : null}
      </div>

      <ul className="space-y-1 text-sm">
        {session.ticket_types.map((ticket) => (
          <li key={ticket.type} className="flex justify-between">
            <span>{ticket.type === 'full' ? 'Inteira' : 'Meia-entrada'}</span>
            <span>{BRL.format(ticket.price)}</span>
          </li>
        ))}
      </ul>

      {/* Slot do TicketPicker (booking-reservation) — esta fatia só exibe
          disponibilidade; a reserva é outra feature. */}
      <button
        type="button"
        disabled={!canReserve}
        className="min-h-11 w-full rounded-md bg-gray-900 px-4 text-sm font-medium text-white disabled:opacity-40"
      >
        {canReserve ? 'Reservar' : STATUS_LABEL[session.status]}
      </button>
    </article>
  );
}
```

### `src/features/catalog/components/ui/index.ts`

```ts
// src/features/catalog/components/ui/index.ts  — editar
export * from './Pagination';
export * from './SessionDetailView';
export * from './ShowCard';
export * from './ShowSessionsView';
```

### `src/features/catalog/components/ShowSessions.tsx`

```tsx
// src/features/catalog/components/ShowSessions.tsx  — novo
'use client';

import { ShowSessionsView } from '@catalog/components/ui';
import { useShowDetail } from '@catalog/hooks/queries';

interface ShowSessionsProps {
  showId: string;
}

export function ShowSessions({ showId }: ShowSessionsProps) {
  const query = useShowDetail(showId);

  if (query.isLoading) {
    return <p className="p-6 text-sm text-gray-500">Carregando espetáculo...</p>;
  }

  if (query.isError || !query.data) {
    return <p className="p-6 text-sm text-red-600">Espetáculo não encontrado.</p>;
  }

  return <ShowSessionsView show={query.data} />;
}
```

### `src/features/catalog/components/SessionDetail.tsx`

```tsx
// src/features/catalog/components/SessionDetail.tsx  — novo
'use client';

import { SessionDetailView } from '@catalog/components/ui';
import { useSessionDetail } from '@catalog/hooks/queries';

interface SessionDetailProps {
  sessionId: string;
}

export function SessionDetail({ sessionId }: SessionDetailProps) {
  const query = useSessionDetail(sessionId);

  if (query.isLoading) {
    return <p className="p-6 text-sm text-gray-500">Carregando sessão...</p>;
  }

  if (query.isError || !query.data) {
    return <p className="p-6 text-sm text-red-600">Sessão não encontrada.</p>;
  }

  return <SessionDetailView session={query.data} />;
}
```

### `src/features/catalog/components/index.ts`

```ts
// src/features/catalog/components/index.ts  — editar
export * from './ui';
export * from './SessionDetail';
export * from './ShowFilters';
export * from './ShowGrid';
export * from './ShowSessions';
```

### `src/features/catalog/components/ui/ShowCard.tsx`

Editar o card criado por `catalog-show-search`: envolver em `Link` para
`/espetaculos/{id}` — fecha "O que habilita" da spec de RF01.

```tsx
// src/features/catalog/components/ui/ShowCard.tsx  — editar
import Link from 'next/link';

// ...

export function ShowCard({ show }: ShowCardProps) {
  // ...priceLabel igual...

  return (
    <Link
      href={`/espetaculos/${show.id}`}
      className="flex flex-col overflow-hidden rounded-lg border border-gray-200"
    >
      {/* mesmo conteúdo interno de antes — trocar <article> por <Link> */}
    </Link>
  );
}
```

### `src/app/espetaculos/[showId]/page.tsx`

```tsx
// src/app/espetaculos/[showId]/page.tsx  — novo
import { dehydrate, HydrationBoundary } from '@tanstack/react-query';

import { ShowSessions } from '@catalog/components';
import { catalogQueryOptions } from '@catalog/hooks/queries';
import { getQueryClient } from '@web/lib/get-query-client';

interface ShowPageProps {
  params: Promise<{ showId: string }>;
}

export default async function ShowPage({ params }: ShowPageProps) {
  const { showId } = await params;
  const queryClient = getQueryClient();

  await queryClient.prefetchQuery(catalogQueryOptions.showDetail(showId));

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ShowSessions showId={showId} />
    </HydrationBoundary>
  );
}
```

### `src/app/sessoes/[sessionId]/page.tsx`

```tsx
// src/app/sessoes/[sessionId]/page.tsx  — novo
import { dehydrate, HydrationBoundary } from '@tanstack/react-query';

import { SessionDetail } from '@catalog/components';
import { catalogQueryOptions } from '@catalog/hooks/queries';
import { getQueryClient } from '@web/lib/get-query-client';

interface SessionPageProps {
  params: Promise<{ sessionId: string }>;
}

export default async function SessionPage({ params }: SessionPageProps) {
  const { sessionId } = await params;
  const queryClient = getQueryClient();

  await queryClient.prefetchQuery(catalogQueryOptions.sessionDetail(sessionId));

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <SessionDetail sessionId={sessionId} />
    </HydrationBoundary>
  );
}
```

---

## 3. Contrato consumido

Contrato-alvo: `docs.ludens/specs/catalog-session-detail/integration.md`. Sem
autenticação — mesma base de `catalog-show-search` (`fetcher`, `endpoints`,
`getQueryClient`).

---

## 4. Estados assíncronos e mensagens

| Estado | Onde | Mensagem |
| --- | --- | --- |
| loading (espetáculo) | `ShowSessions` | "Carregando espetáculo..." |
| erro/404 (espetáculo) | `ShowSessions` | "Espetáculo não encontrado." |
| sem sessão futura | `ShowSessionsView` | "Nenhuma sessão futura à venda no momento." |
| loading (sessão) | `SessionDetail` | "Carregando sessão..." |
| erro/404 (sessão) | `SessionDetail` | "Sessão não encontrada." |
| esgotado / encerrada / cancelada | `SessionDetailView` | rótulo no botão (`STATUS_LABEL`) no lugar de "Reservar", botão desabilitado |

---

## 5. Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-session-detail

# commit 1 — contrato
git add src/routes/endpoints.ts src/features/catalog/schemas src/features/catalog/server \
        src/features/catalog/services
git commit -m "feat(catalog): endpoints, schemas, tipos e service de detalhe da sessao"

# commit 2 — queries com polling
git add src/features/catalog/hooks/queries
git commit -m "feat(catalog): queries de detalhe do espetaculo e da sessao com polling"

# commit 3 — telas e rotas
git add src/features/catalog/components src/app/espetaculos src/app/sessoes
git commit -m "feat(catalog): telas de detalhe da sessao e link da vitrine"

npm run lint && npm run build
```

Depois: `/team-ludens:tbd-pr`.

---

## 6. Ordem entre as superfícies

Depende do backend desta fatia (ou do contrato-alvo). Depende também da fatia
de frontend de `catalog-show-search` mergeada primeiro (edita os mesmos
arquivos: `endpoints.ts`, `schemas/index.ts`, `show.service.ts`,
`query-options.ts`, `useCatalogQueries.ts`, `ShowCard.tsx`) — se as duas
saírem em paralelo, resolver conflito de merge é esperado, não é bloqueio.

---

## 7. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 fechada). Pontos de atenção:

- **Botão "Reservar" é só visual** — `booking-reservation` ainda não existe;
  o slot está marcado no código (`SessionDetailView.tsx`).
- **`refetchInterval: 15_000` gera tráfego constante enquanto a página fica
  aberta** — aceitável para N1 (decisão fechada em spec.md §8); se
  `RNF02`/custo de infra virar problema, é ajuste de configuração, não de
  produto.
