---
name: laravel-queues-jobs
description: Master Laravel queues and job processing with queue drivers, job batching, chaining, failure handling, Horizon monitoring, and advanced queue patterns. Use when implementing background job processing, async tasks, scheduled work, or building scalable job pipelines.
---

# Laravel Queues & Jobs

Comprehensive guide to implementing queued jobs in Laravel, covering queue configuration, job design patterns, batching, chaining, failure handling, monitoring with Horizon, and building robust asynchronous task processing systems.

## When to Use This Skill

- Processing tasks asynchronously in the background
- Sending emails without blocking HTTP responses
- Processing uploaded files and media
- Generating reports and exports
- Making external API calls that can be delayed
- Implementing scheduled or recurring background tasks
- Building job pipelines with dependencies
- Handling long-running operations
- Scaling application performance with async processing
- Implementing retry logic for unreliable operations

## Core Concepts

### 1. Queue Drivers
- **Database**: Store jobs in database table (good for development)
- **Redis**: Fast, reliable queue storage (production recommended)
- **SQS**: Amazon Simple Queue Service
- **Beanstalkd**: Fast work queue protocol
- **Sync**: Process immediately (testing only)

### 2. Job Lifecycle
- Dispatched → Queued → Processed → Completed/Failed
- Jobs can be retried on failure
- Failed jobs stored for inspection and retry
- Middleware can intercept job execution

### 3. Job Patterns
- **Dispatchable**: Jobs that can be dispatched to queues
- **Queueable**: Traits for queue configuration
- **SerializesModels**: Safely serialize Eloquent models
- **InteractsWithQueue**: Access job instance methods

### 4. Queue Priority
- Multiple queues with different priorities
- High-priority jobs processed first
- Dedicated workers for critical queues

### 5. Job Batching
- Group related jobs together
- Track batch progress and completion
- Handle batch failures collectively

## Quick Start

```php
<?php

// Create a job
php artisan make:job ProcessPodcast

// Job class
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Podcast $podcast)
    {
    }

    public function handle(): void
    {
        // Process the podcast
    }
}

// Dispatch the job
ProcessPodcast::dispatch($podcast);

// Run queue worker
php artisan queue:work
```

## Fundamental Patterns

### Pattern 1: Basic Job Structure

```php
<?php

namespace App\Jobs;

use App\Models\User;
use App\Services\VideoEncoder;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class ProcessVideo implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * The number of times the job may be attempted.
     */
    public int $tries = 3;

    /**
     * The number of seconds the job can run before timing out.
     */
    public int $timeout = 120;

    /**
     * Delete the job if its models no longer exist.
     */
    public bool $deleteWhenMissingModels = true;

    /**
     * Create a new job instance.
     */
    public function __construct(
        public Video $video,
        public string $format = 'mp4'
    ) {
    }

    /**
     * Execute the job.
     */
    public function handle(VideoEncoder $encoder): void
    {
        Log::info('Processing video', ['id' => $this->video->id]);

        try {
            $encoder->encode($this->video, $this->format);

            $this->video->update(['status' => 'processed']);

            Log::info('Video processed successfully', ['id' => $this->video->id]);
        } catch (\Exception $e) {
            Log::error('Video processing failed', [
                'id' => $this->video->id,
                'error' => $e->getMessage()
            ]);

            throw $e;
        }
    }

    /**
     * Handle a job failure.
     */
    public function failed(\Throwable $exception): void
    {
        $this->video->update(['status' => 'failed']);

        Log::error('Video processing permanently failed', [
            'id' => $this->video->id,
            'error' => $exception->getMessage()
        ]);
    }
}

// Dispatching examples
ProcessVideo::dispatch($video);
ProcessVideo::dispatch($video, 'webm'); // With custom format
ProcessVideo::dispatchIf($shouldProcess, $video);
ProcessVideo::dispatchUnless($alreadyProcessed, $video);
```

### Pattern 2: Queue Configuration and Connections

