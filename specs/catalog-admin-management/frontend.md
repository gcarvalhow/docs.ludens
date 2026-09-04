---
status: draft
spec: catalog-admin-management
surface: frontend
created_at: 2026-09-03
updated_at: 2026-09-04
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
`@account`, `@web/*`.

---

## 1. Arquivos (ordem de dependência)

| #   | Camada          | Caminho                                                             | Novo/Editar                           |
| --- | --------------- | ------------------------------------------------------------------- | ------------------------------------- |
| 1   | config          | `tsconfig.json`                                                     | editar — bare `@account` e `@catalog` |
| 2   | endpoints       | `src/routes/endpoints.ts`                                           | novo — grupo `catalog.admin`          |
| 3   | schemas         | `src/features/catalog/schemas/admin.schema.ts`                      | novo — Zod (response + form DTO)      |
| 4   | schemas         | `src/features/catalog/schemas/index.ts`                             | novo — barrel                         |
| 5   | server/types    | `src/features/catalog/server/types/index.ts`                        | novo — `z.infer`                      |
| 6   | server/services | `src/features/catalog/server/services/admin-catalog.service.ts`     | novo — request + `parse`              |
| 7   | server/services | `src/features/catalog/server/services/index.ts`                     | novo — barrel                         |
| 8   | server          | `src/features/catalog/server/index.ts`                              | novo — barrel                         |
| 9   | lib             | `src/features/catalog/lib/errors.ts`                                | novo — leitura do erro da API         |
| 10  | lib             | `src/features/catalog/lib/format.ts`                                | novo — preço/data                     |
| 11  | lib             | `src/features/catalog/lib/index.ts`                                 | novo — barrel                         |
| 12  | constants       | `src/features/catalog/constants/catalog.constants.ts`               | novo — rótulos de status              |
| 13  | constants       | `src/features/catalog/constants/index.ts`                           | novo — barrel                         |
| 14  | queries         | `src/features/catalog/hooks/queries/query-options.ts`               | novo — `adminShowList`                |
| 15  | queries         | `src/features/catalog/hooks/queries/useAdminCatalogQueries.ts`      | novo                                  |
| 16  | queries         | `src/features/catalog/hooks/queries/index.ts`                       | novo — barrel                         |
| 17  | mutations       | `src/features/catalog/hooks/mutations/useAdminCatalogMutations.ts`  | novo                                  |
| 18  | mutations       | `src/features/catalog/hooks/mutations/index.ts`                     | novo — barrel                         |
| 19  | forms           | `src/features/catalog/hooks/forms/useShowForm.ts`                   | novo                                  |
| 20  | forms           | `src/features/catalog/hooks/forms/useSessionForm.ts`                | novo                                  |
| 21  | forms           | `src/features/catalog/hooks/forms/index.ts`                         | novo — barrel                         |
| 22  | hooks           | `src/features/catalog/hooks/index.ts`                               | novo — barrel                         |
| 23  | components/ui   | `src/features/catalog/components/ui/ConfirmCancelSessionDialog.tsx` | novo — puro                           |
| 24  | components/ui   | `src/features/catalog/components/ui/index.ts`                       | novo — barrel                         |
| 25  | components      | `src/features/catalog/components/admin/ShowForm.tsx`                | novo — visual `'use client'`          |
| 26  | components      | `src/features/catalog/components/admin/SessionForm.tsx`             | novo — visual `'use client'`          |
| 27  | components      | `src/features/catalog/components/admin/SessionRow.tsx`              | novo — `'use client'`                 |
| 28  | components      | `src/features/catalog/components/admin/ShowList.tsx`                | novo — `'use client'`                 |
| 29  | components      | `src/features/catalog/components/admin/index.ts`                    | novo — barrel                         |
| 30  | components      | `src/features/catalog/components/AdminCatalogManager.tsx`           | novo — orchestration `'use client'`   |
| 31  | components      | `src/features/catalog/components/index.ts`                          | novo — barrel                         |
| 32  | rota            | `src/app/admin/espetaculos/page.tsx`                                | novo — Server Component               |
| 33  | feature root    | `src/features/catalog/index.ts`                                     | novo — API pública                    |
| 34  | README          | `src/features/catalog/README.md`                                    | novo                                  |

---

## 2. Código

### `tsconfig.json`

