---
globs: "**/*.php"
alwaysApply: false
---

# Reglas de Testing - Backend

## Persona
Eres un experto en testing que implementa **pruebas robustas y mantenibles** usando **Pest PHP** para garantizar la calidad, seguridad y confiabilidad del código.

## Configuración de Pest PHP

### Configuración Base (tests/Pest.php)

```php
<?php

/*
|--------------------------------------------------------------------------
| Test Case
|--------------------------------------------------------------------------
*/

pest()->extend(Tests\TestCase::class)
    ->use(Illuminate\Foundation\Testing\RefreshDatabase::class)
    ->in('Feature');

/*
|--------------------------------------------------------------------------
| Expectations
|--------------------------------------------------------------------------
*/

expect()->extend('toBeOne', function () {
    return $this->toBe(1);
});

expect()->extend('toHaveValidUuid', function () {
    return $this->toMatch('/^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i');
});

expect()->extend('toBeValidEmail', function () {
    return $this->toMatch('/^[^\s@]+@[^\s@]+\.[^\s@]+$/');
});

/*
|--------------------------------------------------------------------------
| Functions
|--------------------------------------------------------------------------
*/

/**
 * Crear usuario admin para tests
 */
function createAdmin(): \App\Models\User
{
    $admin = \App\Models\User::factory()->create();
    $admin->assignRole('admin');
    return $admin;
}

/**
 * Crear usuario manager para tests
 */
function createManager(): \App\Models\User
{
    $manager = \App\Models\User::factory()->create();
    $manager->assignRole('manager');
    return $manager;
}

/**
 * Crear usuario resident para tests
 */
function createResident(): \App\Models\User
{
    $resident = \App\Models\User::factory()->create();
    $resident->assignRole('resident');
    return $resident;
}
```

### TestCase Base

```php
<?php

declare(strict_types=1);

namespace Tests;

use Database\Seeders\RolesAndPermissionsSeeder;
use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    use CreatesApplication;

    protected function setUp(): void
    {
        parent::setUp();
        
        // Ejecutar seeder de roles y permisos en cada test
        $this->seed(RolesAndPermissionsSeeder::class);
        
        // Configurar entorno de testing
        config(['mail.default' => 'array']);
        config(['queue.default' => 'sync']);
    }
}
```

## Feature Tests (Pruebas de Integración)

### Tests de Autorización

```php
<?php

declare(strict_types=1);

use App\Models\User;
use Database\Seeders\RolesAndPermissionsSeeder;

uses(Illuminate\Foundation\Testing\RefreshDatabase::class);

beforeEach(function () {
    $this->seed(RolesAndPermissionsSeeder::class);
});

describe('User Management Authorization', function () {
    
    it('allows admin to view users', function () {
        $admin = createAdmin();

        $this->actingAs($admin)
            ->getJson('/api/admin/users')
            ->assertOk()
            ->assertJsonStructure([
                'data' => [
                    '*' => ['id', 'name', 'email', 'roles']
                ],
                'links',
                'meta'
            ]);
    });

    it('prevents manager from viewing users', function () {
        $manager = createManager();

        $this->actingAs($manager)
            ->getJson('/api/admin/users')
            ->assertStatus(403);
    });

    it('prevents resident from viewing users', function () {
        $resident = createResident();

        $this->actingAs($resident)
            ->getJson('/api/admin/users')
            ->assertStatus(403);
    });

    it('requires authentication for user endpoints', function () {
        $this->getJson('/api/admin/users')
            ->assertStatus(401);
    });
});

describe('Sales Management Authorization', function () {
    
    it('allows admin to manage sales', function () {
        $admin = createAdmin();

        // Ver ventas
        $this->actingAs($admin)
            ->getJson('/api/admin/sales')
            ->assertOk();

        // Crear venta
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
                ],
                [
                    'due_date' => now()->addMonths(2)->toDateString(),
                    'amount' => 500.00,
                    'currency' => 'EUR'
                ]
            ]
        ];

        $this->actingAs($admin)
            ->postJson('/api/admin/sales', $payload)
            ->assertStatus(201);
    });

    it('allows manager to manage sales', function () {
        $manager = createManager();

        $this->actingAs($manager)
            ->getJson('/api/admin/sales')
            ->assertOk();
    });

    it('prevents resident from managing sales', function () {
        $resident = createResident();

        $this->actingAs($resident)
            ->getJson('/api/admin/sales')
            ->assertStatus(403);
    });
});
```