```php
<?php

// config/queue.php configuration
return [
    'default' => env('QUEUE_CONNECTION', 'redis'),

    'connections' => [
        'sync' => [
            'driver' => 'sync',
        ],

        'database' => [
            'driver' => 'database',
            'table' => 'jobs',
            'queue' => 'default',
            'retry_after' => 90,
            'after_commit' => false,
        ],

        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
            'queue' => env('REDIS_QUEUE', 'default'),
            'retry_after' => 90,
            'block_for' => null,
            'after_commit' => false,
        ],

        'sqs' => [
            'driver' => 'sqs',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'prefix' => env('SQS_PREFIX'),
            'queue' => env('SQS_QUEUE', 'default'),
            'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
        ],
    ],

    'failed' => [
        'driver' => env('QUEUE_FAILED_DRIVER', 'database-uuids'),
        'database' => env('DB_CONNECTION', 'mysql'),
        'table' => 'failed_jobs',
    ],
];

// Job with specific queue and connection
namespace App\Jobs;

class ProcessOrder implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Order $order)
    {
        // Set specific queue
        $this->onQueue('orders');

        // Set specific connection
        $this->onConnection('redis');
    }

    public function handle(): void
    {
        // Process order
    }
}

// Dispatch with queue configuration
ProcessOrder::dispatch($order)
    ->onQueue('high-priority')
    ->onConnection('redis');

// Delay job execution
ProcessOrder::dispatch($order)
    ->delay(now()->addMinutes(10));

// Chain method calls
ProcessOrder::dispatch($order)
    ->onQueue('orders')
    ->onConnection('redis')
    ->delay(now()->addMinutes(5));
```

### Pattern 3: Job Chaining

```php
<?php

namespace App\Jobs;

use Illuminate\Support\Facades\Bus;

class ProcessOrderPipeline
{
    /**
     * Execute job chain for order processing.
     */
    public static function handle(Order $order): void
    {
        Bus::chain([
            new ValidateOrder($order),
            new ChargePayment($order),
            new SendOrderConfirmation($order),
            new UpdateInventory($order),
            new NotifyShippingDepartment($order),
        ])->dispatch();
    }
}

class ValidateOrder implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Order $order)
    {
    }

    public function handle(): void
    {
        if (!$this->order->isValid()) {
            throw new \Exception('Order validation failed');
        }

        Log::info('Order validated', ['order_id' => $this->order->id]);
    }
}

class ChargePayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Order $order)
    {
    }

    public function handle(PaymentGateway $gateway): void
    {
        $gateway->charge($this->order);

        $this->order->update(['payment_status' => 'charged']);
    }
}

// Chain with catch callback
Bus::chain([
    new ValidateOrder($order),
    new ChargePayment($order),
    new SendOrderConfirmation($order),
])
->catch(function (\Throwable $e) use ($order) {
    Log::error('Order processing chain failed', [
        'order_id' => $order->id,
        'error' => $e->getMessage()
    ]);

    $order->update(['status' => 'failed']);
})
->dispatch();

// Prepend/append to chain
Bus::chain([
    new ProcessOrder($order),
])
->before([
    new ValidateOrder($order),
])
->after([
    new SendNotification($order),
])
->dispatch();
```

### Pattern 4: Job Batching

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Batch;
use Illuminate\Bus\Batchable;
use Illuminate\Support\Facades\Bus;

class ImportUser implements ShouldQueue
{
    use Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public array $userData)
    {
    }

    public function handle(): void
    {
        if ($this->batch()->cancelled()) {
            return;
        }

        User::create($this->userData);
    }
}

// Create and dispatch batch
class UserImportService
{
    public function import(array $users): Batch
    {
        $jobs = collect($users)->map(fn ($user) => new ImportUser($user));

        return Bus::batch($jobs)
            ->name('Import Users')
            ->then(function (Batch $batch) {
                Log::info('All users imported successfully');
            })
            ->catch(function (Batch $batch, \Throwable $e) {
                Log::error('Batch import failed', ['error' => $e->getMessage()]);
            })
            ->finally(function (Batch $batch) {
                Log::info('Batch import completed', [
                    'total' => $batch->totalJobs,
                    'processed' => $batch->processedJobs(),
                    'failed' => $batch->failedJobs,
                ]);
            })
            ->allowFailures()
            ->onConnection('redis')
            ->onQueue('imports')
            ->dispatch();
    }

    /**
     * Check batch status.
     */
    public function checkBatchStatus(string $batchId): array
    {
        $batch = Bus::findBatch($batchId);

        return [
            'id' => $batch->id,
            'name' => $batch->name,
            'total_jobs' => $batch->totalJobs,
            'pending_jobs' => $batch->pendingJobs,
            'processed_jobs' => $batch->processedJobs(),
            'failed_jobs' => $batch->failedJobs,
            'progress' => $batch->progress(),
            'finished' => $batch->finished(),
            'cancelled' => $batch->cancelled(),
        ];
    }

    /**
     * Cancel batch.
     */
    public function cancelBatch(string $batchId): void
    {
        $batch = Bus::findBatch($batchId);
        $batch->cancel();
    }

    /**
     * Add more jobs to batch.
     */
    public function addJobsToBatch(string $batchId, array $users): void
    {
        $batch = Bus::findBatch($batchId);

        $jobs = collect($users)->map(fn ($user) => new ImportUser($user));

        $batch->add($jobs);
    }
}

