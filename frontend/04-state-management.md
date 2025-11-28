---
globs: "resources/js/**/*.ts,resources/js/**/*.tsx,resources/js/**/*.jsx"
alwaysApply: false
---

# Gestión de Estado - Tanstack Query + React Hooks

## Principios de Separación

### Estado del Servidor (Tanstack Query)
```tsx
// ✅ Para datos de API
const { data, isLoading } = useUsersQuery({ page, q: debouncedQuery });
const createMutation = useCreateUserMutation();
```

### Estado Local (React Hooks)
```tsx
// ✅ Para UI y formularios
const [selectedUser, setSelectedUser] = useState<User | null>(null);
const [showFilters, setShowFilters] = useState(false);
```

## Hook de Tabla Reutilizable

### useTableState
```tsx
export function useTableState(options: { debounceDelay?: number } = {}) {
  const { debounceDelay = 300 } = options;
  
  const [query, setQuery] = useState('');
  const [debouncedQuery, setDebouncedQuery] = useState('');
  const [page, setPage] = useState(1);
  const [createOpen, setCreateOpen] = useState(false);
  const [editOpen, setEditOpen] = useState(false);
  const [selected, setSelected] = useState<any | null>(null);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedQuery(query);
      setPage(1);
    }, debounceDelay);
    return () => clearTimeout(timer);
  }, [query, debounceDelay]);

  const resetModals = () => {
    setCreateOpen(false);
    setEditOpen(false);
    setSelected(null);
  };

  return {
    query, setQuery, debouncedQuery,
    page, setPage,
    createOpen, setCreateOpen,
    editOpen, setEditOpen,
    selected, setSelected,
    resetModals,
  };
}
```

## Tanstack Query Patterns

### Query Hook Estándar
```tsx
export function useUsersQuery(params: { page?: number; q?: string; sort?: string }) {
  const queryString = new URLSearchParams();
  if (params.page) queryString.set('page', String(params.page));
  if (params.q) queryString.set('q', params.q);
  if (params.sort) queryString.set('sort', params.sort);

  return useQuery({
    queryKey: ['users', params],
    queryFn: () => request<Paginated<User>>(`/api/admin/users?${queryString}`),
    placeholderData: (prev) => prev,
  });
}
```

### Mutation con Invalidación
```tsx
export function useCreateUserMutation() {
  const queryClient = useQueryClient();
  const { addNotification } = useNotificationsContext();

  return useMutation({
    mutationFn: (input: CreateUserInput) =>
      request('/api/admin/users', { method: 'POST', body: input }),
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
      addNotification({ type: 'success', title: 'Usuario creado' });
    },
    onError: (error: any) => {
      addNotification({ 
        type: 'error', 
        title: 'Error',
        message: error?.body?.message || 'Error al crear usuario'
      });
    },
  });
}
```

### Query con Dependencias
```tsx
export function useSaleQuery(saleId: string | undefined) {
  return useQuery({
    queryKey: ['sales', saleId],
    queryFn: () => request<{ data: Sale }>(`/api/admin/sales/${saleId}`),
    enabled: !!saleId,
    staleTime: 2 * 60 * 1000,
  });
}
```

## Hook Compuesto

### Gestión Completa de Entidad
```tsx
export function useUserManagement() {
  const { hasPermission } = usePermissions();
  const tableState = useTableState();
  
  const usersQuery = useUsersQuery({
    page: tableState.page,
    q: tableState.debouncedQuery,
  });
  
  const createMutation = useCreateUserMutation();
  const updateMutation = useUpdateUserMutation();
  const deleteMutation = useDeleteUserMutation();

  const handleCreate = (data: CreateUserInput) => {
    createMutation.mutate(data, {
      onSuccess: () => tableState.resetModals(),
    });
  };

  return {
    ...tableState,
    users: usersQuery.data?.data || [],
    isLoading: usersQuery.isLoading,
    handleCreate,
    canCreate: hasPermission('users.create'),
    canEdit: hasPermission('users.update'),
  };
}
```

## Optimizaciones

### Prefetching
```tsx
export function usePrefetch() {
  const queryClient = useQueryClient();

  const prefetchUser = (userId: string) => {
    queryClient.prefetchQuery({
      queryKey: ['users', userId],
      queryFn: () => request(`/api/admin/users/${userId}`),
      staleTime: 5 * 60 * 1000,
    });
  };

  return { prefetchUser };
}

// Uso en lista
export function UserRow({ user }: { user: User }) {
  const { prefetchUser } = usePrefetch();
  
  return (
    <tr onMouseEnter={() => prefetchUser(user.id)}>
      <td>{user.name}</td>
    </tr>
  );
}
```

### Background Updates
```tsx
export function useBackgroundSync() {
  const queryClient = useQueryClient();

  useEffect(() => {
    const interval = setInterval(() => {
      queryClient.invalidateQueries({ 
        queryKey: ['dashboard', 'stats'],
        refetchType: 'none',
      });
    }, 5 * 60 * 1000);

    return () => clearInterval(interval);
  }, [queryClient]);
}
```

## Mejores Prácticas

### Keys Consistentes
```tsx
const queryKeys = {
  users: {
    all: ['users'] as const,
    list: (params: any) => [...queryKeys.users.all, 'list', params] as const,
    detail: (id: string) => [...queryKeys.users.all, 'detail', id] as const,
  },
};
```

### Error Handling
```tsx
export function useApiErrorHandler() {
  return (error: any) => {
    switch (error?.status) {
      case 401: router.visit('/login'); break;
      case 403: return { title: 'Sin permisos', type: 'error' };
      default: return { title: 'Error', message: 'Error inesperado', type: 'error' };
    }
  };
}
```
