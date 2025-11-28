---
globs: "**/*.php"
alwaysApply: false
---

# Reglas de Estructura de Código - Backend

## Estándares Fundamentales

### Tipos Estrictos Obligatorios
**TODOS** los archivos PHP deben comenzar con:

```php
<?php

declare(strict_types=1);

namespace App\...;
```

### Beneficios de Strict Types
- Detecta errores de tipo en tiempo de ejecución
- Mejora la seguridad y confiabilidad del código
- Facilita el debugging y mantenimiento

## Modelos Eloquent

### Estructura Base con UUIDs y Soft Deletes

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Resource extends Model
{
    use HasFactory, HasUuids, SoftDeletes;

    protected $keyType = 'string';
    public $incrementing = false;

    protected $fillable = [
        // Campos según necesidades del proyecto
    ];

    protected function casts(): array
    {
        return [
            // Casts según necesidades del proyecto
        ];
    }

    /**
     * Relaciones - Usar tipos de retorno específicos
     */
    public function related(): BelongsTo
    {
        return $this->belongsTo(Related::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(Item::class);
    }

    /**
     * Scopes simples - Lógica compleja va en servicios
     */
    public function scopeActive($query)
    {
        return $query->where('status', 'active');
    }
}
```

### Modelo User con Autorización

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Permission\Traits\HasRoles;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasUuids, HasRoles, SoftDeletes, HasApiTokens;

    protected $keyType = 'string';
    public $incrementing = false;

    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
        ];
    }
}
```

## Controladores (Thin Controllers)

### Estructura Base

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\Api\StoreResourceRequest;
use App\Http\Resources\ResourceResource;
use App\Application\CreateResourceUseCase;
use App\Models\Resource;
use Illuminate\Http\Request;

class ResourceController extends Controller
{
    public function index(Request $request)
    {
        $query = Resource::with(['related']);

        // Filtros simples
        if ($request->filled('search')) {
            $query->where('name', 'like', "%{$request->input('search')}%");
        }

        return ResourceResource::collection($query->paginate(15));
    }

    public function store(StoreResourceRequest $request, CreateResourceUseCase $createResource)
    {
        $resource = $createResource->execute($request->validated());

        return (new ResourceResource($resource))
            ->response()
            ->setStatusCode(201);
    }

    public function show(string $id)
    {
        $resource = Resource::with(['related'])->findOrFail($id);
        
        return new ResourceResource($resource);
    }

    public function destroy(string $id)
    {
        $resource = Resource::findOrFail($id);
        $resource->delete();

        return response()->noContent();
    }
}
```

### Principios para Controladores

1. **Thin Controllers**: Solo recibir, validar, delegar y responder
2. **Autorización Temprana**: Verificar permisos al inicio
3. **Delegación**: Usar casos de uso para lógica compleja
4. **Respuestas Consistentes**: Usar API Resources

## Form Requests

### Validación y Autorización

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
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:resources,email'],
            // Más reglas según necesidades del proyecto
        ];
    }

    public function messages(): array
    {
        return [
            // Mensajes personalizados según necesidades
        ];
    }
}
```

## DTOs (Data Transfer Objects)

### Estructura con Readonly Properties

```php
<?php

declare(strict_types=1);

namespace App\DTOs;

class ResultDTO
{
    public function __construct(
        public readonly bool $success,
        public readonly ?string $message = null,
        public readonly ?array $data = null,
    ) {}

    public static function success(?array $data = null): self
    {
        return new self(success: true, data: $data);
    }

    public static function failure(string $message): self
    {
        return new self(success: false, message: $message);
    }
}
```

## API Resources

### Resource Base

```php
<?php

declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class ResourceResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            
            // Relaciones condicionales
            'related' => new RelatedResource($this->whenLoaded('related')),
            
            // Contadores
            'items_count' => $this->whenCounted('items'),
            
            // Timestamps
            'created_at' => $this->created_at?->toISOString(),
            'updated_at' => $this->updated_at?->toISOString(),
        ];
    }
}
```

## Enums (PHP 8.1+)

### Enum para Estados

```php
<?php

declare(strict_types=1);

namespace App\Enums;

enum Status: string
{
    case Pending = 'pending';
    case Active = 'active';
    case Inactive = 'inactive';

    public function label(): string
    {
        return match($this) {
            self::Pending => 'Pendiente',
            self::Active => 'Activo',
            self::Inactive => 'Inactivo',
        };
    }

    public static function values(): array
    {
        return array_column(self::cases(), 'value');
    }
}
```

## Mejores Prácticas

### 1. Naming Conventions

```php
// ✅ BIEN: Nombres descriptivos
class CreateResourceUseCase {}
class ResourceExportService {}

// ❌ MAL: Nombres genéricos
class ResourceService {}
class Helper {}
```

### 2. Type Hints Completos

```php
// ✅ BIEN: Tipos específicos
public function process(Resource $resource): ResultDTO
{
    // ...
}

// ❌ MAL: Sin tipos
public function process($resource)
{
    // ...
}
```

### 3. Immutabilidad cuando sea posible

```php
// ✅ BIEN: DTOs inmutables
class ResourceData
{
    public function __construct(
        public readonly string $name,
        public readonly string $email,
    ) {}
}
```

### 4. Manejo de Errores Consistente

```php
// ✅ BIEN: Excepciones específicas
if (!$user->can('resource.create')) {
    throw new AccessDeniedHttpException('Insufficient permissions');
}
```