// Nested batching
Bus::batch([
    Bus::batch([
        new ImportUsers($chunk1),
        new ImportUsers($chunk2),
    ])->name('First Batch'),
    Bus::batch([
        new ImportUsers($chunk3),
        new ImportUsers($chunk4),
    ])->name('Second Batch'),
])
->name('Parent Batch')
->dispatch();
```

### Pattern 5: Job Middleware

```php
<?php

namespace App\Jobs\Middleware;

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Redis;

class RateLimited
{
    /**
     * Process the queued job.
     */
    public function handle(object $job, callable $next): void
    {
        Redis::throttle('key')
            ->block(0)
            ->allow(10)
            ->every(60)
            ->then(
                function () use ($job, $next) {
                    $next($job);
                },
                function () use ($job) {
                    $job->release(10);
                }
            );
    }
}

class WithoutOverlapping
{
    /**
     * Prevent job overlap using cache lock.
     */
    public function handle(object $job, callable $next): void
    {
        $lockKey = 'job:' . get_class($job) . ':' . $job->getJobId();

        $lock = Cache::lock($lockKey, 60);

        if ($lock->get()) {
            try {
                $next($job);
            } finally {
                $lock->release();
            }
        } else {
            $job->release(30);
        }
    }
}

// Job using middleware
namespace App\Jobs;

use App\Jobs\Middleware\RateLimited;
use App\Jobs\Middleware\WithoutOverlapping;

class ProcessVideo implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Get the middleware the job should pass through.
     */
    public function middleware(): array
    {
        return [
            new RateLimited,
            new WithoutOverlapping,
        ];
    }

    public function handle(): void
    {
        // Process video
    }
}

// Built-in middleware
use Illuminate\Queue\Middleware\WithoutOverlapping;
use Illuminate\Queue\Middleware\RateLimited as RateLimitedMiddleware;

class SendEmail implements ShouldQueue
{
    public function middleware(): array
    {
        return [
            new WithoutOverlapping($this->user->id),
            (new RateLimitedMiddleware('emails'))
                ->dontRelease()
                ->allow(10)
                ->every(60),
        ];
    }
}
```

### Pattern 6: Handling Job Failures

```php
<?php

namespace App\Jobs;

use App\Models\Order;
use App\Notifications\OrderProcessingFailed;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Notification;

class ProcessOrder implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Number of times to attempt job.
     */
    public int $tries = 5;

    /**
     * Number of seconds to wait before retrying.
     */
    public int $backoff = 30;

    /**
     * Maximum number of exceptions to allow.
     */
    public int $maxExceptions = 3;

    /**
     * Timeout in seconds.
     */
    public int $timeout = 120;

    public function __construct(public Order $order)
    {
    }

    /**
     * Determine the time at which the job should timeout.
     */
    public function retryUntil(): \DateTime
    {
        return now()->addHours(2);
    }

    /**
     * Calculate the number of seconds to wait before retrying.
     */
    public function backoff(): array
    {
        return [30, 60, 120, 300]; // Exponential backoff
    }

    public function handle(): void
    {
        try {
            $this->processOrder();
        } catch (\Exception $e) {
            if ($this->attempts() >= $this->tries) {
                throw $e;
            }

            Log::warning('Order processing failed, will retry', [
                'order_id' => $this->order->id,
                'attempt' => $this->attempts(),
                'error' => $e->getMessage(),
            ]);

            throw $e;
        }
    }

    private function processOrder(): void
    {
        // Process order logic
        if ($this->order->items->isEmpty()) {
            $this->fail(new \Exception('Order has no items'));
            return;
        }

        // ... processing logic
    }

    /**
     * Handle job failure.
     */
    public function failed(\Throwable $exception): void
    {
        $this->order->update([
            'status' => 'failed',
            'failure_reason' => $exception->getMessage(),
        ]);

        Log::error('Order processing permanently failed', [
            'order_id' => $this->order->id,
            'attempts' => $this->attempts(),
            'error' => $exception->getMessage(),
            'trace' => $exception->getTraceAsString(),
        ]);

        Notification::route('mail', config('app.admin_email'))
            ->notify(new OrderProcessingFailed($this->order, $exception));
    }
}

// Retry failed jobs
php artisan queue:retry all
php artisan queue:retry 5 # Retry specific job
php artisan queue:retry --queue=emails # Retry specific queue

// Forget failed jobs
php artisan queue:forget 5
php artisan queue:flush # Forget all
```

### Pattern 7: Unique Jobs

```php
<?php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;

