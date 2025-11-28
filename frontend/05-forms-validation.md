---
globs: "resources/js/**/*.ts,resources/js/**/*.tsx,resources/js/**/*.jsx"
alwaysApply: false
---

# Formularios y Validación - React Hook Form

## Formulario Base

### Estructura Estándar
```tsx
interface FormProps<T> {
  defaultValues?: Partial<T>;
  onSubmit: (values: T) => void;
  isEditing?: boolean;
  submitting?: boolean;
}

export function UserForm({ defaultValues, onSubmit, isEditing, submitting }: FormProps<UserFormValues>) {
  const { t } = useTranslation();
  const { register, handleSubmit, formState: { errors, isValid } } = useForm<UserFormValues>({
    defaultValues: { name: '', email: '', ...defaultValues },
    mode: 'onChange',
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <FormField label={t('common.name')} htmlFor="name" error={errors.name?.message}>
        <Input
          id="name"
          {...register('name', {
            required: t('validation.required', { field: t('common.name') }),
            minLength: { value: 2, message: t('validation.min_length', { min: 2 }) }
          })}
        />
      </FormField>

      <Button type="submit" disabled={!isValid || submitting}>
        {isEditing ? t('common.save') : t('common.create')}
      </Button>
    </form>
  );
}
```

## Formularios con Inertia.js

### useForm de Inertia
```tsx
export default function Profile() {
  const user = usePage<SharedData>().props.auth.user;
  const { data, setData, patch, errors, processing } = useForm({
    name: user.name,
    email: user.email,
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    patch(route('profile.update'), { preserveScroll: true });
  };

  return (
    <form onSubmit={handleSubmit}>
      <Input
        value={data.name}
        onChange={(e) => setData('name', e.target.value)}
        error={errors.name}
      />
      <Button type="submit" disabled={processing}>
        {processing ? 'Guardando...' : 'Guardar'}
      </Button>
    </form>
  );
}
```

## Validación Avanzada

### Validación con Zod
```tsx
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const saleSchema = z.object({
  user_id: z.string().uuid(),
  total: z.number().min(0.01),
  payments: z.array(z.object({
    due_date: z.string(),
    amount: z.number().min(0.01),
  })).min(1),
}).refine((data) => {
  const paymentsTotal = data.payments.reduce((sum, p) => sum + p.amount, 0);
  return Math.abs(paymentsTotal - data.total) < 0.01;
}, { message: 'La suma de pagos debe igual al total', path: ['payments'] });

export function SaleForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(saleSchema),
  });
  
  return <form>{/* campos */}</form>;
}
```

### Validación Asíncrona
```tsx
export function useAsyncValidation() {
  const [isValidating, setIsValidating] = useState(false);
  const [errors, setErrors] = useState<Record<string, string>>({});

  const validateField = async (field: string, value: string, endpoint: string) => {
    if (!value) return;
    
    setIsValidating(true);
    try {
      await request(endpoint, { method: 'POST', body: { [field]: value } });
      setErrors(prev => ({ ...prev, [field]: '' }));
    } catch (error: any) {
      if (error?.status === 422) {
        setErrors(prev => ({ ...prev, [field]: error.body.errors[field][0] }));
      }
    } finally {
      setIsValidating(false);
    }
  };

  return { isValidating, errors, validateField };
}
```

## Componentes de Formulario

### FormField Wrapper
```tsx
interface FormFieldProps {
  label: string;
  htmlFor: string;
  error?: string;
  required?: boolean;
  children: React.ReactNode;
}

export function FormField({ label, htmlFor, error, required, children }: FormFieldProps) {
  return (
    <div className="space-y-2">
      <Label htmlFor={htmlFor}>
        {label}
        {required && <span className="text-destructive ml-1">*</span>}
      </Label>
      {children}
      {error && <p className="text-sm text-destructive">{error}</p>}
    </div>
  );
}
```

### PasswordInput con Toggle
```tsx
export const PasswordInput = React.forwardRef<HTMLInputElement, InputProps>(
  ({ className, ...props }, ref) => {
    const [showPassword, setShowPassword] = useState(false);

    return (
      <div className="relative">
        <Input
          type={showPassword ? "text" : "password"}
          className={cn("pr-10", className)}
          ref={ref}
          {...props}
        />
        <Button
          type="button"
          variant="ghost"
          size="icon"
          className="absolute right-0 top-0 h-9 w-10"
          onClick={() => setShowPassword(!showPassword)}
        >
          {showPassword ? <EyeOff className="h-4 w-4" /> : <Eye className="h-4 w-4" />}
        </Button>
      </div>
    );
  }
);
```

## Manejo de Errores

### Integración con Backend
```tsx
export function ServerValidatedForm() {
  const { setError, clearErrors } = useForm();
  const mutation = useCreateUserMutation();

  const handleSubmit = (data: FormData) => {
    clearErrors();
    mutation.mutate(data, {
      onError: (error: any) => {
        if (error?.status === 422 && error?.body?.errors) {
          Object.entries(error.body.errors).forEach(([field, messages]) => {
            setError(field as any, { type: 'server', message: (messages as string[])[0] });
          });
        }
      },
    });
  };

  return <form onSubmit={handleSubmit(handleSubmit)}>{/* campos */}</form>;
}
```

## Mejores Prácticas

### Validación Progresiva
```tsx
const { trigger } = useForm();

const handleStepChange = async (step: number) => {
  const isValid = await trigger(['name', 'email']); // Validar campos específicos
  if (isValid) setCurrentStep(step + 1);
};
```

### Reset Inteligente
```tsx
useEffect(() => {
  reset({ name: defaultValues?.name || '', email: defaultValues?.email || '' });
}, [defaultValues, reset]);
```