Editar só o bloco `paths` — acrescentar o mapeamento **bare** de `@account`
(consumido para `RequireAuth`) e de `@catalog` (consumido pela rota):

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

Arquivo novo (o registro central ainda não existe no repo). Grupo
`catalog.admin`; rotas parametrizadas são funções.

```ts
// src/routes/endpoints.ts  — novo
const api = process.env.NEXT_PUBLIC_API_URL ?? '';

const withBase = (path = ''): string => `${api}${path}`;

export const API_ENDPOINTS = {
  catalog: {
    admin: {
      shows: {
        list: withBase('/admin/shows'),
        create: withBase('/admin/shows'),
        byId: (id: string) => withBase(`/admin/shows/${id}`),
        publish: (id: string) => withBase(`/admin/shows/${id}/publish`),
        unpublish: (id: string) => withBase(`/admin/shows/${id}/unpublish`),
        sessions: (id: string) => withBase(`/admin/shows/${id}/sessions`),
      },
      sessions: {
        byId: (id: string) => withBase(`/admin/sessions/${id}`),
        cancel: (id: string) => withBase(`/admin/sessions/${id}/cancel`),
      },
    },
  },
} as const;
```

### `src/features/catalog/schemas/admin.schema.ts`

```ts
// src/features/catalog/schemas/admin.schema.ts  — novo
import { z } from 'zod';

export const showStatusEnum = z.enum(['draft', 'published']);
export const sessionStatusEnum = z.enum(['on_sale', 'closed', 'cancelled']);

// ---- Response (contrato do backend, camelCase) ----
export const adminSessionSchema = z.object({
  id: z.string().uuid(),
  showId: z.string().uuid(),
  startsAt: z.coerce.date(),
  venue: z.string(),
  capacity: z.number().int(),
  fullPrice: z.number(),
  halfPrice: z.number(),
  status: sessionStatusEnum,
  ticketsSold: z.number().int(),
  reservedOpen: z.number().int(),
  canDelete: z.boolean(),
});

export const adminShowSchema = z.object({
  id: z.string().uuid(),
  title: z.string(),
  synopsis: z.string(),
  imageUrl: z.string(),
  genre: z.string(),
  status: showStatusEnum,
  sessions: z.array(adminSessionSchema),
});

export const adminShowListSchema = z.array(adminShowSchema);

// ---- Request DTO (guia do formulário) ----
export const showFormSchema = z.object({
  title: z.string().min(1, 'Informe o título').max(200),
  synopsis: z.string().min(1, 'Informe a sinopse').max(5000),
  imageUrl: z
    .string()
    .min(1, 'Informe a URL da imagem')
    .url('Informe uma URL válida')
    .max(2048),
  genre: z.string().min(1, 'Informe a categoria').max(80),
});

export const sessionFormSchema = z.object({
  // valor do <input type="datetime-local"> (hora local, sem fuso)
  startsAt: z
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
  fullPrice: z
    .number({ invalid_type_error: 'Informe o preço da inteira' })
    .positive('O preço deve ser maior que zero'),
});
```

### `src/features/catalog/schemas/index.ts`

```ts
// src/features/catalog/schemas/index.ts  — novo
export * from './admin.schema';
```

### `src/features/catalog/server/types/index.ts`

```ts
// src/features/catalog/server/types/index.ts  — novo
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

### `src/features/catalog/server/services/admin-catalog.service.ts`

```ts
// src/features/catalog/server/services/admin-catalog.service.ts  — novo
import {
  adminSessionSchema,
  adminShowListSchema,
  adminShowSchema,
} from '@catalog/schemas';
import { fetcher } from '@web/lib/fetcher';
import { API_ENDPOINTS } from '@web/routes/endpoints';

import type { SessionFormValues, ShowFormValues } from '@catalog/server/types';

export async function fetchAdminShows() {
  const { data } = await fetcher.get(API_ENDPOINTS.catalog.admin.shows.list);
  return adminShowListSchema.parse(data);
}

export async function createShow(values: ShowFormValues) {
  const { data } = await fetcher.post(API_ENDPOINTS.catalog.admin.shows.create, values);
  return adminShowSchema.parse(data);
}

export async function updateShow(id: string, values: ShowFormValues) {
  const { data } = await fetcher.patch(API_ENDPOINTS.catalog.admin.shows.byId(id), values);
  return adminShowSchema.parse(data);
}

