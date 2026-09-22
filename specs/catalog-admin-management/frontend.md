---
status: draft
spec: catalog-admin-management
surface: frontend
created_at: 2026-09-03
updated_at: 2026-09-10
---

# Gestão de espetáculos e sessões (admin) — Frontend

**Resumo:** área `/admin/espetaculos` da feature `catalog` — o admin lista os
espetáculos (incluindo rascunhos), cria/edita/publica/despublica/exclui
espetáculo e cria/edita/cancela/exclui sessão. "Excluir" fica bloqueado para
sessão com ingresso vendido; o caminho é **cancelar**, com aviso de reembolso.
**RF:** RF08 · **RN:** — (RN04 na exibição de meia) · **Feature frontend:**
`catalog` (área admin)
**Contrato:** `docs.ludens/specs/catalog-admin-management/integration.md`
**Carregar antes:** skill `frontend-architecture` (todos os `references/`).

Stack: **Next.js (App Router) + TypeScript estrito**. Arquivos `.ts`/`.tsx`;
rota em `src/app/admin/espetaculos/page.tsx` (Server Component); componentes/hooks
com estado, handler ou hook de React levam `'use client'`; tipo = `z.infer` do
schema (nunca `interface` manual); `query-options.ts` obrigatório; toda mutation
invalida `adminShowList` + toast de sucesso + toast de erro; `components/ui/` é
apresentacional puro; barrel `index.ts` em toda subpasta. Aliases: `@catalog/*`,
`@catalog`, `@account`, `@web/*`.

> **Revisão de 2026-09-10:** a versão anterior deste documento assumia
> `server/services/` (aninhado), schemas em **camelCase**, um registro
> `API_ENDPOINTS`, e um `fetcher.get/post/patch/delete` que devolve `{ data }`.
> Nenhum dos quatro bate com o código real já mergeado em `identity-auth`
> (feature `account`): services em `services/` (não `server/services/`),
> schemas snake_case iguais ao backend, registro `endpoints` (minúsculo), e
> `fetcher<T>` que devolve `T` direto. Corrigido nesta revisão — inclusive uma
> lacuna real que essa correção expôs: o `fetcher` real não dá acesso ao corpo
> do erro (`{ detail }`) nem ao status HTTP, só um `Error('HTTP <status>')`
> genérico — necessário para as mensagens 409 diferenciadas que esta feature
> pede. Este documento passa a **editar `src/lib/fetcher.ts`** para expor os
> dois (ver §2, primeiro item). `catalog-show-search` e `catalog-session-detail`
> já foram escritas contra o código real; esta revisão alinha esta fatia à
> mesma base.
>
> **Revisão de 2026-09-21:** o código real (já mergeado) foi além desta
> revisão. `genre` (texto livre) virou `genre_id` (UUID) em todo formulário e
> schema, com um `<select>` populado por `useGenreList()` — não um campo de
> texto (`catalog-genre`). O registro `endpoints` real é **plano**, não
> aninhado em `catalog.admin.shows.*`/`catalog.admin.sessions.*`: é
> `endpoints.catalog.shows`, `showById`, `showPublish`, `showUnpublish`,
> `sessions`, `sessionById`, `sessionCancel`, e todos levam o prefixo
> `/api` (`API_BASE = '/api'`) — não existe namespace `/admin` separado, o
> backend unificou leitura pública/admin na mesma URL (ver
> `catalog-admin-management/integration.md`). `AdminShowSummaryResponse` virou
> `adminShowSummarySchema`/`adminShowSchema` (extend), acompanhando o backend.
> `AdminCatalogManager.tsx`, `ShowForm.tsx`, `SessionForm.tsx`, `SessionRow.tsx`,
> `ShowList.tsx`, `RequireAdmin.tsx`, `ConfirmCancelSessionDialog.tsx`,
> `admin.schema.ts`, `show.service.ts`, `useAdminCatalogMutations.ts` existem
> como planejado abaixo. O módulo `catalog` ganhou bem mais componentes desde
> então (`AdminGenreManager.tsx`, `AdminHub.tsx`, `GenreForm.tsx`,
> `GenreTable.tsx`, `ShowSessionsPanel.tsx` e outros) — são código real, mas
> donos/documentados por `catalog-genre`/`catalog-show-search`/
> `catalog-session-detail`, não reproduzidos aqui. Esta revisão corrige só os
> três blocos de código abaixo com o fato mais concreto (endpoints, schema de
> gênero, campo de gênero no `ShowForm`); o resto deste documento (mutations,
> `AdminCatalogManager`, `ShowList`, `SessionRow`, fluxo de cancelar/excluir)
> não foi reconferido linha a linha nesta revisão.

---

## 1. Arquivos (ordem de dependência)

Os itens marcados **"novo (ou editar)"** já podem existir se outra fatia do
módulo `catalog` (`catalog-show-search`, `catalog-session-detail`) mergeou
primeiro — checar antes de sobrescrever; a ordem real entre as três fatias de
frontend não é fixa (só a ordem de *backend* é: `catalog-admin-management`
primeiro).

| #   | Camada        | Caminho                                                             | Novo/Editar                                                              |
| --- | ------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| 1   | lib           | `src/lib/fetcher.ts`                                                | editar — expor `ApiError` (status + corpo)                               |
| 2   | config        | `tsconfig.json`                                                     | editar — bare `@account` e `@catalog`                                    |
| 3   | endpoints     | `src/routes/endpoints.ts`                                           | editar — grupo `catalog.admin` (grupo `catalog` pode já existir)         |
| 4   | schemas       | `src/features/catalog/schemas/admin.schema.ts`                      | novo — Zod (response + form DTO), snake_case                             |
| 5   | schemas       | `src/features/catalog/schemas/index.ts`                             | novo (ou editar) — barrel                                                |
| 6   | server/types  | `src/features/catalog/server/types/admin.types.ts`                  | novo — `z.infer`                                                         |
| 7   | server/types  | `src/features/catalog/server/types/index.ts`                        | novo (ou editar) — barrel                                                |
| 8   | server        | `src/features/catalog/server/index.ts`                              | novo (ou editar) — barrel                                                |
| 9   | services      | `src/features/catalog/services/show.service.ts`                     | novo (ou editar) — acrescenta os métodos admin ao mesmo `catalogService` |
| 10  | services      | `src/features/catalog/services/index.ts`                            | novo (ou editar) — barrel                                                |
| 11  | lib           | `src/features/catalog/lib/errors.ts`                                | novo — leitura do erro via `ApiError`                                    |
| 12  | lib           | `src/features/catalog/lib/format.ts`                                | novo — preço/data                                                        |
| 13  | lib           | `src/features/catalog/lib/index.ts`                                 | novo — barrel                                                            |
| 14  | constants     | `src/features/catalog/constants/catalog.constants.ts`               | novo — rótulos de status                                                 |
| 15  | constants     | `src/features/catalog/constants/index.ts`                           | novo — barrel                                                            |
| 16  | queries       | `src/features/catalog/hooks/queries/query-options.ts`               | novo (ou editar) — acrescenta `adminShowList`                            |
| 17  | queries       | `src/features/catalog/hooks/queries/useCatalogQueries.ts`           | novo (ou editar) — acrescenta `useAdminShowList`                         |
| 18  | queries       | `src/features/catalog/hooks/queries/index.ts`                       | novo (ou editar) — barrel                                                |
| 19  | mutations     | `src/features/catalog/hooks/mutations/useAdminCatalogMutations.ts`  | novo                                                                     |
| 20  | mutations     | `src/features/catalog/hooks/mutations/index.ts`                     | novo — barrel                                                            |
| 21  | forms         | `src/features/catalog/hooks/forms/useShowForm.ts`                   | novo                                                                     |
| 22  | forms         | `src/features/catalog/hooks/forms/useSessionForm.ts`                | novo                                                                     |
| 23  | forms         | `src/features/catalog/hooks/forms/index.ts`                         | novo — barrel                                                            |
| 24  | hooks         | `src/features/catalog/hooks/index.ts`                               | novo (ou editar) — barrel                                                |
| 25  | components/ui | `src/features/catalog/components/ui/ConfirmCancelSessionDialog.tsx` | novo — puro                                                              |
| 26  | components/ui | `src/features/catalog/components/ui/index.ts`                       | novo (ou editar) — barrel                                                |
| 27  | components    | `src/features/catalog/components/admin/ShowForm.tsx`                | novo — visual `'use client'`                                             |
| 28  | components    | `src/features/catalog/components/admin/SessionForm.tsx`             | novo — visual `'use client'`                                             |
| 29  | components    | `src/features/catalog/components/admin/SessionRow.tsx`              | novo — `'use client'`                                                    |
| 30  | components    | `src/features/catalog/components/admin/ShowList.tsx`                | novo — `'use client'`                                                    |
| 31  | components    | `src/features/catalog/components/admin/index.ts`                    | novo — barrel                                                            |
| 32  | components    | `src/features/catalog/components/AdminCatalogManager.tsx`           | novo — orchestration `'use client'`                                      |
| 33  | components    | `src/features/catalog/components/RequireAdmin.tsx`                  | novo — `'use client'` (ver nota abaixo)                                  |
| 34  | components    | `src/features/catalog/components/index.ts`                          | novo (ou editar) — barrel                                                |
| 35  | rota          | `src/app/admin/espetaculos/page.tsx`                                | novo — Server Component                                                  |
| 36  | feature root  | `src/features/catalog/index.ts`                                     | novo (ou editar) — API pública                                           |

