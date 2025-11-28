---
globs: "**/*.php"
alwaysApply: false
---

# Reglas de Características de Laravel - Backend

## Eventos y Listeners

### Implementación de Eventos

```php
<?php

declare(strict_types=1);

namespace App\Events;

use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class ResourceCreated
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly object $resource
    ) {}
}
```

### Listeners con Procesamiento Asíncrono

```php
<?php

declare(strict_types=1);

namespace App\Listeners;

use App\Events\ResourceCreated;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendResourceNotifications implements ShouldQueue
{
    use InteractsWithQueue;

    public string $queue = 'notifications';
    public int $tries = 3;

    public function handle(ResourceCreated $event): void
    {
        // Procesar notificaciones
    }

    public function failed(ResourceCreated $event, \Throwable $exception): void
    {
        \Log::error('Failed to send notifications', [
            'resource_id' => $event->resource->id,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

### Registro de Eventos

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use App\Events\ResourceCreated;
use App\Listeners\SendResourceNotifications;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        ResourceCreated::class => [
            SendResourceNotifications::class,
        ],
    ];
}
```

### Disparar Eventos

```php
use App\Events\ResourceCreated;

// En caso de uso o controlador
ResourceCreated::dispatch($resource);
```

## Jobs (Trabajos en Cola)

### Job Base

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessResource implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 300;

    public function __construct(
        private readonly string $resourceId
    ) {}

    public function handle(): void
    {
        // Procesar recurso
    }

    public function failed(\Throwable $exception): void
    {
        \Log::error('Job failed', [
            'job' => static::class,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

### Programación de Jobs

```php
<?php

declare(strict_types=1);

namespace App\Console;

use App\Jobs\ProcessResource;
use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        $schedule->job(ProcessResource::class)
            ->hourly()
            ->withoutOverlapping()
            ->runInBackground();
    }
}
```

## Notificaciones

### Notificación Base con Cola

```php
<?php

declare(strict_types=1);

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class ResourceCreated extends Notification implements ShouldQueue
{
    use Queueable;

    public string $queue = 'notifications';
    public int $tries = 3;

    public function __construct(
        private readonly object $resource
    ) {}

    public function via(object $notifiable): array
    {
        return ['mail'];
    }

    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('Nuevo recurso creado')
            ->line('Se ha creado un nuevo recurso.')
            ->action('Ver detalles', url('/resources/' . $this->resource->id));
    }
}
```

## Configuración de Colas

### Configuración en config/queue.php

```php
'connections' => [
    'database' => [
        'driver' => 'database',
        'table' => 'jobs',
        'queue' => 'default',
        'retry_after' => 90,
    ],
],

'default' => env('QUEUE_CONNECTION', 'database'),
```

### Worker Configuration

```bash
# Procesar múltiples colas con prioridades
php artisan queue:work --queue=high,default,low --tries=3 --timeout=60

# Procesar solo notificaciones
php artisan queue:work --queue=notifications --tries=3
```

## Cache y Performance

### Service con Cache

```php
<?php

declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\Cache;

class ResourceService
{
    public function getStats(): array
    {
        return Cache::remember('resource.stats', 3600, function () {
            return [
                // Datos cacheados
            ];
        });
    }

    public function clearCache(): void
    {
        Cache::forget('resource.stats');
    }
}
```

## Mejores Prácticas

### 1. Eventos vs Jobs
- **Eventos**: Para notificar que algo pasó
- **Jobs**: Para hacer algo específico

### 2. Configuración de Colas
```php
// Colas por prioridad
'high'         // Operaciones críticas
'default'      // Operaciones normales  
'low'          // Reportes, limpieza
'notifications' // Emails, notificaciones
```

### 3. Manejo de Fallos
```php
public function failed(\Throwable $exception): void
{
    \Log::error('Job failed', [
        'job' => static::class,
        'error' => $exception->getMessage(),
    ]);
}
```

### 4. Testing de Jobs y Eventos
```php
it('dispatches event', function () {
    Event::fake();
    
    // Ejecutar acción
    
    Event::assertDispatched(ResourceCreated::class);
});

it('processes job', function () {
    Queue::fake();
    
    ProcessResource::dispatch();
    
    Queue::assertPushed(ProcessResource::class);
});
```