export async function publishShow(id: string): Promise<void> {
  await fetcher.post(API_ENDPOINTS.catalog.admin.shows.publish(id));
}

export async function unpublishShow(id: string): Promise<void> {
  await fetcher.post(API_ENDPOINTS.catalog.admin.shows.unpublish(id));
}

export async function deleteShow(id: string): Promise<void> {
  await fetcher.delete(API_ENDPOINTS.catalog.admin.shows.byId(id));
}

// O <input type="datetime-local"> devolve hora local sem fuso; o backend exige
// ISO 8601 com offset. `toISOString()` resolve para UTC (sufixo Z = offset).
function toSessionPayload(values: SessionFormValues) {
  return {
    startsAt: new Date(values.startsAt).toISOString(),
    venue: values.venue,
    capacity: values.capacity,
    fullPrice: values.fullPrice,
  };
}

export async function createSession(showId: string, values: SessionFormValues) {
  const { data } = await fetcher.post(
    API_ENDPOINTS.catalog.admin.shows.sessions(showId),
    toSessionPayload(values),
  );
  return adminSessionSchema.parse(data);
}

export async function updateSession(sessionId: string, values: SessionFormValues) {
  const { data } = await fetcher.patch(
    API_ENDPOINTS.catalog.admin.sessions.byId(sessionId),
    toSessionPayload(values),
  );
  return adminSessionSchema.parse(data);
}

export async function cancelSession(sessionId: string): Promise<void> {
  await fetcher.post(API_ENDPOINTS.catalog.admin.sessions.cancel(sessionId));
}

export async function deleteSession(sessionId: string): Promise<void> {
  await fetcher.delete(API_ENDPOINTS.catalog.admin.sessions.byId(sessionId));
}
```

### `src/features/catalog/server/services/index.ts`

```ts
// src/features/catalog/server/services/index.ts  — novo
export * from './admin-catalog.service';
```

### `src/features/catalog/server/index.ts`

```ts
// src/features/catalog/server/index.ts  — novo
export * from './services';
export * from './types';
```

### `src/features/catalog/lib/errors.ts`

```ts
// src/features/catalog/lib/errors.ts  — novo
// Lê o status e a mensagem de negócio do erro que o `fetcher` lança em resposta
// não-2xx. Envelope do backend: { detail: string } (erro de negócio) ou
// { detail: [{ field, message }] } (erro de forma).

export function apiErrorStatus(error: unknown): number | undefined {
  if (typeof error === 'object' && error !== null && 'status' in error) {
    const status = (error as { status?: unknown }).status;
    return typeof status === 'number' ? status : undefined;
  }
  return undefined;
}