class ProcessReport implements ShouldQueue, ShouldBeUnique
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $uniqueFor = 3600; // Lock for 1 hour

    public function __construct(public Report $report)
    {
    }

    /**
     * Get the unique ID for the job.
     */
    public function uniqueId(): string
    {
        return 'report-' . $this->report->id;
    }

    public function handle(): void
    {
        // Generate report
    }
}

// Unique until processing begins
class ProcessVideo implements ShouldQueue, ShouldBeUniqueUntilProcessing
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Video $video)
    {
    }

    public function uniqueId(): string
    {
        return 'video-' . $this->video->id;
    }
}

// Unique via Redis
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateUserProfile implements ShouldQueue, ShouldBeUnique
{
    public int $uniqueFor = 60;

    public function __construct(public User $user)
    {
    }

    public function uniqueId(): string
    {
        return 'update-profile-' . $this->user->id;
    }

    public function uniqueVia(): \Illuminate\Contracts\Cache\Repository
    {
        return Cache::driver('redis');
    }
}
```

### Pattern 8: Priority Queues

```php
<?php

namespace App\Jobs;

// High priority job
class SendUrgentEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public User $user)
    {
        $this->onQueue('high');
    }

    public function handle(): void
    {
        // Send urgent email
    }
}

// Normal priority job
class SendNewsletterEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public User $user)
    {
        $this->onQueue('default');
    }

    public function handle(): void
    {
        // Send newsletter
    }
}

// Low priority job
class GenerateMonthlyReport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Carbon $month)
    {
        $this->onQueue('low');
    }

    public function handle(): void
    {
        // Generate report
    }
}

// Start workers with priority
// Process high priority first, then default, then low
php artisan queue:work --queue=high,default,low

// Dedicated workers for different queues
// Supervisor config for high-priority queue
[program:laravel-worker-high]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/artisan queue:work redis --queue=high --sleep=3 --tries=3
autostart=true
autorestart=true
numprocs=5

[program:laravel-worker-default]
command=php /path/to/artisan queue:work redis --queue=default --sleep=3 --tries=3
numprocs=3

[program:laravel-worker-low]
command=php /path/to/artisan queue:work redis --queue=low --sleep=5 --tries=1
numprocs=1
```

### Pattern 9: Monitoring with Horizon

```php
<?php

// Install Horizon
composer require laravel/horizon
php artisan horizon:install
php artisan migrate

// config/horizon.php
return [
    'prefix' => env('HORIZON_PREFIX', 'horizon:'),

    'middleware' => ['web'],

    'waits' => [
        'redis:default' => 60,
        'redis:critical' => 30,
    ],

    'trim' => [
        'recent' => 60,
        'pending' => 60,
        'completed' => 60,
        'failed' => 10080, // 7 days
        'monitored' => 10080,
    ],

    'environments' => [
        'production' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['critical', 'high', 'default'],
                'balance' => 'auto',
                'processes' => 10,
                'tries' => 3,
                'timeout' => 300,
            ],
            'supervisor-2' => [
                'connection' => 'redis',
                'queue' => ['emails'],
                'balance' => 'simple',
                'processes' => 5,
                'tries' => 3,
                'timeout' => 60,
            ],
        ],

        'local' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'simple',
                'processes' => 3,
                'tries' => 3,
            ],
        ],
    ],
];

// Start Horizon
php artisan horizon

// Horizon commands
php artisan horizon:pause
php artisan horizon:continue
php artisan horizon:terminate
php artisan horizon:status

// Monitor job events in AppServiceProvider
use Laravel\Horizon\Horizon;

public function boot(): void
{
    Horizon::auth(function ($request) {
        return auth()->check() && auth()->user()->isAdmin();
    });
}
```

### Pattern 10: Job Events and Listeners

```php
<?php

namespace App\Providers;

use Illuminate\Queue\Events\JobFailed;
use Illuminate\Queue\Events\JobProcessed;
use Illuminate\Queue\Events\JobProcessing;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\ServiceProvider;

class QueueServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Queue::before(function (JobProcessing $event) {
            Log::info('Job processing started', [
                'connection' => $event->connectionName,
                'job' => $event->job->getName(),
            ]);
        });

        Queue::after(function (JobProcessed $event) {
            Log::info('Job processing completed', [
                'connection' => $event->connectionName,
                'job' => $event->job->getName(),
            ]);
        });

        Queue::failing(function (JobFailed $event) {
            Log::error('Job failed', [
                'connection' => $event->connectionName,
                'job' => $event->job->getName(),
                'exception' => $event->exception->getMessage(),
            ]);

            // Send alert to monitoring service
            app(MonitoringService::class)->alert('Job failed', [
                'job' => $event->job->getName(),
                'exception' => $event->exception->getMessage(),
            ]);
        });

        Queue::looping(function () {
            // Clear cache every iteration
            Cache::forget('queue-stats');
        });
    }
}

