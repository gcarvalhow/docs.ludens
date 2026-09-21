---
status: done
spec: catalog-genre
surface: frontend
created_at: 2026-09-11
---

# Gêneros do catálogo — Frontend

**Resumo:** painel admin de gêneros com CRUD completo (criar, editar,
excluir), consumindo `GET/POST/PUT/DELETE /catalog/genres`. Não existe UI de
ícone (nenhum ícone foi implementado no backend); o gênero é só um nome. O
schema/tipo/serviço de gênero não viraram arquivos dedicados: vivem junto dos
de `show`, na mesma feature `catalog`.

## 1. Arquivos (ordem de dependência)

| # | Caminho |
| --- | --- |
| 1 | `src/routes/endpoints.ts` (`catalog.genres`, `catalog.genreById(id)`) |
| 2 | `src/features/catalog/schemas/show.schema.ts` (`genreSchema`, `genreListSchema`, `genreFormSchema`) |
| 3 | `src/features/catalog/server/types/index.ts` (`Genre`, `GenreList`, `GenreFormValues`) |
| 4 | `src/features/catalog/services/show.service.ts` (`fetchGenres`, `createGenre`, `updateGenre`, `deleteGenre`) |
| 5 | `src/features/catalog/hooks/queries/useCatalogQueries.ts` (`useGenreList`) |
| 6 | `src/features/catalog/hooks/mutations/useGenreMutations.ts` |
| 7 | `src/features/catalog/hooks/forms/useGenreForm.ts` |
| 8 | `src/features/catalog/components/admin/GenreForm.tsx` |
| 9 | `src/features/catalog/components/admin/GenreTable.tsx` |
| 10 | `src/features/catalog/components/AdminGenreManager.tsx` |
| 11 | `src/app/admin/generos/page.tsx` |

Não existem `genre.schema.ts`, `genre.service.ts`, `genre.types.ts`,
`GenreIconBadge.tsx`, `GenreIconPicker.tsx` nem qualquer paleta de ícone —
esses arquivos nunca foram criados.

## 2. Código

### `src/routes/endpoints.ts` — trecho de gênero

```typescript
const CATALOG_BASE = `${API_BASE}/catalog`;
// ...
catalog: {
  // ...
  genres: `${CATALOG_BASE}/genres`,
  genreById: (id: string) => `${CATALOG_BASE}/genres/${id}`,
},
```

### `src/features/catalog/schemas/show.schema.ts` — trecho de gênero

```typescript
// GET /catalog/genres devolve [{ id, name }] — contrato real
// (app/modules/catalog/application/schemas/response.py, GenreResponse).
// Filtro usa o id (UUID); name é o texto exibido.
export const genreSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
});

export const genreListSchema = z.array(genreSchema);

export const genreFormSchema = z.object({
  name: z
    .string()
    .min(1, 'Informe o nome')
    .max(80),
});
```

`showCardSchema` traz `genre_id`/`genre` como campos planos do show (não um
`genreSchema` aninhado):

```typescript
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
```

### `src/features/catalog/services/show.service.ts` — funções de gênero

```typescript
fetchGenres() {
  return fetcher<GenreList>(
    endpoints.catalog.genres,
    { schema: genreListSchema },
  );
},

createGenre(values: GenreFormValues) {
  return fetcher<Genre>(
    endpoints.catalog.genres,
    { method: 'POST', body: values, schema: genreSchema },
  );
},

updateGenre(id: string, values: GenreFormValues) {
  return fetcher<Genre>(
    endpoints.catalog.genreById(id),
    { method: 'PUT', body: values, schema: genreSchema },
  );
},

deleteGenre(id: string) {
  return fetcher<void>(
    endpoints.catalog.genreById(id),
    { method: 'DELETE' },
  );
},
```

### `src/features/catalog/hooks/mutations/useGenreMutations.ts`

