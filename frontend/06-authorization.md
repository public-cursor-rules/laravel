---
globs: "resources/js/**/*.ts,resources/js/**/*.tsx,resources/js/**/*.jsx"
alwaysApply: false
---

# Autorización Frontend - Permisos y Roles

## Hook de Permisos Centralizado

```tsx
// hooks/usePermissions.ts
import { usePage } from '@inertiajs/react';

export function usePermissions() {
  const { auth } = usePage<SharedData>().props;
  const user = auth.user;

  const hasPermission = (permission: string): boolean => {
    return user?.permissions?.includes(permission) ?? false;
  };

  const hasRole = (role: string): boolean => {
    return user?.roles?.includes(role) ?? false;
  };

  const hasAnyPermission = (permissions: string[]): boolean => {
    return permissions.some(p => hasPermission(p));
  };

  return {
    user,
    hasPermission,
    hasRole,
    hasAnyPermission,
    isAdmin: () => hasRole('admin'),
    isManager: () => hasRole('manager'),
    permissions: user?.permissions || [],
    roles: user?.roles || [],
  };
}
```

## Componentes de Autorización

### PermissionGate
```tsx
export function PermissionGate({ 
  permission, 
  fallback = null, 
  children 
}: {
  permission: string | string[];
  fallback?: React.ReactNode;
  children: React.ReactNode;
}) {
  const { hasPermission, hasAnyPermission } = usePermissions();
  
  const permissions = Array.isArray(permission) ? permission : [permission];
  const hasAccess = permissions.length === 1 
    ? hasPermission(permissions[0])
    : hasAnyPermission(permissions);

  return hasAccess ? <>{children}</> : <>{fallback}</>;
}
```

### RoleGuard
```tsx
export function RoleGuard({ roles, children, fallback = null }: {
  roles: string | string[];
  children: React.ReactNode;
  fallback?: React.ReactNode;
}) {
  const { hasRole, hasAnyRole } = usePermissions();
  
  const rolesList = Array.isArray(roles) ? roles : [roles];
  const hasAccess = rolesList.length === 1 ? hasRole(rolesList[0]) : hasAnyRole(rolesList);

  return hasAccess ? <>{children}</> : <>{fallback}</>;
}
```

## Patrones de Uso

### Early Returns en Páginas
```tsx
export default function UsersPage() {
  const { hasPermission } = usePermissions();
  
  if (!hasPermission('users.view')) {
    return (
      <div className="text-center py-8">
        <p>{t('errors.access_denied')}</p>
      </div>
    );
  }
  
  return <UsersList />;
}
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

### Acciones Condicionales
```tsx
export function UserActions({ user }: { user: User }) {
  const { hasPermission } = usePermissions();
  
  if (!hasPermission('users.update') && !hasPermission('users.delete')) {
    return null;
  }

  return (
    <div className="flex gap-2">
      {hasPermission('users.update') && <EditButton />}
      {hasPermission('users.delete') && <DeleteButton />}
    </div>
  );
}
```

## Manejo de Errores

### Error de Autorización
```tsx
export function useApiErrorHandler() {
  const { t } = useTranslation();
  
  return (error: any) => {
    switch (error?.status) {
      case 401:
        router.visit('/login', { replace: true });
        break;
      case 403:
        return { title: t('errors.forbidden'), type: 'error' };
      case 404:
        return { title: t('errors.not_found'), type: 'error' };
      default:
        return { title: t('errors.general'), type: 'error' };
    }
  };
}
```

### ProtectedRoute
```tsx
export function ProtectedRoute({ permission, children }: {
  permission?: string;
  children: React.ReactNode;
}) {
  const { user, hasPermission } = usePermissions();
  
  if (!user) {
    return <LoginRequired />;
  }
  
  if (permission && !hasPermission(permission)) {
    return <AccessDenied />;
  }
  
  return <>{children}</>;
}
```

## Mejores Prácticas

### Composición de Permisos
```tsx
const canEditProfile = user?.id === profileUserId || hasPermission('users.update');
const canDeleteUser = hasPermission('users.delete') && user?.id !== targetUserId;
```

### Autorización en Formularios
```tsx
export function UserForm() {
  const { hasPermission, isAdmin } = usePermissions();
  
  return (
    <form>
      {/* Campos básicos */}
      <Input {...register('name')} />
      
      {/* Campo solo para admin */}
      {isAdmin() && <RoleSelector {...register('roles')} />}
      
      {/* Campo solo con permiso específico */}
      {hasPermission('users.advanced') && <AdvancedSettings />}
    </form>
  );
}
```

### Testing de Autorización
```tsx
// Mock del hook
jest.mock('@/hooks/usePermissions', () => ({
  usePermissions: () => ({
    hasPermission: jest.fn((permission) => permission === 'users.view'),
  }),
}));

it('renders content when user has permission', () => {
  render(
    <PermissionGate permission="users.view">
      <div>Protected Content</div>
    </PermissionGate>
  );
  
  expect(screen.getByText('Protected Content')).toBeInTheDocument();
});
```
