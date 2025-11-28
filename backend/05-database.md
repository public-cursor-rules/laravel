---
globs: "**/*.php"
alwaysApply: false
---

# Reglas de Base de Datos - Backend

## Principios Fundamentales

### UUIDs como Claves Primarias
- **Ventajas**: Únicos globalmente, no secuenciales, seguros
- **Desventajas**: Ligeramente más lentos, mayor espacio
- **Decisión**: Los beneficios de seguridad superan las desventajas

### Soft Deletes Obligatorios
- **Razón**: Preservar integridad referencial y auditoría
- **Implementación**: Todos los modelos principales usan `SoftDeletes`
- **Excepción**: Solo tablas pivot y logs pueden usar hard delete

### Transacciones para Atomicidad
- **Uso**: Operaciones que involucran múltiples tablas
- **Implementación**: `DB::transaction()` en casos de uso complejos

## Migraciones

### Estructura Base con UUIDs

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('table_name', function (Blueprint $table) {
            // Primary key UUID
            $table->uuid('id')->primary();
            
            // Foreign keys como UUIDs
            $table->uuid('foreign_key_id');
            
            // Campos según necesidades del proyecto
            $table->string('name');
            $table->timestamps();
            
            // Soft deletes
            $table->softDeletes();
            
            // Índices para performance
            $table->index('foreign_key_id');
            
            // Foreign key constraints
            $table->foreign('foreign_key_id')
                  ->references('id')
                  ->on('related_table')
                  ->onDelete('restrict');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('table_name');
    }
};
```

## Seeders

### Seeder Base

```php
<?php

declare(strict_types=1);

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // Orden importante: dependencias primero
        $this->call([
            RolesAndPermissionsSeeder::class,
            // Otros seeders según necesidades del proyecto
        ]);
    }
}
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
        // El guard debe ser consistente con la configuración de autenticación del proyecto
        $guard = config('auth.defaults.guard', 'web');

        // Definir permisos según necesidades del proyecto
        $permissions = $this->getPermissions();

        // Crear permisos
        foreach ($permissions as $permissionName) {
            Permission::firstOrCreate([
                'name' => $permissionName,
                'guard_name' => $guard,
            ]);
        }

        // Crear y configurar roles según necesidades del proyecto
        $this->createRoles($guard);
    }

    private function getPermissions(): array
    {
        return [
            // Definir permisos según requisitos del proyecto
            // Formato recomendado: {resource}.{action}
        ];
    }

    private function createRoles(string $guard): void
    {
        // Crear roles según requisitos del proyecto
    }
}
```

## Factories

### Factory Base con UUIDs

```php
<?php

declare(strict_types=1);

namespace Database\Factories;

use App\Models\ModelName;
use Illuminate\Database\Eloquent\Factories\Factory;

class ModelNameFactory extends Factory
{
    protected $model = ModelName::class;

    public function definition(): array
    {
        return [
            // Definir campos según necesidades del proyecto
        ];
    }
}
```

## Consultas Optimizadas

### Eager Loading para Evitar N+1

```php
// ❌ MAL: N+1 queries
$models = Model::all();
foreach ($models as $model) {
    echo $model->relation->field; // Query por cada modelo
}

// ✅ BIEN: Eager loading
$models = Model::with(['relation'])->get();
foreach ($models as $model) {
    echo $model->relation->field; // Sin queries adicionales
}
```

## Transacciones

### Uso de Transacciones

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\DB;

class UseCase
{
    public function execute(array $data)
    {
        return DB::transaction(function () use ($data) {
            // Operaciones atómicas
            // Si alguna falla, se revierte todo
        });
    }
}
```

## Índices y Performance

### Índices en Migraciones

```php
// Índices simples
$table->index('column_name');

// Índices compuestos
$table->index(['column1', 'column2']);

// Índices únicos
$table->unique('column_name');
```

## Mejores Prácticas

### Validación a Nivel de Base de Datos

```php
// En migraciones
$table->decimal('amount', 10, 2)->unsigned(); // No negativos
$table->string('email')->unique(); // Unicidad
$table->foreign('foreign_key_id')
      ->references('id')
      ->on('related_table')
      ->onDelete('restrict'); // Constraints de integridad
```
