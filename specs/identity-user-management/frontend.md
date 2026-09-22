---
status: draft
spec: identity-user-management
surface: frontend
created_at: 2026-09-21
updated_at: 2026-09-21
---

# Gestão de conta de usuário — Frontend

> **Nota de plano (2026-09-21):** nenhum código desta feature existe hoje em
> `web.ludens` — confirmado por auditoria desta sessão (grep por termos de
> autoatendimento de conta não encontrou nada além do que `identity-auth` já
> construiu para login/registro/recuperação de senha). Este documento é um
> **plano de implementação daqui pra frente**, não a descrição de uma feature
> pronta. Todo código abaixo é planejado — arquitetura e convenções reais de
> `web.ludens` (aplicadas a este escopo), não trechos existentes. `status:
> draft` reflete isso; só vira `done` depois que o código for escrito,
> revisado e mergeado. `logic.md` também segue `draft` (pede nova revisão
> conjunta de FE/BE desde o reescopo de 2026-09-17), este plano já assume as
> regras de `logic.md` como fechadas (conforme a própria nota do PO em
> `spec.md` § 9), mas não substitui essa revisão.

**Resumo:** estende a feature `account` (já existente, dona de login/registro
de `identity-auth`) com autoatendimento de conta: editar o próprio nome,
trocar de e-mail (confirmação para o e-mail **atual**, sem formulário na
página de confirmação, só dispara a chamada e mostra o resultado), encerrar
a própria conta (mesmo padrão de confirmação por link) e uma tela
administrativa de listagem paginada de contas. Não cria uma feature nova,
`identity-user-management` no backend e `identity-auth` no backend mapeiam
para a mesma feature `account` no frontend (mesmo domínio de produto: conta
do usuário), exatamente como o próprio `identity-auth/backend.md` já registra
que `user_router.py` é compartilhado entre as duas specs.
**RF:** RF09 (edição de perfil não tem RF numerado próprio, ver `spec.md`
§ 5) · **RN:** — (reforça RNF01 na listagem administrativa)
**Feature frontend:** `account` (extensão)
**Contrato:** `docs.ludens/specs/identity-user-management/integration.md`
**Carregar antes:** skill `frontend-architecture` (todos os `references/`),
`docs.ludens/specs/identity-user-management/{spec.md,logic.md}`,
`identity-auth/backend.md` § 7 e `identity-user-management/backend.md` § 4
(débitos técnicos do backend que **limitam** o que esta tela consegue exibir
corretamente hoje, ver § 7 abaixo).

---

## 1. Arquivos (ordem de dependência)

Todos os arquivos abaixo são **planejados**, nenhum existe ainda. "Editar"
significa editar um arquivo real hoje existente em `web.ludens`; "novo"
significa arquivo a criar.

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | endpoints | `src/routes/endpoints.ts` | editar |
| 2 | schemas | `src/features/account/schemas/auth.schema.ts` | editar |
| 3 | server/types | `src/features/account/server/types/auth.types.ts` | editar |
| 4 | lib (compartilhado) | `src/lib/api-error.ts` | novo |
| 5 | lib (compartilhado) | `src/lib/api-error.test.ts` | novo |
| 6 | services | `src/features/account/services/auth.service.ts` | editar |
| 7 | services (teste) | `src/features/account/services/auth.service.test.ts` | editar |
| 8 | schemas (teste) | `src/features/account/schemas/auth.schema.test.ts` | editar |
| 9 | queries | `src/features/account/hooks/queries/query-options.ts` | editar |
| 10 | queries | `src/features/account/hooks/queries/useUsersQueries.ts` | novo |
| 11 | queries | `src/features/account/hooks/queries/index.ts` | editar |
| 12 | hooks (feature) | `src/features/account/hooks/useUsersFilters.ts` | novo |
| 13 | hooks | `src/features/account/hooks/index.ts` | editar |
| 14 | mutations | `src/features/account/hooks/mutations/useUserMutations.ts` | novo |
| 15 | mutations | `src/features/account/hooks/mutations/index.ts` | editar |
| 16 | forms | `src/features/account/hooks/forms/useUpdateProfileForm.ts` | novo |
| 17 | forms | `src/features/account/hooks/forms/useRequestEmailChangeForm.ts` | novo |
| 18 | forms | `src/features/account/hooks/forms/index.ts` | editar |
| 19 | components/ui | `src/features/account/components/ui/ProfileSummaryCard.tsx` | novo |
| 20 | components/ui | `src/features/account/components/ui/ConfirmResultView.tsx` | novo |
| 21 | components/ui | `src/features/account/components/ui/UsersTable.tsx` | novo |
| 22 | components/ui | `src/features/account/components/ui/index.ts` | editar |
| 23 | components/forms | `src/features/account/components/forms/UpdateProfileForm.tsx` | novo |
| 24 | components/forms | `src/features/account/components/forms/RequestEmailChangeForm.tsx` | novo |
| 25 | components/forms | `src/features/account/components/forms/index.ts` | editar |
| 26 | components | `src/features/account/components/UpdateProfileFormContainer.tsx` | novo |
| 27 | components | `src/features/account/components/RequestEmailChangeContainer.tsx` | novo |
| 28 | components | `src/features/account/components/RequestAccountDeletionContainer.tsx` | novo |
| 29 | components | `src/features/account/components/ProfileContainer.tsx` | novo |
| 30 | components | `src/features/account/components/ConfirmEmailChangeContainer.tsx` | novo |
| 31 | components | `src/features/account/components/ConfirmAccountDeletionContainer.tsx` | novo |
| 32 | components | `src/features/account/components/AdminUsersManager.tsx` | novo |
| 33 | components | `src/features/account/components/index.ts` | editar |
| 34 | components (outra feature) | `src/features/catalog/components/AdminHub.tsx` | editar |
| 35 | rota | `src/app/perfil/page.tsx` | novo |
| 36 | rota | `src/app/confirmar-troca-de-email/page.tsx` | novo |
| 37 | rota | `src/app/confirmar-exclusao-de-conta/page.tsx` | novo |
| 38 | rota | `src/app/admin/usuarios/page.tsx` | novo |
| 39 | README | `src/features/account/README.md` | editar |

Reaproveitados **sem alteração**: `src/features/account/contexts/AuthContext.tsx`
(o `logout()` existente já serve para limpar a sessão local depois de uma
confirmação, ver § 2, arquivo 14, e o risco anotado em § 7);
`src/features/account/components/RequireAuth.tsx`;
`src/features/catalog/components/RequireAdmin.tsx` (reaproveitado via
`@catalog`, não duplicado, gate de admin já existe, não é desta feature);
`src/features/catalog/components/ui/Pagination.tsx` (reaproveitado via
`@catalog/components/ui`, mesmo padrão de `ShowGrid`); `src/app/providers.tsx`;
`src/features/account/{server/index.ts,server/types/index.ts,services/index.ts,index.ts}`
(todos são `export * from './...'`, já reexportam qualquer coisa nova sem
precisar de edição própria).

---

## 2. Código (planejado)

### 1. `src/routes/endpoints.ts` — editar

```ts
// src/routes/endpoints.ts — editar (arquivo inteiro)
const API_BASE = '/api';
const IDENTITY_BASE = `${API_BASE}/identity`;
const AUTH_BASE = `${IDENTITY_BASE}/authentication`;
const CATALOG_BASE = `${API_BASE}/catalog`;

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
    register: `${IDENTITY_BASE}/users`,
    list: `${IDENTITY_BASE}/users`,
    byId: (id: string) => `${IDENTITY_BASE}/users/${id}`,
    // PATCH /identity/users edita o próprio nome — mesma URL de `register`
    // (POST), método diferente decide a ação em user_router.py. Alias
    // separado só por clareza semântica nos services desta feature.
    self: `${IDENTITY_BASE}/users`,
    // POST pede a troca (o link vai pro e-mail ATUAL da conta); PATCH +
    // ?token= confirma — mesma URL, dois verbos (integration.md).
    emailChange: `${IDENTITY_BASE}/users/email/change`,
    // POST pede o encerramento; DELETE + ?token= confirma — mesma URL,
    // dois verbos (integration.md).
    deletion: `${IDENTITY_BASE}/users/deletion`,
  },

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

### 2. `src/features/account/schemas/auth.schema.ts` — editar

```ts
// src/features/account/schemas/auth.schema.ts — editar (mantém tudo que já
// existe; adiciona os schemas novos ao final do arquivo)
import { z } from 'zod';

export const registerSchema = z.object({
  name: z.string().min(1, 'Nome é obrigatório.'),
  cpf: z.string().regex(/^\d{11}$/, 'CPF deve conter exatamente 11 dígitos.'),
  email: z.string().email('E-mail inválido.'),
  password: z.string().min(8, 'A senha deve ter pelo menos 8 caracteres.'),
});