// Listen to specific job events
namespace App\Jobs;

class ProcessOrder implements ShouldQueue
{
    public function handle(): void
    {
        event(new OrderProcessingStarted($this->order));

        // Process order

        event(new OrderProcessingCompleted($this->order));
    }
}
```

## Advanced Patterns

### Pattern 11: Conditional Job Dispatch

```php
<?php

namespace App\Services;

use App\Jobs\SendWelcomeEmail;
use App\Jobs\CreateUserProfile;
use App\Jobs\NotifyAdminOfNewUser;
use Illuminate\Support\Facades\Bus;

class UserRegistrationService
{
    public function register(array $data): User
    {
        $user = User::create($data);

        // Conditional single job dispatch
        SendWelcomeEmail::dispatchIf(
            $user->email_verified_at,
            $user
        );

        SendWelcomeEmail::dispatchUnless(
            $user->is_guest,
            $user
        );

        // Conditional job pipeline
        if ($user->hasCompletedOnboarding()) {
            Bus::chain([
                new CreateUserProfile($user),
                new SendWelcomeEmail($user),
                new NotifyAdminOfNewUser($user),
            ])->dispatch();
        }

        // Dispatch based on plan
        match ($user->plan) {
            'premium' => Bus::chain([
                new SetupPremiumFeatures($user),
                new SendPremiumWelcomeEmail($user),
            ])->dispatch(),
            'basic' => SendBasicWelcomeEmail::dispatch($user),
            default => null,
        };

        return $user;
    }
}
```

### Pattern 12: Dynamic Job Configuration

```php
<?php

namespace App\Jobs;

class ProcessFile implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public File $file,
        ?int $tries = null,
        ?int $timeout = null,
        ?string $queue = null
    ) {
        // Configure based on file size
        if ($file->size > 100 * 1024 * 1024) { // > 100MB
            $this->tries = $tries ?? 5;
            $this->timeout = $timeout ?? 600;
            $this->onQueue($queue ?? 'large-files');
        } else {
            $this->tries = $tries ?? 3;
            $this->timeout = $timeout ?? 60;
            $this->onQueue($queue ?? 'default');
        }
    }

    public function handle(): void
    {
        // Process file
    }
}

// Factory pattern for job configuration
class JobFactory
{
    public static function makeFileProcessingJob(File $file): ProcessFile
    {
        $config = config('queue.file_processing');

        return new ProcessFile(
            file: $file,
            tries: $config['tries'] ?? 3,
            timeout: self::calculateTimeout($file),
            queue: self::determineQueue($file)
        );
    }

    private static function calculateTimeout(File $file): int
    {
        return match (true) {
            $file->size > 500 * 1024 * 1024 => 1800, // 30 min
            $file->size > 100 * 1024 * 1024 => 600,  // 10 min
            default => 120, // 2 min
        };
    }

    private static function determineQueue(File $file): string
    {
        return match ($file->type) {
            'video' => 'video-processing',
            'image' => 'image-processing',
            'document' => 'document-processing',
            default => 'default',
        };
    }
}
```

### Pattern 13: Batch Job Progress Tracking

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Batchable;

class ProcessChunkOfRecords implements ShouldQueue
{
    use Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public Collection $records,
        public string $batchId
    ) {
    }

    public function handle(): void
    {
        foreach ($this->records as $record) {
            if ($this->batch()->cancelled()) {
                return;
            }

            $this->processRecord($record);

            // Update progress
            Cache::increment("batch:{$this->batchId}:processed");
        }
    }

    private function processRecord($record): void
    {
        // Process individual record
    }
}

// Service to manage batch with progress
class BatchProcessingService
{
    public function processBatch(Collection $records): string
    {
        $batchId = (string) Str::uuid();
        $chunks = $records->chunk(100);

        Cache::put("batch:{$batchId}:total", $records->count(), 3600);
        Cache::put("batch:{$batchId}:processed", 0, 3600);

        $jobs = $chunks->map(fn ($chunk) => new ProcessChunkOfRecords($chunk, $batchId));

        Bus::batch($jobs)
            ->name("Batch Processing: {$batchId}")
            ->then(function () use ($batchId) {
                Cache::put("batch:{$batchId}:status", 'completed', 3600);
            })
            ->catch(function () use ($batchId) {
                Cache::put("batch:{$batchId}:status", 'failed', 3600);
            })
            ->dispatch();

        return $batchId;
    }

    public function getProgress(string $batchId): array
    {
        $total = Cache::get("batch:{$batchId}:total", 0);
        $processed = Cache::get("batch:{$batchId}:processed", 0);
        $status = Cache::get("batch:{$batchId}:status", 'processing');

        return [
            'total' => $total,
            'processed' => $processed,
            'percentage' => $total > 0 ? round(($processed / $total) * 100, 2) : 0,
            'status' => $status,
        ];
    }
}
```