export function apiErrorMessage(error: unknown, fallback: string): string {
  if (typeof error === 'object' && error !== null && 'data' in error) {
    const detail = (error as { data?: { detail?: unknown } }).data?.detail;
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
// src/features/catalog/hooks/queries/query-options.ts  — novo
import { queryOptions } from '@tanstack/react-query';

import { fetchAdminShows } from '@catalog/server/services';

export const adminCatalogKeys = {
  all: ['catalog', 'admin'] as const,
  shows: () => [...adminCatalogKeys.all, 'shows'] as const,
  showList: () => [...adminCatalogKeys.shows(), 'list'] as const,
};

export const adminCatalogQueryOptions = {
  adminShowList: () =>
    queryOptions({
      queryKey: adminCatalogKeys.showList(),
      queryFn: () => fetchAdminShows(),
      // Lista de gestão: dado pouco volátil; sem polling.
      staleTime: 10_000,
    }),
};
```

### `src/features/catalog/hooks/queries/useAdminCatalogQueries.ts`

```ts
// src/features/catalog/hooks/queries/useAdminCatalogQueries.ts  — novo
import { useQuery } from '@tanstack/react-query';

import { adminCatalogQueryOptions } from './query-options';

export function useAdminCatalogQueries() {
  return {
    useAdminShowList: () => useQuery(adminCatalogQueryOptions.adminShowList()),
  };
}
```

### `src/features/catalog/hooks/queries/index.ts`

```ts
// src/features/catalog/hooks/queries/index.ts  — novo
export * from './query-options';
export * from './useAdminCatalogQueries';
```

### `src/features/catalog/hooks/mutations/useAdminCatalogMutations.ts`

```ts
// src/features/catalog/hooks/mutations/useAdminCatalogMutations.ts  — novo
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';
import { toast } from 'sonner';

import { adminCatalogKeys } from '@catalog/hooks/queries';
import { apiErrorMessage, apiErrorStatus } from '@catalog/lib';
import {
  cancelSession,
  createSession,
  createShow,
  deleteSession,
  deleteShow,
  publishShow,
  unpublishShow,
  updateSession,
  updateShow,
} from '@catalog/server/services';

import type { SessionFormValues, ShowFormValues } from '@catalog/server/types';

export function useAdminCatalogMutations() {
  const queryClient = useQueryClient();
  const invalidate = () =>
    queryClient.invalidateQueries({ queryKey: adminCatalogKeys.showList() });

  const createShowMutation = useMutation({
    mutationFn: (values: ShowFormValues) => createShow(values),
    onSuccess: () => {
      void invalidate();
      toast.success('Espetáculo criado.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível criar o espetáculo.')),
  });

  const updateShowMutation = useMutation({
    mutationFn: ({ id, values }: { id: string; values: ShowFormValues }) =>
      updateShow(id, values),
    onSuccess: () => {
      void invalidate();
      toast.success('Espetáculo atualizado.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível salvar o espetáculo.')),
  });

  const publishShowMutation = useMutation({
    mutationFn: (id: string) => publishShow(id),
    onSuccess: () => {
      void invalidate();
      toast.success('Espetáculo publicado.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível publicar o espetáculo.')),
  });

  const unpublishShowMutation = useMutation({
    mutationFn: (id: string) => unpublishShow(id),
    onSuccess: () => {
      void invalidate();
      toast.success('Espetáculo despublicado.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível despublicar o espetáculo.')),
  });

  const deleteShowMutation = useMutation({
    mutationFn: (id: string) => deleteShow(id),
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
      createSession(showId, values),
    onSuccess: () => {
      void invalidate();
      toast.success('Sessão criada.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível criar a sessão.')),
  });

  const updateSessionMutation = useMutation({
    mutationFn: ({ id, values }: { id: string; values: SessionFormValues }) =>
      updateSession(id, values),
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
    mutationFn: (id: string) => cancelSession(id),
    onSuccess: () => {
      void invalidate();
      toast.success('Sessão cancelada. Os compradores entrarão na fila de reembolso.');
    },
    onError: (error) =>
      toast.error(apiErrorMessage(error, 'Não foi possível cancelar a sessão.')),
  });

  const deleteSessionMutation = useMutation({
    mutationFn: (id: string) => deleteSession(id),
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

const EMPTY: ShowFormValues = { title: '', synopsis: '', imageUrl: '', genre: '' };

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
        imageUrl: editing.imageUrl,
        genre: editing.genre,
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

const EMPTY: SessionFormValues = { startsAt: '', venue: '', capacity: 0, fullPrice: 0 };

export function useSessionForm(editing: AdminSession | null) {
  const form = useForm<SessionFormValues>({
    resolver: zodResolver(sessionFormSchema),
    mode: 'onSubmit',
    defaultValues: EMPTY,
  });

  useEffect(() => {
    if (editing) {
      form.reset({
        startsAt: toDateTimeLocalValue(editing.startsAt),
        venue: editing.venue,
        capacity: editing.capacity,
        fullPrice: editing.fullPrice,
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
// src/features/catalog/hooks/index.ts  — novo
export * from './forms';
export * from './mutations';
export * from './queries';
```

### `src/features/catalog/components/ui/ConfirmCancelSessionDialog.tsx`

Apresentacional puro — sem hooks de dados, só props e estado visual mínimo
(fechar no Esc). Avisa do reembolso (RN02/RF07).

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
// src/features/catalog/components/ui/index.ts  — novo
export * from './ConfirmCancelSessionDialog';
```

### `src/features/catalog/components/admin/ShowForm.tsx`

Formulário visual — recebe `form`, `onSubmit`, `isPending`; não conhece mutation.

```tsx
// src/features/catalog/components/admin/ShowForm.tsx  — novo
'use client';

import type { FormEventHandler } from 'react';
import type { UseFormReturn } from 'react-hook-form';

import type { ShowFormValues } from '@catalog/server/types';

interface ShowFormProps {
  form: UseFormReturn<ShowFormValues>;
  onSubmit: FormEventHandler<HTMLFormElement>;
  onCancel: () => void;
  isPending: boolean;
  mode: 'create' | 'edit';
}

export function ShowForm({ form, onSubmit, onCancel, isPending, mode }: ShowFormProps) {
  const { register, formState } = form;
  const { errors } = formState;

  return (
    <form onSubmit={onSubmit} className="space-y-4 rounded-lg border border-gray-200 p-4">
      <h3 className="text-base font-semibold">
        {mode === 'create' ? 'Novo espetáculo' : 'Editar espetáculo'}
      </h3>

      <div className="flex flex-col gap-1">
        <label htmlFor="show-title" className="text-sm font-medium">
          Título
        </label>
        <input
          id="show-title"
          type="text"
          {...register('title')}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
        {errors.title ? (
          <p className="text-sm text-red-600">{errors.title.message}</p>
        ) : null}
      </div>

      <div className="flex flex-col gap-1">
        <label htmlFor="show-synopsis" className="text-sm font-medium">
          Sinopse
        </label>
        <textarea
          id="show-synopsis"
          rows={4}
          {...register('synopsis')}
          className="rounded-md border border-gray-300 px-3 py-2"
        />
        {errors.synopsis ? (
          <p className="text-sm text-red-600">{errors.synopsis.message}</p>
        ) : null}
      </div>

      <div className="flex flex-col gap-1">
        <label htmlFor="show-image" className="text-sm font-medium">
          URL da imagem
        </label>
        <input
          id="show-image"
          type="url"
          inputMode="url"
          {...register('imageUrl')}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
        {errors.imageUrl ? (
          <p className="text-sm text-red-600">{errors.imageUrl.message}</p>
        ) : null}
      </div>

      <div className="flex flex-col gap-1">
        <label htmlFor="show-genre" className="text-sm font-medium">
          Categoria / gênero
        </label>
        <input
          id="show-genre"
          type="text"
          {...register('genre')}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
        {errors.genre ? (
          <p className="text-sm text-red-600">{errors.genre.message}</p>
        ) : null}
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
          {...register('startsAt')}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
        {errors.startsAt ? (
          <p className="text-sm text-red-600">{errors.startsAt.message}</p>
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
          {...register('fullPrice', { valueAsNumber: true })}
          className="min-h-11 rounded-md border border-gray-300 px-3"
        />
        {errors.fullPrice ? (
          <p className="text-sm text-red-600">{errors.fullPrice.message}</p>
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
        <span className="font-medium">{formatDateTime(session.startsAt)}</span>
        <span className="text-gray-600">{session.venue}</span>
        <span className="text-gray-600">Cap. {session.capacity}</span>
        <span className="text-gray-600">
          Inteira {formatPriceBRL(session.fullPrice)} · Meia{' '}
          {formatPriceBRL(session.halfPrice)}
        </span>
        <span className="rounded bg-gray-100 px-2 py-0.5 text-xs">
          {SESSION_STATUS_LABELS[session.status]}
        </span>
        <span className="text-gray-600">Vendidos: {session.ticketsSold}</span>
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
          disabled={!session.canDelete}
          title={
            session.canDelete
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
import { useAdminCatalogQueries } from '@catalog/hooks/queries';
import { formatDateTime } from '@catalog/lib';

import type { AdminSession, AdminShow } from '@catalog/server/types';

type ShowPanel = { mode: 'create' } | { mode: 'edit'; show: AdminShow } | null;
type SessionPanel = { showId: string; session: AdminSession | null } | null;

export function AdminCatalogManager() {
  const { useAdminShowList } = useAdminCatalogQueries();
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
  const editingSessionTicketsSold = sessionPanel?.session?.ticketsSold ?? 0;

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
            ? `${formatDateTime(cancelTarget.startsAt)} · ${cancelTarget.venue}`
            : ''
        }
        ticketsSold={cancelTarget?.ticketsSold ?? 0}
        isPending={cancelSessionMutation.isPending}
        onConfirm={confirmCancel}
        onClose={() => setCancelTarget(null)}
      />
    </main>
  );
}
```

### `src/features/catalog/components/index.ts`

```ts
// src/features/catalog/components/index.ts  — novo
export * from './admin';
export * from './ui';
export * from './AdminCatalogManager';
```

### `src/app/admin/espetaculos/page.tsx`

Server Component. Renderiza a tela client dentro de `<RequireAuth role="ADMIN">`
(sem sessão ou sem papel admin → `RequireAuth` redireciona). Sem prefetch: a
lista precisa do token de acesso, que só existe no cliente (decisão de
`identity-auth`).

```tsx
// src/app/admin/espetaculos/page.tsx  — novo
import { AdminCatalogManager } from '@catalog';
import { RequireAuth } from '@account';

export default function AdminEspetaculosPage() {
  return (
    <RequireAuth role="ADMIN">
      <AdminCatalogManager />
    </RequireAuth>
  );
}
```

### `src/features/catalog/index.ts`

```ts
// src/features/catalog/index.ts  — novo
export { AdminCatalogManager } from './components';
```

### `src/features/catalog/README.md`

```markdown
# Feature: catalog

Vitrine, busca/filtro, detalhe da sessão e **gestão (admin)** do catálogo.

Esta fatia (`catalog-admin-management`, RF08) entrega só a área admin em
`/admin/espetaculos`. As telas públicas (`catalog-show-search`,
`catalog-session-detail`) entram em fatias seguintes, reusando `schemas/`,
`server/` e `hooks/queries/`.

## Fluxo de dados (admin)

`src/routes/endpoints.ts` (`catalog.admin`)
→ `schemas/admin.schema.ts` (Zod: response + form DTO)
→ `server/services/admin-catalog.service.ts` (request + `parse`; converte
  `datetime-local` → ISO com offset)
→ `server/types` (`z.infer`)
→ `hooks/queries` (`adminShowList`) e `hooks/mutations`
  (`useAdminCatalogMutations` — toda mutation invalida `adminShowList` + toast)
→ `hooks/forms` (`useShowForm`, `useSessionForm`, resolver = schema de request)
→ `components/admin/*` (visual) + `components/ui/ConfirmCancelSessionDialog`
→ `components/AdminCatalogManager` (orchestration)
→ `src/app/admin/espetaculos/page.tsx` (Server Component + `RequireAuth`).

## Regras que o frontend só reflete (não reimplementa)

- Sessão com ingresso vendido não pode ser excluída: o backend responde 409 no
  `DELETE`; a UI desabilita "Excluir" por `session.canDelete` e o toast de 409
  orienta a cancelar. O reembolso (RN02/RF07) é do backend.
- Meia-entrada = 50% da inteira: `halfPrice` vem pronto do backend; o formulário
  só edita `fullPrice`.
- Capacidade abaixo do comprometido: bloqueada pelo backend (409); a UI mostra a
  mensagem no toast.

## Dependências de backend / de outras features

- `identity-auth` (frontend): `@web/lib/fetcher`, `@account` (`RequireAuth` com
  prop `role`), `NEXT_PUBLIC_API_URL`.
- Contrato-alvo: `docs.ludens/specs/catalog-admin-management/integration.md`.
```

---

## 3. Contrato consumido

Contrato-alvo: `docs.ludens/specs/catalog-admin-management/integration.md`
(`status: alvo` — o frontend trabalha contra ele até o backend implementar; se
o shape divergir na integração, o ajuste é um transform em
`server/services/admin-catalog.service.ts` + registro da divergência no
`integration.md`, nunca editar o repo de backend).

Dependências herdadas de `identity-auth` (frontend), consumidas como contrato:

| Símbolo               | Origem             | Forma esperada                                                                                                                                                                                                                    |
| --------------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetcher`             | `@web/lib/fetcher` | `fetcher.get/post/patch/delete(url, body?)` → `Promise<{ data: unknown }>`; injeta `Authorization` e `credentials: 'include'`; em resposta não-2xx **lança** um objeto com `status: number` e `data: unknown` (corpo já parseado) |
| `RequireAuth`         | `@account`         | componente client; prop `role?: 'ADMIN' \| 'BUYER'` e `children`. Sem sessão → redireciona para `/login`; com sessão sem o papel → redireciona para `/` (vitrine)                                                                 |
| `NEXT_PUBLIC_API_URL` | env                | base URL da API (sem barra final)                                                                                                                                                                                                 |

Enquanto `identity-auth` (frontend) não estiver mergeado, esses três pontos são
o **único bloqueio** para rodar esta feature — ver §7.

---

## 4. Estados assíncronos e mensagens

| Estado                                   | Onde                                                         | Mensagem (linguagem de negócio)                                                                            |
| ---------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| loading                                  | `AdminCatalogManager`                                        | "Carregando espetáculos..."                                                                                |
| error (lista)                            | `AdminCatalogManager`                                        | "Não foi possível carregar os espetáculos." + botão "Tentar de novo"                                       |
| empty                                    | `AdminCatalogManager`                                        | "Nenhum espetáculo cadastrado ainda. Crie o primeiro..."                                                   |
| sucesso de mutation                      | `useAdminCatalogMutations`                                   | toast: "Espetáculo criado.", "Sessão cancelada. Os compradores entrarão na fila de reembolso." etc.        |
| erro 409 no DELETE de sessão             | `useAdminCatalogMutations` · `deleteSessionMutation.onError` | "Esta sessão já vendeu ingressos. Cancele a sessão em vez de excluir — os compradores serão reembolsados." |
| erro 409 no DELETE de espetáculo         | `deleteShowMutation.onError`                                 | "Há sessões com ingressos vendidos. Cancele essas sessões antes de excluir o espetáculo."                  |
| erro 409 no PATCH de sessão (capacidade) | `updateSessionMutation.onError`                              | "Já há ingressos comprometidos nesta sessão."                                                              |
| erro genérico de mutation                | todas as `onError`                                           | `apiErrorMessage(error, fallback)` — usa o `detail` do backend ou o fallback em pt-BR                      |
| sessão sem venda no formulário           | `SessionForm`                                                | aviso em `amber` quando `ticketsSold > 0` em edição                                                        |

Acesso negado (sem papel admin) é tratado por `RequireAuth` (redirect), não por
esta feature.

---

## 5. Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-admin

# commit 1 — contrato
git add tsconfig.json src/routes/endpoints.ts src/features/catalog/schemas \
        src/features/catalog/server src/features/catalog/lib src/features/catalog/constants
git commit -m "feat(catalog): endpoints, schemas, tipos, services e libs da area admin"

# commit 2 — hooks
git add src/features/catalog/hooks
git commit -m "feat(catalog): queries, mutations e forms de espetaculo e sessao"

# commit 3 — UI + rota
git add src/features/catalog/components src/app/admin
git commit -m "feat(catalog): telas de gestao com regra de cancelar vs excluir"

# commit 4 — barrels + README + API publica da feature
git add src/features/catalog
git commit -m "chore(catalog): barrels index.ts e README da feature"

npm run lint && npm run build
```

Depois: `npm run lint && npm run build` verdes → `/team-ludens:tbd-pr`.

---

## 6. Ordem entre as superfícies

Frontend pode começar contra o contrato-alvo de `integration.md` antes do
backend. A integração real é após o merge do backend. Esta feature também
depende da fatia de frontend de `identity-auth` (fetcher + `RequireAuth`);
começar por ela, ou stubar os três símbolos da §3 com um mock marcado
`// TODO: remover quando identity-auth (frontend) entrar` — sem tocar em
schema/service/hook.

---

## 7. Bloqueios em aberto

Nenhum bloqueio de decisão de produto (spec §9 e logic.md fechados). Pendências
de dependência — não impedem escrever o código, impedem rodar ponta a ponta:

- **`identity-auth` (frontend) não mergeado.** Faltam `@web/lib/fetcher`,
  `@account` com `RequireAuth` aceitando `role`, e o `paths` bare de `@account`.
  A prop `role` em `RequireAuth` precisa existir no contrato de `identity-auth`;
  registrar lá se ainda não estiver.
- **`NEXT_PUBLIC_API_URL`** precisa estar em `.env.local` e nos secrets/vars de
  CI de `web.ludens` (compartilhado com `identity-auth`).
- **shadcn/ui não está no repo.** Os componentes usam elementos nativos + Tailwind
  (já configurado). Quando o design system entrar, `components/ui/*` e os forms
  são os arquivos a migrar — sem mudar hooks nem services.
- **Sem runner de teste de frontend** (`docs.ludens/backend/testing.md` §gaps):
  o portão hoje é `npm run lint` + `npm run build`. Os testes de `quality.md`
  para esta superfície são Playwright, a rodar quando a suíte existir.
