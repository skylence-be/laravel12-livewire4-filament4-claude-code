---
name: laravel-caching-strategies
description: Master Laravel caching with Redis, File, Database drivers, cache tags, query caching, model caching, cache invalidation strategies, and Laravel Octane integration. Use when optimizing application performance, reducing database load, or scaling high-traffic applications.
---

# Laravel Caching Strategies

Comprehensive guide to implementing caching strategies in Laravel, covering cache drivers, cache tags, query caching, model caching, cache invalidation patterns, Redis optimization, and Laravel Octane integration for building high-performance applications.

## When to Use This Skill

- Reducing database query load in high-traffic applications
- Caching expensive computations or API responses
- Implementing page caching for static content
- Building scalable applications with Redis
- Optimizing Eloquent query performance
- Implementing fragment caching for views
- Setting up cache invalidation strategies
- Using cache tags for organized cache management
- Integrating with Laravel Octane for persistent caching
- Building distributed caching systems

## Core Concepts

### 1. Cache Drivers
- **Redis**: Fast, in-memory, supports tags (production recommended)
- **Memcached**: Fast, in-memory, no tag support
- **Database**: Persistent, slower than memory stores
- **File**: Simple, good for development
- **Array**: In-memory, only for testing

### 2. Cache Operations
- **Get**: Retrieve cached data
- **Put**: Store data in cache
- **Remember**: Get or store if not exists
- **Forever**: Store indefinitely
- **Forget**: Remove from cache

### 3. Cache Invalidation
- **Time-based**: Expire after duration
- **Event-based**: Invalidate on model changes
- **Tag-based**: Clear related items together
- **Manual**: Explicit flush operations

### 4. Cache Patterns
- **Cache-Aside**: Application manages cache
- **Read-Through**: Cache loads data automatically
- **Write-Through**: Updates cache on write
- **Write-Behind**: Async cache updates

## Quick Start

```php
<?php

// Basic caching
use Illuminate\Support\Facades\Cache;

// Store in cache
Cache::put('key', 'value', 600); // 10 minutes

// Retrieve from cache
$value = Cache::get('key');

// Store if doesn't exist
$value = Cache::remember('key', 3600, function () {
    return DB::table('users')->get();
});

// Remove from cache
Cache::forget('key');

// Check if exists
if (Cache::has('key')) {
    // ...
}

// Configure Redis in .env
CACHE_DRIVER=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

## Fundamental Patterns

### Pattern 1: Basic Cache Operations

```php
<?php

use Illuminate\Support\Facades\Cache;

// Store data
Cache::put('user:1', $user, now()->addHours(1));
Cache::put('posts', $posts, 3600); // Seconds
Cache::put('config', $config, now()->addDay());

// Store forever (until manually removed)
Cache::forever('settings', $settings);

// Store if not present
Cache::add('key', 'value', 600); // Only if key doesn't exist

// Retrieve data
$user = Cache::get('user:1');
$user = Cache::get('user:1', 'default value');
$user = Cache::get('user:1', fn () => User::find(1));

// Remember (get or store)
$posts = Cache::remember('posts', 3600, function () {
    return Post::all();
});

// Remember forever
$categories = Cache::rememberForever('categories', function () {
    return Category::all();
});

// Pull (get and delete)
$value = Cache::pull('key');

// Increment/Decrement
Cache::increment('page_views');
Cache::increment('counter', 5);
Cache::decrement('stock', 1);

// Check existence
if (Cache::has('key')) {
    // Key exists
}

if (Cache::missing('key')) {
    // Key doesn't exist
}

// Delete
Cache::forget('key');
Cache::flush(); // Clear all cache

// Multiple operations
Cache::putMany([
    'key1' => 'value1',
    'key2' => 'value2',
], 600);

$values = Cache::many(['key1', 'key2']);

// Atomic locks
$lock = Cache::lock('process', 10);

if ($lock->get()) {
    // Lock acquired for 10 seconds
    try {
        // Do work
    } finally {
        $lock->release();
    }
}

// Block until lock available
Cache::lock('process', 10)->block(5, function () {
    // Lock acquired, wait max 5 seconds
});
```

### Pattern 2: Cache Tags for Organized Management

```php
<?php

use Illuminate\Support\Facades\Cache;

// Only supported by Redis and Memcached drivers

