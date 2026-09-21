---
status: done
spec: catalog-show-search
surface: frontend
created_at: 2026-09-10
updated_at: 2026-09-21
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

> **Nota histórica.** `schemas/show.schema.ts`, `services/show.service.ts`,
> `hooks/queries/query-options.ts` e `endpoints.ts` **não são exclusivos**
> desta feature: `catalog-genre`, `catalog-session-detail` e
> `catalog-admin-management` foram implementadas no mesmo arquivo
> compartilhado (mesma feature `catalog`). Este documento reproduz só a parte
> relevante à busca (`fetchShows`/`fetchGenres`/filtros); os outros arquivos
> têm bem mais conteúdo do que o mostrado aqui. Reescrito em 2026-09-21 pra
> refletir o código real — ver "Ajustes feitos no `integration.md`" ao final.

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.ts` | editar — grupo `catalog` (compartilhado com as demais specs de `catalog`) |
| 2 | schemas | `src/features/catalog/schemas/show.schema.ts` | editar — `showCardSchema`, `pagedShowsSchema`, `genreSchema`, `genreListSchema` |
| 3 | server/types | `src/features/catalog/server/types/` | editar — `z.infer` dos schemas acima |
| 4 | services | `src/features/catalog/services/show.service.ts` | editar — `catalogService.fetchShows`/`fetchGenres`, tipo `ShowFilterParams` |
| 5 | queries | `src/features/catalog/hooks/queries/query-options.ts` | editar — `catalogQueryOptions.showList`/`genreList` |
| 6 | queries | `src/features/catalog/hooks/queries/useCatalogQueries.ts` | editar |
| 7 | hooks | `src/features/catalog/hooks/useShowFilters.ts` | novo |
| 8 | lib | `src/features/catalog/lib/` | editar — `formatPriceBRL` |
| 9 | components/ui | `src/features/catalog/components/ui/ShowCard.tsx` | novo |
| 10 | components/ui | `src/features/catalog/components/ui/Pagination.tsx` | novo |
| 11 | components | `src/features/catalog/components/ShowFilters.tsx` | novo — `'use client'` |
| 12 | components | `src/features/catalog/components/ShowGrid.tsx` | novo — `'use client'` |
| 13 | lib | `src/lib/get-query-client.ts` | novo — `QueryClient` por request no servidor |
| 14 | rota | `src/app/page.tsx` | editar — vitrine com *prefetch* |

Sem alias novo em `tsconfig.json` — `@catalog/*` já existe.

---

## 2. Código

### `src/routes/endpoints.ts` (recorte relevante)

```ts
const API_BASE = '/api';
const CATALOG_BASE = `${API_BASE}/catalog`;

export const endpoints = {
  // ...auth, users (ver identity-auth)
  catalog: {
    shows: `${CATALOG_BASE}/shows`,
    showById: (id: string) => `${CATALOG_BASE}/shows/${id}`,
    genres: `${CATALOG_BASE}/genres`,
    genreById: (id: string) => `${CATALOG_BASE}/genres/${id}`,
    // ...showPublish, showUnpublish, sessions* — ver catalog-admin-management/catalog-session-detail
  },
} as const;
```

Toda rota de `catalog` (pública e administrativa) é a **mesma URL**; o
backend decide o formato da resposta pelo token do usuário — não existe
namespace `/admin` separado.

### `src/features/catalog/schemas/show.schema.ts` (recorte relevante)

```ts
import { z } from 'zod';

export const showCardSchema = z.object({
  id: z.string().uuid(),
  title: z.string(),
  synopsis_short: z.string(),
  image_url: z.string(),
  genre_id: z.string().uuid(),
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

// GET /catalog/genres devolve [{ id, name }] — filtro usa o id (UUID);
// name é só o texto exibido.
export const genreSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
});

export const genreListSchema = z.array(genreSchema);
```

### `src/features/catalog/services/show.service.ts` (recorte relevante)

`fetcher<T>` não valida a resposta em runtime (contrato da feature); os
schemas Zod só derivam o tipo (`z.infer`). `ShowFilterParams` é estado de UI
(filtro/paginação): mantém os nomes internos `fromDate`/`genre` em camelCase,
mas traduz pro contrato real (`from_date`/`genre_id`) ao montar a query.

```ts
import { fetcher } from '@web/lib/fetcher';
import { endpoints } from '@web/routes/endpoints';

import type { GenreList, PagedShows, ShowCardModel } from '@catalog/server/types';

export type ShowFilterParams = {
  fromDate?: string;
  genre?: string;
  page?: number;
  size?: number;
};

function buildQuery(filters: ShowFilterParams): string {
  const params = new URLSearchParams();

  if (filters.fromDate) {
    // Contrato real é snake_case (from_date) — GET /catalog/shows.
    params.set('from_date', filters.fromDate);
  }
  if (filters.genre) {
    // Contrato real é genre_id (UUID) — GET /catalog/shows.
    params.set('genre_id', filters.genre);
  }
  params.set('page', String(filters.page ?? 1));
  params.set('size', String(filters.size ?? 12));

  return params.toString();
}

// O fetcher não valida em runtime, então upcoming_dates chega como
// string[] apesar do tipo dizer Date[] (via z.coerce.date()).
function reviveShowCard(show: ShowCardModel): ShowCardModel {
  return { ...show, upcoming_dates: show.upcoming_dates.map((date) => new Date(date)) };
}

export const catalogService = {
  async fetchShows(filters: ShowFilterParams = {}) {
    const page = await fetcher<PagedShows>(`${endpoints.catalog.shows}?${buildQuery(filters)}`, {
      method: 'GET',
      skipAuth: true,
    });
    return { ...page, items: page.items.map(reviveShowCard) };
  },

  fetchGenres() {
    return fetcher<GenreList>(endpoints.catalog.genres, { method: 'GET', skipAuth: true });
  },

  // ...createGenre/updateGenre/deleteGenre (catalog-genre), fetchShowById/
  // fetchSessionById (catalog-session-detail), createShow/updateShow/
  // publishShow/... (catalog-admin-management) — mesmo arquivo, fora do
  // escopo deste documento.
};
```

### `src/features/catalog/hooks/queries/query-options.ts` (recorte relevante)

```ts
import { queryOptions } from '@tanstack/react-query';

import { catalogService } from '@catalog/services';

import type { ShowFilterParams } from '@catalog/services/show.service';

export const catalogQueryKeys = {
  all: ['catalog'] as const,
  shows: (filters: ShowFilterParams) => [...catalogQueryKeys.all, 'shows', filters] as const,
  genres: () => [...catalogQueryKeys.all, 'genres'] as const,
  // ...showDetail, sessionDetail, admin.* — outras specs de catalog
};

export const catalogQueryOptions = {
  showList: (filters: ShowFilterParams) =>
    queryOptions({
      queryKey: catalogQueryKeys.shows(filters),
      queryFn: () => catalogService.fetchShows(filters),
    }),
  genreList: () =>
    queryOptions({
      queryKey: catalogQueryKeys.genres(),
      queryFn: () => catalogService.fetchGenres(),
      staleTime: 60_000,
    }),
  // ...showDetail, sessionDetail, adminShowList, adminShow
};
```

### `src/features/catalog/hooks/useShowFilters.ts`

Estado dos filtros (data, gênero, página) sincronizado com a query string.

```ts
'use client';

import { usePathname, useRouter, useSearchParams } from 'next/navigation';
import { useCallback, useMemo } from 'react';

import type { ShowFilterParams } from '@catalog/services';

const DEFAULT_SIZE = 12;

type ShowFilterUpdates = {
  fromDate?: string | undefined;
  genre?: string | undefined;
  page?: number | undefined;
};

export function useShowFilters() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  const filters: ShowFilterParams = useMemo(() => {
    const fromDate = searchParams.get('fromDate');
    const genre = searchParams.get('genre');
    return {
      ...(fromDate ? { fromDate } : {}),
      ...(genre ? { genre } : {}),
      page: Number(searchParams.get('page') ?? '1'),
      size: DEFAULT_SIZE,
    };
  }, [searchParams]);

  const setFilters = useCallback(
    (next: ShowFilterUpdates) => {
      const params = new URLSearchParams(searchParams.toString());
      const merged = { ...filters, ...next };

      if (merged.fromDate) params.set('fromDate', merged.fromDate);
      else params.delete('fromDate');

      if (merged.genre) params.set('genre', merged.genre);
      else params.delete('genre');

      if ('page' in next) params.set('page', String(next.page ?? 1));
      else params.delete('page');

      router.push(`${pathname}?${params.toString()}`);
    },
    [filters, pathname, router, searchParams],
  );

  return { filters, setFilters };
}
```

### `src/features/catalog/components/ui/ShowCard.tsx`

Difere bastante da versão original deste documento: é um `Link` pra rota de
detalhe (`catalog-session-detail`), usa os componentes de UI do design system
(`Card`, `Badge`) em vez de HTML puro, e mostra o gênero como badge.

```tsx
import Link from 'next/link';
import { CalendarDays } from 'lucide-react';

import { Badge } from '@components/ui/badge';
import { Card, CardContent } from '@components/ui/card';

import { formatPriceBRL } from '@catalog/lib';

import type { ShowCardModel } from '@catalog/server/types';

interface ShowCardProps {
  show: ShowCardModel;
}

const SHORT_DATE = new Intl.DateTimeFormat('pt-BR', { day: '2-digit', month: 'short' });

export function ShowCard({ show }: ShowCardProps) {
  const priceLabel =
    show.price_min === show.price_max
      ? formatPriceBRL(show.price_min)
      : `${formatPriceBRL(show.price_min)} – ${formatPriceBRL(show.price_max)}`;

  return (
    <Link href={`/espetaculos/${show.id}`} className="block">
      <Card className="h-full gap-3 border-t-2 border-t-primary/70 transition hover:shadow-md">
        <img src={show.image_url} alt={show.title} className="aspect-[3/4] w-full object-cover" />
        <CardContent className="flex flex-1 flex-col gap-2">
          <div className="flex items-start justify-between gap-2">
            <h3 className="font-heading text-base font-semibold">{show.title}</h3>
            <Badge variant="outline" className="shrink-0 border-primary/40 text-primary">
              {show.genre}
            </Badge>
          </div>
          <p className="line-clamp-2 text-sm text-muted-foreground">{show.synopsis_short}</p>
          <p className="flex items-center gap-1.5 text-xs text-muted-foreground">
            <CalendarDays className="size-3.5 text-primary" />
            {show.upcoming_dates.slice(0, 3).map((date) => SHORT_DATE.format(date)).join(' · ')}
          </p>
          <p className="mt-auto text-sm font-medium">{priceLabel}</p>
        </CardContent>
      </Card>
    </Link>
  );
}
```

### `src/features/catalog/components/ui/Pagination.tsx`

```tsx
import { Button } from '@components/ui/button';

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
      <Button type="button" variant="outline" className="min-h-11 px-3" disabled={page <= 1} onClick={() => onPageChange(page - 1)}>
        Anterior
      </Button>
      <span className="text-sm text-muted-foreground">Página {page} de {totalPages}</span>
      <Button type="button" variant="outline" className="min-h-11 px-3" disabled={page >= totalPages} onClick={() => onPageChange(page + 1)}>
        Próxima
      </Button>
    </div>
  );
}
```

### `src/features/catalog/components/ShowFilters.tsx`

Usa `Input`/`Label`/`Select` do design system, não `<input>`/`<select>` puros
como a versão original do documento — o filtro de gênero é um `Select` do
shadcn, com `value`/`onValueChange`, indexado por `genre.id`.

```tsx
'use client';

import { CalendarDays, Tags } from 'lucide-react';

import { Input } from '@components/ui/input';
import { Label } from '@components/ui/label';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@components/ui/select';

import { useGenreList } from '@catalog/hooks/queries';
import { useShowFilters } from '@catalog/hooks';

const ALL_GENRES = 'all';

export function ShowFilters() {
  const { filters, setFilters } = useShowFilters();
  const genresQuery = useGenreList();

  return (
    <div className="flex flex-wrap items-end gap-4">
      <div className="flex flex-col gap-1.5">
        <Label htmlFor="filter-from-date" className="gap-1.5">
          <CalendarDays className="size-3.5 text-primary" />
          A partir de
        </Label>
        <Input
          id="filter-from-date"
          type="date"
          value={filters.fromDate ?? ''}
          onChange={(event) => setFilters({ fromDate: event.target.value || undefined })}
          className="min-h-11 border-primary/30"
        />
      </div>

      <div className="flex flex-col gap-1.5">
        <Label htmlFor="filter-genre" className="gap-1.5">
          <Tags className="size-3.5 text-primary" />
          Gênero
        </Label>
        <Select
          value={filters.genre ?? ALL_GENRES}
          onValueChange={(value) => setFilters({ genre: value === ALL_GENRES ? undefined : value })}
        >
          <SelectTrigger id="filter-genre" className="min-h-11 w-40 border-primary/30">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value={ALL_GENRES}>Todos</SelectItem>
            {(genresQuery.data ?? []).map((genre) => (
              <SelectItem key={genre.id} value={genre.id}>
                {genre.name}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>
      </div>
    </div>
  );
}
```

### `src/features/catalog/components/ShowGrid.tsx`

Ganhou um cabeçalho ("Espetáculos em cartaz"), skeleton de loading (não texto
"Carregando...") e um `Alert` do design system pro erro — versão bem mais
elaborada do que a original deste documento previa.

```tsx
'use client';

import { RotateCw } from 'lucide-react';

import { Alert, AlertDescription, AlertTitle } from '@components/ui/alert';
import { Button } from '@components/ui/button';
import { Skeleton } from '@components/ui/skeleton';
import { LogoIcon } from '@components/LogoIcon';

import { Pagination, ShowCard } from '@catalog/components/ui';
import { useShowFilters } from '@catalog/hooks';
import { useShowList } from '@catalog/hooks/queries';

import { ShowFilters } from './ShowFilters';

function ShowGridLoadingState() {
  return (
    <div className="grid grid-cols-2 gap-4 md:grid-cols-4">
      {[0, 1, 2, 3].map((key) => (
        <div key={key} className="flex flex-col gap-2">
          <Skeleton className="aspect-[3/4] w-full rounded-xl" />
          <Skeleton className="h-4 w-3/4" />
          <Skeleton className="h-4 w-1/2" />
        </div>
      ))}
    </div>
  );
}

export function ShowGrid() {
  const { filters, setFilters } = useShowFilters();
  const query = useShowList(filters);

  return (
    <section className="mx-auto max-w-5xl space-y-6 p-6">
      <div className="flex items-center gap-3 rounded-2xl border border-border border-b-2 border-b-primary bg-card p-6 sm:p-8">
        <span className="flex size-12 shrink-0 items-center justify-center rounded-full bg-[#fae9e9] p-2.5">
          <LogoIcon className="size-full" />
        </span>
        <div>
          <h1 className="font-heading text-2xl font-semibold tracking-tight sm:text-3xl">Espetáculos em cartaz</h1>
          <p className="text-sm text-muted-foreground sm:text-base">Encontre a próxima sessão do seu espetáculo favorito.</p>
        </div>
      </div>

      <ShowFilters />

      {query.isLoading ? (
        <ShowGridLoadingState />
      ) : query.isError ? (
        <Alert variant="destructive">
          <AlertTitle>Não foi possível carregar os espetáculos</AlertTitle>
          <AlertDescription className="flex flex-col gap-3">
            <span>Verifique sua conexão e tente novamente.</span>
            <Button type="button" variant="outline" size="sm" className="min-h-11 w-fit" onClick={() => void query.refetch()}>
              <RotateCw />
              Tentar de novo
            </Button>
          </AlertDescription>
        </Alert>
      ) : query.data && query.data.items.length === 0 ? (
        <div className="flex flex-col items-center gap-3 rounded-xl border border-dashed border-border p-10 text-center">
          <span className="flex size-12 items-center justify-center rounded-full bg-[#fae9e9] p-2.5">
            <LogoIcon className="size-full" />
          </span>
          <p className="text-sm text-muted-foreground">Nenhum espetáculo em cartaz para esse filtro.</p>
        </div>
      ) : query.data ? (
        <>
          <div className="grid grid-cols-2 gap-4 md:grid-cols-4">
            {query.data.items.map((show) => (
              <ShowCard key={show.id} show={show} />
            ))}
          </div>
          <Pagination page={query.data.page} size={query.data.size} total={query.data.total} onPageChange={(page) => setFilters({ page })} />
        </>
      ) : null}
    </section>
  );
}
```

### `src/lib/get-query-client.ts`

```ts
import { QueryClient } from '@tanstack/react-query';
import { cache } from 'react';

export const getQueryClient = cache(() => new QueryClient());
```

### `src/app/page.tsx`

```tsx
import { dehydrate, HydrationBoundary } from '@tanstack/react-query';

import { ShowGrid } from '@catalog/components';
import { catalogQueryOptions } from '@catalog/hooks/queries';
import type { ShowFilterParams } from '@catalog/services';

import { getQueryClient } from '@web/lib/get-query-client';

interface HomePageProps {
  searchParams: Promise<Record<string, string | string[] | undefined>>;
}

function toFilters(params: Record<string, string | string[] | undefined>): ShowFilterParams {
  const first = (value: string | string[] | undefined) => (Array.isArray(value) ? value[0] : value);
  const filters: ShowFilterParams = { page: Number(first(params.page) ?? '1'), size: 12 };

  const fromDate = first(params.fromDate);
  const genre = first(params.genre);
  if (fromDate) filters.fromDate = fromDate;
  if (genre) filters.genre = genre;

  return filters;
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

Contrato: `docs.ludens/specs/catalog-show-search/integration.md`. Sem
dependência de `identity-auth` (frontend) — rota 100% pública, `skipAuth: true`
em toda chamada.

* `fetcher<T>(path, options): Promise<T>` — em resposta não-2xx lança
  `Error('HTTP <status>')`; `ShowGrid` trata isso como `query.isError`, sem
  ler o corpo do erro (leitura pública, sem regra de negócio a distinguir).
* `endpoints.catalog.shows` / `endpoints.catalog.genres` — path sem
  parâmetro, sob `/api/catalog`.

---

## 4. Estados assíncronos e mensagens

* **loading:** `ShowGrid` mostra um grid de `Skeleton` (não texto).
* **error:** `Alert` "Não foi possível carregar os espetáculos" + "Verifique
  sua conexão e tente novamente." + botão "Tentar de novo".
* **empty (filtro sem resultado):** "Nenhum espetáculo em cartaz para esse
  filtro." (logic.md §1.4), com o ícone da marca.
* **paginação com 1 página só:** `Pagination` não renderiza nada.

---

## 5. Passo a passo TBD (Frontend)

Já mergeado.

```text
feat(catalog): endpoints, schemas, tipos e service de busca de espetaculos
feat(catalog): queries, filtros por URL e query client de servidor
feat(catalog): vitrine com filtros e paginacao + prefetch na home
```

---

## 6. Ordem entre as superfícies

Não depende de nenhuma fatia de frontend (nem `identity-auth`) — só do
backend desta mesma fatia. `catalog-session-detail` é a fatia seguinte no
mesmo módulo (o card já linka pra rota de detalhe dela).

---

## 7. Débitos técnicos registrados

* `catalogService`, `catalogQueryOptions` e `show.schema.ts` acumulam código
  de quatro specs diferentes (`catalog-genre`, `catalog-show-search`,
  `catalog-session-detail`, `catalog-admin-management`) no mesmo arquivo —
  nenhuma dessas specs tem um arquivo isolado só seu.

## 8. Ajustes feitos no `integration.md`

* Filtro passou de `genre: str` pra `genre_id` (UUID); a UI já enviava o id
  desde a implementação real, nunca chegou a mandar nome de gênero como
  texto.
* Lista de gêneros vem de `GET /catalog/genres` como `{id, name}`, consumida
  por um `Select` do design system, não um `<select>` puro com `<option>` por
  string.
* `ShowCard` ganhou navegação pra rota de detalhe (`/espetaculos/{id}`,
  `catalog-session-detail`) e um badge de gênero — não previsto na versão
  original deste documento.
* Loading/erro/vazio usam os componentes do design system (`Skeleton`,
  `Alert`, `Button`), não HTML/Tailwind cru.