Sem `README.md` de feature — não é convenção real do repo (`features/account`
não tem um).

---

## 2. Código

### `src/lib/fetcher.ts`

Editar o arquivo real: `handleResponse` hoje descarta o corpo do erro
(`throw new Error('HTTP ' + status)`), sem dar acesso a `{ detail }` nem ao
status. Esta feature precisa distinguir 409 de outros erros e mostrar a
mensagem do backend — por isso expõe `ApiError`. Mudança aditiva: quem não lê
`.status`/`.data` continua funcionando igual (identity-auth hoje só usa
mensagens fixas em `onError`).

```ts
// src/lib/fetcher.ts  — editar
export class ApiError extends Error {
  status: number;
  data: unknown;

  constructor(status: number, data: unknown) {
    super(`HTTP ${status}`);
    this.name = 'ApiError';
    this.status = status;
    this.data = data;
  }
}

async function handleResponse<T>(response: Response): Promise<T> {
  if (!response.ok) {
    const data = await response.json().catch(() => undefined);
    throw new ApiError(response.status, data);
  }

  if (response.status === 204) {
    return undefined as T;
  }

  return (await response.json()) as T;
}
```

(Resto do arquivo — `getAccessToken`, `setAccessToken`, `refreshAccessToken`,
`fetcher` — sem mudança; só a assinatura de `handleResponse` e a exportação
de `ApiError`.)

### `tsconfig.json`

Editar só o bloco `paths` — acrescentar o mapeamento **bare** de `@account`
(consumido para `RequireAuth`) e de `@catalog` (consumido pela rota). Os
wildcards `@catalog/*`/`@account/*` já existem desde o bootstrap do projeto.

```jsonc
// tsconfig.json  — editar (compilerOptions.paths)
"paths": {
  "@web/*": ["./src/*"],
  "@components/*": ["./src/components/*"],
  "@catalog": ["./src/features/catalog/index.ts"],
  "@catalog/*": ["./src/features/catalog/*"],
  "@booking/*": ["./src/features/booking/*"],
  "@checkout/*": ["./src/features/checkout/*"],
  "@account": ["./src/features/account/index.ts"],
  "@account/*": ["./src/features/account/*"]
}
```

### `src/routes/endpoints.ts`

Sem grupo `admin` aninhado, e sem namespace `/admin`: o backend unificou
leitura pública e admin na mesma URL (decide o formato da resposta pelo
token), então o registro real é **plano**, com prefixo `/api`, e é
compartilhado por esta fatia e por `catalog-show-search`/
`catalog-session-detail`/`catalog-genre`.

```ts
// src/routes/endpoints.ts  — real (trecho relevante a esta fatia)
const API_BASE = '/api';
const CATALOG_BASE = `${API_BASE}/catalog`;

export const endpoints = {
  // ...auth, users...
  catalog: {
    shows: `${CATALOG_BASE}/shows`,
    showById: (id: string) => `${CATALOG_BASE}/shows/${id}`,
    showPublish: (id: string) => `${CATALOG_BASE}/shows/${id}/publish`,
    showUnpublish: (id: string) => `${CATALOG_BASE}/shows/${id}/unpublish`,
    genres: `${CATALOG_BASE}/genres`,
    genreById: (id: string) => `${CATALOG_BASE}/genres/${id}`,
    sessions: `${CATALOG_BASE}/sessions/`,
    sessionById: (id: string) => `${CATALOG_BASE}/sessions/${id}`,
    sessionCancel: (id: string) => `${CATALOG_BASE}/sessions/${id}/cancel`,
  },
} as const;
```

Criar sessão é `POST` em `endpoints.catalog.sessions` (com `show_id` no
corpo, não na URL) — não existe `.../shows/{id}/sessions`.

### `src/features/catalog/schemas/admin.schema.ts`

Campos snake_case — mesma grafia do backend real (`AdminShowResponse`,
`AdminSessionResponse`).

```ts
// src/features/catalog/schemas/admin.schema.ts  — novo
import { z } from 'zod';

export const showStatusEnum = z.enum(['draft', 'published']);
export const sessionStatusEnum = z.enum(['on_sale', 'closed', 'cancelled']);

// ---- Response (contrato do backend) ----
export const adminSessionSchema = z.object({
  id: z.string().uuid(),
  show_id: z.string().uuid(),
  starts_at: z.coerce.date(),
  venue: z.string(),
  capacity: z.number().int(),
  full_price: z.number(),
  half_price: z.number(),
  status: sessionStatusEnum,
  tickets_sold: z.number().int(),
  reserved_open: z.number().int(),
  can_delete: z.boolean(),
});

export const adminShowSummarySchema = z.object({
  id: z.string().uuid(),
  title: z.string(),
  synopsis: z.string(),
  image_url: z.string(),
  genre_id: z.string().uuid(),
  genre: z.string(),
  status: showStatusEnum,
});

// GET /catalog/shows/{id} (com token admin), POST e PUT — detalhe completo,
// com sessions. GET /catalog/shows (lista/busca, mesmo endpoint do público)
// devolve só o resumo acima, sem sessions.
export const adminShowSchema = adminShowSummarySchema.extend({
  sessions: z.array(adminSessionSchema),
});

export const adminShowSummaryListSchema = z.array(adminShowSummarySchema);

// ---- Request DTO (guia do formulário) ----
// Sem image_url: a imagem é atribuída pelo backend na criação (spec.md §6,
// débito técnico de upload real). genre_id vem de um <select> alimentado por
// useGenreList() (catalog-genre) — não é mais um campo de texto livre.
export const showFormSchema = z.object({
  title: z.string().min(1, 'Informe o título').max(200),
  synopsis: z.string().min(1, 'Informe a sinopse').max(5000),
  genre_id: z.string().uuid('Selecione um gênero'),
});

export const sessionFormSchema = z.object({
  // valor do <input type="datetime-local"> (hora local, sem fuso)
  starts_at: z
    .string()
    .min(1, 'Informe a data e a hora')
    .refine((v) => !Number.isNaN(Date.parse(v)), 'Data inválida')
    .refine((v) => new Date(v).getTime() > Date.now(), 'A data da sessão deve ser futura'),
  venue: z.string().min(1, 'Informe o local').max(200),
  capacity: z
    .number({ invalid_type_error: 'Informe a capacidade' })
    .int('A capacidade deve ser um número inteiro')
    .positive('A capacidade deve ser maior que zero')
    .max(100_000),
  full_price: z
    .number({ invalid_type_error: 'Informe o preço da inteira' })
    .positive('O preço deve ser maior que zero'),
});
```