```typescript
export function useGenreMutations() {
  const queryClient = useQueryClient();

  const invalidate = () =>
    queryClient.invalidateQueries({ queryKey: catalogQueryKeys.genres() });

  const createGenreMutation = useMutation({
    mutationFn: (values: GenreFormValues) => catalogService.createGenre(values),
    onSuccess: () => { void invalidate(); toast.success('Gênero criado.'); },
    onError: (error) => toast.error(apiErrorMessage(error, 'Não foi possível criar o gênero.')),
  });

  const updateGenreMutation = useMutation({
    mutationFn: ({ id, values }: { id: string; values: GenreFormValues }) =>
      catalogService.updateGenre(id, values),
    onSuccess: () => { void invalidate(); toast.success('Gênero atualizado.'); },
    onError: (error) => toast.error(apiErrorMessage(error, 'Não foi possível salvar o gênero.')),
  });

  const deleteGenreMutation = useMutation({
    mutationFn: (id: string) => catalogService.deleteGenre(id),
    onSuccess: () => { void invalidate(); toast.success('Gênero excluído.'); },
    onError: (error) => {
      if (apiErrorStatus(error) === 409) {
        toast.error(apiErrorMessage(
          error,
          'Existem espetáculos usando este gênero; altere-os para outro gênero antes de excluir.',
        ));
        return;
      }
      toast.error(apiErrorMessage(error, 'Não foi possível excluir o gênero.'));
    },
  });

  return { createGenreMutation, updateGenreMutation, deleteGenreMutation };
}
```

O 409 de "gênero em uso" (backend recusa excluir gênero referenciado por
algum espetáculo) tem mensagem própria na mutation, não cai no fallback
genérico.

### `src/features/catalog/components/admin/GenreForm.tsx`

Formulário com um único campo (`name`); recebe `mode: 'create' | 'edit'` (só
muda o texto do diálogo que o envolve — o form em si não se comporta
diferente entre os dois modos).

### `src/features/catalog/components/admin/GenreTable.tsx`

Lista os gêneros em cards, cada um com botões "Editar" e "Excluir".

### `src/features/catalog/components/AdminGenreManager.tsx`

Tela admin: lista (`useGenreList`), estado de loading/erro/vazio, e um
`Dialog` que abre o `GenreForm` em modo criar ou editar (`GenrePanel` local:
`{mode:'create'} | {mode:'edit', genre} | null`). Excluir não abre diálogo de
confirmação própria — a mutation já dispara direto e o toast comunica o
resultado (incluindo o 409 de "gênero em uso").

### `src/app/admin/generos/page.tsx`

Server Component que renderiza `<AdminGenreManager>` dentro da área admin.

## 3. Contrato consumido

`GET /catalog/genres` → `GenreList` (`{id, name}[]`). `POST`/`PUT
/catalog/genres/{id}` → `Genre`. `DELETE /catalog/genres/{id}` → 204, ou 409
se o gênero está em uso por algum espetáculo.

## 4. Estados assíncronos e mensagens

* Lista: loading (skeletons) · erro (alerta + "tentar de novo") · vazio
  ("Nenhum gênero cadastrado ainda" + botão "Novo gênero").
* Criar/editar: toast de sucesso ("Gênero criado."/"Gênero atualizado.");
  erro genérico por toast.
* Excluir: toast de sucesso ("Gênero excluído."); 409 vira mensagem
  específica ("Existem espetáculos usando este gênero…"); outro erro cai no
  genérico.

## 5. Passo a passo TBD (Frontend)

```text
git checkout master && git pull && git checkout -b feat/<NN>-catalog-genre-admin
commit 1  feat(catalog): schema, tipos e service de gênero (CRUD)
commit 2  feat(catalog): query e mutations de gênero
commit 3  feat(catalog): formulário, tabela e tela admin de gêneros
```

## 6. Ordem entre as superfícies

Mergeado, junto do backend.

## 7. Bloqueios em aberto

Nenhum.

## 8. Ajustes feitos no `integration.md`

O `frontend.md` original previa `genre.schema.ts`, `genre.service.ts`,
`genre.types.ts` e uma paleta de ícone (`GenreIconBadge`, `GenreIconPicker`,
`GENRE_ICON_OPTIONS`/`GENRE_ICON_MAP`) como arquivos novos, e um `GenreForm`
"sem `mode`" (só criação). Nada disso existe: o schema/tipo/serviço de gênero
ficaram dentro dos arquivos já existentes de `show`, não há UI de ícone, e o
`GenreForm` tem `mode: 'create' | 'edit'` desde o início porque o CRUD saiu
completo.
