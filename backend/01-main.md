---
globs: "**/*.php"
alwaysApply: true
---

# Reglas Principales del Backend

## Resumen Ejecutivo

Este proyecto usa **Laravel con arquitectura hexagonal**, **Spatie Laravel Permission** para autorización, **UUIDs** como claves primarias, **Pest PHP** para testing, y **L5-Swagger** para documentación de APIs.

## Stack Tecnológico

### Core
- **Framework**: Laravel 12.x
- **PHP**: 8.2+ con `declare(strict_types=1);`
- **Base de Datos**: MySQL/PostgreSQL con UUIDs
- **Testing**: Pest PHP
- **Documentación**: L5-Swagger (OpenAPI)

### Librerías Principales
- **spatie/laravel-permission**: Gestión de roles y permisos
- **laravel/sanctum**: Autenticación API
- **inertiajs/inertia-laravel**: SSR con React
- **tightenco/ziggy**: Rutas de Laravel en frontend
- **darkaonline/l5-swagger**: Documentación automática

## Arquitectura del Proyecto

```
app/
├── Domain/                    # 🔵 Lógica de negocio pura
│   ├── Payments/
│   │   ├── RedsysGateway.php         # Interface
│   │   └── RedsysSignature.php       # Interface
│   └── Sales/
│       ├── Contracts/                # Interfaces
│       └── Services/                 # Servicios de dominio
├── Application/               # 🟢 Casos de uso
│   ├── Payments/
│   │   └── ChargePayment.php         # Caso de uso
│   └── Sales/
│       └── ExportSalesUseCase.php    # Caso de uso
├── Infrastructure/            # 🟡 Implementaciones concretas
│   ├── Payments/
│   │   ├── RedsysGatewayImpl.php     # Implementación
│   │   └── RedsysSignatureImpl.php   # Implementación
│   └── Sales/
│       └── EloquentSalesExportRepository.php
├── Http/                     # 🔴 Capa de presentación
│   ├── Controllers/
│   ├── Requests/             # Form Requests
│   ├── Resources/            # API Resources
│   └── Middleware/
├── Models/                   # 🟠 Modelos Eloquent
├── DTOs/                     # 📦 Data Transfer Objects
├── Enums/                    # 🏷️ Enumeraciones
├── Events/                   # 📢 Eventos
├── Listeners/                # 👂 Listeners
├── Jobs/                     # ⚙️ Trabajos en cola
└── Notifications/            # 📧 Notificaciones
```

## Principios Fundamentales

### 1. Arquitectura Hexagonal
- **Dominio**: Independiente de Laravel, solo lógica de negocio
- **Aplicación**: Orquesta casos de uso
- **Infraestructura**: Implementa interfaces del dominio
- **Presentación**: Controladores delgados

### 2. Seguridad por Capas
- **Backend**: Fuente de verdad para autorización
- **Middleware**: Verificación de roles y permisos
- **Form Requests**: Validación y autorización contextual
- **Policies**: Lógica de autorización compleja

### 3. Calidad de Código
- **Tipos Estrictos**: `declare(strict_types=1);` obligatorio
- **PSR-12**: Estándar de codificación
- **Testing**: Cobertura completa con Pest PHP
- **Documentación**: APIs documentadas con Swagger

## Ejemplos de Implementación

### Modelo con UUIDs y Soft Deletes

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Sale extends Model
{
    use HasUuids, SoftDeletes;

    protected $keyType = 'string';
    public $incrementing = false;

    protected $fillable = [
        'code', 'user_id', 'plan_id'
    ];

    protected function casts(): array
    {
        return [
            'sale_date' => 'date',
            'total' => 'decimal:2',
        ];
    }
}
```

### Controlador

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api\Admin;

use App\Application\Sales\CreateSaleUseCase;
use App\Http\Controllers\Controller;
use App\Http\Requests\Api\Admin\StoreSaleRequest;
use App\Http\Resources\SaleResource;

class SaleController extends Controller
{
    public function store(StoreSaleRequest $request, CreateSaleUseCase $createSale)
    {
        // Autorización ya verificada en middleware y FormRequest
        $sale = $createSale->execute($request->validated());

        return (new SaleResource($sale))
            ->response()
            ->setStatusCode(201);
    }
}
```

### Caso de Uso con Transacción

```php
<?php

declare(strict_types=1);

namespace App\Application\Sales;

use App\Events\SaleCreated;
use App\Models\Sale;
use Illuminate\Support\Facades\DB;

class CreateSaleUseCase
{
    public function execute(array $data): Sale
    {
        return DB::transaction(function () use ($data) {
            $sale = Sale::create($data);
            
            // Crear pagos asociados
            foreach ($data['payments'] as $paymentData) {
                $sale->payments()->create($paymentData);
            }

            // Disparar evento
            SaleCreated::dispatch($sale);

            return $sale->load(['user', 'plan', 'payments']);
        });
    }
}
```

### Test con Pest PHP