### `src/features/catalog/schemas/index.ts`

```ts
// src/features/catalog/schemas/index.ts  — novo (ou editar)
export * from './admin.schema';
// export * from './session.schema';  — se catalog-session-detail já mergeou
// export * from './show.schema';     — se catalog-show-search já mergeou
```

### `src/features/catalog/server/types/admin.types.ts`

```ts
// src/features/catalog/server/types/admin.types.ts  — novo
import type { z } from 'zod';

import type {
  adminSessionSchema,
  adminShowSchema,
  sessionFormSchema,
  showFormSchema,
} from '@catalog/schemas';

export type AdminShow = z.infer<typeof adminShowSchema>;
export type AdminSession = z.infer<typeof adminSessionSchema>;
export type ShowFormValues = z.infer<typeof showFormSchema>;
export type SessionFormValues = z.infer<typeof sessionFormSchema>;
```

### `src/features/catalog/server/types/index.ts`

```ts
// src/features/catalog/server/types/index.ts  — novo (ou editar)
export * from './admin.types';
```

### `src/features/catalog/server/index.ts`

```ts
// src/features/catalog/server/index.ts  — novo (ou editar)
export * from './types';
```

### `src/features/catalog/services/show.service.ts`

Acrescenta os métodos admin ao **mesmo** objeto `catalogService` das outras
fatias — um serviço por feature (padrão real de `authService`), não um
objeto por área. Sem `.parse()` em runtime (mesma decisão de
`catalog-show-search`: os schemas Zod só derivam o tipo).

Sem grupo `admin` no service, igual aos endpoints (ver §2 acima): um único
`catalogService`, compartilhado com `catalog-show-search`/
`catalog-session-detail`/`catalog-genre`. `listAdminShows` busca
`GET /catalog/shows?size=48` (o endpoint público, que devolve
`Page<AdminShowSummary>` quando o token é de admin) e devolve só `items` —
sem paginação na UI desta fatia ainda. Criar/editar sessão sempre passa
`show_id` no corpo (`toSessionPayload` recebe `showId` mesmo em edição,
porque o schema é compartilhado e o campo é obrigatório, ainda que o backend
o ignore em `PUT`). Abaixo, só o subconjunto admin (write-path) real;
`fetchShows`/`fetchGenres`/`fetchShowById`/`fetchSessionById` são código
real também, mas donos de `catalog-show-search`/`catalog-session-detail`/
`catalog-genre`, não reproduzidos aqui:

```ts
// src/features/catalog/services/show.service.ts — real, subconjunto admin
import { fetcher } from '@web/lib/fetcher';
import { endpoints } from '@web/routes/endpoints';

import type { AdminSession, AdminShow, AdminShowSummary, SessionFormValues, ShowFormValues } from '@catalog/server/types';

// O <input type="datetime-local"> devolve hora local sem fuso; o backend
// exige ISO 8601 com offset. show_id vai no corpo (SessionRequest) — não
// existe rota aninhada /shows/{id}/sessions.
function toSessionPayload(showId: string, values: SessionFormValues) {
  return {
    show_id: showId,
    starts_at: new Date(values.starts_at).toISOString(),
    venue: values.venue,
    capacity: values.capacity,
    full_price: values.full_price,
  };
}

function reviveSession(session: AdminSession): AdminSession {
  return { ...session, starts_at: new Date(session.starts_at) };
}

function reviveShow(show: AdminShow): AdminShow {
  return { ...show, sessions: show.sessions.map(reviveSession) };
}

export const catalogService = {
  async listAdminShows() {
    const page = await fetcher<{ items: AdminShowSummary[] }>(`${endpoints.catalog.shows}?size=48`, {
      method: 'GET',
    });
    return page.items;
  },

  async getAdminShow(id: string) {
    const show = await fetcher<AdminShow>(endpoints.catalog.showById(id), { method: 'GET' });
    return reviveShow(show);
  },

  async createShow(values: ShowFormValues) {
    const show = await fetcher<AdminShow>(endpoints.catalog.shows, {
      method: 'POST',
      body: JSON.stringify(values),
    });
    return reviveShow(show);
  },

  async updateShow(id: string, values: ShowFormValues) {
    const show = await fetcher<AdminShow>(endpoints.catalog.showById(id), {
      method: 'PUT',
      body: JSON.stringify(values),
    });
    return reviveShow(show);
  },

  publishShow(id: string) {
    return fetcher<void>(endpoints.catalog.showPublish(id), { method: 'POST' });
  },

  unpublishShow(id: string) {
    return fetcher<void>(endpoints.catalog.showUnpublish(id), { method: 'POST' });
  },

  deleteShow(id: string) {
    return fetcher<void>(endpoints.catalog.showById(id), { method: 'DELETE' });
  },

  async createSession(showId: string, values: SessionFormValues) {
    const session = await fetcher<AdminSession>(endpoints.catalog.sessions, {
      method: 'POST',
      body: JSON.stringify(toSessionPayload(showId, values)),
    });
    return reviveSession(session);
  },

  async updateSession(sessionId: string, showId: string, values: SessionFormValues) {
    const session = await fetcher<AdminSession>(endpoints.catalog.sessionById(sessionId), {
      method: 'PUT',
      body: JSON.stringify(toSessionPayload(showId, values)),
    });
    return reviveSession(session);
  },

  cancelSession(sessionId: string) {
    return fetcher<void>(endpoints.catalog.sessionCancel(sessionId), { method: 'POST' });
  },

  deleteSession(sessionId: string) {
    return fetcher<void>(endpoints.catalog.sessionById(sessionId), { method: 'DELETE' });
  },
};
```

### `src/features/catalog/services/index.ts`

```ts
// src/features/catalog/services/index.ts  — novo (ou editar)
export * from './show.service';
```

### `src/features/catalog/lib/errors.ts`

Lê `ApiError` (agora exportada por `@web/lib/fetcher` — ver §2, primeiro
item). Envelope do backend real: `{ detail: string }` (erro de negócio) ou
`{ detail: [{ field, message }] }` (erro de forma).

```ts
// src/features/catalog/lib/errors.ts  — novo
import { ApiError } from '@web/lib/fetcher';

export function apiErrorStatus(error: unknown): number | undefined {
  return error instanceof ApiError ? error.status : undefined;
}

export function apiErrorMessage(error: unknown, fallback: string): string {
  if (error instanceof ApiError) {
    const detail = (error.data as { detail?: unknown } | undefined)?.detail;
    if (typeof detail === 'string' && detail.trim() !== '') {
      return detail;
    }
    if (Array.isArray(detail) && detail.length > 0) {
      const first = detail[0] as { message?: unknown };
      if (typeof first?.message === 'string') {
        return first.message;
      }
    }
  }
  return fallback;
}
```

### `src/features/catalog/lib/format.ts`

```ts
// src/features/catalog/lib/format.ts  — novo
const BRL = new Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' });
const DATE_TIME = new Intl.DateTimeFormat('pt-BR', {
  dateStyle: 'short',
  timeStyle: 'short',
});

export function formatPriceBRL(value: number): string {
  return BRL.format(value);
}

export function formatDateTime(value: Date): string {
  return DATE_TIME.format(value);
}

// Converte um Date para o formato aceito por <input type="datetime-local">
// (yyyy-MM-ddTHH:mm em hora local).
export function toDateTimeLocalValue(value: Date): string {
  const offsetMs = value.getTimezoneOffset() * 60_000;
  return new Date(value.getTime() - offsetMs).toISOString().slice(0, 16);
}
```

### `src/features/catalog/lib/index.ts`

```ts
// src/features/catalog/lib/index.ts  — novo
export * from './errors';
export * from './format';
```

### `src/features/catalog/constants/catalog.constants.ts`

