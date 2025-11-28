---
globs: "resources/js/**/*.ts,resources/js/**/*.tsx,resources/js/**/*.jsx"
alwaysApply: false
---

# Componentes UI - Shadcn/ui + Tailwind CSS

## Configuración Base

### Shadcn/ui Setup (components.json)
```json
{
  "style": "default",
  "tsx": true,
  "tailwind": {
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "ui": "@/components/ui",
    "hooks": "@/hooks"
  }
}
```

### Utilidades (lib/utils.ts)
```typescript
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

export function formatCurrency(amount: number): string {
  return new Intl.NumberFormat('es-ES', { style: 'currency', currency: 'EUR' }).format(amount);
}
```

## Componentes Base Extendidos

### Input con Estados de Error
```tsx
interface InputProps extends React.ComponentProps<"input"> {
  error?: string;
}

export const Input = React.forwardRef<HTMLInputElement, InputProps>(
  ({ className, error, ...props }, ref) => {
    return (
      <div className="space-y-1">
        <input
          className={cn(
            "flex h-9 w-full rounded-md border border-input bg-transparent px-3 py-1 text-sm shadow-xs transition-colors",
            "focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring",
            error && "border-destructive",
            className
          )}
          ref={ref}
          {...props}
        />
        {error && <p className="text-sm text-destructive">{error}</p>}
      </div>
    );
  }
);
```

### ResponsiveTable
```tsx
interface ResponsiveTableProps<T> {
  data: T[];
  columns: Array<{
    key: keyof T;
    header: string;
    cell?: (item: T) => React.ReactNode;
  }>;
  keyExtractor: (item: T) => string;
  isLoading?: boolean;
}

export function ResponsiveTable<T>({ data, columns, keyExtractor, isLoading }: ResponsiveTableProps<T>) {
  if (isLoading) return <TableSkeleton />;
  if (!data.length) return <EmptyState />;

  return (
    <>
      {/* Desktop: Tabla */}
      <div className="hidden md:block">
        <Table>
          <TableHeader>
            <TableRow>
              {columns.map((col) => <TableHead key={String(col.key)}>{col.header}</TableHead>)}
            </TableRow>
          </TableHeader>
          <TableBody>
            {data.map((item) => (
              <TableRow key={keyExtractor(item)}>
                {columns.map((col) => (
                  <TableCell key={String(col.key)}>
                    {col.cell ? col.cell(item) : String(item[col.key])}
                  </TableCell>
                ))}
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </div>

      {/* Mobile: Cards */}
      <div className="md:hidden space-y-4">
        {data.map((item) => (
          <Card key={keyExtractor(item)}>
            <CardContent className="pt-6">
              {columns.map((col) => (
                <div key={String(col.key)} className="flex justify-between py-1">
                  <span className="font-medium text-sm">{col.header}:</span>
                  <span className="text-sm">{col.cell ? col.cell(item) : String(item[col.key])}</span>
                </div>
              ))}
            </CardContent>
          </Card>
        ))}
      </div>
    </>
  );
}
```

### Pagination
```tsx
interface PaginationProps {
  currentPage: number;
  totalPages: number;
  onPageChange: (page: number) => void;
  isLoading?: boolean;
}

export function Pagination({ currentPage, totalPages, onPageChange, isLoading }: PaginationProps) {
  if (totalPages <= 1) return null;

  return (
    <div className="flex items-center justify-center space-x-2">
      <Button
        variant="outline"
        size="sm"
        onClick={() => onPageChange(currentPage - 1)}
        disabled={currentPage <= 1 || isLoading}
      >
        <ChevronLeft className="h-4 w-4" />
      </Button>
      
      <span className="text-sm">
        {currentPage} de {totalPages}
      </span>
      
      <Button
        variant="outline"
        size="sm"
        onClick={() => onPageChange(currentPage + 1)}
        disabled={currentPage >= totalPages || isLoading}
      >
        <ChevronRight className="h-4 w-4" />
      </Button>
    </div>
  );
}
```

## Componentes de Autorización

### PermissionGate
```tsx
interface PermissionGateProps {
  permission: string | string[];
  fallback?: React.ReactNode;
  children: React.ReactNode;
}

export function PermissionGate({ permission, fallback = null, children }: PermissionGateProps) {
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

## Estados de Componentes

### Loading States
```tsx
export function LoadingSpinner({ size = "md" }: { size?: "sm" | "md" | "lg" }) {
  return (
    <div className={cn(
      "animate-spin rounded-full border-2 border-muted border-t-primary",
      { "h-4 w-4": size === "sm", "h-6 w-6": size === "md", "h-8 w-8": size === "lg" }
    )} />
  );
}

export function TableSkeleton() {
  return (
    <div className="space-y-4">
      {Array.from({ length: 5 }).map((_, i) => (
        <div key={i} className="h-16 bg-muted animate-pulse rounded-lg" />
      ))}
    </div>
  );
}
```

### Empty States
```tsx
export function EmptyState({ message, action }: {
  message: string;
  action?: { label: string; onClick: () => void };
}) {
  return (
    <div className="flex flex-col items-center justify-center py-12 text-center">
      <FileX className="h-12 w-12 text-muted-foreground mb-4" />
      <p className="text-muted-foreground mb-6">{message}</p>
      {action && <Button onClick={action.onClick}>{action.label}</Button>}
    </div>
  );
}
```

## Mejores Prácticas

### Extensión de Componentes
```tsx
// ✅ CORRECTO: Extender sin modificar base
export function ActionButton({ isLoading, children, ...props }: ButtonProps & { isLoading?: boolean }) {
  return (
    <Button disabled={isLoading} {...props}>
      {isLoading ? <><LoadingSpinner size="sm" className="mr-2" />Cargando...</> : children}
    </Button>
  );
}
```

### Composición
```tsx
// ✅ CORRECTO: Usar composición
export function UserCard({ user }: { user: User }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{user.name}</CardTitle>
        <CardDescription>{user.email}</CardDescription>
      </CardHeader>
      <CardFooter>
        <PermissionGate permission="users.update">
          <Button size="sm">Editar</Button>
        </PermissionGate>
      </CardFooter>
    </Card>
  );
}
```

### Accesibilidad
```tsx
// ✅ CORRECTO: Componentes accesibles
export function SearchInput({ onSearch }: { onSearch: (query: string) => void }) {
  return (
    <div className="relative">
      <Search className="absolute left-3 top-1/2 transform -translate-y-1/2 h-4 w-4" />
      <Input
        placeholder="Buscar..."
        onKeyDown={(e) => e.key === 'Enter' && onSearch(e.currentTarget.value)}
        className="pl-10"
        aria-label="Campo de búsqueda"
      />
    </div>
  );
}
```