export const loginSchema = z.object({
  email: z.string().email('E-mail inválido.'),
  password: z.string().min(1, 'Senha é obrigatória.'),
});

export const changePasswordSchema = z.object({
  current_password: z.string().min(1, 'Senha atual é obrigatória.'),
  new_password: z.string().min(8, 'A nova senha deve ter pelo menos 8 caracteres.'),
});

export const forgotSchema = z.object({
  email: z.string().email('E-mail inválido.'),
});

export const resetSchema = z.object({
  token: z.string().min(1, 'Token é obrigatório.'),
  password: z.string().min(8, 'A senha deve ter pelo menos 8 caracteres.'),
});

export const tokenResponseSchema = z.object({
  access_token: z.string(),
  expires_in: z.number(),
});

export const userSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
  cpf: z.string(),
  is_admin: z.boolean(),
});

// --- identity-user-management (planejado) ---

export const updateProfileSchema = z.object({
  name: z
    .string()
    .min(1, 'Nome é obrigatório.')
    .max(120, 'Nome deve ter no máximo 120 caracteres.'),
});

// DTO real enviado ao backend — só `new_email` (RequestEmailChangeRequest;
// backend.md § 4 item 2 registra que o backend NÃO pede o endereço duas
// vezes, apesar de `logic.md` § 8 exigir isso como mitigação).
export const requestEmailChangeSchema = z.object({
  new_email: z.string().email('E-mail inválido.').max(254),
});

// Schema do FORM, não do request — mitigação de frontend para o débito
// acima (integration.md, "Lacunas / decisões em aberto"): pede o endereço
// duas vezes na tela; só `new_email` é enviado ao backend (ver o container,
// arquivo 27).
export const requestEmailChangeFormSchema = requestEmailChangeSchema
  .extend({
    confirm_new_email: z.string().email('E-mail inválido.'),
  })
  .refine((data) => data.new_email === data.confirm_new_email, {
    message: 'Os e-mails não coincidem.',
    path: ['confirm_new_email'],
  });

export const messageResponseSchema = z.object({
  message: z.string(),
});

export const pagedUsersSchema = z.object({
  items: z.array(userSchema),
  page: z.number().int(),
  size: z.number().int(),
  total: z.number().int(),
});
```

Ponto de atenção: `userSchema` (já existente) inclui `cpf` porque também
serve `GET /identity/users/{id}` (perfil do próprio usuário, onde CPF É
exibido, só não editável). É o mesmo schema que alimenta `pagedUsersSchema`,
por isso a tela de listagem (§ 2, arquivo 21) precisa **decidir na UI** não
renderizar esse campo, já que o schema/tipo não pode "esconder" um campo que
o backend efetivamente devolve (ver § 7).

### 3. `src/features/account/server/types/auth.types.ts` — editar

```ts
// src/features/account/server/types/auth.types.ts — editar (arquivo inteiro)
import { z } from 'zod';

import {
  changePasswordSchema,
  forgotSchema,
  loginSchema,
  messageResponseSchema,
  pagedUsersSchema,
  registerSchema,
  requestEmailChangeFormSchema,
  requestEmailChangeSchema,
  resetSchema,
  tokenResponseSchema,
  updateProfileSchema,
  userSchema,
} from '@account/schemas/auth.schema';

export type RegisterRequest = z.infer<typeof registerSchema>;
export type LoginRequest = z.infer<typeof loginSchema>;
export type ChangePasswordRequest = z.infer<typeof changePasswordSchema>;
export type ForgotPasswordRequest = z.infer<typeof forgotSchema>;
export type ResetPasswordRequest = z.infer<typeof resetSchema>;

export type TokenResponse = z.infer<typeof tokenResponseSchema>;
export type User = z.infer<typeof userSchema>;

// --- identity-user-management (planejado) ---
export type UpdateProfileRequest = z.infer<typeof updateProfileSchema>;
export type RequestEmailChangeRequest = z.infer<typeof requestEmailChangeSchema>;
export type RequestEmailChangeFormValues = z.infer<typeof requestEmailChangeFormSchema>;
export type MessageResponse = z.infer<typeof messageResponseSchema>;
export type PagedUsers = z.infer<typeof pagedUsersSchema>;
export type UsersListParams = { page: number; size: number };
```

### 4. `src/lib/api-error.ts` — novo

```ts
// src/lib/api-error.ts — novo
import { ApiError } from '@web/lib/fetcher';

/**
 * Extrai a mensagem de negócio do envelope de erro de domínio do backend
 * (`{ "detail": "mensagem" }` para 401/403/404/409/410, ver
 * `identity-user-management/integration.md`, "Erros esperados"). Cai no
 * fallback da mutation quando o payload não tem esse formato (422 de
 * validação, que usa `{ detail: [{ field, message }] }`, ou erro de rede).
 *
 * Não existia utilitário equivalente em `web.ludens` até esta feature, as
 * mutations de `identity-auth` sempre usaram uma string estática por
 * mutation (`useAuthMutations.ts`). Esta feature precisa da mensagem
 * literal do backend porque `logic.md` fecha o texto de vários erros
 * palavra por palavra (ex.: "Você é a única pessoa administradora da
 * plataforma. Convide outra pessoa administradora antes de encerrar sua
 * conta." — reescrever isso no frontend duplicaria o texto aprovado e
 * arrisca divergir dele com o tempo).
 */
export function messageFor(error: unknown, fallback: string): string {
  if (
    error instanceof ApiError &&
    typeof error.data === 'object' &&
    error.data !== null &&
    'detail' in error.data
  ) {
    const detail = (error.data as { detail: unknown }).detail;
    if (typeof detail === 'string') return detail;
  }
  return fallback;
}
```

Importado por caminho direto (`@web/lib/api-error`), não por um barrel,
`src/lib/` neste repo não tem `index.ts` hoje (`fetcher.ts`, `jwt.ts`,
`zod-pt-br.ts` também são importados direto); é a convenção real já
estabelecida para esta pasta, não uma barrel esquecida.

### 5. `src/lib/api-error.test.ts` — novo

```ts
// src/lib/api-error.test.ts — novo
import { ApiError } from '@web/lib/fetcher';
import { messageFor } from './api-error';

describe('messageFor', () => {
  it('usa o detail da ApiError quando é string', () => {
    const error = new ApiError(409, { detail: 'Este e-mail já está em uso.' });
    expect(messageFor(error, 'fallback')).toBe('Este e-mail já está em uso.');
  });

  it('cai no fallback quando detail não é string (422 de validação)', () => {
    const error = new ApiError(422, { detail: [{ field: 'name', message: 'obrigatório' }] });
    expect(messageFor(error, 'fallback')).toBe('fallback');
  });

  it('cai no fallback para erro que não é ApiError', () => {
    expect(messageFor(new Error('rede caiu'), 'fallback')).toBe('fallback');
  });
});
```

### 6. `src/features/account/services/auth.service.ts` — editar

```ts
// src/features/account/services/auth.service.ts — editar (arquivo inteiro)
import { fetcher } from '@web/lib/fetcher';
import { endpoints } from '@web/routes/endpoints';

import type {
  ChangePasswordRequest,
  ForgotPasswordRequest,
  LoginRequest,
  MessageResponse,
  PagedUsers,
  RegisterRequest,
  RequestEmailChangeRequest,
  ResetPasswordRequest,
  TokenResponse,
  UpdateProfileRequest,
  User,
  UsersListParams,
} from '@account/server/types/auth.types';