```ts
// src/features/catalog/constants/catalog.constants.ts  — novo
import type { AdminSession, AdminShow } from '@catalog/server/types';

export const SHOW_STATUS_LABELS: Record<AdminShow['status'], string> = {
  draft: 'Rascunho',
  published: 'Publicado',
};

export const SESSION_STATUS_LABELS: Record<AdminSession['status'], string> = {
  on_sale: 'À venda',
  closed: 'Encerrada',
  cancelled: 'Cancelada',
};
```

### `src/features/catalog/constants/index.ts`

```ts
// src/features/catalog/constants/index.ts  — novo
export * from './catalog.constants';
```

### `src/features/catalog/hooks/queries/query-options.ts`

```ts
// src/features/catalog/hooks/queries/query-options.ts  — novo (ou editar)
import { queryOptions } from '@tanstack/react-query';

import { catalogService } from '@catalog/services';

export const catalogQueryKeys = {
  all: ['catalog'] as const,
  admin: {
    shows: () => [...catalogQueryKeys.all, 'admin', 'shows'] as const,
    showList: () => [...catalogQueryKeys.admin.shows(), 'list'] as const,
  },
};

export const catalogQueryOptions = {
  // ...showList, genreList, showDetail, sessionDetail, se já existirem...

  adminShowList: () =>
    queryOptions({
      queryKey: catalogQueryKeys.admin.showList(),
      queryFn: () => catalogService.listAdminShows(),
      // Lista de gestão: dado pouco volátil; sem polling.
      staleTime: 10_000,
    }),
};
```

### `src/features/catalog/hooks/queries/useCatalogQueries.ts`

```ts
// src/features/catalog/hooks/queries/useCatalogQueries.ts  — novo (ou editar)
export function useAdminShowList() {
  return useQuery(catalogQueryOptions.adminShowList());
}
```

### `src/features/catalog/hooks/queries/index.ts`

```ts
// src/features/catalog/hooks/queries/index.ts  — novo (ou editar)
export * from './query-options';
export * from './useCatalogQueries';
```

### `src/features/catalog/hooks/mutations/useAdminCatalogMutations.ts`

```ts
// src/features/catalog/hooks/mutations/useAdminCatalogMutations.ts  — novo
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';
import { toast } from 'sonner';

import { catalogQueryKeys } from '@catalog/hooks/queries';
import { apiErrorMessage, apiErrorStatus } from '@catalog/lib';
import { catalogService } from '@catalog/services';

import type { SessionFormValues, ShowFormValues } from '@catalog/server/types';

export function useAdminCatalogMutations() {
  const queryClient = useQueryClient();
  const invalidate = () =>
    queryClient.invalidateQueries({ queryKey: catalogQueryKeys.admin.showList() });

  const createShowMutation = useMutation({
    mutationFn: (values: ShowFormValues) => catalogService.createShow(values),
    onSuccess: () => {
      void invalidate();
      toast.success('Espetáculo criado.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível criar o espetáculo.')),
  });

  const updateShowMutation = useMutation({
    mutationFn: ({ id, values }: { id: string; values: ShowFormValues }) =>
      catalogService.updateShow(id, values),
    onSuccess: () => {
      void invalidate();
      toast.success('Espetáculo atualizado.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível salvar o espetáculo.')),
  });

  const publishShowMutation = useMutation({
    mutationFn: (id: string) => catalogService.publishShow(id),
    onSuccess: () => {
      void invalidate();
      toast.success('Espetáculo publicado.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível publicar o espetáculo.')),
  });

  const unpublishShowMutation = useMutation({
    mutationFn: (id: string) => catalogService.unpublishShow(id),
    onSuccess: () => {
      void invalidate();
      toast.success('Espetáculo despublicado.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível despublicar o espetáculo.')),
  });

  const deleteShowMutation = useMutation({
    mutationFn: (id: string) => catalogService.deleteShow(id),
    onSuccess: () => {
      void invalidate();
      toast.success('Espetáculo excluído.');
    },
    onError: (error) => {
      if (apiErrorStatus(error) === 409) {
        toast.error(
          'Há sessões com ingressos vendidos. Cancele essas sessões antes de excluir o espetáculo.',
        );
        return;
      }
      toast.error(apiErrorMessage(error, 'Não foi possível excluir o espetáculo.'));
    },
  });

  const createSessionMutation = useMutation({
    mutationFn: ({ showId, values }: { showId: string; values: SessionFormValues }) =>
      catalogService.createSession(showId, values),
    onSuccess: () => {
      void invalidate();
      toast.success('Sessão criada.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível criar a sessão.')),
  });

  const updateSessionMutation = useMutation({
    mutationFn: ({ id, values }: { id: string; values: SessionFormValues }) =>
      catalogService.updateSession(id, values),
    onSuccess: () => {
      void invalidate();
      toast.success('Sessão atualizada.');
    },
    onError: (error) => {
      if (apiErrorStatus(error) === 409) {
        toast.error(apiErrorMessage(error, 'Já há ingressos comprometidos nesta sessão.'));
        return;
      }
      toast.error(apiErrorMessage(error, 'Não foi possível salvar a sessão.'));
    },
  });

  const cancelSessionMutation = useMutation({
    mutationFn: (id: string) => catalogService.cancelSession(id),
    onSuccess: () => {
      void invalidate();
      toast.success('Sessão cancelada. Os compradores entrarão na fila de reembolso.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível cancelar a sessão.')),
  });

  const deleteSessionMutation = useMutation({
    mutationFn: (id: string) => catalogService.deleteSession(id),
    onSuccess: () => {
      void invalidate();
      toast.success('Sessão excluída.');
    },
    onError: (error) => {
      if (apiErrorStatus(error) === 409) {
        toast.error(
          'Esta sessão já vendeu ingressos. Cancele a sessão em vez de excluir — os compradores serão reembolsados.',
        );
        return;
      }
      toast.error(apiErrorMessage(error, 'Não foi possível excluir a sessão.'));
    },
  });

  return {
    createShowMutation,
    updateShowMutation,
    publishShowMutation,
    unpublishShowMutation,
    deleteShowMutation,
    createSessionMutation,
    updateSessionMutation,
    cancelSessionMutation,
    deleteSessionMutation,
  };
}
```

### `src/features/catalog/hooks/mutations/index.ts`

```ts
// src/features/catalog/hooks/mutations/index.ts  — novo
export * from './useAdminCatalogMutations';
```

### `src/features/catalog/hooks/forms/useShowForm.ts`

```ts
// src/features/catalog/hooks/forms/useShowForm.ts  — novo
'use client';

import { useEffect } from 'react';

import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';

import { showFormSchema } from '@catalog/schemas';

import type { AdminShow, ShowFormValues } from '@catalog/server/types';

const EMPTY: ShowFormValues = { title: '', synopsis: '', genre_id: '' };

export function useShowForm(editing: AdminShow | null) {
  const form = useForm<ShowFormValues>({
    resolver: zodResolver(showFormSchema),
    mode: 'onSubmit',
    defaultValues: EMPTY,
  });

  useEffect(() => {
    if (editing) {
      form.reset({
        title: editing.title,
        synopsis: editing.synopsis,
        genre_id: editing.genre_id,
      });
    } else {
      form.reset(EMPTY);
    }
  }, [editing, form]);

  return form;
}
```

### `src/features/catalog/hooks/forms/useSessionForm.ts`

```ts
// src/features/catalog/hooks/forms/useSessionForm.ts  — novo
'use client';

import { useEffect } from 'react';

import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';

import { sessionFormSchema } from '@catalog/schemas';
import { toDateTimeLocalValue } from '@catalog/lib';

import type { AdminSession, SessionFormValues } from '@catalog/server/types';

const EMPTY: SessionFormValues = { starts_at: '', venue: '', capacity: 0, full_price: 0 };

export function useSessionForm(editing: AdminSession | null) {
  const form = useForm<SessionFormValues>({
    resolver: zodResolver(sessionFormSchema),
    mode: 'onSubmit',
    defaultValues: EMPTY,
  });

  useEffect(() => {
    if (editing) {
      form.reset({
        starts_at: toDateTimeLocalValue(editing.starts_at),
        venue: editing.venue,
        capacity: editing.capacity,
        full_price: editing.full_price,
      });
    } else {
      form.reset(EMPTY);
    }
  }, [editing, form]);

  return form;
}
```