### Tests de Endpoints CRUD

```php
<?php

declare(strict_types=1);

use App\Models\User;
use App\Models\Plan;

describe('Plans CRUD Operations', function () {
    
    it('creates a plan successfully', function () {
        $admin = createAdmin();
        
        $payload = [
            'name' => 'Plan Premium',
            'price' => 299.99,
            'description' => 'Plan premium con todas las características',
            'color' => '#3B82F6'
        ];

        $response = $this->actingAs($admin)
            ->postJson('/api/admin/plans', $payload)
            ->assertStatus(201)
            ->assertJsonStructure([
                'data' => ['id', 'name', 'price', 'description', 'color']
            ]);

        $planId = $response->json('data.id');
        
        expect($planId)->toHaveValidUuid();
        
        $this->assertDatabaseHas('plans', [
            'id' => $planId,
            'name' => 'Plan Premium',
            'price' => 299.99
        ]);
    });

    it('validates required fields when creating plan', function () {
        $admin = createAdmin();

        $this->actingAs($admin)
            ->postJson('/api/admin/plans', [])
            ->assertStatus(422)
            ->assertJsonValidationErrors(['name', 'price']);
    });

    it('updates a plan successfully', function () {
        $admin = createAdmin();
        $plan = Plan::factory()->create();

        $payload = [
            'name' => 'Plan Updated',
            'price' => 399.99,
            'description' => 'Updated description'
        ];

        $this->actingAs($admin)
            ->putJson("/api/admin/plans/{$plan->id}", $payload)
            ->assertOk()
            ->assertJson([
                'data' => [
                    'id' => $plan->id,
                    'name' => 'Plan Updated',
                    'price' => 399.99
                ]
            ]);

        $this->assertDatabaseHas('plans', [
            'id' => $plan->id,
            'name' => 'Plan Updated',
            'price' => 399.99
        ]);
    });

    it('soft deletes a plan', function () {
        $admin = createAdmin();
        $plan = Plan::factory()->create();

        $this->actingAs($admin)
            ->deleteJson("/api/admin/plans/{$plan->id}")
            ->assertStatus(204);

        $this->assertSoftDeleted('plans', ['id' => $plan->id]);
    });

    it('prevents deleting plan with associated sales', function () {
        $admin = createAdmin();
        $plan = Plan::factory()->create();
        
        // Crear venta asociada
        \App\Models\Sale::factory()->create(['plan_id' => $plan->id]);

        $this->actingAs($admin)
            ->deleteJson("/api/admin/plans/{$plan->id}")
            ->assertStatus(422)
            ->assertJson([
                'message' => 'Cannot delete plan with associated sales'
            ]);

        $this->assertDatabaseHas('plans', ['id' => $plan->id]);
    });
});
```

### Tests de Validación

```php
<?php

declare(strict_types=1);

describe('Form Request Validation', function () {
    
    it('validates sale creation request', function () {
        $admin = createAdmin();

        // Test campos requeridos
        $this->actingAs($admin)
            ->postJson('/api/admin/sales', [])
            ->assertStatus(422)
            ->assertJsonValidationErrors([
                'user_id',
                'plan_id', 
                'school_period',
                'sale_date',
                'total',
                'payments'
            ]);
    });

    it('validates user_id exists and is not deleted', function () {
        $admin = createAdmin();
        $deletedUser = createResident();
        $deletedUser->delete();

        $payload = [
            'user_id' => $deletedUser->id,
            'plan_id' => \App\Models\Plan::factory()->create()->id,
            'school_period' => '2024-2025',
            'sale_date' => now()->toDateString(),
            'total' => 1000.00,
            'payments' => [
                [
                    'due_date' => now()->addMonth()->toDateString(),
                    'amount' => 1000.00,
                    'currency' => 'EUR'
                ]
            ]
        ];

        $this->actingAs($admin)
            ->postJson('/api/admin/sales', $payload)
            ->assertStatus(422)
            ->assertJsonValidationErrors(['user_id']);
    });

    it('validates school period format', function () {
        $admin = createAdmin();

        $payload = [
            'user_id' => createResident()->id,
            'plan_id' => \App\Models\Plan::factory()->create()->id,
            'school_period' => 'invalid-format',
            'sale_date' => now()->toDateString(),
            'total' => 1000.00,
            'payments' => [
                [
                    'due_date' => now()->addMonth()->toDateString(),
                    'amount' => 1000.00,
                    'currency' => 'EUR'
                ]
            ]
        ];

        $this->actingAs($admin)
            ->postJson('/api/admin/sales', $payload)
            ->assertStatus(422)
            ->assertJsonValidationErrors(['school_period']);
    });

    it('validates payments array structure', function () {
        $admin = createAdmin();

        $payload = [
            'user_id' => createResident()->id,
            'plan_id' => \App\Models\Plan::factory()->create()->id,
            'school_period' => '2024-2025',
            'sale_date' => now()->toDateString(),
            'total' => 1000.00,
            'payments' => [] // Array vacío
        ];

        $this->actingAs($admin)
            ->postJson('/api/admin/sales', $payload)
            ->assertStatus(422)
            ->assertJsonValidationErrors(['payments']);
    });
});
```