export const authService = {
  register(data: RegisterRequest) {
    return fetcher<TokenResponse>(endpoints.users.register, {
      method: 'POST',
      body: JSON.stringify(data),
      skipAuth: true,
    });
  },

  login(data: LoginRequest) {
    return fetcher<TokenResponse>(endpoints.auth.login, {
      method: 'POST',
      body: JSON.stringify(data),
      skipAuth: true,
    });
  },

  refresh() {
    return fetcher<TokenResponse>(endpoints.auth.refresh, {
      method: 'POST',
      skipAuth: true,
      skipRefresh: true,
    });
  },

  logout() {
    return fetcher<void>(endpoints.auth.logout, { method: 'POST' });
  },

  fetchUserById(id: string) {
    return fetcher<User>(endpoints.users.byId(id), { method: 'GET' });
  },

  changePassword(data: ChangePasswordRequest) {
    return fetcher<void>(endpoints.auth.changePassword, {
      method: 'POST',
      body: JSON.stringify(data),
    });
  },

  forgotPassword(data: ForgotPasswordRequest) {
    return fetcher<MessageResponse>(endpoints.auth.passwordForgot, {
      method: 'POST',
      body: JSON.stringify(data),
      skipAuth: true,
    });
  },

  resetPassword(data: ResetPasswordRequest) {
    return fetcher<void>(endpoints.auth.passwordReset, {
      method: 'POST',
      body: JSON.stringify(data),
      skipAuth: true,
    });
  },

  // --- identity-user-management (planejado) ---

  updateProfile(data: UpdateProfileRequest) {
    return fetcher<User>(endpoints.users.self, {
      method: 'PATCH',
      body: JSON.stringify(data),
    });
  },

  requestEmailChange(data: RequestEmailChangeRequest) {
    return fetcher<MessageResponse>(endpoints.users.emailChange, {
      method: 'POST',
      body: JSON.stringify(data),
    });
  },

  // Rota pública por natureza (integration.md): quem prova identidade é o
  // token na query string, não uma sessão, por isso `skipAuth: true`
  // mesmo que a pessoa esteja com um access token velho em memória nesta
  // aba (não faz diferença, o backend nem olha o header aqui).
  confirmEmailChange(token: string) {
    return fetcher<void>(`${endpoints.users.emailChange}?token=${encodeURIComponent(token)}`, {
      method: 'PATCH',
      skipAuth: true,
    });
  },

  requestAccountDeletion() {
    return fetcher<MessageResponse>(endpoints.users.deletion, { method: 'POST' });
  },

  confirmAccountDeletion(token: string) {
    return fetcher<void>(`${endpoints.users.deletion}?token=${encodeURIComponent(token)}`, {
      method: 'DELETE',
      skipAuth: true,
    });
  },

  listUsers({ page, size }: UsersListParams) {
    return fetcher<PagedUsers>(`${endpoints.users.list}?page=${page}&size=${size}`, {
      method: 'GET',
    });
  },
};
```

### 7. `src/features/account/services/auth.service.test.ts` — editar

```ts
// src/features/account/services/auth.service.test.ts — editar (adiciona
// casos ao describe existente, mesmo estilo dos já escritos)
it('updateProfile bate em PATCH /api/identity/users, autenticado', async () => {
  await authService.updateProfile({ name: 'Maria Nova' });

  expect(mockedFetcher).toHaveBeenCalledWith('/api/identity/users', {
    method: 'PATCH',
    body: JSON.stringify({ name: 'Maria Nova' }),
  });
});

it('requestEmailChange bate em POST /api/identity/users/email/change, autenticado', async () => {
  await authService.requestEmailChange({ new_email: 'novo@example.com' });

  expect(mockedFetcher).toHaveBeenCalledWith('/api/identity/users/email/change', {
    method: 'POST',
    body: JSON.stringify({ new_email: 'novo@example.com' }),
  });
});

it('confirmEmailChange bate em PATCH /api/identity/users/email/change?token=..., sem auth', async () => {
  await authService.confirmEmailChange('tok-123');

  expect(mockedFetcher).toHaveBeenCalledWith(
    '/api/identity/users/email/change?token=tok-123',
    { method: 'PATCH', skipAuth: true },
  );
});

it('requestAccountDeletion bate em POST /api/identity/users/deletion, autenticado', async () => {
  await authService.requestAccountDeletion();

  expect(mockedFetcher).toHaveBeenCalledWith('/api/identity/users/deletion', { method: 'POST' });
});

it('confirmAccountDeletion bate em DELETE /api/identity/users/deletion?token=..., sem auth', async () => {
  await authService.confirmAccountDeletion('tok-456');

  expect(mockedFetcher).toHaveBeenCalledWith(
    '/api/identity/users/deletion?token=tok-456',
    { method: 'DELETE', skipAuth: true },
  );
});

it('listUsers bate em GET /api/identity/users?page=&size=, autenticado', async () => {
  await authService.listUsers({ page: 2, size: 20 });

  expect(mockedFetcher).toHaveBeenCalledWith('/api/identity/users?page=2&size=20', {
    method: 'GET',
  });
});
```

### 8. `src/features/account/schemas/auth.schema.test.ts` — editar

```ts
// adiciona ao describe existente
it('requestEmailChangeFormSchema rejeita quando os e-mails não coincidem', () => {
  const result = requestEmailChangeFormSchema.safeParse({
    new_email: 'a@x.com',
    confirm_new_email: 'b@x.com',
  });

  expect(result.success).toBe(false);
});

it('updateProfileSchema rejeita nome vazio', () => {
  expect(updateProfileSchema.safeParse({ name: '' }).success).toBe(false);
});
```

### 9. `src/features/account/hooks/queries/query-options.ts` — editar

```ts
// src/features/account/hooks/queries/query-options.ts — editar (arquivo inteiro)
import { queryOptions } from '@tanstack/react-query';

import { authService } from '@account/services/auth.service';
import type { UsersListParams } from '@account/server/types/auth.types';

export const accountQueryKeys = {
  all: ['account'] as const,
  currentUser: (userId: string) => [...accountQueryKeys.all, 'currentUser', userId] as const,
  usersList: (params: UsersListParams) => [...accountQueryKeys.all, 'usersList', params] as const,
};

export const accountQueryOptions = {
  currentUser: (userId: string) =>
    queryOptions({
      queryKey: accountQueryKeys.currentUser(userId),
      queryFn: () => authService.fetchUserById(userId),
    }),

  usersList: (params: UsersListParams) =>
    queryOptions({
      queryKey: accountQueryKeys.usersList(params),
      queryFn: () => authService.listUsers(params),
    }),
};
```

### 10. `src/features/account/hooks/queries/useUsersQueries.ts` — novo

```ts
// src/features/account/hooks/queries/useUsersQueries.ts — novo
'use client';

import { useQuery } from '@tanstack/react-query';

import type { UsersListParams } from '@account/server/types/auth.types';
import { accountQueryOptions } from './query-options';

export function useUsersList(params: UsersListParams) {
  return useQuery(accountQueryOptions.usersList(params));
}
```

### 11. `src/features/account/hooks/queries/index.ts` — editar

```ts
export * from './query-options';
export * from './useAccountQueries';
export * from './useUsersQueries';
```

### 12. `src/features/account/hooks/useUsersFilters.ts` — novo

```ts
// src/features/account/hooks/useUsersFilters.ts — novo
'use client';

import { usePathname, useRouter, useSearchParams } from 'next/navigation';
import { useCallback, useMemo } from 'react';

// Mesmo default do backend (PaginationParams: size 1–50, padrão 20), não
// há filtro por nome/e-mail nesta tela (logic.md não pede busca, só
// listagem paginada), então o único estado é a página.
const USERS_PAGE_SIZE = 20;

export function useUsersFilters() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  const params = useMemo(
    () => ({
      page: Number(searchParams.get('page') ?? '1'),
      size: USERS_PAGE_SIZE,
    }),
    [searchParams],
  );

  const setPage = useCallback(
    (page: number) => {
      const next = new URLSearchParams(searchParams.toString());
      next.set('page', String(page));
      router.push(`${pathname}?${next.toString()}`);
    },
    [pathname, router, searchParams],
  );

  return { params, setPage };
}
```

### 13. `src/features/account/hooks/index.ts` — editar

```ts
export * from './forms';
export * from './mutations';
export * from './queries';
export * from './useUsersFilters';
```

### 14. `src/features/account/hooks/mutations/useUserMutations.ts` — novo

```ts
// src/features/account/hooks/mutations/useUserMutations.ts — novo
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';
import { toast } from 'sonner';

import { messageFor } from '@web/lib/api-error';

import { useAuth } from '@account/contexts/AuthContext';
import { accountQueryKeys } from '@account/hooks/queries';
import { authService } from '@account/services/auth.service';