### `src/features/catalog/hooks/forms/index.ts`

```ts
// src/features/catalog/hooks/forms/index.ts  — novo
export * from './useSessionForm';
export * from './useShowForm';
```

### `src/features/catalog/hooks/index.ts`

```ts
// src/features/catalog/hooks/index.ts  — novo (ou editar)
export * from './forms';
export * from './mutations';
export * from './queries';
// export * from './useShowFilters';  — se catalog-show-search já mergeou
```

### `src/features/catalog/components/ui/ConfirmCancelSessionDialog.tsx`

Apresentacional puro — sem hooks de dados, só props e estado visual mínimo
(fechar no Esc). Avisa do reembolso (RN02/RF07). Props em camelCase (é
identificador local do componente, não campo de contrato).

```tsx
// src/features/catalog/components/ui/ConfirmCancelSessionDialog.tsx  — novo
'use client';

import { useEffect } from 'react';

interface ConfirmCancelSessionDialogProps {
  open: boolean;
  sessionLabel: string;
  ticketsSold: number;
  isPending: boolean;
  onConfirm: () => void;
  onClose: () => void;
}

export function ConfirmCancelSessionDialog({
  open,
  sessionLabel,
  ticketsSold,
  isPending,
  onConfirm,
  onClose,
}: ConfirmCancelSessionDialogProps) {
  useEffect(() => {
    if (!open) return;
    const onKey = (event: KeyboardEvent) => {
      if (event.key === 'Escape') onClose();
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [open, onClose]);

  if (!open) return null;

  return (
    <div
      role="presentation"
      onClick={onClose}
      className="fixed inset-0 z-50 flex items-center justify-center bg-black/40 p-4"
    >
      <div
        role="alertdialog"
        aria-modal="true"
        aria-labelledby="cancel-session-title"
        aria-describedby="cancel-session-desc"
        onClick={(event) => event.stopPropagation()}
        className="w-full max-w-md rounded-lg bg-white p-6 shadow-xl"
      >
        <h2 id="cancel-session-title" className="text-lg font-semibold">
          Cancelar a sessão?
        </h2>
        <p id="cancel-session-desc" className="mt-2 text-sm text-gray-700">
          {sessionLabel}
        </p>
        <p className="mt-3 text-sm text-gray-700">
          {ticketsSold > 0
            ? `Esta sessão já vendeu ${ticketsSold} ingresso(s). Ao cancelar, todos os
               compradores entram na fila de reembolso conforme a política vigente, e
               as reservas em aberto são liberadas. A ação não pode ser desfeita.`
            : `A sessão sai da vitrine imediatamente. A ação não pode ser desfeita.`}
        </p>
        <div className="mt-6 flex justify-end gap-3">
          <button
            type="button"
            onClick={onClose}
            disabled={isPending}
            className="min-h-11 rounded-md border border-gray-300 px-4 text-sm font-medium"
          >
            Voltar
          </button>
          <button
            type="button"
            onClick={onConfirm}
            disabled={isPending}
            className="min-h-11 rounded-md bg-red-600 px-4 text-sm font-medium text-white disabled:opacity-60"
          >
            {isPending ? 'Cancelando...' : 'Cancelar a sessão'}
          </button>
        </div>
      </div>
    </div>
  );
}
```

### `src/features/catalog/components/ui/index.ts`

```ts
// src/features/catalog/components/ui/index.ts  — novo (ou editar)
export * from './ConfirmCancelSessionDialog';
// export * from './Pagination';           — se catalog-show-search já mergeou
// export * from './SessionDetailView';    — se catalog-session-detail já mergeou
// export * from './ShowCard';             — se catalog-show-search já mergeou
// export * from './ShowSessionsView';     — se catalog-session-detail já mergeou
```

### `src/features/catalog/components/admin/ShowForm.tsx`

Formulário visual — recebe `form`, `onSubmit`, `isPending`; não conhece
mutation. **shadcn/ui entrou no repo** desde a revisão original deste
documento (contradiz a nota do §7 abaixo, que dizia o contrário) — o
componente real usa `Form`/`FormField`/`Select` do design system, não
elementos nativos com Tailwind cru, e o gênero é um `<Select>` alimentado por
`useGenreList()` (`catalog-genre`), não um campo de texto. Reprodução fiel do
componente real:

```tsx
// src/features/catalog/components/admin/ShowForm.tsx — real
'use client';

import Link from 'next/link';

import { Sparkles } from 'lucide-react';
import type { FormEventHandler } from 'react';
import type { UseFormReturn } from 'react-hook-form';

import { Button } from '@components/ui/button';
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@components/ui/form';
import { Input } from '@components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@components/ui/select';
import { Textarea } from '@components/ui/textarea';

import { useGenreList } from '@catalog/hooks/queries';

import type { ShowFormValues } from '@catalog/server/types';

interface ShowFormProps {
  form: UseFormReturn<ShowFormValues>;
  onSubmit: FormEventHandler<HTMLFormElement>;
  onCancel: () => void;
  isPending: boolean;
  mode: 'create' | 'edit';
}

export function ShowForm({ form, onSubmit, onCancel, isPending, mode }: ShowFormProps) {
  const genresQuery = useGenreList();
  const genres = genresQuery.data ?? [];

  return (
    <Form {...form}>
      <form onSubmit={onSubmit} className="flex flex-col gap-5">
        {mode === 'create' ? (
          <p className="flex items-center gap-2 rounded-lg bg-primary/5 px-3 py-2 text-xs text-muted-foreground">
            <Sparkles className="size-3.5 shrink-0 text-primary" />
            A imagem de capa é sorteada automaticamente do pool padrão assim que o espetáculo é criado.
          </p>
        ) : null}

        <FormField
          control={form.control}
          name="title"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Título</FormLabel>
              <FormControl>
                <Input placeholder="Ex.: Hamlet" disabled={isPending} className="min-h-11" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="synopsis"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Sinopse</FormLabel>
              <FormControl>
                <Textarea rows={4} placeholder="Do que se trata o espetáculo?" disabled={isPending} {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="genre_id"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Categoria / gênero</FormLabel>
              <FormControl>
                <Select value={field.value} onValueChange={field.onChange} disabled={isPending || genres.length === 0}>
                  <SelectTrigger className="min-h-11 w-full">
                    <SelectValue placeholder="Selecione um gênero" />
                  </SelectTrigger>
                  <SelectContent>
                    {genres.map((genre) => (
                      <SelectItem key={genre.id} value={genre.id}>
                        {genre.name}
                      </SelectItem>
                    ))}
                  </SelectContent>
                </Select>
              </FormControl>

              {!genresQuery.isLoading && genres.length === 0 ? (
                <p className="text-xs text-muted-foreground">
                  Nenhum gênero cadastrado —{' '}
                  <Link href="/admin/generos" className="underline underline-offset-2">
                    crie um primeiro
                  </Link>
                  .
                </p>
              ) : null}

              <FormMessage />
            </FormItem>
          )}
        />

        <div className="flex justify-end gap-3 pt-1">
          <Button type="button" variant="outline" className="min-h-11 px-4" onClick={onCancel} disabled={isPending}>
            Cancelar
          </Button>
          <Button type="submit" className="min-h-11 px-5" disabled={isPending}>
            {isPending ? 'Salvando...' : 'Salvar'}
          </Button>
        </div>
      </form>
    </Form>
  );
}
```

### `src/features/catalog/components/admin/SessionForm.tsx`