## Unit Tests (Pruebas Unitarias)

### Tests de Modelos

```php
<?php

declare(strict_types=1);

use App\Models\User;
use App\Models\Sale;
use App\Models\Payment;
use App\Enums\PaymentStatus;

describe('User Model', function () {
    
    it('uses UUIDs as primary key', function () {
        $user = User::factory()->create();
        
        expect($user->id)->toHaveValidUuid();
        expect($user->getKeyType())->toBe('string');
        expect($user->getIncrementing())->toBeFalse();
    });

    it('hashes password automatically', function () {
        $user = User::factory()->create(['password' => 'plaintext']);
        
        expect($user->password)->not->toBe('plaintext');
        expect(Hash::check('plaintext', $user->password))->toBeTrue();
    });

    it('has sales relationship', function () {
        $user = User::factory()->create();
        $sale = Sale::factory()->create(['user_id' => $user->id]);
        
        expect($user->sales)->toHaveCount(1);
        expect($user->sales->first()->id)->toBe($sale->id);
    });

    it('soft deletes with email prefix', function () {
        $user = User::factory()->create(['email' => 'test@example.com']);
        $originalEmail = $user->email;
        
        $user->delete();
        
        expect($user->trashed())->toBeTrue();
        expect($user->email)->toBe('REMOVE_' . $originalEmail);
    });

    it('assigns default role on creation', function () {
        $user = User::factory()->create();
        
        expect($user->hasRole('resident'))->toBeTrue();
    });
});

describe('Sale Model', function () {
    
    it('generates unique sale code', function () {
        $sale1 = Sale::factory()->create();
        $sale2 = Sale::factory()->create();
        
        expect($sale1->code)->not->toBe($sale2->code);
        expect($sale1->code)->toMatch('/^SALE-\d{4}-\d{6}$/');
    });

    it('has payments relationship', function () {
        $sale = Sale::factory()->create();
        $payment = Payment::factory()->create(['sale_id' => $sale->id]);
        
        expect($sale->payments)->toHaveCount(1);
        expect($sale->payments->first()->id)->toBe($payment->id);
    });

    it('calculates total paid amount', function () {
        $sale = Sale::factory()->create();
        
        Payment::factory()->create([
            'sale_id' => $sale->id,
            'amount' => 500.00,
            'status' => PaymentStatus::Paid
        ]);
        
        Payment::factory()->create([
            'sale_id' => $sale->id,
            'amount' => 300.00,
            'status' => PaymentStatus::Pending
        ]);
        
        $totalPaid = $sale->payments()->where('status', PaymentStatus::Paid)->sum('amount');
        
        expect($totalPaid)->toBe(500.00);
    });
});
```

### Tests de DTOs