### Pattern 14: Job Retry Strategies

```php
<?php

namespace App\Jobs;

use Illuminate\Support\Facades\Http;

class FetchExternalData implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 10;

    public function __construct(public string $url)
    {
    }

    /**
     * Exponential backoff with jitter.
     */
    public function backoff(): array
    {
        $backoffs = [];
        for ($i = 0; $i < $this->tries; $i++) {
            $exponential = min(pow(2, $i) * 10, 600); // Max 10 min
            $jitter = rand(0, 10); // Add randomness
            $backoffs[] = $exponential + $jitter;
        }
        return $backoffs;
    }

    /**
     * Determine if exception should trigger retry.
     */
    public function handle(): void
    {
        try {
            $response = Http::timeout(30)
                ->retry(3, 100)
                ->get($this->url);

            if ($response->successful()) {
                $this->processData($response->json());
            } else {
                throw new \Exception("HTTP {$response->status()}");
            }
        } catch (\Illuminate\Http\Client\ConnectionException $e) {
            // Always retry connection errors
            throw $e;
        } catch (\Illuminate\Http\Client\RequestException $e) {
            // Only retry 5xx server errors, not 4xx client errors
            if ($e->response->serverError()) {
                throw $e;
            }

            // Don't retry client errors
            $this->fail($e);
        }
    }

    private function processData(array $data): void
    {
        // Process the fetched data
    }
}
```

### Pattern 15: Scheduled Job Patterns

```php
<?php

namespace App\Console\Kernel;

use App\Jobs\GenerateDailyReport;
use App\Jobs\CleanupOldRecords;
use App\Jobs\SendWeeklyNewsletter;
use Illuminate\Console\Scheduling\Schedule;

class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        // Dispatch jobs at specific times
        $schedule->job(new GenerateDailyReport)
            ->daily()
            ->at('03:00')
            ->timezone('America/New_York')
            ->onOneServer()
            ->runInBackground();

        // Dispatch with custom queue
        $schedule->job(new CleanupOldRecords, 'maintenance')
            ->daily()
            ->at('02:00')
            ->withoutOverlapping(60);

        // Dispatch with parameters
        $schedule->call(function () {
            SendWeeklyNewsletter::dispatch(Carbon::now()->subWeek());
        })->weekly()->sundays()->at('10:00');

        // Conditional dispatch
        $schedule->job(new ProcessPendingOrders)
            ->everyMinute()
            ->when(fn () => Order::where('status', 'pending')->exists());

        // Chain multiple jobs
        $schedule->call(function () {
            Bus::chain([
                new BackupDatabase,
                new CleanupTempFiles,
                new GenerateReport,
            ])->dispatch();
        })->daily()->at('01:00');
    }
}

// Queued scheduled tasks
namespace App\Console\Commands;

class ProcessDailyReports extends Command
{
    protected $signature = 'reports:daily';

    public function handle(): void
    {
        $users = User::where('reports_enabled', true)->get();

        $jobs = $users->map(fn ($user) => new GenerateUserReport($user));

        Bus::batch($jobs)
            ->name('Daily Reports')
            ->dispatch();

        $this->info("Dispatched {$users->count()} report jobs");
    }
}
```

## Real-World Applications

### Application 1: Email Campaign System

```php
<?php

namespace App\Services;

use App\Jobs\SendCampaignEmail;
use App\Models\Campaign;
use Illuminate\Support\Facades\Bus;

class EmailCampaignService
{
    public function send(Campaign $campaign): void
    {
        $subscribers = $campaign->getTargetedSubscribers();

        $jobs = $subscribers->chunk(100)->map(function ($chunk) use ($campaign) {
            return new SendCampaignEmailBatch($campaign, $chunk);
        });

        Bus::batch($jobs)
            ->name("Campaign: {$campaign->name}")
            ->then(function () use ($campaign) {
                $campaign->update(['status' => 'completed', 'sent_at' => now()]);
            })
            ->catch(function () use ($campaign) {
                $campaign->update(['status' => 'failed']);
            })
            ->finally(function () use ($campaign) {
                GenerateCampaignReport::dispatch($campaign);
            })
            ->onQueue('email-campaigns')
            ->dispatch();
    }
}

class SendCampaignEmailBatch implements ShouldQueue
{
    use Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 300;

    public function __construct(
        public Campaign $campaign,
        public Collection $subscribers
    ) {
    }

    public function handle(MailService $mailer): void
    {
        foreach ($this->subscribers as $subscriber) {
            if ($this->batch()->cancelled()) {
                return;
            }

            try {
                $mailer->sendCampaignEmail($this->campaign, $subscriber);

                CampaignLog::create([
                    'campaign_id' => $this->campaign->id,
                    'subscriber_id' => $subscriber->id,
                    'status' => 'sent',
                    'sent_at' => now(),
                ]);
            } catch (\Exception $e) {
                CampaignLog::create([
                    'campaign_id' => $this->campaign->id,
                    'subscriber_id' => $subscriber->id,
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                ]);
            }
        }
    }
}
```

