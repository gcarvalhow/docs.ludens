---
status: draft
spec: catalog-genre
surface: frontend
created_at: 2026-09-11
---

# Gêneros do catálogo — Frontend

**Resumo:** área admin ganha uma tela de gestão de gêneros (só criação, nome + ícone de uma paleta fixa lucide-react); o formulário de criar/editar espetáculo passa a selecionar um gênero de uma lista em vez de digitar texto livre.
**RF:** fortalece RF01 · ajusta RF08 · **RN:** nenhuma RN01–RN05 · **Feature frontend:** `catalog`
**Contrato:** `docs.ludens/specs/catalog-genre/integration.md`
**Carregar antes:** skill `frontend-architecture` (todos os `references/`).

Stack: Next.js (App Router) + TypeScript estrito. Repo lido contra o branch em
review do PR web.ludens#14 (`feat/6-catalog-admin`, área `/admin/espetaculos`
ainda não mergeada em `master`) — esta feature estende essa área.

> **Reconciliado com o parecer de backend** (`backend.md` desta mesma pasta):
> a rota da lista completa de gêneros é `GET /admin/genres` (admin-only, dentro
> do mesmo agrupamento de `/admin/*` que já existe), não `GET /genres/all`
> como uma primeira versão deste documento supôs. `POST /genres` também virou
> `POST /admin/genres`. Ajustado abaixo — não é mais suposição, é o contrato
> que o backend implementa.
>
> Também reconciliado: `GET /genres` (público) deixa de ter shape `{slug,
> label}` e passa a ter o mesmo shape `{id, name, icon}` da lista admin — a
> lacuna de ícone ausente no filtro público que uma primeira leitura deste
> documento apontou **está resolvida** pelo backend. Como consequência, só
> existe **um** schema de gênero (`genreSchema`), não dois.

## Divergência de precedente que este documento segue (não a estrutura "ideal" da skill)