```tsx
// src/features/catalog/components/admin/SessionForm.tsx  — novo
'use client';

import type { FormEventHandler } from 'react';
import type { UseFormReturn } from 'react-hook-form';

import type { SessionFormValues } from '@catalog/server/types';

interface SessionFormProps {
  form: UseFormReturn<SessionFormValues>;
  onSubmit: FormEventHandler<HTMLFormElement>;
  onCancel: () => void;
  isPending: boolean;
  mode: 'create' | 'edit';
  ticketsSold: number;
}

export function SessionForm({
  form,
  onSubmit,
  onCancel,
  isPending,
  mode,
  ticketsSold,
}: SessionFormProps) {
  const { register, formState } = form;
  const { errors } = formState;

  return (
    <form onSubmit={onSubmit} className="space-y-4 rounded-lg border border-gray-200 p-4">
      <h3 className="text-base font-semibold">
        {mode === 'create' ? 'Nova sessão' : 'Editar sessão'}
      </h3>

      {mode === 'edit' && ticketsSold > 0 ? (
        <p className="rounded-md bg-amber-50 p-2 text-sm text-amber-800">
          Esta sessão já vendeu {ticketsSold} ingresso(s). Alterar data/horário
          notifica os compradores. A capacidade não pode ficar abaixo do total já
          comprometido.
        </p>
      ) : null}

      <div className="flex flex-col gap-1">
        <label htmlFor="session-starts" className="text-sm font-medium">
          Data e hora
        </label>
        <input
          id="session-starts"
          type="datetime-local"
          {...register('starts_at')}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
        {errors.starts_at ? (
          <p className="text-sm text-red-600">{errors.starts_at.message}</p>
        ) : null}
      </div>

      <div className="flex flex-col gap-1">
        <label htmlFor="session-venue" className="text-sm font-medium">
          Local
        </label>
        <input
          id="session-venue"
          type="text"
          {...register('venue')}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
        {errors.venue ? (
          <p className="text-sm text-red-600">{errors.venue.message}</p>
        ) : null}
      </div>

      <div className="flex flex-col gap-1">
        <label htmlFor="session-capacity" className="text-sm font-medium">
          Capacidade
        </label>
        <input
          id="session-capacity"
          type="number"
          min={1}
          {...register('capacity', { valueAsNumber: true })}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
        {errors.capacity ? (
          <p className="text-sm text-red-600">{errors.capacity.message}</p>
        ) : null}
      </div>

      <div className="flex flex-col gap-1">
        <label htmlFor="session-price" className="text-sm font-medium">
          Preço da inteira (R$)
        </label>
        <input
          id="session-price"
          type="number"
          min={0}
          step="0.01"
          {...register('full_price', { valueAsNumber: true })}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
        {errors.full_price ? (
          <p className="text-sm text-red-600">{errors.full_price.message}</p>
        ) : null}
        <p className="text-xs text-gray-500">
          A meia-entrada é sempre 50% da inteira (calculada pelo sistema).
        </p>
      </div>

      <div className="flex gap-3">
        <button
          type="submit"
          disabled={isPending}
          className="min-h-11 rounded-md bg-gray-900 px-4 text-sm font-medium text-white disabled:opacity-60"
        >
          {isPending ? 'Salvando...' : 'Salvar'}
        </button>
        <button
          type="button"
          onClick={onCancel}
          disabled={isPending}
          className="min-h-11 rounded-md border border-gray-300 px-4 text-sm font-medium"
        >
          Cancelar
        </button>
      </div>
    </form>
  );
}
```

### `src/features/catalog/components/admin/SessionRow.tsx`

```tsx
// src/features/catalog/components/admin/SessionRow.tsx  — novo
'use client';

import { SESSION_STATUS_LABELS } from '@catalog/constants';
import { formatDateTime, formatPriceBRL } from '@catalog/lib';

import type { AdminSession } from '@catalog/server/types';

interface SessionRowProps {
  session: AdminSession;
  onEdit: () => void;
  onCancel: () => void;
  onDelete: () => void;
}

export function SessionRow({ session, onEdit, onCancel, onDelete }: SessionRowProps) {
  const canCancel = session.status === 'on_sale';

  return (
    <li className="flex flex-wrap items-center justify-between gap-2 border-t border-gray-100 py-2 text-sm">
      <div className="flex flex-wrap items-center gap-x-3 gap-y-1">
        <span className="font-medium">{formatDateTime(session.starts_at)}</span>
        <span className="text-gray-600">{session.venue}</span>
        <span className="text-gray-600">Cap. {session.capacity}</span>
        <span className="text-gray-600">
          Inteira {formatPriceBRL(session.full_price)} · Meia{' '}
          {formatPriceBRL(session.half_price)}
        </span>
        <span className="rounded bg-gray-100 px-2 py-0.5 text-xs">
          {SESSION_STATUS_LABELS[session.status]}
        </span>
        <span className="text-gray-600">Vendidos: {session.tickets_sold}</span>
      </div>

      <div className="flex gap-2">
        <button
          type="button"
          onClick={onEdit}
          className="min-h-11 rounded-md border border-gray-300 px-3 text-sm"
        >
          Editar
        </button>
        <button
          type="button"
          onClick={onCancel}
          disabled={!canCancel}
          title={canCancel ? undefined : 'Só sessões à venda podem ser canceladas'}
          className="min-h-11 rounded-md border border-amber-300 px-3 text-sm text-amber-800 disabled:opacity-40"
        >
          Cancelar sessão
        </button>
        <button
          type="button"
          onClick={onDelete}
          disabled={!session.can_delete}
          title={
            session.can_delete
              ? undefined
              : 'Sessão com ingressos vendidos não pode ser excluída — cancele em vez disso'
          }
          className="min-h-11 rounded-md border border-red-300 px-3 text-sm text-red-700 disabled:opacity-40"
        >
          Excluir
        </button>
      </div>
    </li>
  );
}
```

### `src/features/catalog/components/admin/ShowList.tsx`

```tsx
// src/features/catalog/components/admin/ShowList.tsx  — novo
'use client';

import { SHOW_STATUS_LABELS } from '@catalog/constants';

import { SessionRow } from './SessionRow';

import type { AdminSession, AdminShow } from '@catalog/server/types';

interface ShowListProps {
  shows: AdminShow[];
  onEditShow: (show: AdminShow) => void;
  onPublish: (show: AdminShow) => void;
  onUnpublish: (show: AdminShow) => void;
  onDeleteShow: (show: AdminShow) => void;
  onNewSession: (showId: string) => void;
  onEditSession: (showId: string, session: AdminSession) => void;
  onCancelSession: (session: AdminSession) => void;
  onDeleteSession: (session: AdminSession) => void;
}

export function ShowList({
  shows,
  onEditShow,
  onPublish,
  onUnpublish,
  onDeleteShow,
  onNewSession,
  onEditSession,
  onCancelSession,
  onDeleteSession,
}: ShowListProps) {
  return (
    <div className="space-y-6">
      {shows.map((show) => (
        <section key={show.id} className="rounded-lg border border-gray-200 p-4">
          <header className="flex flex-wrap items-center justify-between gap-2">
            <div className="flex items-center gap-3">
              <h2 className="text-lg font-semibold">{show.title}</h2>
              <span className="rounded bg-gray-100 px-2 py-0.5 text-xs">
                {SHOW_STATUS_LABELS[show.status]}
              </span>
              <span className="text-sm text-gray-500">{show.genre}</span>
            </div>
            <div className="flex gap-2">
              <button
                type="button"
                onClick={() => onEditShow(show)}
                className="min-h-11 rounded-md border border-gray-300 px-3 text-sm"
              >
                Editar
              </button>
              {show.status === 'draft' ? (
                <button
                  type="button"
                  onClick={() => onPublish(show)}
                  className="min-h-11 rounded-md border border-green-300 px-3 text-sm text-green-700"
                >
                  Publicar
                </button>
              ) : (
                <button
                  type="button"
                  onClick={() => onUnpublish(show)}
                  className="min-h-11 rounded-md border border-gray-300 px-3 text-sm"
                >
                  Despublicar
                </button>
              )}
              <button
                type="button"
                onClick={() => onDeleteShow(show)}
                className="min-h-11 rounded-md border border-red-300 px-3 text-sm text-red-700"
              >
                Excluir
              </button>
            </div>
          </header>

          <p className="mt-2 text-sm text-gray-600">{show.synopsis}</p>

          <div className="mt-4">
            <div className="flex items-center justify-between">
              <h3 className="text-sm font-semibold">Sessões</h3>
              <button
                type="button"
                onClick={() => onNewSession(show.id)}
                className="min-h-11 rounded-md border border-gray-300 px-3 text-sm"
              >
                Nova sessão
              </button>
            </div>

            {show.sessions.length === 0 ? (
              <p className="mt-2 text-sm text-gray-500">
                Nenhuma sessão. Um espetáculo sem sessão futura não aparece na
                vitrine.
              </p>
            ) : (
              <ul className="mt-2">
                {show.sessions.map((session) => (
                  <SessionRow
                    key={session.id}
                    session={session}
                    onEdit={() => onEditSession(show.id, session)}
                    onCancel={() => onCancelSession(session)}
                    onDelete={() => onDeleteSession(session)}
                  />
                ))}
              </ul>
            )}
          </div>
        </section>
      ))}
    </div>
  );
}
```