// Store with tags
Cache::tags(['users', 'admins'])->put('user:1', $user, 3600);
Cache::tags(['posts', 'featured'])->put('featured-posts', $posts, 3600);

// Retrieve with tags
$user = Cache::tags(['users'])->get('user:1');
$posts = Cache::tags(['posts', 'featured'])->get('featured-posts');

// Remember with tags
$popularPosts = Cache::tags(['posts', 'popular'])->remember('popular-posts', 3600, function () {
    return Post::popular()->limit(10)->get();
});

// Flush specific tags
Cache::tags(['users'])->flush(); // Remove all user-related cache
Cache::tags(['posts', 'featured'])->flush(); // Remove posts with both tags

// Complex tag usage
class PostService
{
    public function getPost(int $id): Post
    {
        return Cache::tags(['posts', "post:{$id}"])->remember(
            "post:{$id}",
            3600,
            fn () => Post::with(['author', 'comments'])->findOrFail($id)
        );
    }

    public function updatePost(Post $post): void
    {
        $post->save();

        // Invalidate specific post cache
        Cache::tags(["post:{$post->id}"])->flush();

        // Invalidate all posts list
        Cache::tags(['posts'])->flush();
    }

    public function invalidateUserPosts(int $userId): void
    {
        Cache::tags(['posts', "user:{$userId}"])->flush();
    }
}

// Tag-based caching for resources
class CategoryService
{
    private const CACHE_TTL = 3600;

    public function getCategory(int $id): Category
    {
        return Cache::tags(['categories', "category:{$id}"])
            ->remember("category:{$id}", self::CACHE_TTL, function () use ($id) {
                return Category::with('products')->findOrFail($id);
            });
    }

    public function getAllCategories(): Collection
    {
        return Cache::tags(['categories'])
            ->remember('categories:all', self::CACHE_TTL, function () {
                return Category::orderBy('name')->get();
            });
    }

    public function clearCategoryCache(int $id): void
    {
        Cache::tags(["category:{$id}"])->flush();
        Cache::tags(['categories'])->forget('categories:all');
    }
}
```

### Pattern 3: Query Result Caching

```php
<?php

use Illuminate\Support\Facades\Cache;
use Illuminate\Database\Eloquent\Builder;

// Basic query caching
$users = Cache::remember('users:all', 3600, function () {
    return User::all();
});

$activeUsers = Cache::remember('users:active', 3600, function () {
    return User::where('active', true)->get();
});

// Cache complex queries
$statistics = Cache::remember('dashboard:stats', 600, function () {
    return [
        'total_users' => User::count(),
        'active_users' => User::where('active', true)->count(),
        'total_posts' => Post::count(),
        'published_posts' => Post::published()->count(),
        'revenue' => Order::sum('total'),
    ];
});

// Cache with query parameters
class PostRepository
{
    public function getPosts(array $filters = []): Collection
    {
        $cacheKey = 'posts:' . md5(json_encode($filters));

        return Cache::remember($cacheKey, 600, function () use ($filters) {
            $query = Post::query();

            if (isset($filters['status'])) {
                $query->where('status', $filters['status']);
            }

            if (isset($filters['category_id'])) {
                $query->where('category_id', $filters['category_id']);
            }

            return $query->with('author')->get();
        });
    }

    public function getPopularPosts(int $limit = 10): Collection
    {
        return Cache::remember("posts:popular:{$limit}", 3600, function () use ($limit) {
            return Post::withCount('views')
                ->orderBy('views_count', 'desc')
                ->limit($limit)
                ->get();
        });
    }
}

// Cache with relationships
$posts = Cache::remember('posts:with-relations', 3600, function () {
    return Post::with(['author', 'category', 'tags'])
        ->published()
        ->latest()
        ->get();
});

// Pagination with cache
class PostController extends Controller
{
    public function index(Request $request)
    {
        $page = $request->get('page', 1);
        $cacheKey = "posts:page:{$page}";

        $posts = Cache::remember($cacheKey, 600, function () {
            return Post::latest()->paginate(20);
        });

        return view('posts.index', compact('posts'));
    }
}

// Cache macro for Eloquent
Builder::macro('cacheFor', function (int $seconds, string $key = null) {
    $key = $key ?? $this->getCacheKey();

    return Cache::remember($key, $seconds, function () {
        return $this->get();
    });
});