export function useUserMutations() {
  const queryClient = useQueryClient();
  const { userId, logout } = useAuth();

  const updateProfileMutation = useMutation({
    mutationFn: authService.updateProfile,

    onSuccess: async () => {
      if (userId) {
        await queryClient.invalidateQueries({ queryKey: accountQueryKeys.currentUser(userId) });
      }
      toast.success('Perfil atualizado.');
    },

    onError: (error: unknown) => {
      toast.error(messageFor(error, 'Não foi possível atualizar seu perfil.'));
    },
  });

  const requestEmailChangeMutation = useMutation({
    mutationFn: authService.requestEmailChange,

    onSuccess: (response) => {
      // Mensagem já vem pronta do backend, com o e-mail ATUAL da conta
      // interpolado, nunca montar essa frase no frontend
      // (integration.md, "Impacto de UX").
      toast.success(response.message);
    },

    onError: (error: unknown) => {
      toast.error(messageFor(error, 'Não foi possível solicitar a troca de e-mail.'));
    },
  });

  // Disparada só pelas páginas de confirmação (arquivos 30/31), nunca por
  // um formulário, por isso não há `mutateAsync` num handleSubmit aqui.
  const confirmEmailChangeMutation = useMutation({
    mutationFn: authService.confirmEmailChange,

    onSuccess: () => {
      toast.success('E-mail confirmado. Todas as sessões foram encerradas.');
      // O backend já derrubou a sessão (security_stamp), isto só limpa o
      // estado local desta aba. Ver risco anotado em § 7 sobre reaproveitar
      // `logout()` (que também chama POST /auth/logout) aqui.
      void logout();
    },

    onError: (error: unknown) => {
      toast.error(messageFor(error, 'Este link não é mais válido, solicite um novo.'));
    },
  });

  const requestAccountDeletionMutation = useMutation({
    mutationFn: authService.requestAccountDeletion,

    onSuccess: (response) => {
      toast.success(response.message);
    },

    onError: (error: unknown) => {
      // Cobre também o 409 de "único administrador restante", a mensagem
      // vem literal do backend via messageFor (logic.md fecha esse texto).
      toast.error(messageFor(error, 'Não foi possível solicitar o encerramento da conta.'));
    },
  });

  const confirmAccountDeletionMutation = useMutation({
    mutationFn: authService.confirmAccountDeletion,

    onSuccess: () => {
      toast.success('Conta encerrada. Todas as sessões foram encerradas.');
      void logout();
    },

    onError: (error: unknown) => {
      toast.error(messageFor(error, 'Este link não é mais válido, solicite um novo.'));
    },
  });

  return {
    updateProfileMutation,
    requestEmailChangeMutation,
    confirmEmailChangeMutation,
    requestAccountDeletionMutation,
    confirmAccountDeletionMutation,
  };
}
```

### 15. `src/features/account/hooks/mutations/index.ts` — editar

```ts
export * from './useAuthMutations';
export * from './useUserMutations';
```

### 16. `src/features/account/hooks/forms/useUpdateProfileForm.ts` — novo

```ts
// src/features/account/hooks/forms/useUpdateProfileForm.ts — novo
'use client';

import { useEffect } from 'react';
import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';

import { updateProfileSchema } from '@account/schemas/auth.schema';
import type { UpdateProfileRequest, User } from '@account/server/types/auth.types';

export function useUpdateProfileForm(user: User) {
  const form = useForm<UpdateProfileRequest>({
    resolver: zodResolver(updateProfileSchema),
    defaultValues: { name: user.name },
  });

  // Sincroniza o form se o nome mudar por fora (ex.: invalidate depois de
  // salvar), mesmo racional de "reset em edição assíncrona" (references/06).
  useEffect(() => {
    form.reset({ name: user.name });
  }, [user.name, form]);

  return form;
}
```

### 17. `src/features/account/hooks/forms/useRequestEmailChangeForm.ts` — novo

```ts
// src/features/account/hooks/forms/useRequestEmailChangeForm.ts — novo
'use client';

import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';

import { requestEmailChangeFormSchema } from '@account/schemas/auth.schema';
import type { RequestEmailChangeFormValues } from '@account/server/types/auth.types';

export function useRequestEmailChangeForm() {
  return useForm<RequestEmailChangeFormValues>({
    resolver: zodResolver(requestEmailChangeFormSchema),
    defaultValues: { new_email: '', confirm_new_email: '' },
  });
}
```

### 18. `src/features/account/hooks/forms/index.ts` — editar

```ts
export * from './useForgotPasswordForm';
export * from './useLoginForm';
export * from './useRegisterForm';
export * from './useResetPasswordForm';
export * from './useUpdateProfileForm';
export * from './useRequestEmailChangeForm';
```

### 19. `src/features/account/components/ui/ProfileSummaryCard.tsx` — novo

```tsx
// src/features/account/components/ui/ProfileSummaryCard.tsx — novo
import { Badge } from '@components/ui/badge';
import { Card, CardContent, CardHeader, CardTitle } from '@components/ui/card';

import type { User } from '@account/server/types/auth.types';

export function ProfileSummaryCard({ user }: { user: User }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Meus dados</CardTitle>
      </CardHeader>
      <CardContent className="space-y-3 text-sm">
        <div className="flex items-center justify-between">
          <span className="text-muted-foreground">Nome</span>
          <span className="font-medium">{user.name}</span>
        </div>

        <div className="flex items-center justify-between">
          <span className="text-muted-foreground">E-mail</span>
          <span className="font-medium">{user.email}</span>
        </div>

        {/* CPF é só leitura, imutável após o cadastro (spec.md § 6/§ 8).
            Não existe campo editável aqui nem no payload de PATCH. */}
        <div className="flex items-center justify-between">
          <span className="text-muted-foreground">CPF</span>
          <span className="font-medium">{user.cpf}</span>
        </div>

        <div className="flex items-center justify-between">
          <span className="text-muted-foreground">Perfil</span>
          <Badge variant="outline">{user.is_admin ? 'Administrador' : 'Comprador'}</Badge>
        </div>
      </CardContent>
    </Card>
  );
}
```

### 20. `src/features/account/components/ui/ConfirmResultView.tsx` — novo

```tsx
// src/features/account/components/ui/ConfirmResultView.tsx — novo
import Link from 'next/link';
import { CheckCircle2, CircleAlert, Loader2 } from 'lucide-react';

import { Button } from '@components/ui/button';

type ConfirmResultViewProps = {
  status: 'loading' | 'success' | 'error';
  title: string;
  description: string;
  ctaHref?: string;
  ctaLabel?: string;
};

// Visual puro, reaproveitado pelas duas páginas de confirmação (arquivos
// 30/31), a única diferença entre elas é texto e destino do CTA.
export function ConfirmResultView({
  status,
  title,
  description,
  ctaHref,
  ctaLabel,
}: ConfirmResultViewProps) {
  return (
    <div className="flex flex-col items-center gap-3 text-center">
      {status === 'loading' ? <Loader2 className="size-8 animate-spin text-muted-foreground" /> : null}
      {status === 'success' ? <CheckCircle2 className="size-8 text-emerald-600" /> : null}
      {status === 'error' ? <CircleAlert className="size-8 text-destructive" /> : null}

      <h2 className="font-medium">{title}</h2>
      <p className="text-sm text-muted-foreground">{description}</p>

      {ctaHref && ctaLabel ? (
        <Button asChild className="mt-2 w-full">
          <Link href={ctaHref}>{ctaLabel}</Link>
        </Button>
      ) : null}
    </div>
  );
}
```

### 21. `src/features/account/components/ui/UsersTable.tsx` — novo

```tsx
// src/features/account/components/ui/UsersTable.tsx — novo
import { Badge } from '@components/ui/badge';
import { Card, CardContent } from '@components/ui/card';

import type { User } from '@account/server/types/auth.types';

interface UsersTableProps {
  users: User[];
}

// GET /identity/users devolve `cpf` em cada item, deliberadamente NÃO
// renderizado: logic.md § 3 fecha "a listagem expõe nome, e-mail, papel e
// data de criação. Nunca CPF" (RNF01); backend.md § 4 item 1 registra que o
// backend expõe esse campo hoje como bug conhecido (violação de RNF01 em
// produção). Esconder na UI é a mitigação imediata possível sem tocar o
// backend, o dado já trafegou na resposta, mas não aparece na tela.
//
// Também NÃO existe coluna "criado em": `created_at` não vem no shape
// atual (mesmo débito). Não inventar um valor aproximado/placeholder, a
// coluna fica de fora até o backend corrigir (ver frontend.md § 7).
//
// Sem <table> HTML, o design system de web.ludens não tem um componente
// de tabela hoje (só Card/Badge/etc.); segue o mesmo padrão de lista em
// Card já usado por GenreTable/ShowList em `catalog`, não introduz uma
// dependência nova pra esta tela.
export function UsersTable({ users }: UsersTableProps) {
  return (
    <div className="flex flex-col gap-3">
      {users.map((user) => (
        <Card key={user.id}>
          <CardContent className="flex flex-wrap items-center justify-between gap-3">
            <div>
              <p className="font-medium">{user.name}</p>
              <p className="text-sm text-muted-foreground">{user.email}</p>
            </div>
            <Badge variant="outline">{user.is_admin ? 'Administrador' : 'Comprador'}</Badge>
          </CardContent>
        </Card>
      ))}
    </div>
  );
}
```

### 22. `src/features/account/components/ui/index.ts` — editar

```ts
export * from './AuthShell';
export * from './ProfileSummaryCard';
export * from './ConfirmResultView';
export * from './UsersTable';
```

### 23. `src/features/account/components/forms/UpdateProfileForm.tsx` — novo

```tsx
// src/features/account/components/forms/UpdateProfileForm.tsx — novo
'use client';

import type { UseFormReturn } from 'react-hook-form';

import { Button } from '@components/ui/button';
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@components/ui/form';
import { Input } from '@components/ui/input';

import type { UpdateProfileRequest } from '@account/server/types/auth.types';

type UpdateProfileFormProps = {
  form: UseFormReturn<UpdateProfileRequest>;
  onSubmit: () => void;
  isPending: boolean;
};