### `src/features/catalog/components/admin/index.ts`

```ts
// src/features/catalog/components/admin/index.ts  — novo
export * from './SessionForm';
export * from './SessionRow';
export * from './ShowForm';
export * from './ShowList';
```

### `src/features/catalog/components/AdminCatalogManager.tsx`

Orchestration — conecta query, mutations e forms; trata loading/error/empty;
controla os painéis de formulário e o diálogo de cancelamento.

```tsx
// src/features/catalog/components/AdminCatalogManager.tsx  — novo
'use client';

import { useState } from 'react';

import { SessionForm, ShowForm, ShowList } from '@catalog/components/admin';
import { ConfirmCancelSessionDialog } from '@catalog/components/ui';
import { useSessionForm, useShowForm } from '@catalog/hooks/forms';
import { useAdminCatalogMutations } from '@catalog/hooks/mutations';
import { useAdminShowList } from '@catalog/hooks/queries';
import { formatDateTime } from '@catalog/lib';

import type { AdminSession, AdminShow } from '@catalog/server/types';

type ShowPanel = { mode: 'create' } | { mode: 'edit'; show: AdminShow } | null;
type SessionPanel = { showId: string; session: AdminSession | null } | null;

export function AdminCatalogManager() {
  const query = useAdminShowList();
  const {
    createShowMutation,
    updateShowMutation,
    publishShowMutation,
    unpublishShowMutation,
    deleteShowMutation,
    createSessionMutation,
    updateSessionMutation,
    cancelSessionMutation,
    deleteSessionMutation,
  } = useAdminCatalogMutations();

  const [showPanel, setShowPanel] = useState<ShowPanel>(null);
  const [sessionPanel, setSessionPanel] = useState<SessionPanel>(null);
  const [cancelTarget, setCancelTarget] = useState<AdminSession | null>(null);

  const showForm = useShowForm(showPanel?.mode === 'edit' ? showPanel.show : null);
  const sessionForm = useSessionForm(sessionPanel?.session ?? null);

  const showPending = createShowMutation.isPending || updateShowMutation.isPending;
  const sessionPending =
    createSessionMutation.isPending || updateSessionMutation.isPending;

  const submitShow = showForm.handleSubmit((values) => {
    if (showPanel?.mode === 'edit') {
      updateShowMutation.mutate(
        { id: showPanel.show.id, values },
        { onSuccess: () => setShowPanel(null) },
      );
    } else {
      createShowMutation.mutate(values, { onSuccess: () => setShowPanel(null) });
    }
  });

  const submitSession = sessionForm.handleSubmit((values) => {
    if (!sessionPanel) return;
    if (sessionPanel.session) {
      updateSessionMutation.mutate(
        { id: sessionPanel.session.id, values },
        { onSuccess: () => setSessionPanel(null) },
      );
    } else {
      createSessionMutation.mutate(
        { showId: sessionPanel.showId, values },
        { onSuccess: () => setSessionPanel(null) },
      );
    }
  });

  const confirmCancel = () => {
    if (!cancelTarget) return;
    cancelSessionMutation.mutate(cancelTarget.id, {
      onSuccess: () => setCancelTarget(null),
    });
  };

  if (query.isLoading) {
    return <p className="p-6 text-sm text-gray-500">Carregando espetáculos...</p>;
  }

  if (query.isError) {
    return (
      <div className="p-6 text-sm">
        <p className="text-red-600">Não foi possível carregar os espetáculos.</p>
        <button
          type="button"
          onClick={() => void query.refetch()}
          className="mt-2 min-h-11 rounded-md border border-gray-300 px-4"
        >
          Tentar de novo
        </button>
      </div>
    );
  }

  const shows = query.data ?? [];
  const editingSessionTicketsSold = sessionPanel?.session?.tickets_sold ?? 0;

  return (
    <main className="mx-auto max-w-4xl space-y-6 p-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Espetáculos e sessões</h1>
        <button
          type="button"
          onClick={() => setShowPanel({ mode: 'create' })}
          className="min-h-11 rounded-md bg-gray-900 px-4 text-sm font-medium text-white"
        >
          Novo espetáculo
        </button>
      </div>

      {showPanel ? (
        <ShowForm
          form={showForm}
          onSubmit={submitShow}
          onCancel={() => setShowPanel(null)}
          isPending={showPending}
          mode={showPanel.mode}
        />
      ) : null}

      {sessionPanel ? (
        <SessionForm
          form={sessionForm}
          onSubmit={submitSession}
          onCancel={() => setSessionPanel(null)}
          isPending={sessionPending}
          mode={sessionPanel.session ? 'edit' : 'create'}
          ticketsSold={editingSessionTicketsSold}
        />
      ) : null}

      {shows.length === 0 ? (
        <p className="rounded-lg border border-dashed border-gray-300 p-8 text-center text-sm text-gray-500">
          Nenhum espetáculo cadastrado ainda. Crie o primeiro para começar a
          montar o catálogo.
        </p>
      ) : (
        <ShowList
          shows={shows}
          onEditShow={(show) => setShowPanel({ mode: 'edit', show })}
          onPublish={(show) => publishShowMutation.mutate(show.id)}
          onUnpublish={(show) => unpublishShowMutation.mutate(show.id)}
          onDeleteShow={(show) => deleteShowMutation.mutate(show.id)}
          onNewSession={(showId) => setSessionPanel({ showId, session: null })}
          onEditSession={(showId, session) => setSessionPanel({ showId, session })}
          onCancelSession={(session) => setCancelTarget(session)}
          onDeleteSession={(session) => deleteSessionMutation.mutate(session.id)}
        />
      )}

      <ConfirmCancelSessionDialog
        open={cancelTarget !== null}
        sessionLabel={
          cancelTarget
            ? `${formatDateTime(cancelTarget.starts_at)} · ${cancelTarget.venue}`
            : ''
        }
        ticketsSold={cancelTarget?.tickets_sold ?? 0}
        isPending={cancelSessionMutation.isPending}
        onConfirm={confirmCancel}
        onClose={() => setCancelTarget(null)}
      />
    </main>
  );
}
```

### `src/features/catalog/components/RequireAdmin.tsx`

O `RequireAuth` real (`identity-auth`, `@account/components/RequireAuth.tsx`)
só sabe "logado ou não" — recebe só `children`, **não** tem prop `role`
(`User` real não tem enum `Role`, é `is_admin: bool`, e `RequireAuth` nem
carrega os dados do usuário, só `isAuthenticated`/`isLoading` do token). Não
dá pra usar `<RequireAuth role="ADMIN">`. Em vez de editar um componente de
`identity`, este arquivo compõe: `RequireAuth` cuida do login, um gate local
busca o usuário atual (`useCurrentUser`, que já existe em `@account`) e
decide por `is_admin`.

