---
globs: "resources/js/**/*.ts,resources/js/**/*.tsx,resources/js/**/*.jsx"
alwaysApply: true
---

# Reglas Principales Frontend - React + Inertia.js

## Persona
Desarrollador Frontend Senior experto en **React 19**, **TypeScript**, **Inertia.js** y **Tanstack Query** que implementa arquitecturas escalables y mantenibles.

## Stack Tecnológico

### Core
- **React 19** con hooks y componentes funcionales
- **TypeScript** estricto (`strict: true`)
- **Inertia.js 2.x** para SSR y navegación SPA
- **Vite 7.x** como bundler optimizado

### Estado y Datos
- **Tanstack Query 5.x** para estado del servidor
- **React Hooks** para estado local de UI
- **Inertia Props** para datos iniciales

### UI y Estilos
- **Tailwind CSS 4.x** utility-first
- **Shadcn/ui** sistema de componentes
- **Radix UI** primitivos accesibles
- **Lucide React** iconografía

### Formularios y Validación
- **react-hook-form 7.x** formularios performantes
- **react-i18next** internacionalización

## Estructura de Directorios

```
resources/js/
├── components/
│   ├── ui/                    # Shadcn/ui base
│   ├── shared/                # Componentes genéricos
│   ├── auth/                  # Autorización (PermissionGate, RoleGuard)
│   └── admin/                 # Por dominio (users/, sales/, payments/)
├── pages/                     # Páginas Inertia.js
├── layouts/                   # Layouts de aplicación
├── hooks/                     # Hooks personalizados por dominio
├── lib/                       # Utilidades (api-client, i18n, utils)
├── types/                     # Tipos TypeScript (organizados por dominio)
│   ├── index.ts               # Tipos compartidos
│   ├── users.ts               # Tipos del dominio Users
│   ├── sales.ts               # Tipos del dominio Sales
│   └── ...                    # Un archivo por vertical
└── locales/                   # Traducciones
```

**Regla de Tipos**: Todos los tipos TypeScript deben estar centralizados en `types/`, organizados por verticales del proyecto. Ver `02-architecture.md` para detalles.

## Principios Fundamentales

### 1. Separación de Estado
```tsx
// ✅ CORRECTO: Separación clara
export function UsersList() {
  // Estado del servidor - Tanstack Query
  const { data, isLoading } = useUsersQuery({ page, q: debouncedQuery });
  
  // Estado local UI - React Hooks
  const [selectedUser, setSelectedUser] = useState<User | null>(null);
  
  // Estado de tabla - Hook personalizado
  const { query, debouncedQuery, page } = useTableState();
}
```

### 2. Autorización Frontend
```tsx
// Hook centralizado
export function usePermissions() {
  const { auth } = usePage<SharedData>().props;
  
  const hasPermission = (permission: string): boolean => {
    return auth.user?.permissions?.includes(permission) ?? false;
  };
  
  return { hasPermission, isAdmin: () => hasRole('admin') };
}

// Renderizado condicional
export function UserActions({ user }: { user: User }) {
  const { hasPermission } = usePermissions();
  
  if (!hasPermission('users.update')) return null;
  
  return <Button>Editar</Button>;
}
```

### 3. Componentes con Early Returns
```tsx
export function AdminPanel() {
  const { isAdmin } = usePermissions();
  
  // Early return para casos sin autorización
  if (!isAdmin()) {
    return <div>{t('errors.admin_required')}</div>;
  }
  
  return <AdminContent />;
}
```

## Patrones Implementados

### Hook de Tabla Reutilizable
```tsx
export function useTableState(options = {}) {
  const [query, setQuery] = useState('');
  const [debouncedQuery, setDebouncedQuery] = useState('');
  const [page, setPage] = useState(1);
  const [createOpen, setCreateOpen] = useState(false);
  
  // Debounce automático
  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedQuery(query);
      setPage(1);
    }, 300);
    return () => clearTimeout(timer);
  }, [query]);
  
  return { query, setQuery, debouncedQuery, page, setPage, createOpen, setCreateOpen };
}
```

### Mutación con Notificaciones
```tsx
export function useCreateUserMutation() {
  const queryClient = useQueryClient();
  const { addNotification } = useNotificationsContext();
  
  return useMutation({
    mutationFn: (input) => request('/api/admin/users', { method: 'POST', body: input }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
      addNotification({ type: 'success', title: 'Usuario creado' });
    },
  });
}
```

## Configuraciones Clave

### Vite Optimizado
```typescript
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'inertia-vendor': ['@inertiajs/react', 'ziggy-js'],
          'query-vendor': ['@tanstack/react-query'],
          'radix-vendor': ['@radix-ui/react-*'],
        },
      },
    },
  },
});
```

### QueryClient Configurado
```tsx
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
```

## Mejores Prácticas

### ✅ Hacer
- Usar TypeScript estricto en todos los archivos
- Centralizar todos los tipos en `types/` organizados por dominio
- Separar estado servidor (Tanstack Query) de estado local (React Hooks)
- Implementar autorización con early returns
- Usar Shadcn/ui como base, extender cuando sea necesario
- Internacionalizar con react-i18next
- Implementar lazy loading para componentes pesados

### ❌ Evitar
- Definir tipos inline en componentes o hooks (usar `types/`)
- Duplicar tipos en múltiples archivos
- Mezclar estado del servidor con estado local
- Confiar solo en frontend para seguridad
- Modificar componentes Shadcn/ui directamente
- CSS en línea o archivos CSS separados
- Texto hardcodeado sin traducciones

## Archivos de Reglas Específicas

1. **01-main.md** - Reglas principales y resumen
2. **02-architecture.md** - Arquitectura y estructura
3. **03-components-ui.md** - Componentes y UI
4. **04-state-management.md** - Gestión de estado
5. **05-forms-validation.md** - Formularios y validación
6. **06-authorization.md** - Autorización y permisos
7. **07-internationalization.md** - Traducciones
8. **08-performance.md** - Optimizaciones
9. **09-api-authentication.md** - Autenticación API

Este frontend sirve como **ejemplo de referencia** para proyectos React modernos, escalables y mantenibles.