### Application 2: File Processing Pipeline

```php
<?php

namespace App\Services;

use App\Jobs\ValidateFile;
use App\Jobs\ProcessFile;
use App\Jobs\GenerateThumbnails;
use App\Jobs\ExtractMetadata;
use App\Jobs\NotifyFileProcessed;
use Illuminate\Support\Facades\Bus;

class FileProcessingService
{
    public function process(UploadedFile $file, User $user): void
    {
        $fileRecord = File::create([
            'user_id' => $user->id,
            'filename' => $file->getClientOriginalName(),
            'path' => $file->store('uploads'),
            'size' => $file->getSize(),
            'mime_type' => $file->getMimeType(),
            'status' => 'processing',
        ]);

        Bus::chain([
            new ValidateFile($fileRecord),
            new ProcessFile($fileRecord),
            new GenerateThumbnails($fileRecord),
            new ExtractMetadata($fileRecord),
            new NotifyFileProcessed($fileRecord, $user),
        ])
        ->catch(function (\Throwable $e) use ($fileRecord) {
            $fileRecord->update([
                'status' => 'failed',
                'error' => $e->getMessage(),
            ]);

            NotifyFileProcessingFailed::dispatch($fileRecord);
        })
        ->onQueue('file-processing')
        ->dispatch();
    }
}

class ProcessFile implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $timeout = 600;

    public function __construct(public File $file)
    {
    }

    public function handle(FileProcessor $processor): void
    {
        $result = $processor->process($this->file);

        $this->file->update([
            'processed_path' => $result['path'],
            'status' => 'processed',
        ]);
    }

    public function failed(\Throwable $exception): void
    {
        $this->file->update(['status' => 'failed']);
    }
}
```

## Performance Best Practices

### Practice 1: Optimize Job Serialization

```php
<?php

// BAD: Serializes entire collection
class ProcessUsers implements ShouldQueue
{
    public function __construct(public Collection $users)
    {
    }
}

// GOOD: Only serialize IDs, load in handle()
class ProcessUsers implements ShouldQueue
{
    public function __construct(public array $userIds)
    {
    }

    public function handle(): void
    {
        $users = User::whereIn('id', $this->userIds)->get();
        // Process users
    }
}

// GOOD: Use SerializesModels for single models
class ProcessUser implements ShouldQueue
{
    use SerializesModels;

    public function __construct(public User $user)
    {
        // Only user ID is serialized
    }
}
```

### Practice 2: Use Job Batching for Bulk Operations

```php
<?php

// BAD: Dispatch thousands of individual jobs
foreach ($users as $user) {
    SendEmail::dispatch($user);
}

// GOOD: Batch jobs
$jobs = $users->map(fn ($user) => new SendEmail($user));

Bus::batch($jobs)
    ->onQueue('emails')
    ->dispatch();

// BETTER: Chunk and batch
$users->chunk(100)->each(function ($chunk) {
    SendEmailBatch::dispatch($chunk);
});
```

### Practice 3: Use Horizon for Monitoring

```php
<?php

// Monitor queue metrics
use Laravel\Horizon\Contracts\JobRepository;
use Laravel\Horizon\Contracts\MetricsRepository;

class QueueMonitorService
{
    public function __construct(
        private JobRepository $jobs,
        private MetricsRepository $metrics
    ) {
    }

    public function getStats(): array
    {
        return [
            'recent_jobs' => $this->jobs->getRecent(),
            'failed_jobs' => $this->jobs->getFailed(),
            'wait_time' => $this->metrics->waitTimeFor('redis:default'),
            'throughput' => $this->metrics->throughputFor('redis:default'),
        ];
    }
}
```

## Common Pitfalls

### Pitfall 1: Forgetting to Run Queue Worker