```tsx
// src/features/catalog/components/RequireAdmin.tsx  — novo
'use client';

import { useEffect, type ReactNode } from 'react';

import { useRouter } from 'next/navigation';

import { RequireAuth, useCurrentUser } from '@account';

interface RequireAdminProps {
  children: ReactNode;
}

function AdminGate({ children }: { children: ReactNode }) {
  const router = useRouter();
  const { data: user, isLoading } = useCurrentUser();

  useEffect(() => {
    if (!isLoading && user && !user.is_admin) {
      router.replace('/');
    }
  }, [isLoading, user, router]);

  if (isLoading || !user || !user.is_admin) {
    return null;
  }

  return <>{children}</>;
}

export function RequireAdmin({ children }: RequireAdminProps) {
  return (
    <RequireAuth>
      <AdminGate>{children}</AdminGate>
    </RequireAuth>
  );
}
```

### `src/features/catalog/components/index.ts`

```ts
// src/features/catalog/components/index.ts  — novo (ou editar)
export * from './ui';
export * from './admin';
export * from './AdminCatalogManager';
export * from './RequireAdmin';
// export * from './ShowFilters';   — se catalog-show-search já mergeou
// export * from './ShowGrid';      — se catalog-show-search já mergeou
// export * from './SessionDetail'; — se catalog-session-detail já mergeou
// export * from './ShowSessions';  — se catalog-session-detail já mergeou
```

### `src/app/admin/espetaculos/page.tsx`

Server Component. Renderiza a tela client dentro de `<RequireAdmin>` (sem
sessão → `RequireAuth` interno redireciona para `/login`; logado mas
`is_admin=false` → `RequireAdmin` redireciona para `/`). Sem prefetch: o gate
de admin precisa do token de acesso, que só existe no cliente.

```tsx
// src/app/admin/espetaculos/page.tsx  — novo
import { AdminCatalogManager, RequireAdmin } from '@catalog';

export default function AdminEspetaculosPage() {
  return (
    <RequireAdmin>
      <AdminCatalogManager />
    </RequireAdmin>
  );
}
```

### `src/features/catalog/index.ts`

Segue o barrel real de `features/account/index.ts` (`export *` de cada
subpasta).

```ts
// src/features/catalog/index.ts  — novo (ou editar)
export * from './components';
export * from './constants';
export * from './hooks';
export * from './lib';
export * from './schemas';
export * from './server';
export * from './services';
```

---

## 3. Contrato consumido

Contrato: `docs.ludens/specs/catalog-admin-management/integration.md`
(`status: canônico` — reflete o código real já mergeado; se o shape divergir
na integração, o ajuste é um transform em `services/show.service.ts` +
registro da divergência no `integration.md`, nunca editar o repo de backend).

Dependências herdadas de `identity-auth` (frontend, já mergeado), consumidas
como contrato:

| Símbolo               | Origem             | Forma esperada                                                                                                                                                                   |
| --------------------- | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetcher`             | `@web/lib/fetcher` | `fetcher<T>(path, options): Promise<T>`; injeta `Authorization` e `credentials: 'include'`; em resposta não-2xx lança `ApiError` (`status`, `data` = corpo já parseado — ver §2) |
| `RequireAuth`         | `@account`         | componente client; só `children` — sem prop `role`. Sem sessão → redireciona para `/login`. Não sabe distinguir admin de comprador (ver `RequireAdmin`, §2)                      |
| `useCurrentUser`      | `@account`         | hook client (`useQuery`); `{ data: User \                                                                                                                                        |
| `NEXT_PUBLIC_API_URL` | env                | base URL da API (sem barra final)                                                                                                                                                |

---

## 4. Estados assíncronos e mensagens

| Estado                                 | Onde                                                         | Mensagem (linguagem de negócio)                                                                             |
| -------------------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| loading                                | `AdminCatalogManager`                                        | "Carregando espetáculos..."                                                                                 |
| error (lista)                          | `AdminCatalogManager`                                        | "Não foi possível carregar os espetáculos." + botão "Tentar de novo"                                        |
| empty                                  | `AdminCatalogManager`                                        | "Nenhum espetáculo cadastrado ainda. Crie o primeiro..."                                                    |
| sucesso de mutation                    | `useAdminCatalogMutations`                                   | toast: "Espetáculo criado.", "Sessão cancelada. Os compradores entrarão na fila de reembolso." etc.         |
| erro 409 no DELETE de sessão           | `useAdminCatalogMutations` · `deleteSessionMutation.onError` | "Esta sessão já vendeu ingressos. Cancele a sessão em vez de excluir — os compradores serão reembolsados."  |
| erro 409 no DELETE de espetáculo       | `deleteShowMutation.onError`                                 | "Há sessões com ingressos vendidos. Cancele essas sessões antes de excluir o espetáculo."                   |
| erro 409 no PUT de sessão (capacidade) | `updateSessionMutation.onError`                              | "Já há ingressos comprometidos nesta sessão."                                                               |
| erro genérico de mutation              | todas as `onError`                                           | `apiErrorMessage(error, fallback)` — usa o `detail` do backend (via `ApiError.data`) ou o fallback em pt-BR |
| sessão sem venda no formulário         | `SessionForm`                                                | aviso em `amber` quando `ticketsSold > 0` em edição                                                         |

Acesso negado (sem sessão, ou logado sem `is_admin`) é tratado por
`RequireAdmin` (redirect), não por esta feature no sentido de tratamento de
erro de mutation.

---

## 5. Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-admin

# commit 1 — infra compartilhada
git add src/lib/fetcher.ts tsconfig.json src/routes/endpoints.ts
git commit -m "feat(catalog): expor ApiError no fetcher e endpoints admin"

# commit 2 — contrato da feature
git add src/features/catalog/schemas src/features/catalog/server \
        src/features/catalog/services src/features/catalog/lib src/features/catalog/constants
git commit -m "feat(catalog): schemas, tipos, services e libs da area admin"

# commit 3 — hooks
git add src/features/catalog/hooks
git commit -m "feat(catalog): queries, mutations e forms de espetaculo e sessao"

# commit 4 — UI + rota
git add src/features/catalog/components src/app/admin
git commit -m "feat(catalog): telas de gestao com regra de cancelar vs excluir"

# commit 5 — barrels + API publica da feature
git add src/features/catalog/index.ts
git commit -m "chore(catalog): barrel index.ts da feature"

npm run lint && npm run build
```

Depois: `npm run lint && npm run build` verdes → `/team-ludens:tbd-pr`.

---

## 6. Ordem entre as superfícies

Frontend pode começar contra o contrato-alvo de `integration.md` antes do
backend. A integração real é após o merge do backend. `identity-auth`
(frontend) já está mergeado — `fetcher`, `RequireAuth` e `useCurrentUser`
já existem, sem stub necessário.

---

## 7. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 e logic.md fechados). Pendências
de dependência — não impedem escrever o código, impedem rodar ponta a ponta:

- **`NEXT_PUBLIC_API_URL`** precisa estar em `.env.local` e nos secrets/vars de
  CI de `web.ludens` (compartilhado com `identity-auth`).
- ~~shadcn/ui não está no repo~~ — **entrou desde esta revisão.** `ShowForm.tsx`
  já usa `Form`/`FormField`/`Select`/`Button`/`Input`/`Textarea` de
  `@components/ui/*` (ver §2). O resto dos componentes desta fatia
  (`SessionForm.tsx`, `ShowList.tsx`, `SessionRow.tsx`,
  `ConfirmCancelSessionDialog.tsx`, `AdminCatalogManager.tsx`) não foi
  reconferido nesta revisão — é provável que também tenham migrado para
  shadcn/ui; tratar os blocos de código correspondentes acima como
  desatualizados até serem reconferidos.