```php
<?php

use App\Models\User;

it('allows admin to create sales', function () {
    $admin = User::factory()->create();
    $admin->assignRole('admin');

    $payload = [
        'user_id' => createResident()->id,
        'plan_id' => \App\Models\Plan::factory()->create()->id,
        'school_period' => '2024-2025',
        'sale_date' => now()->toDateString(),
        'total' => 1000.00,
        'payments' => [
            [
                'due_date' => now()->addMonth()->toDateString(),
                'amount' => 500.00,
                'currency' => 'EUR'
            ]
        ]
    ];

    $this->actingAs($admin)
        ->postJson('/api/admin/sales', $payload)
        ->assertStatus(201)
        ->assertJsonStructure([
            'data' => ['id', 'code', 'total', 'user', 'plan', 'payments']
        ]);
});
```

## Sistema de Roles y Permisos

El proyecto utiliza **Spatie Laravel Permission** para gestionar roles y permisos. La implementación específica de roles y permisos dependerá de los requisitos de cada proyecto.

### Nomenclatura de Permisos
Formato recomendado: `{resource}.{action}`

```php
// Ejemplos de nomenclatura (adaptar según necesidades del proyecto)
'users.view'        // Ver usuarios
'users.create'      // Crear usuarios
'users.update'      // Actualizar usuarios
'users.delete'      // Eliminar usuarios
'resource.view'     // Ver recurso
'resource.create'   // Crear recurso
'resource.update'   // Actualizar recurso
'resource.delete'   // Eliminar recurso
```

### Implementación en Rutas

```php
// Rutas protegidas por rol específico
Route::middleware(['auth:sanctum', 'role:nombre-rol'])
    ->prefix('admin')
    ->group(function () {
        Route::get('/resource', [ResourceController::class, 'index'])
            ->middleware('permission:resource.view');
    });

// Rutas protegidas por permiso (múltiples roles pueden tener el permiso)
Route::middleware(['auth:sanctum'])
    ->prefix('admin')
    ->group(function () {
        Route::get('/resource', [ResourceController::class, 'index'])
            ->middleware('permission:resource.view');
    });

// Combinación de rol y permiso
Route::middleware(['auth:sanctum', 'role:nombre-rol', 'permission:resource.view'])
    ->get('/resource', [ResourceController::class, 'index']);
```

### Mejores Prácticas
- Definir roles y permisos en seeders dedicados
- Usar permisos granulares en lugar de roles para control fino
- Verificar permisos tanto en middleware como en FormRequests/Policies
- Documentar la estructura de roles y permisos del proyecto

## Configuraciones Importantes

### Base de Datos
- **Primary Keys**: UUIDs en todas las tablas principales
- **Soft Deletes**: Implementado en todos los modelos
- **Foreign Keys**: Con constraints apropiados
- **Índices**: Optimizados para consultas comunes

### Colas y Jobs
- **Configuración**: Redis/Database para colas
- **Prioridades**: high, default, low, notifications
- **Reintentos**: Configurados por tipo de job
- **Monitoreo**: Logs detallados de fallos

### Notificaciones
- **Canales**: Mail principalmente
- **Colas**: Procesamiento asíncrono
- **Templates**: Consistentes y profesionales

## Comandos Útiles

### Desarrollo
```bash
# Servidor de desarrollo con colas
composer run dev

# Testing con coverage
php artisan test --coverage

# Generar documentación
php artisan l5-swagger:generate

# Procesar colas
php artisan queue:work --queue=high,default,low
```

### Producción
```bash
# Optimizaciones
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Migraciones
php artisan migrate --force

# Seeders
php artisan db:seed --class=RolesAndPermissionsSeeder
```

## Archivos de Reglas Específicas

Este proyecto incluye reglas detalladas en archivos separados:

1. **02-architecture.md**: Arquitectura hexagonal y separación de capas
2. **03-code-structure.md**: Estructura de código, modelos, controladores
3. **04-database.md**: Migraciones, seeders, consultas optimizadas
4. **05-laravel-features.md**: Eventos, jobs, notificaciones, colas
5. **06-api-documentation.md**: APIs RESTful y documentación Swagger
6. **07-testing.md**: Testing con Pest PHP, factories, mocks
7. **08-security-authorization.md**: Seguridad, roles y permisos

## Flujo de Desarrollo

### 1. Nueva Funcionalidad
1. Crear interface en `Domain/`
2. Implementar caso de uso en `Application/`
3. Crear implementación en `Infrastructure/`
4. Desarrollar controlador en `Http/Controllers/`
5. Escribir tests en `tests/`
6. Documentar API con Swagger

### 2. Autorización
1. Definir permisos en seeder
2. Agregar middleware a rutas
3. Implementar autorización en FormRequest
4. Crear tests de autorización

### 3. Testing
1. Unit tests para lógica de dominio
2. Feature tests para endpoints
3. Tests de autorización por rol
4. Tests de validación

## Mejores Prácticas

### ✅ Hacer
- Usar tipos estrictos en todos los archivos
- Implementar soft deletes en modelos principales
- Crear DTOs para transferencia de datos complejos
- Usar transacciones para operaciones atómicas
- Documentar todos los endpoints de API
- Escribir tests para toda funcionalidad nueva

### ❌ Evitar
- Lógica de negocio en controladores
- Consultas N+1 sin eager loading
- Hardcodear valores de configuración
- Exponer información sensible en APIs
- Omitir validación de autorización
- Crear archivos sin tipos estrictos