// Usage
$posts = Post::where('status', 'published')
    ->cacheFor(3600, 'posts:published')
    ->get();
```

### Pattern 4: Model Caching with Events

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Cache;

class Post extends Model
{
    /**
     * Boot the model.
     */
    protected static function booted(): void
    {
        // Clear cache when post is created
        static::created(function (Post $post) {
            Cache::tags(['posts'])->flush();
            Cache::forget('posts:recent');
        });

        // Clear cache when post is updated
        static::updated(function (Post $post) {
            Cache::tags(['posts', "post:{$post->id}"])->flush();
            Cache::forget("post:{$post->id}");
        });

        // Clear cache when post is deleted
        static::deleted(function (Post $post) {
            Cache::tags(['posts', "post:{$post->id}"])->flush();
        });
    }

    /**
     * Get cached post.
     */
    public static function findCached(int $id): ?self
    {
        return Cache::tags(['posts', "post:{$id}"])
            ->remember("post:{$id}", 3600, function () use ($id) {
                return static::with(['author', 'category'])->find($id);
            });
    }

    /**
     * Get cached posts list.
     */
    public static function allCached(): Collection
    {
        return Cache::tags(['posts'])
            ->remember('posts:all', 3600, function () {
                return static::with('author')->get();
            });
    }
}

// Cache trait for reusable model caching
namespace App\Traits;

trait Cacheable
{
    /**
     * Boot the trait.
     */
    protected static function bootCacheable(): void
    {
        static::saved(function ($model) {
            $model->clearCache();
        });

        static::deleted(function ($model) {
            $model->clearCache();
        });
    }

    /**
     * Get cache key for model.
     */
    public function getCacheKey(): string
    {
        return static::class . ':' . $this->getKey();
    }

    /**
     * Get model from cache.
     */
    public static function findCached($id)
    {
        $key = static::class . ':' . $id;

        return Cache::remember($key, 3600, function () use ($id) {
            return static::find($id);
        });
    }

    /**
     * Clear model cache.
     */
    public function clearCache(): void
    {
        Cache::forget($this->getCacheKey());
        Cache::tags([Str::plural(class_basename($this))])->flush();
    }
}

// Usage
class Post extends Model
{
    use Cacheable;
}

$post = Post::findCached(1);
```

### Pattern 5: Cache Invalidation Strategies

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Cache;

class CacheService
{
    /**
     * Time-based invalidation.
     */
    public function getWithExpiration(string $key, int $ttl, callable $callback)
    {
        return Cache::remember($key, $ttl, $callback);
    }

    /**
     * Event-based invalidation.
     */
    public function invalidateOnEvent(string $event, array $cacheKeys): void
    {
        Event::listen($event, function () use ($cacheKeys) {
            foreach ($cacheKeys as $key) {
                Cache::forget($key);
            }
        });
    }

    /**
     * Dependency-based invalidation.
     */
    public function getWithDependencies(string $key, array $dependencies, callable $callback)
    {
        $dependencyHash = md5(serialize($dependencies));
        $cacheKey = "{$key}:{$dependencyHash}";

        return Cache::remember($cacheKey, 3600, $callback);
    }

    /**
     * Versioned cache keys.
     */
    public function getVersioned(string $key, callable $callback)
    {
        $version = Cache::get('cache:version', 1);
        $versionedKey = "{$key}:v{$version}";

        return Cache::remember($versionedKey, 3600, $callback);
    }

    /**
     * Increment version to invalidate all caches.
     */
    public function incrementVersion(): void
    {
        Cache::increment('cache:version');
    }

    /**
     * Cascading invalidation.
     */
    public function invalidateCascade(string $pattern): void
    {
        if (Cache::getStore() instanceof RedisStore) {
            $redis = Cache::getRedis();
            $keys = $redis->keys($pattern);

            foreach ($keys as $key) {
                Cache::forget($key);
            }
        }
    }

    /**
     * Lazy invalidation (mark stale, refresh on next access).
     */
    public function markStale(string $key): void
    {
        Cache::put("{$key}:stale", true, 60);
    }

    public function getOrRefresh(string $key, callable $callback)
    {
        if (Cache::has("{$key}:stale")) {
            Cache::forget("{$key}:stale");
            Cache::forget($key);
        }

        return Cache::remember($key, 3600, $callback);
    }
}