export function UpdateProfileForm({ form, onSubmit, isPending }: UpdateProfileFormProps) {
  return (
    <Form {...form}>
      <form className="flex flex-col gap-4" onSubmit={onSubmit}>
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Nome</FormLabel>
              <FormControl>
                <Input type="text" autoComplete="name" disabled={isPending} {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" className="w-fit" disabled={isPending}>
          {isPending ? 'Salvando...' : 'Salvar nome'}
        </Button>
      </form>
    </Form>
  );
}
```

### 24. `src/features/account/components/forms/RequestEmailChangeForm.tsx` — novo

```tsx
// src/features/account/components/forms/RequestEmailChangeForm.tsx — novo
'use client';

import type { UseFormReturn } from 'react-hook-form';

import { Button } from '@components/ui/button';
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@components/ui/form';
import { Input } from '@components/ui/input';

import type { RequestEmailChangeFormValues } from '@account/server/types/auth.types';

type RequestEmailChangeFormProps = {
  form: UseFormReturn<RequestEmailChangeFormValues>;
  onSubmit: () => void;
  isPending: boolean;
};

export function RequestEmailChangeForm({ form, onSubmit, isPending }: RequestEmailChangeFormProps) {
  return (
    <Form {...form}>
      <form className="flex flex-col gap-4" onSubmit={onSubmit}>
        <FormField
          control={form.control}
          name="new_email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Novo e-mail</FormLabel>
              <FormControl>
                <Input type="email" autoComplete="email" disabled={isPending} {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="confirm_new_email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Confirme o novo e-mail</FormLabel>
              <FormControl>
                <Input type="email" autoComplete="off" disabled={isPending} {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" className="w-full" disabled={isPending}>
          {isPending ? 'Enviando...' : 'Enviar link de confirmação'}
        </Button>
      </form>
    </Form>
  );
}
```

### 25. `src/features/account/components/forms/index.ts` — editar

```ts
export * from './ForgotPasswordForm';
export * from './LoginForm';
export * from './RegisterForm';
export * from './ResetPasswordForm';
export * from './UpdateProfileForm';
export * from './RequestEmailChangeForm';
```

### 26. `src/features/account/components/UpdateProfileFormContainer.tsx` — novo

```tsx
// src/features/account/components/UpdateProfileFormContainer.tsx — novo
'use client';

import { UpdateProfileForm } from '@account/components/forms';
import { useUpdateProfileForm } from '@account/hooks/forms';
import { useUserMutations } from '@account/hooks/mutations';
import type { User } from '@account/server/types/auth.types';

export function UpdateProfileFormContainer({ user }: { user: User }) {
  const form = useUpdateProfileForm(user);
  const { updateProfileMutation } = useUserMutations();

  const onSubmit = form.handleSubmit(async (data) => {
    try {
      await updateProfileMutation.mutateAsync(data);
    } catch {
      // O toast de erro já é tratado na mutation.
    }
  });

  return (
    <UpdateProfileForm form={form} onSubmit={onSubmit} isPending={updateProfileMutation.isPending} />
  );
}
```

### 27. `src/features/account/components/RequestEmailChangeContainer.tsx` — novo

```tsx
// src/features/account/components/RequestEmailChangeContainer.tsx — novo
'use client';

import { useState } from 'react';

import { Button } from '@components/ui/button';
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@components/ui/dialog';

import { RequestEmailChangeForm } from '@account/components/forms';
import { useRequestEmailChangeForm } from '@account/hooks/forms';
import { useUserMutations } from '@account/hooks/mutations';

export function RequestEmailChangeContainer({ currentEmail }: { currentEmail: string }) {
  const [open, setOpen] = useState(false);
  const form = useRequestEmailChangeForm();
  const { requestEmailChangeMutation } = useUserMutations();

  const onSubmit = form.handleSubmit(async (data) => {
    try {
      // DTO real só tem `new_email`, `confirm_new_email` não sai desta
      // tela (backend.md § 4 item 2; requestEmailChangeSchema).
      await requestEmailChangeMutation.mutateAsync({ new_email: data.new_email });
      form.reset();
      setOpen(false);
    } catch {
      // O toast de erro já é tratado na mutation.
    }
  });

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button type="button" variant="outline">
          Trocar e-mail
        </Button>
      </DialogTrigger>

      <DialogContent>
        <DialogHeader>
          <DialogTitle>Trocar e-mail</DialogTitle>
          <DialogDescription>
            Enviaremos um link de confirmação para o e-mail ATUAL da sua conta
            ({currentEmail}), não para o endereço novo. A troca só vale
            depois que você abrir esse link, e ao confirmar todas as suas
            sessões são encerradas.
          </DialogDescription>
        </DialogHeader>

        <RequestEmailChangeForm
          form={form}
          onSubmit={onSubmit}
          isPending={requestEmailChangeMutation.isPending}
        />
      </DialogContent>
    </Dialog>
  );
}
```

Este texto no `DialogDescription`, escrito **antes** de qualquer submit, é o
que cobre a exigência de `logic.md` § 4 ("a tela precisa dizer que o link foi
para o e-mail ATUAL, a pessoa vai procurar na caixa errada se a tela não
disser isso"); o toast de sucesso (mutation, arquivo 14) reforça o mesmo fato
com o texto literal do backend.

### 28. `src/features/account/components/RequestAccountDeletionContainer.tsx` — novo

```tsx
// src/features/account/components/RequestAccountDeletionContainer.tsx — novo
'use client';

import { useState } from 'react';
import { TriangleAlert } from 'lucide-react';

import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from '@components/ui/alert-dialog';
import { Button } from '@components/ui/button';

import { useUserMutations } from '@account/hooks/mutations';

// Mesmo padrão de ConfirmCancelSessionDialog (catalog), toda ação
// destrutiva leva AlertDialog (references/11-ux-principles.md #8).
export function RequestAccountDeletionContainer() {
  const [open, setOpen] = useState(false);
  const { requestAccountDeletionMutation } = useUserMutations();

  const onConfirm = async () => {
    try {
      await requestAccountDeletionMutation.mutateAsync();
      setOpen(false);
    } catch {
      // O toast de erro já é tratado na mutation (ex.: único administrador
      // restante, 409, mensagem literal do backend via messageFor).
    }
  };

  return (
    <>
      <Button type="button" variant="destructive" onClick={() => setOpen(true)}>
        Encerrar minha conta
      </Button>

      <AlertDialog
        open={open}
        onOpenChange={(next) => {
          if (!next && !requestAccountDeletionMutation.isPending) setOpen(false);
        }}
      >
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle className="flex items-center gap-2">
              <TriangleAlert className="size-5 text-destructive" />
              Encerrar sua conta?
            </AlertDialogTitle>

            <AlertDialogDescription>
              Enviaremos um link de confirmação para o e-mail da sua conta,
              a conta só é encerrada depois que você abrir esse link.
            </AlertDialogDescription>

            <AlertDialogDescription>
              Ao confirmar: seu histórico de pedidos e o pedido de
              cancelamento/reembolso deixam de estar ao seu alcance.
              Ingressos já emitidos para sessões futuras continuam valendo na
              porta (o que é validado é o código do ingresso, não a conta).
            </AlertDialogDescription>
          </AlertDialogHeader>

          <AlertDialogFooter>
            <AlertDialogCancel
              disabled={requestAccountDeletionMutation.isPending}
              onClick={() => setOpen(false)}
            >
              Voltar
            </AlertDialogCancel>

            <AlertDialogAction
              disabled={requestAccountDeletionMutation.isPending}
              onClick={(event) => {
                event.preventDefault();
                void onConfirm();
              }}
            >
              {requestAccountDeletionMutation.isPending ? 'Enviando...' : 'Encerrar conta'}
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </>
  );
}
```

Este texto de aviso é o que cobre `logic.md` § 1 ("Encerra a própria conta"),
passo 2: mostrar **antes de qualquer confirmação** o que se perde e o que
continua valendo.

### 29. `src/features/account/components/ProfileContainer.tsx` — novo

```tsx
// src/features/account/components/ProfileContainer.tsx — novo
'use client';

import { Alert, AlertDescription, AlertTitle } from '@components/ui/alert';
import { Skeleton } from '@components/ui/skeleton';

import { useCurrentUser } from '@account/hooks/queries';
import { ProfileSummaryCard } from '@account/components/ui';

import { RequestAccountDeletionContainer } from './RequestAccountDeletionContainer';
import { RequestEmailChangeContainer } from './RequestEmailChangeContainer';
import { UpdateProfileFormContainer } from './UpdateProfileFormContainer';

export function ProfileContainer() {
  const { data: user, isLoading, isError } = useCurrentUser();

  if (isLoading) {
    return (
      <div className="mx-auto max-w-2xl space-y-6 p-6">
        <Skeleton className="h-48 w-full" />
        <Skeleton className="h-24 w-full" />
      </div>
    );
  }

  if (isError || !user) {
    return (
      <div className="mx-auto max-w-2xl p-6">
        <Alert variant="destructive">
          <AlertTitle>Não foi possível carregar seus dados</AlertTitle>
          <AlertDescription>Verifique sua conexão e tente novamente.</AlertDescription>
        </Alert>
      </div>
    );
  }

  return (
    <div className="mx-auto max-w-2xl space-y-6 p-6">
      <ProfileSummaryCard user={user} />
      <UpdateProfileFormContainer user={user} />

      <div className="flex flex-wrap items-center gap-3">
        <RequestEmailChangeContainer currentEmail={user.email} />
        <RequestAccountDeletionContainer />
      </div>
    </div>
  );
}
```

Não usa `RequireAuth`/gate aqui dentro, quem gateia é a página
(`src/app/perfil/page.tsx`, arquivo 35), mesmo padrão de
`AdminHub`/`RequireAdmin` em `catalog`.

### 30. `src/features/account/components/ConfirmEmailChangeContainer.tsx` — novo

```tsx
// src/features/account/components/ConfirmEmailChangeContainer.tsx — novo
'use client';

import { useEffect, useRef } from 'react';
import { useSearchParams } from 'next/navigation';

import { ConfirmResultView } from '@account/components/ui';
import { useUserMutations } from '@account/hooks/mutations';
import { messageFor } from '@web/lib/api-error';

export function ConfirmEmailChangeContainer() {
  const searchParams = useSearchParams();
  const token = searchParams.get('token');
  const { confirmEmailChangeMutation } = useUserMutations();

  // Guarda contra disparo duplo (Strict Mode roda efeitos duas vezes em
  // dev), o token é de uso único, uma segunda chamada devolveria 410
  // mesmo tendo confirmado com sucesso na primeira.
  const firedRef = useRef(false);

  useEffect(() => {
    if (!token || firedRef.current) return;
    firedRef.current = true;
    confirmEmailChangeMutation.mutate(token);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [token]);

  if (!token) {
    return (
      <ConfirmResultView
        status="error"
        title="Link inválido"
        description="Este link de confirmação está incompleto. Solicite a troca de e-mail novamente em Meu Perfil."
        ctaHref="/perfil"
        ctaLabel="Ir para meu perfil"
      />
    );
  }

  if (confirmEmailChangeMutation.isError) {
    return (
      <ConfirmResultView
        status="error"
        title="Não foi possível confirmar"
        description={messageFor(
          confirmEmailChangeMutation.error,
          'Este link não é mais válido, solicite um novo.',
        )}
        ctaHref="/perfil"
        ctaLabel="Ir para meu perfil"
      />
    );
  }

  if (confirmEmailChangeMutation.isSuccess) {
    return (
      <ConfirmResultView
        status="success"
        title="E-mail confirmado"
        description="Seu e-mail foi atualizado e todas as suas sessões foram encerradas. Entre novamente com o e-mail novo."
        ctaHref="/login"
        ctaLabel="Ir para o login"
      />
    );
  }

  return (
    <ConfirmResultView
      status="loading"
      title="Confirmando a troca de e-mail..."
      description="Aguarde um instante."
    />
  );
}
```

### 31. `src/features/account/components/ConfirmAccountDeletionContainer.tsx` — novo

```tsx
// src/features/account/components/ConfirmAccountDeletionContainer.tsx — novo
'use client';

import { useEffect, useRef } from 'react';
import { useSearchParams } from 'next/navigation';

import { ConfirmResultView } from '@account/components/ui';
import { useUserMutations } from '@account/hooks/mutations';
import { messageFor } from '@web/lib/api-error';

export function ConfirmAccountDeletionContainer() {
  const searchParams = useSearchParams();
  const token = searchParams.get('token');
  const { confirmAccountDeletionMutation } = useUserMutations();

  const firedRef = useRef(false);

  useEffect(() => {
    if (!token || firedRef.current) return;
    firedRef.current = true;
    confirmAccountDeletionMutation.mutate(token);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [token]);

  if (!token) {
    return (
      <ConfirmResultView
        status="error"
        title="Link inválido"
        description="Este link de confirmação está incompleto. Solicite o encerramento novamente em Meu Perfil."
        ctaHref="/perfil"
        ctaLabel="Ir para meu perfil"
      />
    );
  }

  if (confirmAccountDeletionMutation.isError) {
    return (
      <ConfirmResultView
        status="error"
        title="Não foi possível confirmar"
        description={messageFor(
          confirmAccountDeletionMutation.error,
          'Este link não é mais válido, solicite um novo.',
        )}
        ctaHref="/perfil"
        ctaLabel="Ir para meu perfil"
      />
    );
  }

  if (confirmAccountDeletionMutation.isSuccess) {
    return (
      <ConfirmResultView
        status="success"
        title="Conta encerrada"
        description="Sua conta foi encerrada e todas as suas sessões foram encerradas."
        ctaHref="/"
        ctaLabel="Ir para o início"
      />
    );
  }

  return (
    <ConfirmResultView
      status="loading"
      title="Confirmando o encerramento..."
      description="Aguarde um instante."
    />
  );
}
```

### 32. `src/features/account/components/AdminUsersManager.tsx` — novo

```tsx
// src/features/account/components/AdminUsersManager.tsx — novo
'use client';

import { RotateCw } from 'lucide-react';

import { Alert, AlertDescription, AlertTitle } from '@components/ui/alert';
import { Button } from '@components/ui/button';
import { Skeleton } from '@components/ui/skeleton';

// Reaproveita Pagination de `catalog`, mesmo componente que já pagina a
// vitrine (ShowGrid). Não duplicar um `Pagination` dentro de `account`
// (references/01-architecture.md, "Verifique se já existe").
import { Pagination } from '@catalog/components/ui';

import { UsersTable } from '@account/components/ui';
import { useUsersFilters } from '@account/hooks';
import { useUsersList } from '@account/hooks/queries';

export function AdminUsersManager() {
  const { params, setPage } = useUsersFilters();
  const query = useUsersList(params);

  return (
    <main className="mx-auto max-w-4xl space-y-6 p-6">
      <div>
        <h1 className="font-heading text-2xl font-semibold tracking-tight">
          Contas da plataforma
        </h1>
        <p className="text-sm text-muted-foreground">
          Nome, e-mail e papel de quem tem conta. Tela de leitura, nenhuma
          ação sobre a conta listada.
        </p>
      </div>

      {query.isLoading ? (
        <div className="space-y-3">
          <Skeleton className="h-16 w-full" />
          <Skeleton className="h-16 w-full" />
          <Skeleton className="h-16 w-full" />
        </div>
      ) : query.isError ? (
        <Alert variant="destructive">
          <AlertTitle>Não foi possível carregar as contas</AlertTitle>
          <AlertDescription className="flex flex-col gap-3">
            <span>Verifique sua conexão e tente novamente.</span>
            <Button
              type="button"
              variant="outline"
              size="sm"
              className="w-fit"
              onClick={() => void query.refetch()}
            >
              <RotateCw />
              Tentar de novo
            </Button>
          </AlertDescription>
        </Alert>
      ) : query.data && query.data.items.length === 0 ? (
        <p className="text-sm text-muted-foreground">Nenhuma conta ativa encontrada.</p>
      ) : query.data ? (
        <>
          <UsersTable users={query.data.items} />
          <Pagination
            page={query.data.page}
            size={query.data.size}
            total={query.data.total}
            onPageChange={setPage}
          />
        </>
      ) : null}
    </main>
  );
}
```

### 33. `src/features/account/components/index.ts` — editar

```ts
export * from './ForgotPasswordFormContainer';
export * from './LoginFormContainer';
export * from './RegisterFormContainer';
export * from './RequireAuth';
export * from './ResetPasswordFormContainer';
export * from './forms';
export * from './ui';

// identity-user-management (planejado)
export * from './UpdateProfileFormContainer';
export * from './RequestEmailChangeContainer';
export * from './RequestAccountDeletionContainer';
export * from './ProfileContainer';
export * from './ConfirmEmailChangeContainer';
export * from './ConfirmAccountDeletionContainer';
export * from './AdminUsersManager';
```

### 34. `src/features/catalog/components/AdminHub.tsx` — editar

```tsx
// diff conceitual, só o necessário para adicionar a opção "Usuários"
import { Users } from 'lucide-react'; // adicionar ao import de lucide-react existente

interface HubOption {
  href?: string;
  title: string;
  description: string;
  icon: 'shows' | 'tags' | 'seats' | 'sales' | 'users'; // adicionar 'users'
}

const OPTIONS: HubOption[] = [
  // ...entradas existentes (espetáculos, gêneros, assentos, vendas)...
  {
    href: '/admin/usuarios',
    title: 'Usuários',
    description: 'Veja quem tem conta na plataforma e quem administra junto com você.',
    icon: 'users',
  },
];

function OptionIcon({ icon }: { icon: HubOption['icon'] }) {
  if (icon === 'shows') return <Drama className="size-full" />;
  if (icon === 'tags') return <Tags className="size-full" />;
  if (icon === 'seats') return <Armchair className="size-full" />;
  if (icon === 'users') return <Users className="size-full" />; // novo ramo
  return <Ticket className="size-full" />;
}
```

Edição cross-feature deliberada: `AdminHub` é o índice único de navegação
admin (já lista `catalog`/gêneros); sem esta edição, `/admin/usuarios` fica
órfã (inalcançável pela UI). É a mesma lógica de reaproveitar `RequireAdmin`
e `Pagination`, estender o que já existe em vez de duplicar um hub próprio
dentro de `account`.

### 35. `src/app/perfil/page.tsx` — novo

```tsx
// src/app/perfil/page.tsx — novo
import { ProfileContainer, RequireAuth } from '@account/components';

export default function PerfilPage() {
  return (
    <RequireAuth>
      <ProfileContainer />
    </RequireAuth>
  );
}
```

### 36. `src/app/confirmar-troca-de-email/page.tsx` — novo

```tsx
// src/app/confirmar-troca-de-email/page.tsx — novo
//
// URL fixada pelo backend já mergeado: `notification/shared/templates.py`
// (handle_email_change_requested) monta o link do e-mail como
// `{frontend_base_url}/confirmar-troca-de-email?token=...`, este caminho
// não é uma escolha de frontend, é um contrato já em produção
// (identity-user-management/backend.md, arquivo 18/19).
import { Suspense } from 'react';

import { AuthShell } from '@account/components/ui';
import { ConfirmEmailChangeContainer } from '@account/components';

export default function ConfirmarTrocaDeEmailPage() {
  return (
    <AuthShell title="Confirmar troca de e-mail" description="Só um instante...">
      <Suspense fallback={null}>
        <ConfirmEmailChangeContainer />
      </Suspense>
    </AuthShell>
  );
}
```

### 37. `src/app/confirmar-exclusao-de-conta/page.tsx` — novo

```tsx
// src/app/confirmar-exclusao-de-conta/page.tsx — novo
// Mesma observação do arquivo 36: URL fixada por
// handle_account_deletion_requested (backend já mergeado).
import { Suspense } from 'react';

import { AuthShell } from '@account/components/ui';
import { ConfirmAccountDeletionContainer } from '@account/components';

export default function ConfirmarExclusaoDeContaPage() {
  return (
    <AuthShell title="Confirmar encerramento de conta" description="Só um instante...">
      <Suspense fallback={null}>
        <ConfirmAccountDeletionContainer />
      </Suspense>
    </AuthShell>
  );
}
```

### 38. `src/app/admin/usuarios/page.tsx` — novo

```tsx
// src/app/admin/usuarios/page.tsx — novo
import { RequireAdmin } from '@catalog';
import { AdminUsersManager } from '@account/components';

export default function AdminUsuariosPage() {
  return (
    <RequireAdmin>
      <AdminUsersManager />
    </RequireAdmin>
  );
}
```

### 39. `src/features/account/README.md` — editar

```markdown
# Account

Feature responsável pelos fluxos de autenticação e conta do usuário.

## Responsabilidades

- Login
- Registro
- Recuperação de senha
- Redefinição de senha
- Sessão autenticada
- Refresh de access token
- Logout
- Consulta do usuário autenticado
- Proteção de conteúdo autenticado
- **(planejado)** Edição do próprio perfil (nome)
- **(planejado)** Troca de e-mail com confirmação por link ao e-mail atual
- **(planejado)** Encerramento de conta por autosserviço, confirmado por link
- **(planejado)** Listagem administrativa paginada de contas

## Estrutura

- `components/`: orchestration (`*FormContainer.tsx`, `*Container.tsx`,
  ligam hook + mutation + navegação/estado local) e `RequireAuth`
- `components/forms/`: formulário visual puro (`*Form.tsx`)
- `components/ui/`: `AuthShell`, `ProfileSummaryCard`, `ConfirmResultView`,
  `UsersTable`, todos apresentacionais puros
- `contexts/`: contexto de autenticação
- `hooks/forms/`: formulários com React Hook Form
- `hooks/mutations/`: mutations com TanStack Query (`useAuthMutations`,
  sessão; `useUserMutations`, perfil/e-mail/exclusão de conta)
- `hooks/queries/`: queries da conta (usuário atual, listagem admin)
- `hooks/useUsersFilters.ts`: paginação da listagem admin via URL
- `schemas/`: validação com Zod
- `server/types/`: tipos derivados dos schemas
- `services/`: integração com endpoints de `identity-auth` e
  `identity-user-management` (mesmo `user_router.py` no backend)

## Rotas

- `/login`
- `/registro`
- `/recuperar-senha`
- `/redefinir-senha`
- `/perfil` **(planejado)**, autenticado
- `/confirmar-troca-de-email` **(planejado)**, pública, token na query
- `/confirmar-exclusao-de-conta` **(planejado)**, pública, token na query
- `/admin/usuarios` **(planejado)**, admin

## Observação sobre a listagem administrativa

`GET /identity/users` devolve `cpf` e não devolve `created_at`
(`identity-user-management/backend.md` § 4 item 1, violação de RNF01 em
produção). O frontend esconde `cpf` na UI e não tem como mostrar
`created_at`; isso é uma limitação do backend, não do frontend, não
inventar aqui um workaround que simule os dois.

## Testes

`schemas/auth.schema.test.ts`, `services/auth.service.test.ts` e
`src/lib/fetcher.test.ts` (Jest, `npm run test`). Fluxos de UI (Playwright)
ainda não entraram no repo (nenhum comando `playwright` existe no
`package.json` hoje), ver
`frontend-architecture/references/13-testing.md`.
```

---

## 3. Contrato consumido

`docs.ludens/specs/identity-user-management/integration.md`. Shapes reais
(snake_case, sem transform, mesma convenção já usada por `identity-auth`
em `web.ludens`):

| Rota | Auth | Request | Response |
| --- | --- | --- | --- |
| `PATCH /identity/users` | Bearer | `{ name }` | 200 `{ id, name, email, cpf, is_admin }` |
| `POST /identity/users/email/change` | Bearer | `{ new_email }` | 202 `{ message }` |
| `PATCH /identity/users/email/change?token=...` | pública | — | 204 |
| `POST /identity/users/deletion` | Bearer | — | 202 `{ message }` |
| `DELETE /identity/users/deletion?token=...` | pública | — | 204 |
| `GET /identity/users?page=&size=` | Bearer + admin | — | 200 `{ items: User[], page, size, total }` |
| `GET /identity/users/{id}` | Bearer | — | 200 `{ id, name, email, cpf, is_admin }` |

Envelope de erro 4xx: os dois formatos já documentados em
`identity-auth/integration.md`, `{ detail: [{ field, message }] }` para 422
de validação, `{ detail: "mensagem" }` para erro de domínio (401/403/404/
409/410). `messageFor` (§ 2, arquivo 4) só sabe ler o segundo formato, o
primeiro continua exibido campo a campo via `<FormMessage />` do
react-hook-form, nunca via toast.

**Nenhuma service faz `.parse()` de schema em runtime**, segue o padrão já
estabelecido em `auth.service.ts`/`show.service.ts` (`fetcher<T>()` com
generic TS, sem validação de shape em runtime; débito rastreado como
"issue #11" no próprio código, não introduzido nem corrigido por esta
feature). Os schemas Zod desta spec (`updateProfileSchema`,
`requestEmailChangeSchema`, `pagedUsersSchema`, etc.) servem para: (a)
resolver de formulário (`zodResolver`) e (b) `z.infer` como fonte dos tipos
em `server/types/`, não para validar a resposta HTTP.

---

## 4. Estados assíncronos e mensagens

| Estado | Onde | Mensagem |
| --- | --- | --- |
| loading, perfil | `ProfileContainer` | skeletons (mesma estrutura dimensional do card real) |
| loading, salvar nome | `UpdateProfileForm` | botão "Salvando..." |
| loading, pedir troca de e-mail | `RequestEmailChangeForm` | botão "Enviando..." |
| loading, pedir exclusão | `RequestAccountDeletionContainer` | botão do `AlertDialogAction` "Enviando..." |
| loading, confirmar troca/exclusão | `ConfirmResultView` (status `loading`) | "Confirmando a troca de e-mail..." / "Confirmando o encerramento..." |
| loading, listagem admin | `AdminUsersManager` | 3 skeletons de linha |
| sucesso, nome atualizado | toast (mutation) | "Perfil atualizado." |
| sucesso, pedido de troca de e-mail | toast (mutation) | **texto literal do backend**: "Enviamos um link de confirmação para \<e-mail atual\>." (nunca montado no frontend) |
| sucesso, pedido de exclusão | toast (mutation) | **texto literal do backend**: "Enviamos um link de confirmação para \<e-mail\>." |
| sucesso, confirmar troca de e-mail | toast + `ConfirmResultView` (status `success`) | "E-mail confirmado. Todas as suas sessões foram encerradas." + CTA "Ir para o login" |
| sucesso, confirmar exclusão | toast + `ConfirmResultView` (status `success`) | "Conta encerrada. Todas as suas sessões foram encerradas." + CTA "Ir para o início" |
| erro, atualizar perfil | toast (`messageFor`) | 422 de forma via `<FormMessage/>`; fallback "Não foi possível atualizar seu perfil." |
| erro, pedir troca de e-mail | toast (`messageFor`) | 409 "Este e-mail já está em uso." (literal do backend) / fallback "Não foi possível solicitar a troca de e-mail." |
| erro, pedir exclusão | toast (`messageFor`) | 409 "Você é a única pessoa administradora da plataforma..." (literal, único administrador) / fallback genérico |
| erro, confirmar troca/exclusão | `ConfirmResultView` (status `error`) + toast (`messageFor`) | 410 "Este link não é mais válido, solicite um novo." (literal) |
| erro, token ausente na página de confirmação | `ConfirmResultView` (status `error`), sem disparar mutation | "Este link de confirmação está incompleto..." |
| erro, listagem admin | `AdminUsersManager` (`Alert` + botão "Tentar de novo") | "Não foi possível carregar as contas" |
| empty, listagem admin | `AdminUsersManager` | "Nenhuma conta ativa encontrada." |
| 403, não-admin em `/admin/usuarios` | `RequireAdmin` (reaproveitado) | redireciona para `/`, mesmo comportamento já existente para `/admin/generos`/`/admin/espetaculos` |

---

## 5. Passo a passo TBD (Frontend)

```bash
git checkout master && git pull && git checkout -b feat/09-account-self-service

# commit 1 — contrato (endpoints, schemas, types, lib compartilhada)
git add src/routes/endpoints.ts src/lib/api-error.ts src/lib/api-error.test.ts \
        src/features/account/schemas src/features/account/server

git commit -m "feat(account): contrato de perfil, troca de e-mail e exclusão de conta"

# commit 2 — services + testes
git add src/features/account/services

git commit -m "feat(account): services de perfil, troca de e-mail, exclusão e listagem admin"

# commit 3 — hooks (queries, mutations, forms)
git add src/features/account/hooks

git commit -m "feat(account): hooks de perfil, troca de e-mail, exclusão e listagem admin"

# commit 4 — UI (components/ui, components/forms, orchestration)
git add src/features/account/components src/features/catalog/components/AdminHub.tsx

git commit -m "feat(account): telas de perfil, troca de e-mail, exclusão e listagem admin"

# commit 5 — rotas
git add src/app/perfil src/app/confirmar-troca-de-email \
        src/app/confirmar-exclusao-de-conta src/app/admin/usuarios

git commit -m "feat(account): rotas de perfil, confirmação de e-mail/exclusão e listagem admin"

# commit 6 — README
git add src/features/account/README.md

git commit -m "docs(account): documenta perfil, troca de e-mail, exclusão e listagem admin"
```

Depois: `npm run build && npm run lint && npm run test` verdes,
`/team-ludens:tbd-pr`.

---

## 6. Ordem entre as superfícies

O backend desta feature **já está mergeado** (`identity-user-management/
backend.md`, `status: done`), diferente de `identity-auth`, que foi
construído contra um contrato-alvo antes do backend existir. Isso muda a
ordem de risco aqui: o frontend não está apostando em um contrato que ainda
pode mudar, está implementando contra rotas reais já em produção. A única
cautela é justamente o inverso do caso comum: **não copiar cegamente o shape
de `UserResponse`** (que inclui `cpf`) para a listagem sem aplicar a
mitigação de UI da § 7, aqui o contrato real é o problema, não a ausência
dele.

---

## 7. Riscos e pontos de atenção

1. **Bloqueante herdado do backend, não desta implementação:** a listagem
   administrativa não pode mostrar `created_at` porque o backend não
   devolve esse campo hoje (`identity-user-management/backend.md` § 4 item
   1). Não inventar um placeholder ("data desconhecida", string vazia
   fingindo ser data), a coluna simplesmente não existe até o backend ser
   corrigido. Esconder `cpf` na UI (feito em `UsersTable`, arquivo 21) é
   mitigação suficiente para RNF01 do lado do frontend, mas **o dado ainda
   trafega na resposta HTTP**, isso só fecha de verdade quando o backend
   criar o `AdminUserResponse`/`UserSummaryResponse` sugerido em
   `backend.md` § 4 item 1.
2. **Dupla digitação do novo e-mail é só mitigação de frontend**
   (`requestEmailChangeFormSchema`), não uma garantia real, o backend
   nunca valida a igualdade dos dois campos, só recebe `new_email`
   (`backend.md` § 4 item 2). Se alguém digitar o mesmo endereço errado nos
   dois campos, o `refine()` não pega nada. É a mitigação possível hoje, não
   a solução completa que `logic.md` § 8 originalmente desenhou.
3. **`useUserMutations().logout()` reaproveitado nas duas mutations de
   confirmação** (arquivo 14) chama `POST /identity/authentication/logout`
   além de limpar o token local, nessa altura o refresh token já foi
   invalidado no backend pela própria confirmação (`security_stamp`
   regenerado), então essa chamada extra tende a ser um no-op inofensivo
   (o `try/finally` de `AuthContext.logout()` limpa o estado local mesmo se
   o POST falhar). Alternativa mais "correta" seria expor um
   `clearSession()` só-local em `AuthContext`, sem bater na rede, não foi
   feita essa adição para não editar o contrato de `identity-auth` sem
   necessidade; se a call extra incomodar em code review, é uma mudança de
   3 linhas em `AuthContext.tsx`, não em nenhum dos arquivos desta feature.
4. **Página de confirmação assume que o clique vem de fora da aplicação**
   (link de e-mail, possivelmente sessão anônima nesta aba), por isso não
   usa `RequireAuth` e as duas rotas são públicas (`skipAuth: true` nos
   services, arquivo 6), espelhando exatamente `integration.md`. Se a mesma
   pessoa abrir o link numa aba onde já está autenticada, o bootstrap do
   `AuthProvider` ainda roda `refresh()` normalmente; não há necessidade de
   tratamento especial além do que já existe, mas vale um teste manual
   desse caso específico ao implementar.
5. **Ref-guard (`firedRef`) nos dois containers de confirmação (arquivos
   30/31)** existe porque o token é de uso único, sem essa guarda, o
   double-invoke de efeito do React Strict Mode em desenvolvimento
   consumiria o token na primeira chamada e devolveria 410 (falso "link
   expirado") na segunda, mascarando um sucesso real como erro. Não pular
   essa guarda achando que é só boilerplate.
6. **`RequireAdmin`/`Pagination` são reaproveitados de `@catalog`, não
   duplicados**, se `identity-user-management` crescer a ponto de
   justificar seu próprio "hub" administrativo, mover esses dois para uma
   camada verdadeiramente compartilhada (`src/components/`) é uma decisão
   futura, não deste recorte.
7. **`messageFor` é a única peça verdadeiramente nova em `src/lib/`** desta
   spec, todas as mutations de `identity-auth` hoje usam string estática
   no `onError`, ignorando o `detail` do backend. Não foi retrofitado
   `useAuthMutations.ts` para usar `messageFor` (fora de escopo desta
   feature); se isso virar um padrão desejado para o resto de `account`, é
   uma limpeza separada a decidir com o time, não algo que este plano força.

---

## 8. Bloqueios em aberto

Nenhum bloqueio de **produto**, `spec.md` § 9 fecha as três pendências do
reescopo de 2026-09-17. `logic.md` segue `status: draft` (pede revisão
conjunta de FE/BE), mas este plano já assume as regras ali como decisão
fechada, seguindo a mesma leitura que `identity-user-management/backend.md`
já fez.

Bloqueio técnico herdado do backend, não desta implementação: a listagem
administrativa não pode ser publicada exatamente como `logic.md` § 3 descreve
enquanto `backend.md` § 4 item 1 (CPF exposto / `created_at` ausente) não for
corrigido, ver § 7 item 1 acima. O restante da feature (perfil, troca de
e-mail, exclusão de conta) não tem bloqueio de contrato: o backend está
implementado e as rotas batem com o que este plano consome.