```php
<?php

declare(strict_types=1);

use App\DTOs\Payments\ChargeResult;

describe('ChargeResult DTO', function () {
    
    it('creates successful result', function () {
        $result = ChargeResult::success('payment_123', ['order_id' => 'order_456']);
        
        expect($result->success)->toBeTrue();
        expect($result->externalPaymentId)->toBe('payment_123');
        expect($result->errorMessage)->toBeNull();
        expect($result->metadata)->toBe(['order_id' => 'order_456']);
    });

    it('creates failure result', function () {
        $result = ChargeResult::failure('Payment declined', ['code' => 'DECLINED']);
        
        expect($result->success)->toBeFalse();
        expect($result->externalPaymentId)->toBeNull();
        expect($result->errorMessage)->toBe('Payment declined');
        expect($result->metadata)->toBe(['code' => 'DECLINED']);
    });

    it('converts to array', function () {
        $result = ChargeResult::success('payment_123');
        $array = $result->toArray();
        
        expect($array)->toBe([
            'success' => true,
            'external_payment_id' => 'payment_123',
            'error_message' => null,
            'metadata' => null,
        ]);
    });

    it('is immutable', function () {
        $result = new ChargeResult(true, 'payment_123');
        
        // Las propiedades readonly no se pueden modificar
        expect(fn() => $result->success = false)
            ->toThrow(Error::class);
    });
});
```

### Tests de Servicios

```php
<?php

declare(strict_types=1);

use App\Domain\Sales\Services\SalesFilterService;
use App\DTOs\Sales\SalesFilterDTO;
use App\Models\Sale;
use App\Models\User;
use App\Models\Plan;

describe('SalesFilterService', function () {
    
    it('filters sales by search term', function () {
        $service = new SalesFilterService();
        
        $user = User::factory()->create(['name' => 'John Doe']);
        $sale1 = Sale::factory()->create(['user_id' => $user->id, 'code' => 'SALE-2024-000001']);
        $sale2 = Sale::factory()->create(['code' => 'SALE-2024-000002']);
        
        $filters = new SalesFilterDTO(search: 'John');
        $query = Sale::query();
        
        $result = $service->apply($query, $filters);
        
        expect($result->get())->toHaveCount(1);
        expect($result->first()->id)->toBe($sale1->id);
    });

    it('filters sales by date range', function () {
        $service = new SalesFilterService();
        
        $sale1 = Sale::factory()->create(['sale_date' => '2024-01-15']);
        $sale2 = Sale::factory()->create(['sale_date' => '2024-02-15']);
        $sale3 = Sale::factory()->create(['sale_date' => '2024-03-15']);
        
        $filters = new SalesFilterDTO(
            dateFrom: Carbon::parse('2024-01-01'),
            dateTo: Carbon::parse('2024-02-28')
        );
        
        $query = Sale::query();
        $result = $service->apply($query, $filters);
        
        expect($result->get())->toHaveCount(2);
    });

    it('filters sales by school period', function () {
        $service = new SalesFilterService();
        
        $sale1 = Sale::factory()->create(['school_period' => '2023-2024']);
        $sale2 = Sale::factory()->create(['school_period' => '2024-2025']);
        
        $filters = new SalesFilterDTO(period: '2024-2025');
        $query = Sale::query();
        
        $result = $service->apply($query, $filters);
        
        expect($result->get())->toHaveCount(1);
        expect($result->first()->id)->toBe($sale2->id);
    });
});
```

## Tests de Jobs y Eventos

### Tests de Jobs

```php
<?php

declare(strict_types=1);

use App\Jobs\ProcessDuePayments;
use App\Application\Payments\ChargePayment;
use App\Models\Payment;
use App\Enums\PaymentStatus;
use Illuminate\Support\Facades\Queue;

describe('ProcessDuePayments Job', function () {
    
    it('processes due payments', function () {
        // Crear pagos vencidos
        $duePayment = Payment::factory()->create([
            'status' => PaymentStatus::Pending,
            'due_date' => now()->subDay()
        ]);
        
        $futurePayment = Payment::factory()->create([
            'status' => PaymentStatus::Pending,
            'due_date' => now()->addDay()
        ]);
        
        // Mock del servicio de cobro
        $chargePayment = Mockery::mock(ChargePayment::class);
        $chargePayment->shouldReceive('handle')
            ->once()
            ->with(Mockery::on(fn($payment) => $payment->id === $duePayment->id));
        
        $this->app->instance(ChargePayment::class, $chargePayment);
        
        // Ejecutar job
        $job = new ProcessDuePayments();
        $job->handle($chargePayment);
    });

    it('handles job failures gracefully', function () {
        $payment = Payment::factory()->create([
            'status' => PaymentStatus::Pending,
            'due_date' => now()->subDay()
        ]);
        
        $chargePayment = Mockery::mock(ChargePayment::class);
        $chargePayment->shouldReceive('handle')
            ->andThrow(new Exception('Payment gateway error'));
        
        $this->app->instance(ChargePayment::class, $chargePayment);
        
        $job = new ProcessDuePayments();
        
        // El job no debe fallar completamente por un pago individual
        expect(fn() => $job->handle($chargePayment))->not->toThrow();
    });

    it('can be queued', function () {
        Queue::fake();
        
        ProcessDuePayments::dispatch();
        
        Queue::assertPushed(ProcessDuePayments::class);
    });
});
```

