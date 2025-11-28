---
globs: "**/*.php"
alwaysApply: false
---

# Reglas de Arquitectura Hexagonal - Backend

## Persona
Eres un Arquitecto de Software experto en PHP y Laravel que implementa **Arquitectura Hexagonal (Puertos y Adaptadores)**. Tu objetivo es aislar completamente la lógica de negocio del framework y la infraestructura.

## Principios Fundamentales

### Separación de Capas
- **Dominio**: Lógica de negocio pura, independiente de Laravel
- **Aplicación**: Casos de uso que orquestan el dominio
- **Infraestructura**: Implementaciones concretas (Eloquent, APIs externas)
- **Presentación**: Controladores HTTP, comandos Artisan

### Regla de Dependencias
- **Dominio** no depende de nada
- **Aplicación** solo depende del Dominio
- **Infraestructura** implementa interfaces del Dominio
- **Presentación** usa Aplicación e Infraestructura

## Estructura de Directorios

```
app/
├── Domain/                    # Capa de Dominio
│   ├── {Context}/
│   │   ├── Contracts/         # Interfaces (Puertos)
│   │   ├── Services/          # Servicios de dominio
│   │   ├── Entities/          # Entidades de negocio (si aplica)
│   │   └── ValueObjects/      # Objetos de valor (si aplica)
├── Application/               # Capa de Aplicación
│   └── {Context}/
│       ├── UseCases/          # Casos de uso específicos
│       └── Services/          # Servicios de aplicación
├── Infrastructure/            # Capa de Infraestructura
│   └── {Context}/
│       ├── Repositories/      # Implementaciones con Eloquent
│       ├── Services/          # Servicios externos
│       └── Adapters/          # Adaptadores a APIs externas
└── Http/Controllers/          # Capa de Presentación
```

## Ejemplos de Implementación

### 1. Interface de Dominio (Puerto)

```php
<?php

declare(strict_types=1);

namespace App\Domain\Payments;

use App\DTOs\Payments\ChargeResult;

interface RedsysGateway
{
    /**
     * Realiza un cargo usando un token almacenado del proveedor Redsys.
     */
    public function charge(string $token, string $amount, string $currency, array $metadata = []): ChargeResult;
}
```

### 2. Implementación de Infraestructura (Adaptador)

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Payments;

use App\Domain\Payments\RedsysGateway;
use App\Domain\Payments\RedsysSignature;
use App\DTOs\Payments\ChargeResult;
use Illuminate\Support\Facades\Http;

class RedsysGatewayImpl implements RedsysGateway
{
    public function __construct(private readonly RedsysSignature $signer)
    {
    }

    public function charge(string $token, string $amount, string $currency, array $metadata = []): ChargeResult
    {
        // Implementación específica de Redsys
        $order = now()->format('ymdHis');
        $merchantParams = [
            'DS_MERCHANT_MERCHANTCODE' => config('redsys.merchant_code'),
            'DS_MERCHANT_TERMINAL' => config('redsys.terminal'),
            'DS_MERCHANT_CURRENCY' => config('redsys.currency'),
            'DS_MERCHANT_ORDER' => $order,
            'DS_MERCHANT_AMOUNT' => (string) ((int) round(((float) $amount) * 100)),
            'DS_MERCHANT_TRANSACTIONTYPE' => '0',
            'DS_MERCHANT_IDENTIFIER' => $token,
        ];
        
        // ... resto de la implementación
    }
}
```

### 3. Caso de Uso de Aplicación

```php
<?php

declare(strict_types=1);

namespace App\Application\Payments;

use App\Domain\Payments\RedsysGateway;
use App\Enums\PaymentStatus;
use App\Models\Payment;

class ChargePayment
{
    public function __construct(private readonly RedsysGateway $gateway)
    {
    }