O código real de `catalog` (web.ludens) já estabeleceu um padrão que diverge
do "ideal" de `references/01`/`08` — por regra da skill mestra ("se divergir
do que o código real diz, ele vence"), sigo o precedente real:

1. **`services/` na raiz da feature**, não `server/services/` — só
   `server/types/` existe sob `server/` (ver `show.service.ts`,
   `admin.types.ts`). Gênero ganha `genre.service.ts` na raiz de `services/`.
2. **Um objeto de serviço por arquivo/recurso REST** (`genreService`), não
   funções soltas.
3. **`useCatalogQueries.ts` e `useAdminCatalogMutations.ts` únicos** para a
   feature inteira — estendidos, não duplicados em arquivos `useGenre*.ts`.
4. **`genre.schema.ts` próprio** (não dentro de `admin.schema.ts`) porque
   gênero não é subconjunto de "admin" — tem shape público também.

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
|---|---|---|---|
| 1 | endpoints | `src/routes/endpoints.ts` | editar |
| 2 | schemas | `src/features/catalog/schemas/genre.schema.ts` | novo |
| 3 | schemas | `src/features/catalog/schemas/admin.schema.ts` | editar |
| 4 | server/types | `src/features/catalog/server/types/genre.types.ts` | novo |
| 5 | server/types | `src/features/catalog/server/types/index.ts` | editar |
| 6 | services | `src/features/catalog/services/genre.service.ts` | novo |
| 7 | services | `src/features/catalog/services/index.ts` | editar |
| 8 | queries | `src/features/catalog/hooks/queries/query-options.ts` | editar |
| 9 | queries | `src/features/catalog/hooks/queries/useCatalogQueries.ts` | editar |
| 10 | mutations | `src/features/catalog/hooks/mutations/useAdminCatalogMutations.ts` | editar |
| 11 | forms | `src/features/catalog/hooks/forms/useGenreForm.ts` | novo |
| 12 | forms | `src/features/catalog/hooks/forms/index.ts` | editar |
| 13 | forms | `src/features/catalog/hooks/forms/useShowForm.ts` | editar |
| 14 | constants | `src/features/catalog/constants/catalog.constants.ts` | editar |
| 15 | components/ui (shadcn) | `src/components/ui/select.tsx` | novo (`npx shadcn add select`) |
| 16 | components/ui | `src/features/catalog/components/ui/GenreIconBadge.tsx` | novo |
| 17 | components/ui | `src/features/catalog/components/ui/GenreIconPicker.tsx` | novo |
| 18 | components/ui | `src/features/catalog/components/ui/index.ts` | editar |
| 19 | components | `src/features/catalog/components/admin/GenreForm.tsx` | novo |
| 20 | components | `src/features/catalog/components/admin/GenreList.tsx` | novo |
| 21 | components | `src/features/catalog/components/admin/index.ts` | editar |
| 22 | components | `src/features/catalog/components/AdminGenreManager.tsx` | novo |
| 23 | components | `src/features/catalog/components/admin/ShowForm.tsx` | editar |
| 24 | components | `src/features/catalog/components/AdminCatalogManager.tsx` | editar |
| 25 | components | `src/features/catalog/components/index.ts` | editar |
| 26 | rota | `src/app/admin/generos/page.tsx` | novo |

## 2. Código

### `src/routes/endpoints.ts` — editar (adiciona o grupo `catalog.genres`)

```ts
const GENRES_BASE = '/genres';

// dentro do objeto `catalog` já existente, ao lado de `admin`:
catalog: {
  genres: {
    list: GENRES_BASE,                  // GET público — filtro da vitrine (RF01)
    adminList: `/admin${GENRES_BASE}`,  // GET admin-only — lista completa
    create: `/admin${GENRES_BASE}`,     // POST admin-only
  },
  admin: { shows: { /* já existe */ }, sessions: { /* já existe */ } },
},
```

### `src/features/catalog/schemas/genre.schema.ts` — novo

```ts
import { z } from 'zod';

// Paleta fixa — cada key corresponde a um componente lucide-react listado
// em GENRE_ICON_OPTIONS (constants/catalog.constants.ts). O backend só
// valida forma (não-vazio, minúsculo, kebab-case) — a paleta em si é
// decisão de frontend/design system, não existe lista fechada no servidor.
export const genreIconEnum = z.enum([
  'drama', 'laugh', 'ghost', 'baby', 'music-4', 'disc-3',
  'mic-2', 'popcorn', 'clapperboard', 'sparkles', 'heart',
  'swords', 'crown', 'moon', 'sun', 'flame', 'feather',
  'wand-2', 'party-popper', 'puzzle', 'book-open',
  'guitar', 'piano', 'star',
]);

// Único shape de gênero — usado tanto pelo filtro público (GET /genres)
// quanto pela lista completa do admin (GET /admin/genres). O backend
// devolve {id, name, icon} nos dois casos (reconciliado com backend.md).
export const genreSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  icon: genreIconEnum,
});

export const genreListSchema = z.array(genreSchema);

export const createGenreFormSchema = z.object({
  name: z.string().min(1, 'Informe o nome do gênero').max(80),
  icon: genreIconEnum, // sem .optional() — ícone é obrigatório
});
```

`schemas/index.ts`: adicionar `export * from './genre.schema';`.

### `src/features/catalog/schemas/admin.schema.ts` — editar (trecho)

```ts
// import novo no topo do arquivo:
import { genreSchema } from './genre.schema';

// dentro do schema de AdminShow existente:
// antes: genre: z.string(),
genre: genreSchema,

// dentro do showFormSchema existente:
// antes: genre: z.string().min(1, 'Informe a categoria').max(80),
genre_id: z.string().uuid('Selecione um gênero'),
```

### `src/features/catalog/server/types/genre.types.ts` — novo

```ts
import type { z } from 'zod';

import type {
  genreSchema,
  createGenreFormSchema,
  genreIconEnum,
} from '@catalog/schemas';

export type Genre = z.infer<typeof genreSchema>;
export type CreateGenreFormValues = z.infer<typeof createGenreFormSchema>;
export type GenreIcon = z.infer<typeof genreIconEnum>;
```

`server/types/index.ts`: adicionar `export * from './genre.types';`.

### `src/features/catalog/services/genre.service.ts` — novo

```ts
import { fetcher } from '@web/lib/fetcher';
import { endpoints } from '@web/routes/endpoints';

import { genreListSchema } from '@catalog/schemas';

import type { CreateGenreFormValues, Genre } from '@catalog/server/types';

export const genreService = {
  async listPublicGenres() {
    const data = await fetcher<unknown>(endpoints.catalog.genres.list, { method: 'GET' });
    return genreListSchema.parse(data);
  },

  async listAllGenres() {
    const data = await fetcher<unknown>(endpoints.catalog.genres.adminList, { method: 'GET' });
    return genreListSchema.parse(data);
  },

  createGenre(values: CreateGenreFormValues) {
    return fetcher<Genre>(endpoints.catalog.genres.create, {
      method: 'POST',
      body: JSON.stringify(values),
    });
  },
};
```

`services/index.ts`: adicionar `export * from './genre.service';`.

### `src/features/catalog/hooks/queries/query-options.ts` — editar (trecho)

```ts
export const catalogQueryKeys = {
  all: ['catalog'] as const,

  admin: {
    shows: () => [...catalogQueryKeys.all, 'admin', 'shows'] as const,
    showList: () => [...catalogQueryKeys.admin.shows(), 'list'] as const,
    genres: () => [...catalogQueryKeys.all, 'admin', 'genres'] as const,
    genreList: () => [...catalogQueryKeys.admin.genres(), 'list'] as const,
  },

  public: {
    genres: () => [...catalogQueryKeys.all, 'public', 'genres'] as const,
    genreList: () => [...catalogQueryKeys.public.genres(), 'list'] as const,
  },
};

export const catalogQueryOptions = {
  adminShowList: () => queryOptions({ /* já existe */ }),

  adminGenreList: () =>
    queryOptions({
      queryKey: catalogQueryKeys.admin.genreList(),
      queryFn: () => genreService.listAllGenres(),
      // gênero é criado pelo próprio admin e precisa aparecer no form de
      // espetáculo logo em seguida — staleTime curto, invalidada pela
      // mutation de criação.
      staleTime: 10_000,
    }),

  publicGenreList: () =>
    queryOptions({
      queryKey: catalogQueryKeys.public.genreList(),
      queryFn: () => genreService.listPublicGenres(),
      staleTime: 60_000,
    }),
};
```

### `src/features/catalog/hooks/queries/useCatalogQueries.ts` — editar (trecho)

```ts
export function useAdminShowList() { /* já existe */ }

export function useAdminGenreList() {
  return useQuery(catalogQueryOptions.adminGenreList());
}

export function usePublicGenreList() {
  return useQuery(catalogQueryOptions.publicGenreList());
}
```

> `usePublicGenreList` é escrito agora (a logic exige que o frontend "saiba"
> das duas listas) mas **não tem consumidor de UI nesta entrega** — a
> vitrine/busca pública (RF01) ainda não existe em `web.ludens`
> (`src/app/page.tsx` é placeholder hoje). Fica pronto pra quando RF01 for
> implementada, não é código morto para remover.

### `src/features/catalog/hooks/mutations/useAdminCatalogMutations.ts` — editar (trecho, dentro do hook existente)

```ts
const createGenreMutation = useMutation({
  mutationFn: (values: CreateGenreFormValues) => genreService.createGenre(values),

  onSuccess: () => {
    void queryClient.invalidateQueries({ queryKey: catalogQueryKeys.admin.genreList() });
    toast.success('Gênero criado.');
  },

  onError: (error) => {
    if (apiErrorStatus(error) === 409) {
      toast.error('Esse gênero já existe.');
      return;
    }
    toast.error(apiErrorMessage(error, 'Não foi possível criar o gênero.'));
  },
});

// incluir createGenreMutation no objeto retornado pelo hook
```

### `src/features/catalog/hooks/forms/useGenreForm.ts` — novo

```ts
'use client';

import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';

import { createGenreFormSchema } from '@catalog/schemas';
import type { CreateGenreFormValues } from '@catalog/server/types';

const EMPTY: Partial<CreateGenreFormValues> = { name: '' }; // icon sem default — obrigatório escolher

export function useGenreForm() {
  return useForm<CreateGenreFormValues>({
    resolver: zodResolver(createGenreFormSchema),
    mode: 'onSubmit',
    defaultValues: EMPTY as CreateGenreFormValues,
  });
}
```

Sem `editing`/`reset` por prop — não existe edição nesta entrega (spec §6).

`hooks/forms/index.ts`: adicionar `export * from './useGenreForm';`.

### `src/features/catalog/hooks/forms/useShowForm.ts` — editar (trecho)

```ts
const EMPTY: ShowFormValues = { title: '', synopsis: '', genre_id: '' };
// ...
form.reset({ title: editing.title, synopsis: editing.synopsis, genre_id: editing.genre.id });
```

### `src/features/catalog/constants/catalog.constants.ts` — editar (adiciona)

```ts
import {
  Drama, Laugh, Ghost, Baby, Music4, Disc3, Mic2, Popcorn,
  Clapperboard, Sparkles, Heart, Swords, Crown, Moon, Sun,
  Flame, Feather, Wand2, PartyPopper, Puzzle, BookOpen,
  Guitar, Piano, Star, VenetianMask,
  type LucideIcon,
} from 'lucide-react';

import type { GenreIcon } from '@catalog/server/types';

// Paleta fixa oferecida na criação de gênero (spec §8 — admin escolhe, não é
// atribuído automaticamente). Toda key aqui precisa existir em
// genreIconEnum (schemas/genre.schema.ts). Nomes de ícone conferidos contra
// lucide-react ^1.43.0 (package.json); "máscara de teatro" é VenetianMask,
// não Masks (não existe na lib).
export const GENRE_ICON_OPTIONS: { value: GenreIcon; label: string; Icon: LucideIcon }[] = [
  { value: 'drama', label: 'Drama', Icon: VenetianMask },
  { value: 'laugh', label: 'Comédia', Icon: Laugh },
  { value: 'ghost', label: 'Terror', Icon: Ghost },
  { value: 'baby', label: 'Infantil', Icon: Baby },
  { value: 'music-4', label: 'Musical', Icon: Music4 },
  { value: 'disc-3', label: 'Dança', Icon: Disc3 },
  { value: 'mic-2', label: 'Stand-up', Icon: Mic2 },
  { value: 'popcorn', label: 'Entretenimento', Icon: Popcorn },
  { value: 'clapperboard', label: 'Performance', Icon: Clapperboard },
  { value: 'sparkles', label: 'Circo', Icon: Sparkles },
  { value: 'heart', label: 'Romance', Icon: Heart },
  { value: 'swords', label: 'Aventura', Icon: Swords },
  { value: 'crown', label: 'Clássico', Icon: Crown },
  { value: 'moon', label: 'Noturno', Icon: Moon },
  { value: 'sun', label: 'Família', Icon: Sun },
  { value: 'flame', label: 'Suspense', Icon: Flame },
  { value: 'feather', label: 'Poesia', Icon: Feather },
  { value: 'wand-2', label: 'Fantasia', Icon: Wand2 },
  { value: 'party-popper', label: 'Festa', Icon: PartyPopper },
  { value: 'puzzle', label: 'Experimental', Icon: Puzzle },
  { value: 'book-open', label: 'Literário', Icon: BookOpen },
  { value: 'guitar', label: 'Música ao vivo', Icon: Guitar },
  { value: 'piano', label: 'Instrumental', Icon: Piano },
  { value: 'star', label: 'Especial', Icon: Star },
] as const;

export const GENRE_ICON_MAP: Record<GenreIcon, LucideIcon> = Object.fromEntries(
  GENRE_ICON_OPTIONS.map((o) => [o.value, o.Icon]),
) as Record<GenreIcon, LucideIcon>;
```

### `src/components/ui/select.tsx` — novo, gerado via `npx shadcn add select`

Não existe `Select` em `src/components/ui/` hoje. Gerar via CLI (style
`radix-nova`, já configurado em `components.json`) em vez de escrever à mão,
pra garantir consistência de versão com os outros primitivos (`dialog.tsx`,
`form.tsx`) — mesmo padrão de `data-slot`, `cn` do pacote `"cn"`, paleta
`border-border`/`bg-background`/`focus-visible:ring-ring/50`, itens com
`min-h-11` (alvo de toque acessível).

### `src/features/catalog/components/ui/GenreIconBadge.tsx` — novo

```tsx
'use client';

import { GENRE_ICON_MAP } from '@catalog/constants';
import type { GenreIcon } from '@catalog/server/types';

interface GenreIconBadgeProps {
  icon: GenreIcon;
  className?: string;
}

export function GenreIconBadge({ icon, className }: GenreIconBadgeProps) {
  const Icon = GENRE_ICON_MAP[icon];
  return <Icon className={className ?? 'size-4'} aria-hidden />;
}
```

Reutilizado em `GenreList`, `GenreForm` (preview do ícone escolhido) e no
`SelectItem` de gênero dentro do `ShowForm`.

### `src/features/catalog/components/ui/GenreIconPicker.tsx` — novo

Grid de botões (`role="radiogroup"`, cada opção `role="radio"` com
`aria-checked`), 100% visual — recebe `value`/`onChange`, sem query/mutation.
Usa `GENRE_ICON_OPTIONS` pra renderizar as ~24 opções em grid responsivo
(`grid-cols-4 sm:grid-cols-6`), cada célula `min-h-11 min-w-11` e
`aria-label={option.label}` (ícone sozinho não tem texto visível).

`components/ui/index.ts`: adicionar `export * from './GenreIconBadge';` e
`export * from './GenreIconPicker';`.

### `src/features/catalog/components/admin/GenreForm.tsx` — novo

Mesmo contrato de props que `ShowForm.tsx` (`form`, `onSubmit`, `onCancel`,
`isPending`), sem `mode` (só existe "create"). Campo `name` (`Input`,
`min-h-11`), campo `icon` ligado ao `GenreIconPicker` via `FormField`/
`FormControl`. `FormMessage` em ambos os campos.

### `src/features/catalog/components/admin/GenreList.tsx` — novo

Lista simples (sem ações — spec §6: só criação). Cada linha: `GenreIconBadge`
+ nome. Recebe `genres: Genre[]` via props.

`components/admin/index.ts`: adicionar `export * from './GenreForm';` e
`export * from './GenreList';`.

### `src/features/catalog/components/AdminGenreManager.tsx` — novo (orchestration, espelha `AdminCatalogManager.tsx`)

- `useAdminGenreList()` para os dados; `useAdminCatalogMutations()` (só usa
  `createGenreMutation`); `useGenreForm()`.
- Estados: **loading** (skeleton), **error** (`Alert` destructive + "Tentar
  de novo" chamando `query.refetch()`), **empty** ("Nenhum gênero cadastrado
  ainda" + CTA "Novo gênero") — mesmo padrão literal de `AdminCatalogManager`.
- Dialog de criação, `onOpenChange` bloqueado durante `isPending`.
- Ao suceder: fecha dialog + reseta form (lista atualiza via invalidate).

`components/index.ts`: adicionar `export * from './AdminGenreManager';`.

### `src/features/catalog/components/admin/ShowForm.tsx` — editar

Troca o `FormField name="genre"` (Input texto livre) por `FormField
name="genre_id"` ligado a `Select`/`SelectContent`/`SelectItem`, cada item
mostrando `GenreIconBadge` + nome. Props novas:

```ts
interface ShowFormProps {
  form: UseFormReturn<ShowFormValues>;
  onSubmit: FormEventHandler<HTMLFormElement>;
  onCancel: () => void;
  isPending: boolean;
  mode: 'create' | 'edit';
  genres: Genre[];        // novo — lista completa, injetada pelo orchestrator
  genresLoading: boolean; // novo
}
```

Caso de borda obrigatório (`logic.md` §5, "Nenhum gênero cadastrado ainda",
`[fechada]`): se `!genresLoading && genres.length === 0`, não renderizar um
`<Select>` vazio sem explicação — mostrar "Nenhum gênero cadastrado. Cadastre
um gênero antes de criar um espetáculo." com link pra `/admin/generos`, e
desabilitar o submit nesse estado.

### `src/features/catalog/components/AdminCatalogManager.tsx` — editar

Adicionar `const genresQuery = useAdminGenreList();`, passar
`genres={genresQuery.data ?? []}` e `genresLoading={genresQuery.isLoading}`
pro `<ShowForm />`.

### `src/app/admin/generos/page.tsx` — novo

```tsx
import { AdminGenreManager, RequireAdmin } from '@catalog';

export default function AdminGenresPage() {
  return (
    <RequireAdmin>
      <AdminGenreManager />
    </RequireAdmin>
  );
}
```

Não existe layout/nav compartilhado entre rotas `/admin/*` hoje — nenhuma
página linka pra lá ainda. Sugestão não bloqueante: um link no header do
`AdminCatalogManager` apontando pra `/admin/generos` e vice-versa (decisão de
produto/UX, registrar como pendência, não inventar um shell de admin novo
fora de escopo).

## 3. Contrato consumido

Aponta `docs.ludens/specs/catalog-genre/integration.md`. Enquanto o backend
não implementa, o frontend trabalha contra o contrato-alvo ali descrito
(já reconciliado com `backend.md` desta pasta — sem suposição pendente).

## 4. Estados assíncronos e mensagens

| Estado | Onde | Mensagem |
|---|---|---|
| loading (lista de gêneros) | `AdminGenreManager.tsx` | skeleton |
| error (lista de gêneros) | `AdminGenreManager.tsx` | "Não foi possível carregar os gêneros. Tente de novo." |
| empty (nenhum gênero) | `AdminGenreManager.tsx` | "Nenhum gênero cadastrado ainda." + CTA |
| erro de duplicidade (criar gênero) | `useAdminCatalogMutations.ts` → toast | "Esse gênero já existe." |
| erro genérico (criar gênero) | `useAdminCatalogMutations.ts` → toast | mensagem do backend via `apiErrorMessage` |
| formulário de espetáculo sem gênero cadastrado | `ShowForm.tsx` | "Nenhum gênero cadastrado. Cadastre um gênero antes de criar um espetáculo." + link |

## 5. Passo a passo TBD (Frontend)

```
git checkout master && git pull && git checkout -b feat/<NN>-catalog-genre
# commit 1 — contrato
git add src/routes/endpoints.ts src/features/catalog/schemas src/features/catalog/server src/features/catalog/services && git commit -m "feat(catalog): endpoints, schemas, tipos e service de gênero"
# commit 2 — hooks
git add src/features/catalog/hooks src/features/catalog/constants.ts && git commit -m "feat(catalog): queries, mutations e form de gênero"
# commit 3 — UI + rota
git add src/components/ui/select.tsx src/features/catalog/components src/app/admin/generos && git commit -m "feat(catalog): tela de gestão de gêneros e seleção no form de espetáculo"
# commit 4 — barrels
git add src/features/catalog && git commit -m "chore(catalog): barrels index.ts de gênero"
```
Depois: `npm run lint && npm run build` verdes → `/team-ludens:tbd-pr`.

## 6. Ordem entre as superfícies

Frontend pode começar contra o contrato-alvo de `integration.md` (já
reconciliado, sem suposição pendente). A integração real (trocar mock/contra
API real) é só depois do merge do backend — em especial depois da migration
`0003_catalog_genre.py` rodar, porque o form de espetáculo depende de
`GET /admin/genres` não vir vazio.

## 7. Bloqueios em aberto

- Nenhuma navegação cruzada entre `/admin/espetaculos` e `/admin/generos`
  existe hoje — decisão de produto/UX pendente (não bloqueia esta fatia, só
  registra a lacuna).
- Depende da migration de backend (`0003_catalog_genre.py`) já ter rodado em
  qualquer ambiente onde este código for testado manualmente — senão
  `GET /admin/genres` vem vazio e o form de espetáculo trava no caso de
  borda "nenhum gênero cadastrado" mesmo havendo espetáculos antigos.