```php
// WRONG: Jobs dispatched but worker not running
ProcessOrder::dispatch($order);
// Job sits in queue forever

// CORRECT: Ensure worker is running
php artisan queue:work

// PRODUCTION: Use supervisor to keep workers running
[program:laravel-worker]
command=php /path/to/artisan queue:work redis --sleep=3 --tries=3
autostart=true
autorestart=true
```

### Pitfall 2: Not Handling Job Failures

```php
// WRONG: No failure handling
class ProcessPayment implements ShouldQueue
{
    public function handle(): void
    {
        // Process payment
    }
}

// CORRECT: Implement failed() method
class ProcessPayment implements ShouldQueue
{
    public function handle(): void
    {
        // Process payment
    }

    public function failed(\Throwable $exception): void
    {
        // Update order status
        // Send notification
        // Log error
    }
}
```

### Pitfall 3: Serializing Large Objects

```php
// WRONG: Serialize entire collection
class ProcessOrders implements ShouldQueue
{
    public function __construct(public Collection $orders)
    {
    }
}

// CORRECT: Pass IDs only
class ProcessOrders implements ShouldQueue
{
    public function __construct(public array $orderIds)
    {
    }

    public function handle(): void
    {
        $orders = Order::whereIn('id', $this->orderIds)->get();
    }
}
```

### Pitfall 4: Not Setting Timeouts

```php
// WRONG: No timeout, job runs forever
class ProcessLargeFile implements ShouldQueue
{
    public function handle(): void
    {
        // Long-running operation
    }
}

// CORRECT: Set appropriate timeout
class ProcessLargeFile implements ShouldQueue
{
    public int $timeout = 600; // 10 minutes

    public function handle(): void
    {
        // Long-running operation
    }
}
```

### Pitfall 5: Not Using Job Middleware

```php
// WRONG: No rate limiting
class SendEmail implements ShouldQueue
{
    public function handle(): void
    {
        // Could overwhelm email service
    }
}

// CORRECT: Use rate limiting middleware
class SendEmail implements ShouldQueue
{
    public function middleware(): array
    {
        return [
            (new RateLimited('emails'))
                ->allow(100)
                ->every(60),
        ];
    }
}
```

## Testing

```php
<?php

namespace Tests\Feature;

use App\Jobs\ProcessOrder;
use App\Models\Order;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\Queue;
use Tests\TestCase;

class JobTest extends TestCase
{
    use RefreshDatabase;

    public function test_job_is_dispatched()
    {
        Queue::fake();

        $order = Order::factory()->create();

        ProcessOrder::dispatch($order);

        Queue::assertPushed(ProcessOrder::class);
        Queue::assertPushed(ProcessOrder::class, fn ($job) => $job->order->id === $order->id);
    }

    public function test_job_is_not_dispatched_when_condition_false()
    {
        Queue::fake();

        $order = Order::factory()->create(['status' => 'cancelled']);

        ProcessOrder::dispatchIf($order->status === 'pending', $order);

        Queue::assertNothingPushed();
    }

    public function test_job_processes_order_correctly()
    {
        $order = Order::factory()->create(['status' => 'pending']);

        $job = new ProcessOrder($order);
        $job->handle();

        $this->assertEquals('processed', $order->fresh()->status);
    }

    public function test_job_chain_dispatched()
    {
        Bus::fake();

        $order = Order::factory()->create();

        Bus::chain([
            new ValidateOrder($order),
            new ProcessOrder($order),
            new SendConfirmation($order),
        ])->dispatch();

        Bus::assertChained([
            ValidateOrder::class,
            ProcessOrder::class,
            SendConfirmation::class,
        ]);
    }

    public function test_batch_dispatched()
    {
        Bus::fake();

        $users = User::factory()->count(10)->create();
        $jobs = $users->map(fn ($user) => new SendEmail($user));

        Bus::batch($jobs)->dispatch();

        Bus::assertBatched(fn ($batch) => $batch->jobs->count() === 10);
    }
}
```

## Resources

- **Laravel Queue Documentation**: https://laravel.com/docs/queues
- **Laravel Horizon**: https://laravel.com/docs/horizon
- **Redis**: https://redis.io/
- **Supervisor**: http://supervisord.org/
- **AWS SQS**: https://aws.amazon.com/sqs/
- **Laravel Queue Monitor**: Package for tracking job execution

## Best Practices Summary

1. **Always run queue workers** in production with supervisor
2. **Use Redis** for production queue driver
3. **Implement failed() method** for all critical jobs
4. **Set appropriate timeouts** and retry attempts
5. **Use job batching** for bulk operations
6. **Monitor queues** with Horizon or similar tools
7. **Serialize minimal data** in job constructors
8. **Use job middleware** for rate limiting and locking
9. **Test job dispatch** and execution logic
10. **Handle job failures** gracefully with notifications