// Cache warming
class CacheWarmupService
{
    /**
     * Warm up frequently accessed data.
     */
    public function warmup(): void
    {
        // Warm up user data
        $users = User::all();
        Cache::put('users:all', $users, 3600);

        // Warm up popular posts
        $popularPosts = Post::popular()->limit(10)->get();
        Cache::put('posts:popular', $popularPosts, 3600);

        // Warm up categories
        $categories = Category::all();
        Cache::put('categories:all', $categories, 3600);
    }

    /**
     * Schedule warmup.
     */
    public function scheduleWarmup(): void
    {
        // In app/Console/Kernel.php
        $schedule->call(function () {
            app(CacheWarmupService::class)->warmup();
        })->hourly();
    }
}
```

### Pattern 6: Redis-Specific Optimizations

```php
<?php

use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Cache;

// Use Redis directly for advanced features
class RedisService
{
    /**
     * Store with Redis data structures.
     */
    public function useRedisDataStructures(): void
    {
        // Lists
        Redis::lpush('queue:emails', json_encode($email));
        $email = json_decode(Redis::rpop('queue:emails'));

        // Sets
        Redis::sadd('users:online', $userId);
        Redis::srem('users:online', $userId);
        $onlineUsers = Redis::smembers('users:online');

        // Sorted sets (leaderboard)
        Redis::zadd('leaderboard', $score, $userId);
        $topUsers = Redis::zrevrange('leaderboard', 0, 9, 'WITHSCORES');

        // Hashes
        Redis::hmset("user:{$id}", [
            'name' => $name,
            'email' => $email,
        ]);
        $user = Redis::hgetall("user:{$id}");

        // HyperLogLog (unique counts)
        Redis::pfadd('visitors:today', $userId);
        $uniqueVisitors = Redis::pfcount('visitors:today');
    }

    /**
     * Implement rate limiting with Redis.
     */
    public function rateLimit(string $key, int $maxAttempts, int $decayMinutes): bool
    {
        $attempts = Redis::incr($key);

        if ($attempts === 1) {
            Redis::expire($key, $decayMinutes * 60);
        }

        return $attempts <= $maxAttempts;
    }

    /**
     * Distributed locking with Redis.
     */
    public function executeWithLock(string $lockKey, callable $callback, int $timeout = 10)
    {
        $lock = Cache::lock($lockKey, $timeout);

        if ($lock->get()) {
            try {
                return $callback();
            } finally {
                $lock->release();
            }
        }

        return false;
    }

    /**
     * Cache aside pattern with Redis.
     */
    public function getCacheAside(string $key, callable $loader, int $ttl = 3600)
    {
        $value = Redis::get($key);

        if ($value === null) {
            $value = $loader();
            Redis::setex($key, $ttl, serialize($value));
        } else {
            $value = unserialize($value);
        }

        return $value;
    }

    /**
     * Pipeline multiple Redis operations.
     */
    public function pipeline(): void
    {
        Redis::pipeline(function ($pipe) {
            $pipe->set('key1', 'value1');
            $pipe->set('key2', 'value2');
            $pipe->incr('counter');
        });
    }

    /**
     * Use Redis pub/sub.
     */
    public function pubSub(): void
    {
        // Publish
        Redis::publish('channel', json_encode($message));

        // Subscribe
        Redis::subscribe(['channel'], function ($message) {
            // Handle message
        });
    }
}
```

### Pattern 7: Fragment Caching in Views

```php
<?php

// Blade directive for cache
namespace App\Providers;

use Illuminate\Support\Facades\Blade;
use Illuminate\Support\Facades\Cache;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // @cache directive
        Blade::directive('cache', function ($expression) {
            return "<?php if (! app('cache')->has({$expression})) : ?>";
        });

        Blade::directive('endcache', function ($expression) {
            $key = $expression;
            return "<?php endif; echo app('cache')->get({$key}); ?>";
        });
    }
}

// Usage in Blade templates
@cache('sidebar', 3600)
    <div class="sidebar">
        @foreach($categories as $category)
            <a href="{{ route('category', $category) }}">
                {{ $category->name }}
            </a>
        @endforeach
    </div>
@endcache

// View composer with caching
namespace App\View\Composers;

