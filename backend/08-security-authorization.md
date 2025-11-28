---
globs: "**/*.php"
alwaysApply: false
---

# Reglas de Seguridad y Autorización - Backend

## Principios de Seguridad

### Defensa en Profundidad
- **Backend**: Fuente de verdad, validación y aplicación de reglas
- **Frontend**: Reflejo de permisos para UX, nunca para seguridad
- **Base de Datos**: Constraints y validaciones adicionales
- **Middleware**: Múltiples capas de verificación

### Principio de Menor Privilegio
- Cada usuario tiene **solo** los permisos mínimos necesarios
- Cada endpoint requiere el permiso **específico** para su función

## Configuración de Spatie Laravel Permission

### Modelos Personalizados con UUIDs

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Spatie\Permission\Models\Permission as SpatiePermission;

class Permission extends SpatiePermission
{
    use HasUuids;

    protected $keyType = 'string';
    public $incrementing = false;
}
```

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Spatie\Permission\Models\Role as SpatieRole;

class Role extends SpatieRole
{
    use HasUuids;

    protected $keyType = 'string';
    public $incrementing = false;
}
```

### Modelo User con Autorización

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\SoftDeletes;
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasUuids, HasRoles, SoftDeletes;

    protected $keyType = 'string';
    public $incrementing = false;
}
```

## Sistema de Permisos

### Nomenclatura Consistente
Formato: `{resource}.{action}`

```php
// Ejemplos de permisos
'resource.view'        // Ver recurso
'resource.create'      // Crear recurso
'resource.update'      // Actualizar recurso
'resource.delete'      // Eliminar recurso
```

### Seeder de Roles y Permisos

```php
<?php

declare(strict_types=1);

namespace Database\Seeders;

use Spatie\Permission\Models\Permission;
use Spatie\Permission\Models\Role;
use Illuminate\Database\Seeder;

class RolesAndPermissionsSeeder extends Seeder
{
    public function run(): void
    {
        $guard = config('auth.defaults.guard', 'web');

        // Definir permisos según necesidades del proyecto
        $permissions = [
            // Definir permisos según requisitos del proyecto
        ];

        // Crear permisos
        foreach ($permissions as $permissionName) {
            Permission::firstOrCreate([
                'name' => $permissionName,
                'guard_name' => $guard,
            ]);
        }

        // Crear roles y asignar permisos según necesidades del proyecto
    }
}
```

## Middleware de Autorización

### Middleware Personalizado para Permisos

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;

class EnsureHasPermission
{
    public function handle(Request $request, Closure $next, string $permission)
    {
        $user = $request->user();

        if (!$user) {
            throw new AccessDeniedHttpException('Unauthenticated.');
        }

        if (!$user->can($permission)) {
            throw new AccessDeniedHttpException('This action is unauthorized.');
        }

        return $next($request);
    }
}
```

### Registro en Kernel

```php
protected $middlewareAliases = [
    'permission' => \App\Http\Middleware\EnsureHasPermission::class,
];
```

## Implementación en Rutas

### Agrupación por Nivel de Autorización

```php
<?php

use Illuminate\Support\Facades\Route;

// Rutas protegidas por rol específico
Route::middleware(['auth:sanctum', 'role:nombre-rol'])
    ->prefix('admin')
    ->group(function () {
        Route::get('/resources', [ResourceController::class, 'index'])
            ->middleware('permission:resource.view');
    });

// Rutas protegidas por permiso (múltiples roles pueden tener el permiso)
Route::middleware(['auth:sanctum'])
    ->prefix('admin')
    ->group(function () {
        Route::get('/resources', [ResourceController::class, 'index'])
            ->middleware('permission:resource.view');
    });
```

## Autorización en Controladores

### Verificación Temprana

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\Api\StoreResourceRequest;

class ResourceController extends Controller
{
    public function store(StoreResourceRequest $request)
    {
        // La autorización ya se verificó en:
        // 1. Middleware de ruta
        // 2. FormRequest authorize() method
        
        $validated = $request->validated();
        // Procesar creación
    }
}
```

## Form Requests con Autorización

### Autorización Contextual

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests\Api;

use Illuminate\Foundation\Http\FormRequest;

class StoreResourceRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('resource.create') ?? false;
    }

    public function rules(): array
    {
        return [
            // Reglas de validación
        ];
    }
}
```

## Testing de Autorización

### Tests de Permisos

```php
<?php

use App\Models\User;

it('user has correct permissions', function () {
    $user = User::factory()->create();
    $user->assignRole('role-name');

    expect($user->can('resource.view'))->toBeTrue();
    expect($user->can('resource.create'))->toBeFalse();
});
```

### Tests de Endpoints Protegidos

```php
it('rejects unauthenticated access', function () {
    $this->getJson('/api/admin/resources')->assertStatus(401);
});

it('rejects unauthorized users', function () {
    $user = User::factory()->create();
    $user->assignRole('role-without-permission');

    $this->actingAs($user)
        ->getJson('/api/admin/resources')
        ->assertStatus(403);
});
```

## Mejores Prácticas

### 1. Nunca Confiar Solo en el Frontend
```php
// ❌ MAL: Solo verificación en frontend
// ✅ BIEN: Verificación en múltiples capas
Route::middleware(['auth:sanctum', 'permission:resource.view'])
    ->get('/resources', [ResourceController::class, 'index']);
```

### 2. Usar Transacciones para Operaciones Críticas
```php
public function store(StoreResourceRequest $request)
{
    return DB::transaction(function () use ($request) {
        // Operaciones atómicas
    });
}
```

### 3. Logging de Acciones Sensibles
```php
public function destroy(Resource $resource)
{
    \Log::info('Resource deletion', [
        'user_id' => auth()->id(),
        'resource_id' => $resource->id,
    ]);

    $resource->delete();
}
```

### 4. Rate Limiting en Endpoints Sensibles
```php
Route::middleware(['auth:sanctum', 'throttle:10,1'])
    ->delete('/resources/{resource}', [ResourceController::class, 'destroy']);
```

## Configuración de Seguridad Adicional

### Sanctum para APIs
```php
// config/sanctum.php
'expiration' => 60 * 24, // 24 horas
```

### Headers de Seguridad
```php
// En middleware o servidor web
'X-Content-Type-Options' => 'nosniff',
'X-Frame-Options' => 'DENY',
'X-XSS-Protection' => '1; mode=block',
'Strict-Transport-Security' => 'max-age=31536000; includeSubDomains',
```
