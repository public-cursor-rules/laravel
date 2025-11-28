---
globs: "resources/js/**/*.ts,resources/js/**/*.tsx,resources/js/**/*.jsx"
alwaysApply: false
---

# API y Autenticación - Cliente HTTP + Hooks

## Configuración de Query Keys

```typescript
// lib/query-keys.ts
export enum QueryKey {
  USERS = 'users',
  SALES = 'sales',
  SALE_PAYMENTS = 'sale-payments',
  PORTAL_PAYMENTS = 'portal-payments',
  ENTITY = 'entity',
}

export const queryKeys = {
  users: {
    all: [QueryKey.USERS] as const,
    list: (params: any) => [QueryKey.USERS, 'list', params] as const,
    detail: (id: string) => [QueryKey.USERS, 'detail', id] as const,
  },
  sales: {
    all: [QueryKey.SALES] as const,
    list: (params: any) => [QueryKey.SALES, 'list', params] as const,
  },
  salePayments: {
    list: (saleId: string) => [QueryKey.SALE_PAYMENTS, saleId] as const,
  },
  portalPayments: {
    all: [QueryKey.PORTAL_PAYMENTS] as const,
  },
  entity: {
    all: [QueryKey.ENTITY] as const,
    list: (params: any) => [QueryKey.ENTITY, 'list', params] as const,
  },
} as const;
```

## Cliente API Base

```typescript
// lib/api-client.ts
import { router } from '@inertiajs/react';

function getCookie(name: string): string | null {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  return parts.length === 2 ? parts.pop()!.split(';').shift() || null : null;
}

export async function request<T = unknown>(path: string, options: RequestOptions = {}): Promise<T> {
  const method = (options.method || 'GET').toUpperCase();
  const isMutation = method !== 'GET';

  if (isMutation) await fetch('/sanctum/csrf-cookie', { method: 'GET', credentials: 'include' });

  const headers: HeadersInit = { Accept: 'application/json', ...(options.headers || {}) };
  if (isMutation) {
    const xsrfCookie = getCookie('XSRF-TOKEN');
    if (xsrfCookie) (headers as Record<string, string>)['X-XSRF-TOKEN'] = decodeURIComponent(xsrfCookie);
  }

  let body = options.body;
  if (isMutation && body && typeof body === 'object' && !(body instanceof FormData)) {
    body = JSON.stringify(body);
    (headers as Record<string, string>)['Content-Type'] = 'application/json';
  }

  const response = await fetch(path, { ...options, method, headers, body, credentials: 'include' });

  if (!response.ok) {
    const body = await response.json().catch(() => ({}));
    if (response.status === 401) {
      localStorage.removeItem('auth_token');
      sessionStorage.clear();
      router.visit(window.location.pathname.startsWith('/portal') ? '/portal/login' : '/login', 
        { method: 'get', replace: true, preserveState: false });
    }
    const error = new Error(`${response.status}`);
    (error as any).status = response.status;
    (error as any).body = body;
    throw error;
  }

  return method === 'DELETE' || response.status === 204 ? undefined : response.json();
}
```

## Patrones de Hooks

### Query Hook
```typescript
// hooks/users/useUsersQuery.ts
import { queryKeys } from '@/lib/query-keys';

export function useUsersQuery(params: UseUsersParams) {
  const query = new URLSearchParams();
  if (params.page) query.set('page', String(params.page));
  if (params.q) query.set('q', params.q);
  if (params.sort) query.set('sort', params.sort);

  return useQuery({
    queryKey: queryKeys.users.list(params),
    queryFn: () => request<Paginated<UserListItem>>(`/api/admin/users?${query.toString()}`),
    placeholderData: (prev) => prev,
  });
}
```

### Mutation Hook
```typescript
// hooks/users/useCreateUserMutation.ts
import { queryKeys } from '@/lib/query-keys';

export function useCreateUserMutation() {
  const queryClient = useQueryClient();
  const { addNotification } = useNotificationsContext();
  const { t } = useTranslation();

  return useMutation({
    mutationFn: (input: CreateUserInput) =>
      request('/api/admin/users', { method: 'POST', body: input }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.users.all });
      addNotification({
        type: 'success',
        title: t('admin.users.notifications.created.title'),
        message: t('admin.users.notifications.created.message'),
      });
    },
    onError: (error: any) => {
      addNotification({
        type: 'error',
        title: t('admin.users.notifications.error.title'),
        message: error?.body?.message || t('admin.users.notifications.error.message'),
      });
    },
  });
}
```

### Export Hook
```typescript
// hooks/sales/useSalesExport.ts
export function useSalesExport() {
  const toast = useToast();
  const { t } = useTranslation();

  return useMutation({
    mutationFn: async (params: ExportSalesParams) => {
      const { blob, filename } = await requestFileDownload('/api/admin/sales/export', params);
      downloadFile(blob, filename);
    },
    onSuccess: () => toast.success(t('admin.sales.export_success')),
    onError: () => toast.error(t('admin.sales.export_error')),
  });
}
```

## Utilidades

### File Download
```typescript
// lib/file-download.ts
export function downloadFile(data: Blob, filename: string): void {
  const url = window.URL.createObjectURL(data);
  const link = document.createElement('a');
  link.href = url;
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  window.URL.revokeObjectURL(url);
}

export async function requestFileDownload(path: string, params: Record<string, string> = {}): Promise<BlobResponse> {
  const qs = new URLSearchParams(params).toString();
  const response = await fetch(`${path}${qs ? `?${qs}` : ''}`, { credentials: 'include' });
  if (!response.ok) throw new Error(`${response.status}`);
  const blob = await response.blob();
  const filename = response.headers.get('Content-Disposition')?.match(/filename="(.+)"/)?.[1] || 'download';
  return { blob, filename };
}
```

### Toast Hook
```typescript
// hooks/useToast.ts
export function useToast() {
  const { addNotification } = useNotificationsContext();
  return {
    success: (title: string, message?: string) => addNotification({ type: 'success', title, message }),
    error: (title: string, message?: string) => addNotification({ type: 'error', title, message }),
    warning: (title: string, message?: string) => addNotification({ type: 'warning', title, message }),
    info: (title: string, message?: string) => addNotification({ type: 'info', title, message }),
  };
}
```

## Invalidación de Queries

```typescript
export function useRetryPaymentMutation(saleId?: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => request(`/api/admin/payments/${id}/retry`, { method: 'POST' }),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: queryKeys.sales.all });
      if (saleId) qc.invalidateQueries({ queryKey: queryKeys.salePayments.list(saleId) });
    },
  });
}
```

## Configuración QueryClient

```typescript
// app.tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      retry: (failureCount, error: any) => {
        if ([401, 403].includes(error?.status)) return false;
        return failureCount < 3;
      },
    },
  },
});

createInertiaApp({
  setup({ el, App, props }) {
    const root = createRoot(el);
    root.render(
      <QueryClientProvider client={queryClient}>
        <App {...props} />
      </QueryClientProvider>
    );
  },
});
```