class SidebarComposer
{
    public function compose(View $view): void
    {
        $categories = Cache::remember('sidebar:categories', 3600, function () {
            return Category::withCount('posts')->get();
        });

        $view->with('categories', $categories);
    }
}

// Register composer
View::composer('partials.sidebar', SidebarComposer::class);

// Cache entire view
public function show(Post $post)
{
    $html = Cache::remember("post:{$post->id}:view", 3600, function () use ($post) {
        return view('posts.show', compact('post'))->render();
    });

    return response($html);
}
```

### Pattern 8: API Response Caching

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;

class CacheApiResponse
{
    /**
     * Handle an incoming request.
     */
    public function handle(Request $request, Closure $next, int $ttl = 300)
    {
        // Only cache GET requests
        if ($request->method() !== 'GET') {
            return $next($request);
        }

        $cacheKey = $this->getCacheKey($request);

        // Check if cached
        if (Cache::has($cacheKey)) {
            $response = Cache::get($cacheKey);

            return response($response['content'], $response['status'])
                ->withHeaders(array_merge($response['headers'], [
                    'X-Cache' => 'HIT',
                ]));
        }

        // Get fresh response
        $response = $next($request);

        // Cache successful responses
        if ($response->isSuccessful()) {
            Cache::put($cacheKey, [
                'content' => $response->getContent(),
                'status' => $response->getStatusCode(),
                'headers' => $response->headers->all(),
            ], $ttl);
        }

        $response->headers->set('X-Cache', 'MISS');

        return $response;
    }

    /**
     * Generate cache key from request.
     */
    private function getCacheKey(Request $request): string
    {
        $userId = $request->user()?->id ?? 'guest';
        $url = $request->fullUrl();

        return "api:response:{$userId}:" . md5($url);
    }
}

// Apply middleware
Route::middleware(['cache.response:600'])->group(function () {
    Route::get('/api/posts', [PostController::class, 'index']);
});

// Controller-level caching
class PostController extends Controller
{
    public function index(Request $request)
    {
        $cacheKey = 'api:posts:' . md5(json_encode($request->all()));

        return Cache::remember($cacheKey, 600, function () {
            return PostResource::collection(Post::paginate(20));
        });
    }

    public function show(Post $post)
    {
        $cacheKey = "api:post:{$post->id}";

        return Cache::remember($cacheKey, 3600, function () use ($post) {
            return new PostResource($post->load(['author', 'comments']));
        });
    }
}
```

### Pattern 9: Cache with Laravel Octane

```php
<?php

// Octane tables for persistent in-memory cache
use Illuminate\Support\Facades\Cache;

class OctaneCacheService
{
    /**
     * Use Octane table for fast access.
     */
    public function useOctaneTable(): void
    {
        // Store in Octane table
        Cache::driver('octane')->put('key', 'value', 3600);

        // Retrieve from Octane table
        $value = Cache::driver('octane')->get('key');

        // Octane table persists across requests in same worker
        // Perfect for application-level cache
    }

    /**
     * Warm cache on Octane worker start.
     */
    public function warmOnWorkerStart(): void
    {
        // In Octane service provider
        Octane::tick('warm-cache', function () {
            Cache::driver('octane')->put('config', config('app'), 3600);
        })->seconds(3600);
    }

    /**
     * Use Octane for shared state.
     */
    public function sharedState(): void
    {
        // Share state across requests in same worker
        $users = Cache::driver('octane')->rememberForever('active_users', function () {
            return User::where('active', true)->get();
        });
    }
}

// config/cache.php
'stores' => [
    'octane' => [
        'driver' => 'octane',
    ],
],

// Use with Octane
Cache::store('octane')->put('key', 'value');
```

### Pattern 10: Cache Monitoring and Analytics

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Redis;

class CacheMonitoringService
{
    /**
     * Track cache hit/miss ratio.
     */
    public function trackCacheHit(string $key, bool $hit): void
    {
        $metric = $hit ? 'cache:hits' : 'cache:misses';
        Redis::incr($metric);
        Redis::incr("{$metric}:{$key}");
    }

