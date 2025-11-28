---
globs: "**/*.php"
alwaysApply: false
---

# Reglas de API y Documentación - Backend

## Principios de Diseño de API

### RESTful Design
- **Recursos**: Sustantivos en plural (`/resources`, `/items`)
- **Verbos HTTP**: GET (leer), POST (crear), PUT (actualizar), DELETE (eliminar)
- **Códigos de Estado**: Usar códigos HTTP apropiados
- **Consistencia**: Estructura uniforme en todas las respuestas

### Versionado
- **URL**: `/api/v1/resources` (preferido para APIs públicas)
- **Header**: `Accept: application/vnd.api+json;version=1` (alternativo)

## Documentación con L5-Swagger

### Configuración Base

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

/**
 * @OA\Info(
 *   version="1.0.0",
 *   title="API",
 *   description="API del sistema"
 * )
 * 
 * @OA\Server(
 *   url="/api",
 *   description="Servidor principal de API"
 * )
 * 
 * @OA\SecurityScheme(
 *   securityScheme="sanctum",
 *   type="http",
 *   scheme="bearer",
 *   bearerFormat="JWT"
 * )
 */
class AppServiceProvider extends ServiceProvider
{
    // ...
}
```

### Documentación de Controlador

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Resources\ResourceResource;
use Illuminate\Http\Request;

/**
 * @OA\Tag(
 *   name="Resources",
 *   description="Gestión de recursos"
 * )
 */
class ResourceController extends Controller
{
    /**
     * @OA\Get(
     *   path="/api/resources",
     *   summary="Listar recursos",
     *   tags={"Resources"},
     *   security={{"sanctum": {}}},
     *   @OA\Parameter(
     *     name="page",
     *     in="query",
     *     @OA\Schema(type="integer", default=1)
     *   ),
     *   @OA\Response(
     *     response=200,
     *     description="Lista de recursos",
     *     @OA\JsonContent(
     *       @OA\Property(property="data", type="array", @OA\Items(ref="#/components/schemas/Resource")),
     *       @OA\Property(property="links", type="object"),
     *       @OA\Property(property="meta", type="object")
     *     )
     *   )
     * )
     */
    public function index(Request $request)
    {
        // Implementación
    }

    /**
     * @OA\Post(
     *   path="/api/resources",
     *   summary="Crear recurso",
     *   tags={"Resources"},
     *   security={{"sanctum": {}}},
     *   @OA\RequestBody(
     *     required=true,
     *     @OA\JsonContent(ref="#/components/schemas/ResourceRequest")
     *   ),
     *   @OA\Response(
     *     response=201,
     *     description="Recurso creado",
     *     @OA\JsonContent(@OA\Property(property="data", ref="#/components/schemas/Resource"))
     *   ),
     *   @OA\Response(response=422, description="Errores de validación")
     * )
     */
    public function store(Request $request)
    {
        // Implementación
    }
}
```

### Schema Genérico

```php
/**
 * @OA\Schema(
 *   schema="Resource",
 *   type="object",
 *   required={"id", "name"},
 *   @OA\Property(property="id", type="string", format="uuid"),
 *   @OA\Property(property="name", type="string"),
 *   @OA\Property(property="created_at", type="string", format="date-time"),
 *   @OA\Property(property="updated_at", type="string", format="date-time")
 * )
 */
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
            'relation' => new RelationResource($this->whenLoaded('relation')),
            // Contadores
            'items_count' => $this->whenCounted('items'),
            // Timestamps
            'created_at' => $this->created_at?->toISOString(),
            'updated_at' => $this->updated_at?->toISOString(),
        ];
    }
}
```

## Manejo de Errores

### Exception Handler

```php
<?php

declare(strict_types=1);

namespace App\Exceptions;

use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;
use Illuminate\Http\JsonResponse;
use Illuminate\Validation\ValidationException;
use Throwable;

class Handler extends ExceptionHandler
{
    public function render($request, Throwable $e): JsonResponse
    {
        if ($request->is('api/*')) {
            return $this->handleApiException($request, $e);
        }

        return parent::render($request, $e);
    }

    private function handleApiException($request, Throwable $e): JsonResponse
    {
        $status = 500;
        $message = 'Internal Server Error';
        $errors = null;

        if ($e instanceof ValidationException) {
            $status = 422;
            $message = 'The given data was invalid.';
            $errors = $e->errors();
        } elseif ($e instanceof \Illuminate\Auth\AuthenticationException) {
            $status = 401;
            $message = 'Unauthenticated.';
        }

        $response = ['message' => $message, 'status' => $status];

        if ($errors) {
            $response['errors'] = $errors;
        }

        return response()->json($response, $status);
    }
}
```

## Comandos de Documentación

```bash
# Generar documentación Swagger
php artisan l5-swagger:generate

# Publicar configuración
php artisan vendor:publish --provider="L5Swagger\L5SwaggerServiceProvider"
```
