---
globs: "resources/js/**/*.ts,resources/js/**/*.tsx,resources/js/**/*.jsx"
alwaysApply: false
---

# Arquitectura Frontend - React + Inertia.js

## Configuración Principal

### App Setup (app.tsx)
```tsx
import { createInertiaApp } from '@inertiajs/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      retry: (failureCount, error: any) => {
        if (error?.status === 401 || error?.status === 403) return false;
        return failureCount < 3;
      },
    },
  },
});

createInertiaApp({
  title: (title) => `${title} - Cohousing`,
  resolve: (name) => resolvePageComponent(`./pages/${name}.tsx`, import.meta.glob('./pages/**/*.tsx')),
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

### Organización de Tipos TypeScript

**Regla obligatoria**: Todos los tipos TypeScript deben estar centralizados en la carpeta `types/`, organizados por verticales del proyecto.

```
resources/js/types/
├── index.ts              # Tipos compartidos (SharedData, User, etc.)
├── users.ts              # Tipos del dominio Users
├── sales.ts              # Tipos del dominio Sales
├── payments.ts           # Tipos del dominio Payments
├── clients.ts            # Tipos del dominio Clients
└── portal.ts             # Tipos del dominio Portal
```

**Ejemplo de estructura**:
```typescript
// types/index.ts - Tipos compartidos
export interface SharedData {
  auth: { user: User };
  ziggy: Config & { location: string };
  sidebarOpen: boolean;
}

export interface User {
  id: string;
  name: string;
  email: string;
  roles: string[];
  permissions: string[];
}

// types/users.ts - Tipos del dominio Users
export interface UserListItem {
  id: string;
  name: string;
  email: string;
  roles: string[];
}

export interface CreateUserInput {
  name: string;
  email: string;
  password: string;
  role_ids: string[];
}

export interface UpdateUserInput extends Partial<CreateUserInput> {
  id: string;
}

// types/sales.ts - Tipos del dominio Sales
export interface Sale {
  id: string;
  client_id: string;
  total: number;
  status: 'pending' | 'completed' | 'cancelled';
}

export interface CreateSaleInput {
  client_id: string;
  items: SaleItem[];
}
```

**Reglas**:
- ❌ **NUNCA** definir tipos inline en componentes o hooks
- ❌ **NUNCA** duplicar tipos en múltiples archivos
- ✅ **SIEMPRE** importar tipos desde `types/`
- ✅ **SIEMPRE** exportar tipos desde el archivo correspondiente a su dominio

## Patrones de Componentes

### Página con Layout
```tsx
export default function UsersPage() {
  const { t } = useTranslation();
  const { hasPermission } = usePermissions();

  if (!hasPermission('users.view')) {
    return <UnauthorizedMessage />;
  }

  return (
    <AppLayout breadcrumbs={breadcrumbs}>
      <Head title={t('admin.users.title')} />
      <UserTable />
    </AppLayout>
  );
}
```

### Componente con Autorización
```tsx
export function UserActions({ user }: { user: User }) {
  const { hasPermission } = usePermissions();
  
  if (!hasPermission('users.view')) return null;
  
  return (
    <div className="flex gap-2">
      {hasPermission('users.update') && <EditButton />}
      {hasPermission('users.delete') && <DeleteButton />}
    </div>
  );
}
```

## Navegación con Inertia

### Navegación Básica
```tsx
import { Link, router } from '@inertiajs/react';

// Link para navegación SPA
<Link href="/admin/users" prefetch>Usuarios</Link>

// Navegación programática
router.visit('/admin/users', {
  method: 'get',
  preserveState: false,
});
```

### Navegación Condicional
```tsx
export function AppSidebar() {
  const { hasPermission } = usePermissions();
  
  const navItems = [
    { title: 'Dashboard', href: '/dashboard' },
    ...(hasPermission('users.view') ? [{ title: 'Usuarios', href: '/admin/users' }] : []),
    ...(hasPermission('sales.view') ? [{ title: 'Ventas', href: '/admin/sales' }] : []),
  ];

  return <NavMain items={navItems} />;
}
```

## Cliente API

### Request con Manejo de Errores
```tsx
export async function request<T>(path: string, options: RequestOptions = {}): Promise<T> {
  const response = await fetch(path, {
    ...options,
    credentials: 'include',
    headers: {
      'Accept': 'application/json',
      'X-XSRF-TOKEN': getCsrfToken(),
      ...options.headers,
    },
  });

  if (!response.ok) {
    if (response.status === 401) {
      router.visit('/login', { replace: true });
    }
    throw new Error(`${response.status}`);
  }

  return response.json();
}
```

## Mejores Prácticas

### Estructura de Archivos
- Agrupar por **feature** no por tipo
- **Hooks específicos** por dominio (`hooks/users/`, `hooks/sales/`)
- **Componentes reutilizables** en `components/shared/`
- **Tipos TypeScript** centralizados en `types/` organizados por dominio

### Performance
- **Lazy loading** para componentes pesados
- **Code splitting** por vendors
- **Prefetching** en hover para navegación

### Autorización
- **Early returns** para verificación de permisos
- **Renderizado condicional** para mejor UX
- **Backend como fuente de verdad** siempre