    /**
     * Get cache statistics.
     */
    public function getStats(): array
    {
        $hits = Redis::get('cache:hits') ?? 0;
        $misses = Redis::get('cache:misses') ?? 0;
        $total = $hits + $misses;

        return [
            'hits' => $hits,
            'misses' => $misses,
            'total' => $total,
            'hit_ratio' => $total > 0 ? round(($hits / $total) * 100, 2) : 0,
        ];
    }

    /**
     * Monitor cache size.
     */
    public function getCacheSize(): array
    {
        $info = Redis::info('memory');

        return [
            'used_memory' => $info['used_memory_human'] ?? 'N/A',
            'used_memory_peak' => $info['used_memory_peak_human'] ?? 'N/A',
        ];
    }

    /**
     * Get most accessed keys.
     */
    public function getTopKeys(int $limit = 10): array
    {
        $pattern = 'cache:hits:*';
        $keys = Redis::keys($pattern);

        $scores = [];
        foreach ($keys as $key) {
            $scores[$key] = (int) Redis::get($key);
        }

        arsort($scores);

        return array_slice($scores, 0, $limit, true);
    }

    /**
     * Clear old cache entries.
     */
    public function clearStaleCache(): int
    {
        $count = 0;

        // Clear entries older than 7 days
        $keys = Redis::keys('*');
        foreach ($keys as $key) {
            $ttl = Redis::ttl($key);
            if ($ttl === -1) { // No expiration
                Redis::expire($key, 604800); // Set to 7 days
                $count++;
            }
        }

        return $count;
    }
}
```

## Advanced Patterns

### Pattern 11: Multi-Level Caching

```php
<?php

class MultiLevelCache
{
    /**
     * Try L1 (memory), then L2 (Redis), then source.
     */
    public function get(string $key, callable $loader)
    {
        // L1: Application memory (Octane)
        static $l1Cache = [];

        if (isset($l1Cache[$key])) {
            return $l1Cache[$key];
        }

        // L2: Redis
        $value = Cache::get($key);

        if ($value !== null) {
            $l1Cache[$key] = $value;
            return $value;
        }

        // L3: Source
        $value = $loader();

        // Populate caches
        Cache::put($key, $value, 3600);
        $l1Cache[$key] = $value;

        return $value;
    }
}
```

### Pattern 12: Cache Stampede Prevention

```php
<?php

class CacheStampedeProtection
{
    /**
     * Prevent cache stampede with early recomputation.
     */
    public function get(string $key, int $ttl, callable $loader)
    {
        $cacheKey = "stampede:{$key}";
        $cache = Cache::get($cacheKey);

        if ($cache && $cache['expires_at'] > now()->addSeconds($ttl * 0.1)) {
            return $cache['value'];
        }

        // Recompute in background if near expiration
        if ($cache && $cache['expires_at'] <= now()->addSeconds($ttl * 0.1)) {
            dispatch(function () use ($key, $ttl, $loader) {
                $this->recompute($key, $ttl, $loader);
            })->afterResponse();

            return $cache['value'];
        }

        // Cache miss, compute now
        return $this->recompute($key, $ttl, $loader);
    }

    private function recompute(string $key, int $ttl, callable $loader)
    {
        $value = $loader();

        Cache::put("stampede:{$key}", [
            'value' => $value,
            'expires_at' => now()->addSeconds($ttl),
        ], $ttl);

        return $value;
    }
}
```

### Pattern 13: Distributed Cache Locking

```php
<?php

class DistributedCacheLock
{
    public function executeOnce(string $key, callable $callback, int $timeout = 60)
    {
        $lock = Cache::lock("lock:{$key}", $timeout);

        try {
            if ($lock->get()) {
                return $callback();
            }

            // Wait for lock and use cached result
            return $lock->block(30, function () use ($key) {
                return Cache::get($key);
            });
        } finally {
            optional($lock)->release();
        }
    }
}
```

## Real-World Applications

### Application 1: E-commerce Product Catalog

```php
<?php

class ProductCatalogService
{
    public function getProduct(int $id): Product
    {
        return Cache::tags(['products', "product:{$id}"])
            ->remember("product:{$id}", 3600, function () use ($id) {
                return Product::with(['images', 'variants', 'reviews'])
                    ->findOrFail($id);
            });
    }

    public function getPopularProducts(): Collection
    {
        return Cache::tags(['products', 'popular'])
            ->remember('products:popular', 3600, function () {
                return Product::withCount('orders')
                    ->orderBy('orders_count', 'desc')
                    ->limit(20)
                    ->get();
            });
    }

