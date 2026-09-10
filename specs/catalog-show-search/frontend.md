---
status: draft
spec: catalog-show-search
surface: frontend
created_at: 2026-09-10
updated_at: 2026-09-10
---

# Busca e filtro de espetáculos — Frontend

**Resumo:** rota `/` (home) — vitrine pública de espetáculos com sessão
futura, filtro por data e gênero, paginação simples. Primeira tela do app sem
autenticação; primeira rota do repo com *prefetch* de React Query no servidor.
**RF:** RF01 · **RN:** — · **Feature frontend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-show-search/integration.md`
**Carregar antes:** skill `frontend-architecture`.

Stack: **Next.js (App Router) + TypeScript estrito**. Arquivos `.ts`/`.tsx`;
componentes/hooks com estado, handler ou hook de React levam `'use client'`;
tipo = `z.infer` do schema (nunca `interface` manual para o que vem da API);
barrel `index.ts` em toda subpasta. Aliases: `@catalog/*`, `@web/*`.

> **Nota de consistência (resolvida em 2026-09-10):** `catalog-admin-management/
> frontend.md` chegou a assumir `server/services/` (aninhado), schemas em
> camelCase, um `fetcher.get/post/patch/delete` que devolve `{ data }`, e um
> registro `API_ENDPOINTS`. Já foi corrigido para `services/` (não aninhado),
> schemas snake_case, `fetcher<T>` devolvendo `T` direto, e o registro real
> `endpoints` (minúsculo). Este documento sempre seguiu o código real.

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.ts` | editar — acrescentar grupo `catalog` |
| 2 | schemas | `src/features/catalog/schemas/show.schema.ts` | novo |
| 3 | schemas | `src/features/catalog/schemas/index.ts` | novo — barrel |
| 4 | server/types | `src/features/catalog/server/types/show.types.ts` | novo |
| 5 | server/types | `src/features/catalog/server/types/index.ts` | novo — barrel |
| 6 | server | `src/features/catalog/server/index.ts` | novo — barrel |
| 7 | services | `src/features/catalog/services/show.service.ts` | novo |
| 8 | services | `src/features/catalog/services/index.ts` | novo — barrel |
| 9 | queries | `src/features/catalog/hooks/queries/query-options.ts` | novo |
| 10 | queries | `src/features/catalog/hooks/queries/useCatalogQueries.ts` | novo |
| 11 | queries | `src/features/catalog/hooks/queries/index.ts` | novo — barrel |
| 12 | hooks | `src/features/catalog/hooks/useShowFilters.ts` | novo |
| 13 | hooks | `src/features/catalog/hooks/index.ts` | novo — barrel |
| 14 | components/ui | `src/features/catalog/components/ui/ShowCard.tsx` | novo — puro |
| 15 | components/ui | `src/features/catalog/components/ui/Pagination.tsx` | novo — puro |
| 16 | components/ui | `src/features/catalog/components/ui/index.ts` | novo — barrel |
| 17 | components | `src/features/catalog/components/ShowFilters.tsx` | novo — `'use client'` |
| 18 | components | `src/features/catalog/components/ShowGrid.tsx` | novo — `'use client'` |
| 19 | components | `src/features/catalog/components/index.ts` | novo — barrel |
| 20 | feature root | `src/features/catalog/index.ts` | novo — API pública |
| 21 | lib | `src/lib/get-query-client.ts` | novo — QueryClient por request no servidor |
| 22 | rota | `src/app/page.tsx` | editar — troca o placeholder pela vitrine com *prefetch* |

Sem alias novo em `tsconfig.json` — `@catalog/*` já existe (registrado desde
o bootstrap do projeto, antes de qualquer feature de catálogo).

---

## 2. Código

### `src/routes/endpoints.ts`

Editar o arquivo real (grupos `auth`/`users` já existem — ver
`identity-auth`). Acrescentar o grupo `catalog`:

```ts
// src/routes/endpoints.ts  — editar
const AUTH_BASE = '/auth';
const USERS_BASE = '/users';

export const endpoints = {
  auth: {
    login: `${AUTH_BASE}/login`,
    refresh: `${AUTH_BASE}/refresh`,
    logout: `${AUTH_BASE}/logout`,
    changePassword: `${AUTH_BASE}/password/change`,
    passwordForgot: `${AUTH_BASE}/password/forgot`,
    passwordReset: `${AUTH_BASE}/password/reset`,
  },
  users: {
    register: `${USERS_BASE}/register`,
    list: USERS_BASE,
    byId: (id: string) => `${USERS_BASE}/${id}`,
  },
  catalog: {
    shows: '/shows',
    genres: '/genres',
  },
} as const;
```

### `src/features/catalog/schemas/show.schema.ts`

Campos snake_case — mesma grafia do backend (§2 nota de consistência).

```ts
// src/features/catalog/schemas/show.schema.ts  — novo
import { z } from 'zod';

export const showCardSchema = z.object({
  id: z.string().uuid(),
  title: z.string(),
  synopsis_short: z.string(),
  image_url: z.string(),
  genre: z.string(),
  upcoming_dates: z.array(z.coerce.date()),
  price_min: z.number(),
  price_max: z.number(),
});

export const pagedShowsSchema = z.object({
  items: z.array(showCardSchema),
  page: z.number().int(),
  size: z.number().int(),
  total: z.number().int(),
});

export const genreListSchema = z.array(z.string());
```

### `src/features/catalog/schemas/index.ts`

```ts
// src/features/catalog/schemas/index.ts  — novo
export * from './show.schema';
```

### `src/features/catalog/server/types/show.types.ts`

```ts
// src/features/catalog/server/types/show.types.ts  — novo
import type { z } from 'zod';

import type { genreListSchema, pagedShowsSchema, showCardSchema } from '@catalog/schemas';

export type ShowCard = z.infer<typeof showCardSchema>;
export type PagedShows = z.infer<typeof pagedShowsSchema>;
export type GenreList = z.infer<typeof genreListSchema>;
```

### `src/features/catalog/server/types/index.ts`

```ts
// src/features/catalog/server/types/index.ts  — novo
export * from './show.types';
```

### `src/features/catalog/server/index.ts`

```ts
// src/features/catalog/server/index.ts  — novo
export * from './types';
```

### `src/features/catalog/services/show.service.ts`

Segue `authService` real (`services/auth.service.ts`): função por operação,
`fetcher<T>` tipado, sem `.parse()` em runtime — os schemas Zod só derivam o
tipo (`z.infer`), a validação de contrato é o próprio TypeScript. `ShowFilters`
é estado de UI (filtro/paginação), não vem da API — por isso é um `type`
comum, não um `z.infer`.

```ts
// src/features/catalog/services/show.service.ts  — novo
import { fetcher } from '@web/lib/fetcher';
import { endpoints } from '@web/routes/endpoints';

import type { GenreList, PagedShows } from '@catalog/server/types';

export type ShowFilters = {
  fromDate?: string; // yyyy-mm-dd
  genre?: string;
  page?: number;
  size?: number;
};

function buildQuery(filters: ShowFilters): string {
  const params = new URLSearchParams();
  if (filters.fromDate) params.set('fromDate', filters.fromDate);
  if (filters.genre) params.set('genre', filters.genre);
  params.set('page', String(filters.page ?? 1));
  params.set('size', String(filters.size ?? 12));
  return params.toString();
}

export const catalogService = {
  fetchShows(filters: ShowFilters = {}) {
    return fetcher<PagedShows>(`${endpoints.catalog.shows}?${buildQuery(filters)}`, {
      method: 'GET',
      skipAuth: true,
    });
  },

  fetchGenres() {
    return fetcher<GenreList>(endpoints.catalog.genres, {
      method: 'GET',
      skipAuth: true,
    });
  },
};
```

### `src/features/catalog/services/index.ts`

```ts
// src/features/catalog/services/index.ts  — novo
export * from './show.service';
```

### `src/features/catalog/hooks/queries/query-options.ts`

```ts
// src/features/catalog/hooks/queries/query-options.ts  — novo
import { queryOptions } from '@tanstack/react-query';

import { catalogService } from '@catalog/services';

import type { ShowFilters } from '@catalog/services';

export const catalogQueryKeys = {
  all: ['catalog'] as const,
  shows: (filters: ShowFilters) => [...catalogQueryKeys.all, 'shows', filters] as const,
  genres: () => [...catalogQueryKeys.all, 'genres'] as const,
};

export const catalogQueryOptions = {
  showList: (filters: ShowFilters) =>
    queryOptions({
      queryKey: catalogQueryKeys.shows(filters),
      queryFn: () => catalogService.fetchShows(filters),
    }),
  genreList: () =>
    queryOptions({
      queryKey: catalogQueryKeys.genres(),
      queryFn: () => catalogService.fetchGenres(),
      // Lista de gêneros muda só quando o admin publica algo novo.
      staleTime: 60_000,
    }),
};
```

### `src/features/catalog/hooks/queries/useCatalogQueries.ts`

Um hook nomeado por consulta — mesmo padrão de `useCurrentUser()` em
`features/account/hooks/queries/useAccountQueries.ts` (não uma fábrica que
devolve um objeto de hooks).

```ts
// src/features/catalog/hooks/queries/useCatalogQueries.ts  — novo
'use client';

import { useQuery } from '@tanstack/react-query';

import { catalogQueryOptions } from './query-options';

import type { ShowFilters } from '@catalog/services';

export function useShowList(filters: ShowFilters) {
  return useQuery(catalogQueryOptions.showList(filters));
}

export function useGenreList() {
  return useQuery(catalogQueryOptions.genreList());
}
```

### `src/features/catalog/hooks/queries/index.ts`

```ts
// src/features/catalog/hooks/queries/index.ts  — novo
export * from './query-options';
export * from './useCatalogQueries';
```

### `src/features/catalog/hooks/useShowFilters.ts`

Estado dos filtros (data, gênero, página) sincronizado com a query string —
não é query nem mutation nem form, por isso fica na raiz de `hooks/`, não em
uma das três subpastas.

```ts
// src/features/catalog/hooks/useShowFilters.ts  — novo
'use client';

import { usePathname, useRouter, useSearchParams } from 'next/navigation';
import { useCallback, useMemo } from 'react';

import type { ShowFilters } from '@catalog/services';

const DEFAULT_SIZE = 12;

export function useShowFilters() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  const filters: ShowFilters = useMemo(
    () => ({
      fromDate: searchParams.get('fromDate') ?? undefined,
      genre: searchParams.get('genre') ?? undefined,
      page: Number(searchParams.get('page') ?? '1'),
      size: DEFAULT_SIZE,
    }),
    [searchParams],
  );

  const setFilters = useCallback(
    (next: Partial<Pick<ShowFilters, 'fromDate' | 'genre' | 'page'>>) => {
      const params = new URLSearchParams(searchParams.toString());
      const merged = { ...filters, ...next };

      if (merged.fromDate) params.set('fromDate', merged.fromDate);
      else params.delete('fromDate');

      if (merged.genre) params.set('genre', merged.genre);
      else params.delete('genre');

      // Trocar data/gênero reseta a página; mudar só a página não.
      if ('page' in next) params.set('page', String(next.page ?? 1));
      else params.delete('page');

      router.push(`${pathname}?${params.toString()}`);
    },
    [filters, pathname, router, searchParams],
  );

  return { filters, setFilters };
}
```

### `src/features/catalog/hooks/index.ts`

```ts
// src/features/catalog/hooks/index.ts  — novo
export * from './queries';
export * from './useShowFilters';
```

### `src/features/catalog/components/ui/ShowCard.tsx`

Apresentacional puro. `<img>` nativo, não `next/image` — a feature admin
explicitamente não faz upload/otimização de imagem, só aceita URL (spec
`catalog-admin-management` §6); usar `next/image` exigiria liberar domínios
arbitrários em `next.config`, fora de escopo aqui.

```tsx
// src/features/catalog/components/ui/ShowCard.tsx  — novo
import type { ShowCard as ShowCardModel } from '@catalog/server/types';

interface ShowCardProps {
  show: ShowCardModel;
}

const BRL = new Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' });
const SHORT_DATE = new Intl.DateTimeFormat('pt-BR', { day: '2-digit', month: 'short' });

export function ShowCard({ show }: ShowCardProps) {
  const priceLabel =
    show.price_min === show.price_max
      ? BRL.format(show.price_min)
      : `${BRL.format(show.price_min)} – ${BRL.format(show.price_max)}`;

  return (
    <article className="flex flex-col overflow-hidden rounded-lg border border-gray-200">
      <img
        src={show.image_url}
        alt={show.title}
        className="aspect-[3/4] w-full object-cover"
      />
      <div className="flex flex-1 flex-col gap-2 p-3">
        <h3 className="text-base font-semibold">{show.title}</h3>
        <p className="line-clamp-2 text-sm text-gray-600">{show.synopsis_short}</p>
        <p className="text-xs text-gray-500">
          {show.upcoming_dates.slice(0, 3).map((date) => SHORT_DATE.format(date)).join(' · ')}
        </p>
        <p className="mt-auto text-sm font-medium">{priceLabel}</p>
      </div>
    </article>
  );
}
```

### `src/features/catalog/components/ui/Pagination.tsx`

```tsx
// src/features/catalog/components/ui/Pagination.tsx  — novo
interface PaginationProps {
  page: number;
  size: number;
  total: number;
  onPageChange: (page: number) => void;
}

export function Pagination({ page, size, total, onPageChange }: PaginationProps) {
  const totalPages = Math.max(1, Math.ceil(total / size));
  if (totalPages <= 1) return null;

  return (
    <div className="flex items-center justify-center gap-3">
      <button
        type="button"
        disabled={page <= 1}
        onClick={() => onPageChange(page - 1)}
        className="min-h-11 rounded-md border border-gray-300 px-3 text-sm disabled:opacity-40"
      >
        Anterior
      </button>
      <span className="text-sm text-gray-600">
        Página {page} de {totalPages}
      </span>
      <button
        type="button"
        disabled={page >= totalPages}
        onClick={() => onPageChange(page + 1)}
        className="min-h-11 rounded-md border border-gray-300 px-3 text-sm disabled:opacity-40"
      >
        Próxima
      </button>
    </div>
  );
}
```

### `src/features/catalog/components/ui/index.ts`

```ts
// src/features/catalog/components/ui/index.ts  — novo
export * from './Pagination';
export * from './ShowCard';
```

### `src/features/catalog/components/ShowFilters.tsx`

```tsx
// src/features/catalog/components/ShowFilters.tsx  — novo
'use client';

import { useGenreList } from '@catalog/hooks/queries';
import { useShowFilters } from '@catalog/hooks';

export function ShowFilters() {
  const { filters, setFilters } = useShowFilters();
  const genresQuery = useGenreList();

  return (
    <div className="flex flex-wrap items-end gap-4">
      <div className="flex flex-col gap-1">
        <label htmlFor="filter-from-date" className="text-sm font-medium">
          A partir de
        </label>
        <input
          id="filter-from-date"
          type="date"
          value={filters.fromDate ?? ''}
          onChange={(event) => setFilters({ fromDate: event.target.value || undefined })}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
      </div>

      <div className="flex flex-col gap-1">
        <label htmlFor="filter-genre" className="text-sm font-medium">
          Gênero
        </label>
        <select
          id="filter-genre"
          value={filters.genre ?? ''}
          onChange={(event) => setFilters({ genre: event.target.value || undefined })}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        >
          <option value="">Todos</option>
          {(genresQuery.data ?? []).map((genre) => (
            <option key={genre} value={genre}>
              {genre}
            </option>
          ))}
        </select>
      </div>
    </div>
  );
}
```

### `src/features/catalog/components/ShowGrid.tsx`

Orchestration — conecta filtros e a query da lista; trata loading/error/empty;
pagina.

```tsx
// src/features/catalog/components/ShowGrid.tsx  — novo
'use client';

import { Pagination, ShowCard } from '@catalog/components/ui';
import { useShowFilters } from '@catalog/hooks';
import { useShowList } from '@catalog/hooks/queries';

import { ShowFilters } from './ShowFilters';

export function ShowGrid() {
  const { filters, setFilters } = useShowFilters();
  const query = useShowList(filters);

  return (
    <section className="mx-auto max-w-5xl space-y-6 p-6">
      <ShowFilters />

      {query.isLoading ? (
        <p className="text-sm text-gray-500">Carregando espetáculos...</p>
      ) : query.isError ? (
        <div className="text-sm">
          <p className="text-red-600">Não foi possível carregar os espetáculos.</p>
          <button
            type="button"
            onClick={() => void query.refetch()}
            className="mt-2 min-h-11 rounded-md border border-gray-300 px-4"
          >
            Tentar de novo
          </button>
        </div>
      ) : query.data && query.data.items.length === 0 ? (
        <p className="rounded-lg border border-dashed border-gray-300 p-8 text-center text-sm text-gray-500">
          Nenhum espetáculo em cartaz para esse filtro.
        </p>
      ) : query.data ? (
        <>
          <div className="grid grid-cols-2 gap-4 md:grid-cols-4">
            {query.data.items.map((show) => (
              <ShowCard key={show.id} show={show} />
            ))}
          </div>
          <Pagination
            page={query.data.page}
            size={query.data.size}
            total={query.data.total}
            onPageChange={(page) => setFilters({ page })}
          />
        </>
      ) : null}
    </section>
  );
}
```

### `src/features/catalog/components/index.ts`

```ts
// src/features/catalog/components/index.ts  — novo
export * from './ui';
export * from './ShowFilters';
export * from './ShowGrid';
```

### `src/features/catalog/index.ts`

Segue o barrel real de `features/account/index.ts` (`export *` de cada
subpasta; sem `contexts` aqui — a vitrine não usa nenhum).

```ts
// src/features/catalog/index.ts  — novo
export * from './components';
export * from './hooks';
export * from './schemas';
export * from './server';
export * from './services';
```

### `src/lib/get-query-client.ts`

Primeira rota do repo a fazer *prefetch* no servidor — `Providers.tsx` cria um
`QueryClient` por sessão de navegador (`useState`), que não serve para o
servidor. Padrão recomendado pelo TanStack Query para o App Router: um
`QueryClient` por request, deduplicado com `cache()` do React.

```ts
// src/lib/get-query-client.ts  — novo
import { QueryClient } from '@tanstack/react-query';
import { cache } from 'react';

export const getQueryClient = cache(() => new QueryClient());
```

### `src/app/page.tsx`

Editar — troca o placeholder (`// Vitrine (catalog). As features entram aqui...`)
pela vitrine real. Server Component; lê `searchParams` (Promise no App Router
atual — `await`), faz *prefetch* da lista e dos gêneros, hidrata para o
client.

```tsx
// src/app/page.tsx  — editar
import { dehydrate, HydrationBoundary } from '@tanstack/react-query';

import { ShowGrid } from '@catalog/components';
import { catalogQueryOptions } from '@catalog/hooks/queries';
import { getQueryClient } from '@web/lib/get-query-client';

import type { ShowFilters } from '@catalog/services';

interface HomePageProps {
  searchParams: Promise<Record<string, string | string[] | undefined>>;
}

function toFilters(params: Record<string, string | string[] | undefined>): ShowFilters {
  const first = (value: string | string[] | undefined) => (Array.isArray(value) ? value[0] : value);
  return {
    fromDate: first(params.fromDate),
    genre: first(params.genre),
    page: Number(first(params.page) ?? '1'),
    size: 12,
  };
}

export default async function HomePage({ searchParams }: HomePageProps) {
  const filters = toFilters(await searchParams);
  const queryClient = getQueryClient();

  await Promise.all([
    queryClient.prefetchQuery(catalogQueryOptions.showList(filters)),
    queryClient.prefetchQuery(catalogQueryOptions.genreList()),
  ]);

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ShowGrid />
    </HydrationBoundary>
  );
}
```

---

## 3. Contrato consumido

Contrato-alvo: `docs.ludens/specs/catalog-show-search/integration.md`. Sem
dependência de `identity-auth` (frontend) — rota 100% pública, `skipAuth: true`
em toda chamada.

| Símbolo | Origem | Forma esperada |
| --- | --- | --- |
| `fetcher` | `@web/lib/fetcher` | `fetcher<T>(path, options): Promise<T>`; em resposta não-2xx lança `Error('HTTP <status>')` — `ShowGrid` trata isso como `query.isError`, sem ler o corpo do erro (não há regra de negócio a distinguir aqui, é leitura pública) |
| `endpoints` | `@web/routes/endpoints` | objeto `as const`; grupo `catalog.shows` / `catalog.genres` (path sem parâmetro) |

---

## 4. Estados assíncronos e mensagens

| Estado | Onde | Mensagem |
| --- | --- | --- |
| loading | `ShowGrid` | "Carregando espetáculos..." |
| error | `ShowGrid` | "Não foi possível carregar os espetáculos." + botão "Tentar de novo" |
| empty (filtro sem resultado) | `ShowGrid` | "Nenhum espetáculo em cartaz para esse filtro." (logic.md §1.4) |
| paginação com 1 página só | `Pagination` | componente não renderiza nada |

---

## 5. Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-show-search

# commit 1 — contrato
git add src/routes/endpoints.ts src/features/catalog/schemas src/features/catalog/server \
        src/features/catalog/services
git commit -m "feat(catalog): endpoints, schemas, tipos e service de busca de espetaculos"

# commit 2 — hooks
git add src/features/catalog/hooks src/lib/get-query-client.ts
git commit -m "feat(catalog): queries, filtros por URL e query client de servidor"

# commit 3 — UI + rota
git add src/features/catalog/components src/app/page.tsx
git commit -m "feat(catalog): vitrine com filtros e paginacao + prefetch na home"

# commit 4 — barrel da feature
git add src/features/catalog/index.ts
git commit -m "chore(catalog): barrel index.ts da feature"

npm run lint && npm run build
```

Depois: `npm run lint && npm run build` verdes → `/team-ludens:tbd-pr`.

---

## 6. Ordem entre as superfícies

Não depende de nenhuma fatia de frontend (nem `identity-auth`) — só do backend
desta mesma fatia (contra o contrato-alvo, se o backend ainda não tiver
mergeado). `catalog-session-detail` (frontend) é a fatia seguinte no mesmo
módulo, e consome `ShowCard`/formatação de preço em comum se fizer sentido na
hora (avaliar duplicação vs. extrair para `lib/` nessa fatia).

---

## 7. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 fechada). Pontos de atenção:

- **`catalog-admin-management` (frontend) ainda não mergeado** (spec já
  corrigida, ver nota no topo) — sem ele não há dado real para a vitrine
  mostrar (a API responde lista vazia, não erro).
- **`getQueryClient`/*prefetch* é padrão novo no repo** — nenhuma rota atual
  faz isso; primeira vez que `HydrationBoundary` aparece no `web.ludens`.
  Validar no code review que `Providers.tsx` (o `QueryClient` de sessão) não é
  reaproveitado no servidor.
- **Sem runner de teste de frontend** (mesma lacuna registrada em
  `catalog-admin-management/quality.md`) — o portão hoje é `npm run lint` +
  `npm run build`; os casos de `quality.md` desta fatia usam Jest (unit, já
  configurado no repo) e ficam marcados como Playwright quando a suíte E2E
  existir.