    public function handle(Payment $payment): void
    {
        // Lógica de aplicación que orquesta el dominio
        if ($payment->status !== PaymentStatus::Pending) {
            return;
        }

        $result = $this->gateway->charge(
            token: $method->token,
            amount: (string) $payment->amount,
            currency: $payment->currency,
            metadata: ['payment_id' => $payment->id]
        );

        // Actualizar estado basado en resultado
        if ($result->success) {
            $payment->status = PaymentStatus::Paid;
            $payment->paid_at = now();
        }
    }
}
```

### 4. Controlador (Presentación)

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api\Admin;

use App\Application\Sales\ExportSalesUseCase;
use App\Http\Controllers\Controller;

class SaleController extends Controller
{
    public function export(Request $request, ExportSalesUseCase $exportUseCase)
    {
        // Verificar autorización
        $this->authorize('sales.view');
        
        // Delegar al caso de uso
        $result = $exportUseCase->execute($request->validated());
        
        // Devolver respuesta
        return response($result['content'])
            ->header('Content-Type', $result['mimeType'])
            ->header('Content-Disposition', 'attachment; filename="' . $result['filename'] . '"');
    }
}
```

## Inyección de Dependencias

### Service Provider

```php
<?php

namespace App\Providers;

use App\Domain\Payments\RedsysGateway;
use App\Infrastructure\Payments\RedsysGatewayImpl;
use App\Domain\Sales\Contracts\SalesExportRepositoryInterface;
use App\Infrastructure\Sales\EloquentSalesExportRepository;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Binding de interfaces a implementaciones concretas
        $this->app->bind(RedsysGateway::class, RedsysGatewayImpl::class);
        $this->app->bind(SalesExportRepositoryInterface::class, EloquentSalesExportRepository::class);
        
        // Singletons para servicios stateless
        $this->app->singleton(SalesFilterService::class);
    }
}
```

## Reglas de Implementación

### Dominio
- ✅ **DEBE** ser independiente de Laravel
- ✅ **DEBE** usar interfaces para dependencias externas
- ✅ **DEBE** contener solo lógica de negocio pura
- ❌ **NO DEBE** importar clases de Eloquent, HTTP, o cualquier framework

### Aplicación
- ✅ **DEBE** orquestar casos de uso
- ✅ **DEBE** usar inyección de dependencias
- ✅ **PUEDE** usar modelos Eloquent para persistencia
- ✅ **DEBE** manejar transacciones cuando sea necesario

### Infraestructura
- ✅ **DEBE** implementar interfaces del dominio
- ✅ **PUEDE** usar cualquier tecnología (Eloquent, HTTP, etc.)
- ✅ **DEBE** manejar errores específicos de infraestructura

### Presentación
- ✅ **DEBE** ser "delgada" (thin)
- ✅ **DEBE** delegar lógica a la capa de aplicación
- ✅ **DEBE** manejar autorización
- ✅ **DEBE** transformar datos para respuesta (Resources)

## Testing

### Unit Tests para Dominio
```php
it('calculates payment amount correctly', function () {
    $calculator = new PaymentCalculator();
    
    $result = $calculator->calculate(100.50, 'EUR');
    
    expect($result)->toBe('10050'); // Céntimos
});
```

### Feature Tests para Casos de Uso
```php
it('charges payment successfully', function () {
    $gateway = Mockery::mock(RedsysGateway::class);
    $gateway->shouldReceive('charge')->andReturn(new ChargeResult(true));
    
    $useCase = new ChargePayment($gateway);
    $payment = Payment::factory()->create();
    
    $useCase->handle($payment);
    
    expect($payment->status)->toBe(PaymentStatus::Paid);
});
```

## Beneficios de esta Arquitectura

1. **Testabilidad**: Fácil mockear dependencias
2. **Flexibilidad**: Cambiar implementaciones sin afectar lógica
3. **Mantenibilidad**: Separación clara de responsabilidades
4. **Escalabilidad**: Agregar nuevas funcionalidades sin romper existentes
5. **Independencia**: Lógica de negocio no acoplada al framework