    public function invalidateProduct(int $id): void
    {
        Cache::tags(["product:{$id}"])->flush();
        Cache::tags(['popular'])->flush();
    }
}
```

### Application 2: Social Media Feed

```php
<?php

class FeedService
{
    public function getUserFeed(int $userId): array
    {
        return Cache::remember("feed:user:{$userId}", 300, function () use ($userId) {
            return Post::whereIn('user_id', function ($query) use ($userId) {
                $query->select('following_id')
                    ->from('followers')
                    ->where('follower_id', $userId);
            })
            ->with(['author', 'media'])
            ->latest()
            ->limit(50)
            ->get();
        });
    }

    public function clearUserFeed(int $userId): void
    {
        Cache::forget("feed:user:{$userId}");
    }
}
```

## Performance Best Practices

### Practice 1: Choose Right Driver

```php
// Development: File
CACHE_DRIVER=file

// Production: Redis
CACHE_DRIVER=redis

// Testing: Array
CACHE_DRIVER=array
```

### Practice 2: Use Tags Wisely

```php
// Good: Specific tags
Cache::tags(['posts', "post:{$id}"])->put($key, $value);

// Bad: Too many tags
Cache::tags(['posts', 'admin', 'public', 'featured', 'recent'])->put($key, $value);
```

### Practice 3: Set Appropriate TTLs

```php
// Static data: Long TTL
Cache::remember('countries', 86400, $loader);

// Dynamic data: Short TTL
Cache::remember('cart', 300, $loader);

// User-specific: Medium TTL
Cache::remember("user:{$id}", 3600, $loader);
```

## Common Pitfalls

### Pitfall 1: Not Using Tags

```php
// WRONG: Hard to invalidate
Cache::put('posts', $posts);

// CORRECT: Use tags
Cache::tags(['posts'])->put('all', $posts);
```

### Pitfall 2: Caching Too Much

```php
// WRONG: Cache entire collection
$users = Cache::remember('users', 3600, fn () => User::all());

// CORRECT: Cache specific queries
$activeUsers = Cache::remember('users:active', 3600, fn () =>
    User::where('active', true)->get()
);
```

### Pitfall 3: No Cache Invalidation

```php
// WRONG: Never clear cache
Cache::forever('settings', $settings);

// CORRECT: Clear on update
Settings::updated(fn () => Cache::forget('settings'));
```

### Pitfall 4: Race Conditions

```php
// WRONG: No locking
$value = Cache::get('counter');
Cache::put('counter', $value + 1);

// CORRECT: Atomic operation
Cache::increment('counter');
```

### Pitfall 5: Serialization Issues

```php
// WRONG: Cache closure
Cache::put('callback', fn () => 'value');

// CORRECT: Cache serializable data
Cache::put('value', 'value');
```

## Testing

```php
<?php

test('cache works correctly', function () {
    Cache::put('key', 'value', 60);

    expect(Cache::get('key'))->toBe('value');
});

test('cache tags work', function () {
    Cache::tags(['posts'])->put('post:1', 'data', 60);

    expect(Cache::tags(['posts'])->get('post:1'))->toBe('data');

    Cache::tags(['posts'])->flush();

    expect(Cache::tags(['posts'])->has('post:1'))->toBeFalse();
});

test('cache remember works', function () {
    $value = Cache::remember('test', 60, fn () => 'computed');

    expect($value)->toBe('computed');
});
```

## Resources

- **Laravel Cache Documentation**: https://laravel.com/docs/cache
- **Redis Documentation**: https://redis.io/documentation
- **Laravel Octane**: https://laravel.com/docs/octane
- **Cache Best Practices**: Performance optimization guides

## Best Practices Summary

1. **Use Redis in production** for performance and features
2. **Implement cache tags** for organized cache management
3. **Set appropriate TTLs** based on data volatility
4. **Clear cache on data changes** to maintain consistency
5. **Use cache locks** to prevent race conditions
6. **Monitor cache hit ratios** to optimize strategy
7. **Cache at multiple levels** for best performance
8. **Prevent cache stampede** with early recomputation
9. **Use Octane tables** for persistent app-level cache
10. **Test cache logic** thoroughly to avoid bugs
