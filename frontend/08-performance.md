---
globs: "resources/js/**/*.ts,resources/js/**/*.tsx,resources/js/**/*.jsx"
alwaysApply: false
---

# Performance Frontend - Optimizaciones

## Configuración Vite

### Code Splitting Optimizado
```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'inertia-vendor': ['@inertiajs/react', 'ziggy-js'],
          'query-vendor': ['@tanstack/react-query'],
          'radix-vendor': ['@radix-ui/react-dropdown-menu', '@radix-ui/react-dialog'],
        },
      },
    },
    chunkSizeWarningLimit: 1000,
  },
});
```

## Lazy Loading

### Componentes Pesados
```tsx
import { lazy, Suspense } from 'react';

const AnalyticsDashboard = lazy(() => import('@/components/dashboard/AnalyticsDashboard'));

export function Dashboard() {
  const { hasPermission } = usePermissions();
  
  return (
    <div>
      {hasPermission('dashboard.analytics') && (
        <Suspense fallback={<Skeleton className="h-96 w-full" />}>
          <AnalyticsDashboard />
        </Suspense>
      )}
    </div>
  );
}
```

### Intersection Observer
```tsx
export function useIntersectionObserver(options = {}) {
  const [isIntersecting, setIsIntersecting] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const observer = new IntersectionObserver(
      ([entry]) => setIsIntersecting(entry.isIntersecting),
      { threshold: 0.1, ...options }
    );

    observer.observe(element);
    return () => observer.unobserve(element);
  }, []);

  return { ref, isIntersecting };
}
```

## Optimizaciones React

### Memoización Estratégica
```tsx
export const UserCard = memo(function UserCard({ user, onEdit }: {
  user: User;
  onEdit: (user: User) => void;
}) {
  const handleEdit = useCallback(() => onEdit(user), [onEdit, user]);
  
  const displayName = useMemo(() => 
    `${user.name} (${user.roles.join(', ')})`, 
    [user.name, user.roles]
  );

  return (
    <Card>
      <CardTitle>{displayName}</CardTitle>
      <Button onClick={handleEdit}>Editar</Button>
    </Card>
  );
});
```

### Debouncing
```tsx
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}
```

## Tanstack Query Optimizations

### Prefetching Inteligente
```tsx
export function usePrefetch() {
  const queryClient = useQueryClient();

  const prefetchUser = useCallback((userId: string) => {
    queryClient.prefetchQuery({
      queryKey: ['users', userId],
      queryFn: () => request(`/api/admin/users/${userId}`),
      staleTime: 5 * 60 * 1000,
    });
  }, [queryClient]);

  return { prefetchUser };
}

// Uso en hover
<tr onMouseEnter={() => prefetchUser(user.id)}>
```

### Background Sync
```tsx
export function useBackgroundSync() {
  const queryClient = useQueryClient();

  useEffect(() => {
    const interval = setInterval(() => {
      queryClient.invalidateQueries({ 
        queryKey: ['dashboard'],
        refetchType: 'none' 
      });
    }, 5 * 60 * 1000);

    return () => clearInterval(interval);
  }, [queryClient]);
}
```

### Infinite Queries
```tsx
export function useInfinitePayments() {
  return useInfiniteQuery({
    queryKey: ['payments', 'infinite'],
    queryFn: ({ pageParam = 1 }) => 
      request(`/api/admin/payments?page=${pageParam}`),
    getNextPageParam: (lastPage) => {
      const { current_page, last_page } = lastPage.meta;
      return current_page < last_page ? current_page + 1 : undefined;
    },
    initialPageParam: 1,
  });
}
```

## Optimizaciones de Bundle

### Tree Shaking
```tsx
// ✅ CORRECTO: Importaciones específicas
import { format } from 'date-fns/format';
import { parseISO } from 'date-fns/parseISO';

// ❌ INCORRECTO: Importación completa
import * as dateFns from 'date-fns';
```

### Dynamic Imports
```tsx
export function ExportButton() {
  const handleExport = async () => {
    const { exportToCSV } = await import('@/lib/csv-export');
    const data = await fetchData();
    exportToCSV(data, 'export.csv');
  };

  return <Button onClick={handleExport}>Exportar</Button>;
}
```

## Monitoreo

### Performance Hook (desarrollo)
```tsx
export function usePerformanceMonitor(componentName: string) {
  useEffect(() => {
    if (import.meta.env.DEV) {
      const start = performance.now();
      return () => {
        const time = performance.now() - start;
        if (time > 16) console.warn(`Slow render: ${componentName} ${time.toFixed(2)}ms`);
      };
    }
  }, [componentName]);
}
```

## Mejores Prácticas

### Bundle Analysis
```bash
npm run build
npx vite-bundle-analyzer dist
```

### Preloading Crítico
```html
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin />
```

### Métricas Web Vitals
```tsx
import { getCLS, getFID, getLCP } from 'web-vitals';

export function initWebVitals() {
  if (import.meta.env.PROD) {
    getCLS(console.log);
    getFID(console.log);
    getLCP(console.log);
  }
}
```
