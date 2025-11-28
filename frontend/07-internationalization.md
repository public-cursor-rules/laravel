---
globs: "resources/js/**/*.ts,resources/js/**/*.tsx,resources/js/**/*.jsx"
alwaysApply: false
---

# Internacionalización - react-i18next

## Configuración Base

### Setup (lib/i18n.ts)
```typescript
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';
import es from '../locales/es/translation.json';

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    resources: { es: { translation: es } },
    lng: 'es',
    fallbackLng: 'es',
    debug: import.meta.env.DEV,
    interpolation: { escapeValue: false },
    detection: {
      order: ['localStorage', 'navigator'],
      caches: ['localStorage'],
    },
  });
```

## Estructura de Traducciones

### Organización Jerárquica
```json
{
  "common": {
    "save": "Guardar",
    "cancel": "Cancelar",
    "delete": "Eliminar",
    "loading": "Cargando...",
    "search": "Buscar..."
  },
  "navigation": {
    "dashboard": "Panel de Control",
    "users": "Usuarios",
    "sales": "Ventas"
  },
  "admin": {
    "users": {
      "title": "Gestión de Usuarios",
      "create": "Crear Usuario",
      "notifications": {
        "created": {
          "title": "Usuario Creado",
          "message": "{{name}} ha sido creado exitosamente"
        }
      }
    }
  },
  "validation": {
    "required": "{{field}} es requerido",
    "email_invalid": "Email inválido",
    "min_length": "Mínimo {{min}} caracteres"
  },
  "errors": {
    "access_denied": "Acceso Denegado",
    "forbidden": "Sin permisos",
    "not_found": "No encontrado"
  }
}
```

## Uso en Componentes

### Hook Básico
```tsx
export function WelcomeMessage({ userName }: { userName: string }) {
  const { t } = useTranslation();
  
  return (
    <div>
      <h1>{t('dashboard.welcome', { name: userName })}</h1>
      <p>{t('dashboard.subtitle')}</p>
    </div>
  );
}
```

### Formularios Internacionalizados
```tsx
export function UserForm() {
  const { t } = useTranslation();
  const { register, formState: { errors } } = useForm();

  return (
    <form>
      <FormField
        label={t('admin.users.name')}
        error={errors.name?.message}
      >
        <Input
          {...register('name', {
            required: t('validation.required', { field: t('admin.users.name') })
          })}
        />
      </FormField>
    </form>
  );
}
```

### Componente Trans para HTML
```tsx
import { Trans } from 'react-i18next';

// JSON: "agreement": "Acepto los <link>términos</link>"
export function TermsAgreement() {
  return (
    <Trans
      i18nKey="terms.agreement"
      components={{
        link: <Link href="/terms" className="text-primary hover:underline" />
      }}
    />
  );
}
```

## Formateo Localizado

### Hook de Formatters
```tsx
export function useFormatters() {
  const { i18n } = useTranslation();
  
  const formatCurrency = (amount: number): string => {
    return new Intl.NumberFormat(i18n.language, {
      style: 'currency',
      currency: 'EUR',
    }).format(amount);
  };

  const formatDate = (date: string | Date): string => {
    return new Intl.DateTimeFormat(i18n.language, {
      year: 'numeric',
      month: 'long',
      day: 'numeric',
    }).format(new Date(date));
  };

  return { formatCurrency, formatDate };
}
```

### Selector de Idioma
```tsx
export function LanguageSelector() {
  const { i18n } = useTranslation();
  
  const languages = [
    { code: 'es', name: 'Español', flag: '🇪🇸' },
    { code: 'en', name: 'English', flag: '🇺🇸' },
  ];

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="sm">
          <Globe className="h-4 w-4 mr-2" />
          {languages.find(l => l.code === i18n.language)?.name}
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        {languages.map((lang) => (
          <DropdownMenuItem
            key={lang.code}
            onClick={() => i18n.changeLanguage(lang.code)}
          >
            {lang.flag} {lang.name}
          </DropdownMenuItem>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

## Mejores Prácticas

### Keys Organizadas
```tsx
// ✅ CORRECTO: Jerárquicas
t('admin.users.notifications.created.title')
t('validation.required', { field: 'nombre' })

// ❌ INCORRECTO: Planas
t('userCreatedTitle')
t('nameRequired')
```

### Interpolación Segura
```tsx
// ✅ CORRECTO
const message = t('welcome', { name: user.name, count: users.length });

// ❌ INCORRECTO
const message = `Bienvenido ${user.name}`;
```

### Pluralización
```json
{
  "items": {
    "count_0": "Sin elementos",
    "count_1": "{{count}} elemento", 
    "count_other": "{{count}} elementos"
  }
}
```

```tsx
const { t } = useTranslation();
return <p>{t('items.count', { count })}</p>;
```