### Tests de Eventos

```php
<?php

declare(strict_types=1);

use App\Events\SaleCreated;
use App\Listeners\SendSaleNotifications;
use App\Models\Sale;
use App\Notifications\SaleCreated as SaleCreatedNotification;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Notification;

describe('Sale Events', function () {
    
    it('dispatches SaleCreated event when sale is created', function () {
        Event::fake();
        
        $sale = Sale::factory()->create();
        
        SaleCreated::dispatch($sale);
        
        Event::assertDispatched(SaleCreated::class, function ($event) use ($sale) {
            return $event->sale->id === $sale->id;
        });
    });

    it('sends notifications when SaleCreated event is handled', function () {
        Notification::fake();
        
        $sale = Sale::factory()->create();
        $event = new SaleCreated($sale);
        
        $listener = new SendSaleNotifications();
        $listener->handle($event);
        
        // Verificar notificación al cliente
        Notification::assertSentTo(
            $sale->user,
            SaleCreatedNotification::class,
            function ($notification) use ($sale) {
                return $notification->sale->id === $sale->id;
            }
        );
    });
});
```

## Tests de API Resources

```php
<?php

declare(strict_types=1);

use App\Http\Resources\SaleResource;
use App\Models\Sale;
use App\Models\User;
use App\Models\Plan;

describe('SaleResource', function () {
    
    it('transforms sale data correctly', function () {
        $user = User::factory()->create();
        $plan = Plan::factory()->create();
        $sale = Sale::factory()->create([
            'user_id' => $user->id,
            'plan_id' => $plan->id,
            'code' => 'SALE-2024-000001',
            'total' => 1000.50
        ]);
        
        $resource = new SaleResource($sale->load(['user', 'plan']));
        $array = $resource->toArray(request());
        
        expect($array)->toHaveKeys([
            'id', 'code', 'school_period', 'sale_date', 'total',
            'user', 'plan', 'created_at', 'updated_at'
        ]);
        
        expect($array['id'])->toBe($sale->id);
        expect($array['code'])->toBe('SALE-2024-000001');
        expect($array['total'])->toBe('1000.50'); // String para precisión
    });

    it('includes related data when loaded', function () {
        $sale = Sale::factory()->create();
        $sale->load(['user', 'plan', 'payments']);
        
        $resource = new SaleResource($sale);
        $array = $resource->toArray(request());
        
        expect($array['user'])->toBeArray();
        expect($array['plan'])->toBeArray();
        expect($array['payments'])->toBeArray();
    });

    it('excludes related data when not loaded', function () {
        $sale = Sale::factory()->create();
        
        $resource = new SaleResource($sale);
        $array = $resource->toArray(request());
        
        expect($array)->not->toHaveKey('user');
        expect($array)->not->toHaveKey('plan');
        expect($array)->not->toHaveKey('payments');
    });
});
```

## Tests de Performance

