---
status: done
spec: identity-auth
surface: frontend
created_at: 2026-09-03
updated_at: 2026-09-04
---

# Cadastro e autenticação do comprador — Frontend

> **Nota de reescopo (2026-09-11):** ver mesma nota em `backend.md`. Telas de
> cadastro saem pra `identity-user-management`; falta adicionar aqui a tela de
> alteração de e-mail (nova). Conteúdo abaixo ainda reflete o escopo antigo.

**Resumo:** feature `account` com registro, login, "esqueci a senha", redefinição
por link e sessão que se mantém entre visitas. O access token vive só em memória
(nunca `localStorage`); o refresh acontece via cookie `HttpOnly` — o `fetcher`
injeta o `Authorization`, tenta `refresh` uma vez em 401 e repete a chamada.
`AuthContext` guarda `user`/`isAuthenticated`; `RequireAuth` protege as
subárvores autenticadas.
**RF:** RF09 · **RN:** — (reforça RNF01) · **Feature frontend:** `account`
**Contrato:** `docs.ludens/specs/identity-auth/integration.md`
**Carregar antes:** skill `frontend-architecture` (todos os `references/`).

> **Revisão de 2026-09-17 — documento desatualizado, não implementar/copiar
> como está.** O `integration.md` foi corrigido para refletir o backend real
> (`api.ludens`, PR #8): rotas migraram de `/auth/...`/`/users/...` para
> `/identity/...`/`/identity/users/...`; o contrato inteiro é **snake_case**
> (`access_token`, `expires_in`, `current_password`, `new_password`), não
> camelCase; e não existe campo `role` — só o booleano `is_admin`. Este
> documento e o código já publicado em `web.ludens` ainda assumem a versão
> antiga (camelCase, `/auth`, `/users`, `role`). Débito técnico registrado em
> `docs.ludens/team/overview.md` — revisar `web.ludens` contra o
> `integration.md` atualizado antes de confiar nos trechos de código abaixo.

Stack: **Next.js (App Router) + TypeScript estrito**. Arquivos `.ts`/`.tsx`;
rotas em `src/app/**/page.tsx` (Server Components; `await searchParams`);
componentes/hooks com estado, handler ou hook de React levam `'use client'`; tipo
= `z.infer` do schema (nunca `interface` manual); `query-options.ts` obrigatório;
toda mutation invalida query + toast de sucesso + toast de erro;
`components/ui/` é apresentacional puro; barrel `index.ts` em toda subpasta.
Aliases: `@account/*`, `@web/*`, `@components/*`.

Este é o primeiro código de feature do repositório: entram junto dois arquivos de
infra compartilhada — `src/routes/endpoints.ts` e `src/lib/` (`fetcher`,
`api-error`) — mais a edição de `src/app/providers.tsx` e um `.env` novo.

---

## 1. Arquivos (ordem de dependência)

| # | Camada | Caminho | Novo/Editar |
| --- | --- | --- | --- |
| 1 | env | `web.ludens/.env.example` e `web.ludens/.env.local` | novo |
| 2 | endpoints | `src/routes/endpoints.ts` | novo |
| 3 | lib | `src/lib/fetcher.ts` | novo |
| 4 | lib | `src/lib/api-error.ts` | novo |
| 5 | lib | `src/lib/index.ts` | novo |
| 6 | schemas | `src/features/account/schemas/auth.schema.ts` | novo |
| 7 | schemas | `src/features/account/schemas/index.ts` | novo |
| 8 | server/types | `src/features/account/server/types/index.ts` | novo |
| 9 | server/services | `src/features/account/server/services/auth.service.ts` | novo |
| 10 | server/services | `src/features/account/server/services/index.ts` | novo |
| 11 | server | `src/features/account/server/index.ts` | novo |
| 12 | contexts | `src/features/account/contexts/AuthContext.tsx` | novo |
| 13 | contexts | `src/features/account/contexts/index.ts` | novo |
| 14 | queries | `src/features/account/hooks/queries/query-options.ts` | novo |
| 15 | queries | `src/features/account/hooks/queries/useAccountQueries.ts` | novo |
| 16 | queries | `src/features/account/hooks/queries/index.ts` | novo |
| 17 | mutations | `src/features/account/hooks/mutations/useAuthMutations.ts` | novo |
| 18 | mutations | `src/features/account/hooks/mutations/index.ts` | novo |
| 19 | forms | `src/features/account/hooks/forms/useRegisterForm.ts` | novo |
| 20 | forms | `src/features/account/hooks/forms/useLoginForm.ts` | novo |
| 21 | forms | `src/features/account/hooks/forms/useResetPasswordForm.ts` | novo |
| 22 | forms | `src/features/account/hooks/forms/index.ts` | novo |
| 23 | hooks | `src/features/account/hooks/index.ts` | novo |
| 24 | lib (feature) | `src/features/account/lib/cpf-mask.ts` | novo |
| 25 | lib (feature) | `src/features/account/lib/index.ts` | novo |
| 26 | components/ui | `src/features/account/components/ui/AuthCard.tsx` | novo |
| 27 | components/ui | `src/features/account/components/ui/index.ts` | novo |
| 28 | components | `src/features/account/components/RegisterForm.tsx` | novo |
| 29 | components | `src/features/account/components/LoginForm.tsx` | novo |
| 30 | components | `src/features/account/components/ForgotPasswordForm.tsx` | novo |
| 31 | components | `src/features/account/components/ResetPasswordForm.tsx` | novo |
| 32 | components | `src/features/account/components/RequireAuth.tsx` | novo |
| 33 | components | `src/features/account/components/index.ts` | novo |
| 34 | rota | `src/app/registro/page.tsx` | novo |
| 35 | rota | `src/app/login/page.tsx` | novo |
| 36 | rota | `src/app/recuperar-senha/page.tsx` | novo |
| 37 | rota | `src/app/redefinir-senha/page.tsx` | novo |
| 38 | providers | `src/app/providers.tsx` | editar |
| 39 | barrel | `src/features/account/index.ts` | novo |
| 40 | README | `src/features/account/README.md` | novo |

---

## 2. Código

### 1. `web.ludens/.env.example` e `web.ludens/.env.local` — novo

```bash
# web.ludens/.env.example (e copie para .env.local) — novo
# Base da API (api.ludens). O src/routes/endpoints.ts monta as URLs a partir daqui.
NEXT_PUBLIC_API_URL=http://localhost:8000
```

### 2. `src/routes/endpoints.ts` — novo

```ts
// src/routes/endpoints.ts — novo
const API_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:8000';
const withBase = (path = '') => `${API_URL}${path}`;

export const API_ENDPOINTS = {
  auth: {
    login: withBase('/auth/login'),
    refresh: withBase('/auth/refresh'),
    logout: withBase('/auth/logout'),
    passwordChange: withBase('/auth/password/change'),
    passwordForgot: withBase('/auth/password/forgot'),
    passwordReset: withBase('/auth/password/reset'),
  },
  users: {
    register: withBase('/users'),
    list: (page: number, size: number) => withBase(`/users?page=${page}&size=${size}`),
    byId: (id: string) => withBase(`/users/${id}`),
  },
} as const;
```

### 3. `src/lib/fetcher.ts` — novo

```ts
// src/lib/fetcher.ts — novo
// Wrapper primitivo sobre fetch: injeta Authorization a partir do token em
// memória, manda o cookie de refresh (credentials: 'include') e, num 401, tenta
// o refresh UMA vez e repete a chamada. Nenhuma feature fala com fetch direto.

type TokenAccessor = {
  get: () => string | null;
};

type RefreshHandler = () => Promise<string | null>;

let accessor: TokenAccessor = { get: () => null };
let refreshHandler: RefreshHandler | null = null;

/** Ligado pelo AuthContext no boot do provider. */
export function configureAuth(next: TokenAccessor, onRefresh: RefreshHandler): void {
  accessor = next;
  refreshHandler = onRefresh;
}

export class ApiError extends Error {
  readonly status: number;
  readonly payload: unknown;

  constructor(status: number, payload: unknown, message: string) {
    super(message);
    this.name = 'ApiError';
    this.status = status;
    this.payload = payload;
  }
}

function messageFromPayload(payload: unknown): string {
  if (payload && typeof payload === 'object' && 'detail' in payload) {
    const detail = (payload as { detail: unknown }).detail;
    if (Array.isArray(detail) && detail.length > 0) {
      const first = detail[0] as { message?: unknown };
      if (first && typeof first.message === 'string') return first.message;
    }
    if (typeof detail === 'string') return detail;
  }
  return 'Não foi possível completar a solicitação.';
}

async function request<T>(url: string, init: RequestInit, allowRetry = true): Promise<T> {
  const headers = new Headers(init.headers);
  headers.set('Accept', 'application/json');
  if (init.body !== undefined) headers.set('Content-Type', 'application/json');
  const token = accessor.get();
  if (token) headers.set('Authorization', `Bearer ${token}`);

  const response = await fetch(url, { ...init, headers, credentials: 'include' });

  if (response.status === 401 && allowRetry && refreshHandler) {
    const refreshed = await refreshHandler();
    if (refreshed) return request<T>(url, init, false);
  }

  const isJson = response.headers.get('content-type')?.includes('application/json') ?? false;
  const payload: unknown = isJson ? await response.json() : null;

  if (!response.ok) {
    throw new ApiError(response.status, payload, messageFromPayload(payload));
  }
  return payload as T;
}

export const fetcher = {
  get: <T>(url: string): Promise<T> => request<T>(url, { method: 'GET' }),
  post: <T>(url: string, body?: unknown): Promise<T> =>
    request<T>(url, {
      method: 'POST',
      body: body === undefined ? undefined : JSON.stringify(body),
    }),
};
```

### 4. `src/lib/api-error.ts` — novo

```ts
// src/lib/api-error.ts — novo
import { ApiError } from './fetcher';

/** Mensagem de erro em pt-BR para toast: usa a do backend quando existir,
 *  senão o fallback da mutation. Nunca expõe status/stack. */
export function messageFor(error: unknown, fallback: string): string {
  if (error instanceof ApiError && error.message) return error.message;
  return fallback;
}
```

### 5. `src/lib/index.ts` — novo

```ts
// src/lib/index.ts — novo
export { ApiError, configureAuth, fetcher } from './fetcher';
export { messageFor } from './api-error';
```

### 6. `src/features/account/schemas/auth.schema.ts` — novo

```ts
// src/features/account/schemas/auth.schema.ts — novo
import { z } from 'zod';

// Enum fechado do domínio (mesmos valores do backend).
export const roleSchema = z.enum(['BUYER', 'ADMIN']);

// CPF: validado por forma (11 dígitos). O dígito verificador é regra de negócio
// do backend (RF09). A máscara é só de exibição — o valor enviado é limpo.
const hasElevenDigits = (value: string) => value.replace(/\D/g, '').length === 11;

// --- Request (DTO) ---
export const registerRequestSchema = z.object({
  name: z.string().trim().min(1, 'Informe seu nome').max(120, 'No máximo 120 caracteres'),
  cpf: z.string().refine(hasElevenDigits, 'Informe os 11 dígitos do CPF'),
  email: z.string().trim().toLowerCase().email('E-mail inválido').max(254),
  password: z.string().min(8, 'A senha precisa de ao menos 8 caracteres').max(128),
});

export const loginRequestSchema = z.object({
  email: z.string().trim().toLowerCase().email('E-mail inválido'),
  password: z.string().min(1, 'Informe sua senha'),
});

export const changePasswordRequestSchema = z.object({
  currentPassword: z.string().min(1, 'Informe a senha atual'),
  newPassword: z.string().min(8, 'A nova senha precisa de ao menos 8 caracteres').max(128),
});

export const forgotPasswordRequestSchema = z.object({
  email: z.string().trim().toLowerCase().email('E-mail inválido'),
});

export const resetPasswordRequestSchema = z.object({
  token: z.string().min(1, 'Link de redefinição inválido ou incompleto'),
  password: z.string().min(8, 'A senha precisa de ao menos 8 caracteres').max(128),
});

// --- Response ---
export const tokenResponseSchema = z.object({
  accessToken: z.string().min(1),
  expiresIn: z.number().int().positive(),
});

export const userSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  email: z.string(),
  cpf: z.string(),
  role: roleSchema,
});

export const messageResponseSchema = z.object({
  message: z.string(),
});
```

### 7. `src/features/account/schemas/index.ts` — novo

```ts
// src/features/account/schemas/index.ts — novo
export {
  roleSchema,
  registerRequestSchema,
  loginRequestSchema,
  changePasswordRequestSchema,
  forgotPasswordRequestSchema,
  resetPasswordRequestSchema,
  tokenResponseSchema,
  userSchema,
  messageResponseSchema,
} from './auth.schema';
```

### 8. `src/features/account/server/types/index.ts` — novo

```ts
// src/features/account/server/types/index.ts — novo
import type { z } from 'zod';

import type {
  roleSchema,
  registerRequestSchema,
  loginRequestSchema,
  changePasswordRequestSchema,
  forgotPasswordRequestSchema,
  resetPasswordRequestSchema,
  tokenResponseSchema,
  userSchema,
  messageResponseSchema,
} from '@account/schemas';

export type Role = z.infer<typeof roleSchema>;

export type RegisterDTO = z.infer<typeof registerRequestSchema>;
export type LoginDTO = z.infer<typeof loginRequestSchema>;
export type ChangePasswordDTO = z.infer<typeof changePasswordRequestSchema>;
export type ForgotPasswordDTO = z.infer<typeof forgotPasswordRequestSchema>;
export type ResetPasswordDTO = z.infer<typeof resetPasswordRequestSchema>;

export type TokenResponse = z.infer<typeof tokenResponseSchema>;
export type User = z.infer<typeof userSchema>;
export type MessageResponse = z.infer<typeof messageResponseSchema>;
```

### 9. `src/features/account/server/services/auth.service.ts` — novo

```ts
// src/features/account/server/services/auth.service.ts — novo
import { fetcher } from '@web/lib';
import { API_ENDPOINTS } from '@web/routes/endpoints';

import {
  userSchema,
  messageResponseSchema,
  tokenResponseSchema,
} from '@account/schemas';

import type {
  ChangePasswordDTO,
  ForgotPasswordDTO,
  LoginDTO,
  RegisterDTO,
  ResetPasswordDTO,
} from '@account/server/types';

export async function register(payload: RegisterDTO) {
  const data = await fetcher.post(API_ENDPOINTS.users.register, payload);
  return tokenResponseSchema.parse(data);
}

export async function login(payload: LoginDTO) {
  const data = await fetcher.post(API_ENDPOINTS.auth.login, payload);
  return tokenResponseSchema.parse(data);
}

export async function refresh() {
  const data = await fetcher.post(API_ENDPOINTS.auth.refresh);
  return tokenResponseSchema.parse(data);
}

export async function logout(): Promise<void> {
  await fetcher.post(API_ENDPOINTS.auth.logout);
}

export async function fetchUserById(id: string) {
  const data = await fetcher.get(API_ENDPOINTS.users.byId(id));
  return userSchema.parse(data);
}

export async function changePassword(payload: ChangePasswordDTO): Promise<void> {
  await fetcher.post(API_ENDPOINTS.auth.passwordChange, payload);
}

export async function forgotPassword(payload: ForgotPasswordDTO) {
  const data = await fetcher.post(API_ENDPOINTS.auth.passwordForgot, payload);
  return messageResponseSchema.parse(data);
}

export async function resetPassword(payload: ResetPasswordDTO): Promise<void> {
  await fetcher.post(API_ENDPOINTS.auth.passwordReset, payload);
}
```

### 10. `src/features/account/server/services/index.ts` — novo

```ts
// src/features/account/server/services/index.ts — novo
export {
  register,
  login,
  refresh,
  logout,
  fetchMe,
  changePassword,
  forgotPassword,
  resetPassword,
} from './auth.service';
```

### 11. `src/features/account/server/index.ts` — novo

```ts
// src/features/account/server/index.ts — novo
export * from './services';
export type * from './types';
```

### 12. `src/features/account/contexts/AuthContext.tsx` — novo

```tsx
// src/features/account/contexts/AuthContext.tsx — novo
'use client';

import {
  createContext,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useRef,
  useState,
  type ReactNode,
} from 'react';

import { configureAuth } from '@web/lib';
import { API_ENDPOINTS } from '@web/routes/endpoints';

import { tokenResponseSchema } from '@account/schemas';

import type { User } from '@account/server/types';

type AuthContextValue = {
  accessToken: string | null;
  user: User | null;
  isAuthenticated: boolean;
  isBootstrapping: boolean;
  setSession: (token: string) => void;
  setUser: (user: User | null) => void;
  clearSession: () => void;
};

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [accessToken, setAccessToken] = useState<string | null>(null);
  const [user, setUser] = useState<User | null>(null);
  const [isBootstrapping, setIsBootstrapping] = useState(true);

  // Espelho síncrono do token para o fetcher ler sem re-render.
  const tokenRef = useRef<string | null>(null);
  tokenRef.current = accessToken;

  const clearSession = useCallback(() => {
    setAccessToken(null);
    setUser(null);
  }, []);

  const setSession = useCallback((token: string) => {
    setAccessToken(token);
  }, []);

  // Refresh silencioso via cookie HttpOnly. Não passa pelo fetcher (evita
  // recursão de 401). Devolve o novo token ou null.
  const runRefresh = useCallback(async (): Promise<string | null> => {
    try {
      const response = await fetch(API_ENDPOINTS.auth.refresh, {
        method: 'POST',
        credentials: 'include',
      });
      if (!response.ok) {
        clearSession();
        return null;
      }
      const parsed = tokenResponseSchema.parse(await response.json());
      setAccessToken(parsed.accessToken);
      return parsed.accessToken;
    } catch {
      clearSession();
      return null;
    }
  }, [clearSession]);

  // Liga o fetcher ao token em memória + tenta restaurar a sessão no primeiro load.
  useEffect(() => {
    configureAuth({ get: () => tokenRef.current }, runRefresh);
    let active = true;
    void (async () => {
      await runRefresh().catch(() => null);
      if (active) setIsBootstrapping(false);
    })();
    return () => {
      active = false;
    };
  }, [runRefresh]);

  const value = useMemo<AuthContextValue>(
    () => ({
      accessToken,
      user,
      isAuthenticated: Boolean(accessToken),
      isBootstrapping,
      setSession,
      setUser,
      clearSession,
    }),
    [accessToken, user, isBootstrapping, setSession, clearSession],
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth precisa estar dentro de <AuthProvider>.');
  return ctx;
}
```

### 13. `src/features/account/contexts/index.ts` — novo

```ts
// src/features/account/contexts/index.ts — novo
export { AuthProvider, useAuth } from './AuthContext';
```

### 14. `src/features/account/hooks/queries/query-options.ts` — novo

```ts
// src/features/account/hooks/queries/query-options.ts — novo
import { queryOptions } from '@tanstack/react-query';

import { fetchMe } from '@account/server/services';

export const accountQueryKeys = {
  all: ['account'] as const,
  me: () => [...accountQueryKeys.all, 'me'] as const,
};

export const accountQueryOptions = {
  me: (enabled: boolean) =>
    queryOptions({
      queryKey: accountQueryKeys.me(),
      queryFn: fetchMe,
      enabled,
      staleTime: 5 * 60 * 1000,
      retry: false,
    }),
};
```

### 15. `src/features/account/hooks/queries/useAccountQueries.ts` — novo

```ts
// src/features/account/hooks/queries/useAccountQueries.ts — novo
'use client';

import { useQuery } from '@tanstack/react-query';

import { useAuth } from '@account/contexts';

import { accountQueryOptions } from './query-options';

export function useAccountQueries() {
  const { isAuthenticated } = useAuth();

  return {
    useMe: () => useQuery(accountQueryOptions.me(isAuthenticated)),
  };
}
```

### 16. `src/features/account/hooks/queries/index.ts` — novo

```ts
// src/features/account/hooks/queries/index.ts — novo
export { accountQueryKeys, accountQueryOptions } from './query-options';
export { useAccountQueries } from './useAccountQueries';
```

### 17. `src/features/account/hooks/mutations/useAuthMutations.ts` — novo

```ts
// src/features/account/hooks/mutations/useAuthMutations.ts — novo
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';

import { messageFor } from '@web/lib';

import { useAuth } from '@account/contexts';
import { accountQueryKeys } from '@account/hooks/queries';
import {
  changePassword,
  forgotPassword,
  login,
  logout,
  register,
  resetPassword,
} from '@account/server/services';

import type {
  ChangePasswordDTO,
  ForgotPasswordDTO,
  LoginDTO,
  RegisterDTO,
  ResetPasswordDTO,
} from '@account/server/types';

export function useAuthMutations() {
  const queryClient = useQueryClient();
  const router = useRouter();
  const { setSession, clearSession } = useAuth();

  const dropAccountCache = () => {
    clearSession();
    queryClient.removeQueries({ queryKey: accountQueryKeys.all });
  };

  const registerMutation = useMutation({
    mutationFn: (data: RegisterDTO) => register(data),
    onSuccess: async (token) => {
      setSession(token.accessToken);
      await queryClient.invalidateQueries({ queryKey: accountQueryKeys.me() });
      toast.success('Conta criada. Bem-vindo à Ludens!');
    },
    onError: (error: unknown) => {
      toast.error(messageFor(error, 'Não foi possível concluir o cadastro.'));
    },
  });

  const loginMutation = useMutation({
    mutationFn: (data: LoginDTO) => login(data),
    onSuccess: async (token) => {
      setSession(token.accessToken);
      await queryClient.invalidateQueries({ queryKey: accountQueryKeys.me() });
      toast.success('Login efetuado.');
    },
    onError: (error: unknown) => {
      toast.error(messageFor(error, 'E-mail ou senha inválidos.'));
    },
  });

  const logoutMutation = useMutation({
    mutationFn: () => logout(),
    onSuccess: () => {
      dropAccountCache();
      toast.success('Você saiu da sua conta.');
      router.replace('/login');
    },
    onError: () => {
      // Mesmo se a chamada falhar, a sessão local vai embora.
      dropAccountCache();
      toast.error('Sessão encerrada localmente.');
      router.replace('/login');
    },
  });

  const changePasswordMutation = useMutation({
    mutationFn: (data: ChangePasswordDTO) => changePassword(data),
    onSuccess: () => {
      dropAccountCache();
      toast.success('Senha alterada. Entre novamente com a nova senha.');
      router.replace('/login');
    },
    onError: (error: unknown) => {
      toast.error(messageFor(error, 'A senha atual não confere.'));
    },
  });

  const forgotPasswordMutation = useMutation({
    mutationFn: (data: ForgotPasswordDTO) => forgotPassword(data),
    onSuccess: (result) => {
      toast.success(result.message);
    },
    onError: () => {
      // Resposta neutra também no erro — não revela se o e-mail existe.
      toast.error('Não foi possível concluir agora. Tente novamente em instantes.');
    },
  });

  const resetPasswordMutation = useMutation({
    mutationFn: (data: ResetPasswordDTO) => resetPassword(data),
    onSuccess: () => {
      toast.success('Senha redefinida. Faça login com a nova senha.');
      router.replace('/login');
    },
    onError: (error: unknown) => {
      toast.error(messageFor(error, 'Este link não é mais válido, solicite um novo.'));
    },
  });

  return {
    register: registerMutation,
    login: loginMutation,
    logout: logoutMutation,
    changePassword: changePasswordMutation,
    forgotPassword: forgotPasswordMutation,
    resetPassword: resetPasswordMutation,
  };
}
```

### 18. `src/features/account/hooks/mutations/index.ts` — novo

```ts
// src/features/account/hooks/mutations/index.ts — novo
export { useAuthMutations } from './useAuthMutations';
```

### 19. `src/features/account/hooks/forms/useRegisterForm.ts` — novo

```ts
// src/features/account/hooks/forms/useRegisterForm.ts — novo
'use client';

import { zodResolver } from '@hookform/resolvers/zod';
import { useRouter } from 'next/navigation';
import { useForm } from 'react-hook-form';

import { registerRequestSchema } from '@account/schemas';
import { useAuthMutations } from '@account/hooks/mutations';

import type { RegisterDTO } from '@account/server/types';

export function useRegisterForm({ next = '/' }: { next?: string } = {}) {
  const router = useRouter();
  const { register } = useAuthMutations();

  const form = useForm<RegisterDTO>({
    resolver: zodResolver(registerRequestSchema),
    mode: 'onSubmit',
    defaultValues: { name: '', cpf: '', email: '', password: '' },
  });

  const handleSubmit = form.handleSubmit(async (values) => {
    // Máscara é só de exibição: envia CPF só com dígitos.
    await register.mutateAsync({ ...values, cpf: values.cpf.replace(/\D/g, '') });
    router.replace(next);
  });

  return { form, handleSubmit, isPending: register.isPending };
}
```

### 20. `src/features/account/hooks/forms/useLoginForm.ts` — novo

```ts
// src/features/account/hooks/forms/useLoginForm.ts — novo
'use client';

import { zodResolver } from '@hookform/resolvers/zod';
import { useRouter } from 'next/navigation';
import { useForm } from 'react-hook-form';

import { loginRequestSchema } from '@account/schemas';
import { useAuthMutations } from '@account/hooks/mutations';

import type { LoginDTO } from '@account/server/types';

export function useLoginForm({ next }: { next: string }) {
  const router = useRouter();
  const { login } = useAuthMutations();

  const form = useForm<LoginDTO>({
    resolver: zodResolver(loginRequestSchema),
    mode: 'onSubmit',
    defaultValues: { email: '', password: '' },
  });

  const handleSubmit = form.handleSubmit(async (values) => {
    await login.mutateAsync(values);
    router.replace(next);
  });

  return { form, handleSubmit, isPending: login.isPending };
}
```

### 21. `src/features/account/hooks/forms/useResetPasswordForm.ts` — novo

```ts
// src/features/account/hooks/forms/useResetPasswordForm.ts — novo
'use client';

import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';

import { resetPasswordRequestSchema } from '@account/schemas';
import { useAuthMutations } from '@account/hooks/mutations';

import type { ResetPasswordDTO } from '@account/server/types';

export function useResetPasswordForm({ token }: { token: string }) {
  const { resetPassword } = useAuthMutations();

  const form = useForm<ResetPasswordDTO>({
    resolver: zodResolver(resetPasswordRequestSchema),
    mode: 'onSubmit',
    defaultValues: { token, password: '' },
  });

  const handleSubmit = form.handleSubmit(async (values) => {
    await resetPassword.mutateAsync(values);
  });

  return { form, handleSubmit, isPending: resetPassword.isPending };
}
```

### 22. `src/features/account/hooks/forms/index.ts` — novo

```ts
// src/features/account/hooks/forms/index.ts — novo
export { useRegisterForm } from './useRegisterForm';
export { useLoginForm } from './useLoginForm';
export { useResetPasswordForm } from './useResetPasswordForm';
```

### 23. `src/features/account/hooks/index.ts` — novo

```ts
// src/features/account/hooks/index.ts — novo
export * from './queries';
export * from './mutations';
export * from './forms';
```

### 24. `src/features/account/lib/cpf-mask.ts` — novo

```ts
// src/features/account/lib/cpf-mask.ts — novo
/** Máscara de exibição do CPF (000.000.000-00). Nunca usada no valor enviado. */
export function formatCpf(value: string): string {
  const digits = value.replace(/\D/g, '').slice(0, 11);
  return digits
    .replace(/^(\d{3})(\d)/, '$1.$2')
    .replace(/^(\d{3})\.(\d{3})(\d)/, '$1.$2.$3')
    .replace(/^(\d{3})\.(\d{3})\.(\d{3})(\d)/, '$1.$2.$3-$4');
}
```

### 25. `src/features/account/lib/jwt.ts` — novo

```ts
// src/features/account/lib/jwt.ts — novo
/** Lê só o claim `sub` (id) do payload do access token — sem verificar
 *  assinatura, isso é sempre responsabilidade do backend. Usado pelo
 *  AuthContext pra saber "quem sou eu" sem round-trip (não existe mais
 *  GET /auth/me; GET /users/{id} exige o id). */
export function decodeAccessTokenSub(token: string): string | null {
  try {
    const [, payload] = token.split('.');
    if (!payload) return null;
    const json = atob(payload.replace(/-/g, '+').replace(/_/g, '/'));
    const claims = JSON.parse(json) as { sub?: string };
    return claims.sub ?? null;
  } catch {
    return null;
  }
}
```

### 26. `src/features/account/lib/index.ts` — novo

```ts
// src/features/account/lib/index.ts — novo
export { formatCpf } from './cpf-mask';
export { decodeAccessTokenSub } from './jwt';
```

### 27. `src/features/account/components/ui/AuthCard.tsx` — novo

```tsx
// src/features/account/components/ui/AuthCard.tsx — novo
'use client';

import type { ReactNode } from 'react';

export interface AuthCardProps {
  title: string;
  subtitle?: string;
  children: ReactNode;
  footer?: ReactNode;
}

/** Layout puro de tela de autenticação. Sem hook de dados, sem service. */
export function AuthCard({ title, subtitle, children, footer }: AuthCardProps) {
  return (
    <main className="mx-auto flex min-h-screen w-full max-w-md flex-col justify-center gap-6 px-4 py-10">
      <header className="space-y-1">
        <h1 className="text-2xl font-semibold">{title}</h1>
        {subtitle ? <p className="text-sm text-gray-600">{subtitle}</p> : null}
      </header>
      {children}
      {footer ? <div className="text-sm text-gray-600">{footer}</div> : null}
    </main>
  );
}
```

### 28. `src/features/account/components/ui/index.ts` — novo

```ts
// src/features/account/components/ui/index.ts — novo
export { AuthCard } from './AuthCard';
export type { AuthCardProps } from './AuthCard';
```

### 29. `src/features/account/components/RegisterForm.tsx` — novo

```tsx
// src/features/account/components/RegisterForm.tsx — novo
'use client';

import Link from 'next/link';

import { AuthCard } from '@account/components/ui';
import { useRegisterForm } from '@account/hooks/forms';
import { formatCpf } from '@account/lib';

export function RegisterForm() {
  const { form, handleSubmit, isPending } = useRegisterForm();
  const { register, formState } = form;
  const { errors } = formState;

  return (
    <AuthCard
      title="Criar conta"
      subtitle="CPF, e-mail e senha para comprar ingressos e acompanhar seu histórico."
      footer={
        <>
          Já tem conta?{' '}
          <Link className="underline" href="/login">
            Entrar
          </Link>
        </>
      }
    >
      <form className="space-y-4" onSubmit={handleSubmit} noValidate>
        <div className="space-y-1">
          <label htmlFor="name" className="block text-sm font-medium">
            Nome
          </label>
          <input
            id="name"
            className="min-h-11 w-full rounded border px-3 py-2"
            autoComplete="name"
            {...register('name')}
          />
          {errors.name ? (
            <p className="text-sm text-red-600" role="alert">
              {errors.name.message}
            </p>
          ) : null}
        </div>

        <div className="space-y-1">
          <label htmlFor="cpf" className="block text-sm font-medium">
            CPF
          </label>
          <input
            id="cpf"
            inputMode="numeric"
            className="min-h-11 w-full rounded border px-3 py-2"
            {...register('cpf', {
              onChange: (event) => {
                event.target.value = formatCpf(event.target.value);
              },
            })}
          />
          {errors.cpf ? (
            <p className="text-sm text-red-600" role="alert">
              {errors.cpf.message}
            </p>
          ) : null}
        </div>

        <div className="space-y-1">
          <label htmlFor="email" className="block text-sm font-medium">
            E-mail
          </label>
          <input
            id="email"
            type="email"
            className="min-h-11 w-full rounded border px-3 py-2"
            autoComplete="email"
            {...register('email')}
          />
          {errors.email ? (
            <p className="text-sm text-red-600" role="alert">
              {errors.email.message}
            </p>
          ) : null}
        </div>

        <div className="space-y-1">
          <label htmlFor="password" className="block text-sm font-medium">
            Senha
          </label>
          <input
            id="password"
            type="password"
            className="min-h-11 w-full rounded border px-3 py-2"
            autoComplete="new-password"
            {...register('password')}
          />
          {errors.password ? (
            <p className="text-sm text-red-600" role="alert">
              {errors.password.message}
            </p>
          ) : null}
        </div>

        <button
          type="submit"
          disabled={isPending}
          className="min-h-11 w-full rounded bg-black px-4 py-2 text-white disabled:opacity-60"
        >
          {isPending ? 'Criando...' : 'Criar conta'}
        </button>
      </form>
    </AuthCard>
  );
}
```

### 30. `src/features/account/components/LoginForm.tsx` — novo

```tsx
// src/features/account/components/LoginForm.tsx — novo
'use client';

import Link from 'next/link';
import { useSearchParams } from 'next/navigation';

import { AuthCard } from '@account/components/ui';
import { useLoginForm } from '@account/hooks/forms';

export function LoginForm() {
  const params = useSearchParams();
  const next = params.get('next') ?? '/';
  const { form, handleSubmit, isPending } = useLoginForm({ next });
  const { register, formState } = form;
  const { errors } = formState;

  return (
    <AuthCard
      title="Entrar"
      subtitle="Acesse sua conta para reservar ingressos e ver suas compras."
      footer={
        <div className="flex flex-col gap-1">
          <span>
            Não tem conta?{' '}
            <Link className="underline" href="/registro">
              Criar conta
            </Link>
          </span>
          <Link className="underline" href="/recuperar-senha">
            Esqueci minha senha
          </Link>
        </div>
      }
    >
      <form className="space-y-4" onSubmit={handleSubmit} noValidate>
        <div className="space-y-1">
          <label htmlFor="email" className="block text-sm font-medium">
            E-mail
          </label>
          <input
            id="email"
            type="email"
            className="min-h-11 w-full rounded border px-3 py-2"
            autoComplete="email"
            {...register('email')}
          />
          {errors.email ? (
            <p className="text-sm text-red-600" role="alert">
              {errors.email.message}
            </p>
          ) : null}
        </div>

        <div className="space-y-1">
          <label htmlFor="password" className="block text-sm font-medium">
            Senha
          </label>
          <input
            id="password"
            type="password"
            className="min-h-11 w-full rounded border px-3 py-2"
            autoComplete="current-password"
            {...register('password')}
          />
          {errors.password ? (
            <p className="text-sm text-red-600" role="alert">
              {errors.password.message}
            </p>
          ) : null}
        </div>

        <button
          type="submit"
          disabled={isPending}
          className="min-h-11 w-full rounded bg-black px-4 py-2 text-white disabled:opacity-60"
        >
          {isPending ? 'Entrando...' : 'Entrar'}
        </button>
      </form>
    </AuthCard>
  );
}
```

### 31. `src/features/account/components/ForgotPasswordForm.tsx` — novo

```tsx
// src/features/account/components/ForgotPasswordForm.tsx — novo
'use client';

import Link from 'next/link';
import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';

import { AuthCard } from '@account/components/ui';
import { forgotPasswordRequestSchema } from '@account/schemas';
import { useAuthMutations } from '@account/hooks/mutations';

import type { ForgotPasswordDTO } from '@account/server/types';

export function ForgotPasswordForm() {
  const { forgotPassword } = useAuthMutations();
  const form = useForm<ForgotPasswordDTO>({
    resolver: zodResolver(forgotPasswordRequestSchema),
    mode: 'onSubmit',
    defaultValues: { email: '' },
  });
  const { register, formState } = form;

  const handleSubmit = form.handleSubmit(async (values) => {
    await forgotPassword.mutateAsync(values);
  });

  return (
    <AuthCard
      title="Recuperar senha"
      subtitle="Informe seu e-mail e enviaremos um link para redefinir a senha."
      footer={
        <Link className="underline" href="/login">
          Voltar para o login
        </Link>
      }
    >
      {forgotPassword.isSuccess ? (
        <p className="rounded border border-gray-200 bg-gray-50 p-4 text-sm text-gray-700">
          Se houver uma conta com esse e-mail, enviamos um link de redefinição.
          O link expira em 1 hora.
        </p>
      ) : (
        <form className="space-y-4" onSubmit={handleSubmit} noValidate>
          <div className="space-y-1">
            <label htmlFor="email" className="block text-sm font-medium">
              E-mail
            </label>
            <input
              id="email"
              type="email"
              className="min-h-11 w-full rounded border px-3 py-2"
              autoComplete="email"
              {...register('email')}
            />
            {formState.errors.email ? (
              <p className="text-sm text-red-600" role="alert">
                {formState.errors.email.message}
              </p>
            ) : null}
          </div>

          <button
            type="submit"
            disabled={forgotPassword.isPending}
            className="min-h-11 w-full rounded bg-black px-4 py-2 text-white disabled:opacity-60"
          >
            {forgotPassword.isPending ? 'Enviando...' : 'Enviar link'}
          </button>
        </form>
      )}
    </AuthCard>
  );
}
```

### 32. `src/features/account/components/ResetPasswordForm.tsx` — novo

```tsx
// src/features/account/components/ResetPasswordForm.tsx — novo
'use client';

import Link from 'next/link';

import { AuthCard } from '@account/components/ui';
import { useResetPasswordForm } from '@account/hooks/forms';

export function ResetPasswordForm({ token }: { token: string }) {
  const { form, handleSubmit, isPending } = useResetPasswordForm({ token });
  const { register, formState } = form;

  if (!token) {
    return (
      <AuthCard
        title="Link inválido"
        subtitle="Este link de redefinição está incompleto ou expirou."
        footer={
          <Link className="underline" href="/recuperar-senha">
            Solicitar um novo link
          </Link>
        }
      >
        <p className="text-sm text-gray-600">
          Abra o link mais recente que você recebeu por e-mail ou solicite outro.
        </p>
      </AuthCard>
    );
  }

  return (
    <AuthCard
      title="Definir nova senha"
      subtitle="Escolha uma senha de ao menos 8 caracteres."
      footer={
        <Link className="underline" href="/login">
          Voltar para o login
        </Link>
      }
    >
      <form className="space-y-4" onSubmit={handleSubmit} noValidate>
        <input type="hidden" {...register('token')} />
        <div className="space-y-1">
          <label htmlFor="password" className="block text-sm font-medium">
            Nova senha
          </label>
          <input
            id="password"
            type="password"
            className="min-h-11 w-full rounded border px-3 py-2"
            autoComplete="new-password"
            {...register('password')}
          />
          {formState.errors.password ? (
            <p className="text-sm text-red-600" role="alert">
              {formState.errors.password.message}
            </p>
          ) : null}
        </div>

        <button
          type="submit"
          disabled={isPending}
          className="min-h-11 w-full rounded bg-black px-4 py-2 text-white disabled:opacity-60"
        >
          {isPending ? 'Salvando...' : 'Redefinir senha'}
        </button>
      </form>
    </AuthCard>
  );
}
```

### 33. `src/features/account/components/RequireAuth.tsx` — novo

```tsx
// src/features/account/components/RequireAuth.tsx — novo
'use client';

import { useEffect, type ReactNode } from 'react';
import { usePathname, useRouter } from 'next/navigation';

import { useAuth } from '@account/contexts';
import { useAccountQueries } from '@account/hooks/queries';

/** Protege uma subárvore autenticada. Sem sessão, redireciona para
 *  /login?next=<destino>. Enquanto a sessão é restaurada, segura a UI. */
export function RequireAuth({ children }: { children: ReactNode }) {
  const router = useRouter();
  const pathname = usePathname();
  const { isAuthenticated, isBootstrapping } = useAuth();
  const { useMe } = useAccountQueries();
  const { data: user, isLoading, isError } = useMe();

  useEffect(() => {
    if (!isBootstrapping && !isAuthenticated) {
      router.replace(`/login?next=${encodeURIComponent(pathname)}`);
    }
  }, [isBootstrapping, isAuthenticated, pathname, router]);

  if (isBootstrapping || (isAuthenticated && isLoading)) {
    return <p className="p-6 text-sm text-gray-600">Carregando sua conta...</p>;
  }

  if (!isAuthenticated || isError || !user) {
    return (
      <p className="p-6 text-sm text-gray-600">
        Sua sessão expirou. Redirecionando para o login...
      </p>
    );
  }

  return <>{children}</>;
}
```

### 34. `src/features/account/components/index.ts` — novo

```ts
// src/features/account/components/index.ts — novo
export * from './ui';
export { RegisterForm } from './RegisterForm';
export { LoginForm } from './LoginForm';
export { ForgotPasswordForm } from './ForgotPasswordForm';
export { ResetPasswordForm } from './ResetPasswordForm';
export { RequireAuth } from './RequireAuth';
```

### 35. `src/app/registro/page.tsx` — novo

```tsx
// src/app/registro/page.tsx — novo
import { RegisterForm } from '@account/components';

export default function RegistroPage() {
  return <RegisterForm />;
}
```

### 36. `src/app/login/page.tsx` — novo

```tsx
// src/app/login/page.tsx — novo
import { Suspense } from 'react';

import { LoginForm } from '@account/components';

// LoginForm usa useSearchParams (lê ?next=) — precisa de fronteira de Suspense.
export default function LoginPage() {
  return (
    <Suspense fallback={null}>
      <LoginForm />
    </Suspense>
  );
}
```

### 37. `src/app/recuperar-senha/page.tsx` — novo

```tsx
// src/app/recuperar-senha/page.tsx — novo
import { ForgotPasswordForm } from '@account/components';

export default function RecuperarSenhaPage() {
  return <ForgotPasswordForm />;
}
```

### 38. `src/app/redefinir-senha/page.tsx` — novo

```tsx
// src/app/redefinir-senha/page.tsx — novo
import { ResetPasswordForm } from '@account/components';

export default async function RedefinirSenhaPage({
  searchParams,
}: {
  searchParams: Promise<{ token?: string }>;
}) {
  const { token } = await searchParams;
  return <ResetPasswordForm token={token ?? ''} />;
}
```

### 39. `src/app/providers.tsx` — editar

```tsx
// src/app/providers.tsx — editar (arquivo inteiro)
'use client';

import { useState, type ReactNode } from 'react';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Toaster } from 'sonner';

import { AuthProvider } from '@account/contexts';

export function Providers({ children }: { children: ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());

  return (
    <QueryClientProvider client={queryClient}>
      <AuthProvider>{children}</AuthProvider>
      <Toaster position="top-right" />
    </QueryClientProvider>
  );
}
```

### 40. `src/features/account/index.ts` — novo

```ts
// src/features/account/index.ts — novo
export { AuthProvider, useAuth } from './contexts';
export { useAccountQueries, useAuthMutations, accountQueryKeys } from './hooks';
export {
  RequireAuth,
  LoginForm,
  RegisterForm,
  ForgotPasswordForm,
  ResetPasswordForm,
  AuthCard,
} from './components';
export type { User, Role, TokenResponse } from './server/types';
```

### 41. `src/features/account/README.md` — novo

````markdown
# Feature: account

Registro, login, recuperação de senha e sessão do comprador (RF09). Consome o
módulo `identity` do `api.ludens` — contrato em
`docs.ludens/specs/identity-auth/integration.md`.

## Fluxo de dados

```
src/routes/endpoints.ts (auth.*)
  → schemas/auth.schema.ts (Zod: request DTO + response)
  → server/types (z.infer)
  → server/services/auth.service.ts (fetcher + parse)
  → contexts/AuthContext (token em memória + refresh silencioso)
  → hooks/queries (me) · hooks/mutations (register/login/logout/...) · hooks/forms
  → components/ (RegisterForm, LoginForm, ForgotPasswordForm, ResetPasswordForm, RequireAuth)
  → src/app/{registro,login,recuperar-senha,redefinir-senha}/page.tsx
```

## Decisões

- **Access token só em memória.** Nunca `localStorage`/`sessionStorage` (RNF01,
  `frontend-architecture` reference/12). Ao recarregar a página, o
  `AuthProvider` chama `POST /auth/refresh` uma vez (cookie `HttpOnly`) para
  restaurar a sessão; `isBootstrapping` segura a UI protegida até isso resolver.
- **Refresh transparente.** O `src/lib/fetcher.ts` injeta `Authorization`, manda
  o cookie (`credentials: 'include'`) e, em 401, tenta `refresh` uma vez e repete
  a chamada. Se o refresh falhar, limpa a sessão e o `RequireAuth` manda para
  `/login`.
- **Logout / troca / redefinição de senha derrubam todos os dispositivos.** O
  backend regenera o `security_stamp`; o frontend trata qualquer 401 seguinte
  como anônimo.
- **Máscara de CPF só na exibição.** O valor enviado é limpo (`replace(/\D/g,
  '')`) no `onSubmit` do `useRegisterForm`.
- **"Esqueci a senha" é sempre neutro.** Sucesso e erro mostram a mesma
  mensagem; a tela nunca revela se o e-mail existe.
````

---

## 3. Contrato consumido

`docs.ludens/specs/identity-auth/integration.md` (contrato-alvo enquanto o backend
não implementou). Shapes consumidos, todos em camelCase:

| Rota | Request | Response |
| --- | --- | --- |
| `POST /users` | `{ name, cpf, email, password }` | `201 { accessToken, expiresIn }` + `Set-Cookie` |
| `POST /auth/login` | `{ email, password }` | `200 { accessToken, expiresIn }` + `Set-Cookie` |
| `POST /auth/refresh` | — (cookie) | `200 { accessToken, expiresIn }` + `Set-Cookie` |
| `POST /auth/logout` | — | `204` |
| `POST /auth/password/change` | `{ currentPassword, newPassword }` | `204` |
| `POST /auth/password/forgot` | `{ email }` | `202 { message }` |
| `POST /auth/password/reset` | `{ token, password }` | `204` |
| `GET /users/{id}` | — | `200 { id, name, email, cpf, role }` |

Envelope de erro 4xx: `{ "detail": [ { "field": string, "message": string } ] }`
— o `fetcher` extrai `detail[0].message` para o toast. Se ao integrar o shape
divergir, o ajuste é um transform em `auth.service.ts` + registro da divergência
no `integration.md` — nunca editar o `api.ludens`.

---

## 4. Estados assíncronos e mensagens

| Estado | Onde | Mensagem (linguagem de negócio) |
| --- | --- | --- |
| loading (submit) | cada `*Form.tsx` | botão "Criando..." / "Entrando..." / "Salvando...", desabilitado |
| loading (sessão) | `RequireAuth` | "Carregando sua conta..." |
| error — cadastro | toast (mutation) | backend: "Este CPF já possui cadastro." / "Este e-mail já está em uso." / "CPF inválido." — fallback "Não foi possível concluir o cadastro." |
| error — login | toast (mutation) | "E-mail ou senha inválidos." |
| error — troca de senha | toast (mutation) | "A senha atual não confere." |
| error — redefinição | toast (mutation) | "Este link não é mais válido, solicite um novo." |
| error — token ausente | `ResetPasswordForm` (`!token`) | tela "Link inválido" + CTA "Solicitar um novo link" |
| 401 em rota protegida | `fetcher` → `refresh` 1x → `RequireAuth` | "Sua sessão expirou. Redirecionando para o login..." |
| sucesso "esqueci a senha" | painel + toast | "Se houver uma conta com esse e-mail, enviamos um link de redefinição. O link expira em 1 hora." |

---

## 5. Passo a passo TBD (Frontend)

```bash
git checkout master && git pull && git checkout -b feat/09-account-auth

# commit 1 — infra + contrato
git add web.ludens/.env.example web.ludens/.env.local src/routes/endpoints.ts \
        src/lib src/features/account/schemas src/features/account/server
git commit -m "feat(account): endpoints, fetcher com refresh, schemas e services de auth"

# commit 2 — contexto + hooks
git add src/features/account/contexts src/features/account/hooks \
        src/features/account/lib
git commit -m "feat(account): AuthContext, queries, mutations e forms de auth"

# commit 3 — UI + rotas
git add src/features/account/components src/app/registro src/app/login \
        src/app/recuperar-senha src/app/redefinir-senha src/app/providers.tsx
git commit -m "feat(account): telas de login, registro, recuperação e RequireAuth"

# commit 4 — barrels + README
git add src/features/account/index.ts src/features/account/README.md
git commit -m "chore(account): barrels index.ts e README da feature"
```

Depois: `npm run lint && npm run build` verdes → `/team-ludens:tbd-pr`.

---

## 6. Ordem entre as superfícies

Frontend começa contra o contrato-alvo do `integration.md` antes do backend — o
`fetcher`, o `AuthContext` e os schemas não dependem do backend pronto. A
integração real (cookies `Secure`, CORS com `allow_credentials`, shapes finais) é
depois do merge do backend.

---

## 7. Bloqueios em aberto

Nenhum. Decisões de produto fechadas no `spec.md` (§8) e no `logic.md` (§5).

---

## 8. Ajustes feitos no `integration.md`

Mesmos ajustes registrados no `backend.md` §8 (mantido `status: alvo`):

- Envelope de erro 4xx `{ "detail": [ { "field", "message" } ] }`.
- `POST /auth/password/change` → corpo `{ currentPassword, newPassword }`.
- `expiresIn` em segundos; `role` = `"BUYER"` | `"ADMIN"`.
- Sem prefixo/versionamento de rota no N1; base = `NEXT_PUBLIC_API_URL`.
- `POST /auth/logout` também apaga o cookie `refresh_token` (`Path=/auth`).