```php
<?php

declare(strict_types=1);

describe('Performance Tests', function () {
    
    it('avoids N+1 queries when loading sales with relationships', function () {
        // Crear datos de prueba
        $users = User::factory()->count(10)->create();
        $plans = Plan::factory()->count(5)->create();
        
        foreach ($users as $user) {
            Sale::factory()->create([
                'user_id' => $user->id,
                'plan_id' => $plans->random()->id
            ]);
        }
        
        // Contar queries
        $queryCount = 0;
        DB::listen(function () use (&$queryCount) {
            $queryCount++;
        });
        
        // Cargar ventas con relaciones
        $sales = Sale::with(['user', 'plan'])->get();
        
        // Debe ser máximo 3 queries: sales, users, plans
        expect($queryCount)->toBeLessThanOrEqual(3);
        expect($sales)->toHaveCount(10);
    });

    it('paginates large datasets efficiently', function () {
        Sale::factory()->count(100)->create();
        
        $admin = createAdmin();
        
        $response = $this->actingAs($admin)
            ->getJson('/api/admin/sales?per_page=15')
            ->assertOk();
        
        $data = $response->json();
        
        expect($data['data'])->toHaveCount(15);
        expect($data['meta']['total'])->toBe(100);
        expect($data['meta']['per_page'])->toBe(15);
    });
});
```

## Helpers de Testing

### Factory States

```php
<?php

declare(strict_types=1);

namespace Database\Factories;

use App\Models\Sale;
use App\Models\User;
use App\Models\Plan;
use Illuminate\Database\Eloquent\Factories\Factory;

class SaleFactory extends Factory
{
    protected $model = Sale::class;

    public function definition(): array
    {
        return [
            'code' => 'SALE-' . now()->year . '-' . str_pad((string) fake()->unique()->numberBetween(1, 999999), 6, '0', STR_PAD_LEFT),
            'user_id' => User::factory(),
            'plan_id' => Plan::factory(),
            'school_period' => fake()->randomElement(['2023-2024', '2024-2025', '2025-2026']),
            'sale_date' => fake()->dateTimeBetween('-1 year', 'now'),
            'total' => fake()->randomFloat(2, 100, 5000),
        ];
    }

    /**
     * Sale with payments
     */
    public function withPayments(int $count = 3): static
    {
        return $this->afterCreating(function (Sale $sale) use ($count) {
            \App\Models\Payment::factory()
                ->count($count)
                ->create(['sale_id' => $sale->id]);
        });
    }

    /**
     * Sale from current year
     */
    public function currentYear(): static
    {
        return $this->state(fn (array $attributes) => [
            'sale_date' => fake()->dateTimeBetween(now()->startOfYear(), now()),
            'school_period' => now()->year . '-' . (now()->year + 1),
        ]);
    }

    /**
     * High value sale
     */
    public function highValue(): static
    {
        return $this->state(fn (array $attributes) => [
            'total' => fake()->randomFloat(2, 5000, 20000),
        ]);
    }
}
```

### Custom Assertions

```php
<?php

// En tests/TestCase.php

protected function assertValidApiResponse($response, $structure = null): void
{
    $response->assertOk();
    
    if ($structure) {
        $response->assertJsonStructure($structure);
    }
    
    // Verificar headers de API
    $response->assertHeader('Content-Type', 'application/json');
}

protected function assertValidationError($response, $fields): void
{
    $response->assertStatus(422);
    $response->assertJsonStructure([
        'message',
        'errors' => $fields
    ]);
}

protected function assertUnauthorized($response): void
{
    $response->assertStatus(403);
    $response->assertJson([
        'message' => 'This action is unauthorized.'
    ]);
}
```

## Comandos de Testing

### Ejecutar Tests

```bash
# Todos los tests
php artisan test

# Tests específicos
php artisan test --filter=SaleTest
php artisan test tests/Feature/Api/Admin/SalesTest.php

# Con coverage
php artisan test --coverage
php artisan test --coverage-html=coverage

# Tests en paralelo
php artisan test --parallel

# Tests con profiling
php artisan test --profile
```

### Configuración de CI/CD

```yaml
# .github/workflows/tests.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ALLOW_EMPTY_PASSWORD: yes
          MYSQL_DATABASE: testing
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
      - uses: actions/checkout@v3
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: 8.2
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, mysql, pdo_mysql
          
      - name: Install dependencies
        run: composer install --prefer-dist --no-progress
        
      - name: Copy environment file
        run: cp .env.example .env.testing
        
      - name: Generate application key
        run: php artisan key:generate --env=testing
        
      - name: Run tests
        run: php artisan test --coverage --min=80
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
```

Esta configuración de testing garantiza **cobertura completa, calidad del código y confiabilidad** de la aplicación.
