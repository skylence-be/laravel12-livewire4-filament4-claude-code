---
name: laravel-testing-patterns
description: Master Laravel testing with Pest 4 syntax, feature tests, unit tests, database testing, factories, mocking, browser testing with Dusk, and comprehensive testing strategies. Use when writing tests, implementing TDD, or building robust test suites.
---

# Laravel Testing Patterns

Comprehensive guide to testing Laravel applications using Pest 4, feature tests, unit tests, database testing, factories, mocking, browser automation, and test-driven development practices for building reliable applications.

## When to Use This Skill

- Writing feature tests for HTTP endpoints and controllers
- Creating unit tests for services and business logic
- Testing database operations and Eloquent models
- Using factories and seeders for test data generation
- Mocking external services and APIs
- Implementing browser tests with Laravel Dusk
- Setting up continuous testing in CI/CD pipelines
- Testing authentication and authorization
- Validating API responses and JSON structures
- Debugging and troubleshooting failing tests

## Core Concepts

### 1. Test Types
- **Feature Tests**: Test complete features through HTTP layer
- **Unit Tests**: Test individual classes and methods in isolation
- **Browser Tests**: Test JavaScript interactions with Dusk
- **Integration Tests**: Test multiple components working together

### 2. Test Structure (Arrange-Act-Assert)
- **Arrange**: Set up test data and preconditions
- **Act**: Execute the code under test
- **Assert**: Verify the expected outcomes

### 3. Database Testing
- Use in-memory SQLite for fast tests
- Refresh database between tests
- Use factories for consistent test data
- Seed databases when needed

### 4. Test Isolation
- Tests should be independent
- Each test creates its own data
- Database reset between tests
- No shared state between tests

### 5. Pest vs PHPUnit
- Pest provides cleaner, more readable syntax
- Built on top of PHPUnit
- Better for BDD-style tests
- Laravel 11+ includes Pest by default

## Quick Start

```php
<?php

// Install Pest
composer require pestphp/pest --dev --with-all-dependencies
php artisan pest:install

// Feature test example - tests/Feature/UserTest.php
test('user can register', function () {
    $response = $this->post('/register', [
        'name' => 'John Doe',
        'email' => 'john@example.com',
        'password' => 'password',
        'password_confirmation' => 'password',
    ]);

    $response->assertStatus(302);
    $this->assertDatabaseHas('users', ['email' => 'john@example.com']);
});

// Run tests
php artisan test
./vendor/bin/pest
```

## Fundamental Patterns

### Pattern 1: Basic Pest Test Structure

```php
<?php

// tests/Feature/PostTest.php

use App\Models\Post;
use App\Models\User;
use function Pest\Laravel\{get, post, put, delete, actingAs};

test('guest cannot create posts', function () {
    $response = post('/posts', [
        'title' => 'Test Post',
        'content' => 'Test content',
    ]);

    $response->assertStatus(401);
});

test('authenticated user can create post', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->post('/posts', [
            'title' => 'My First Post',
            'content' => 'This is the content',
        ])
        ->assertStatus(302)
        ->assertSessionHas('success');

    $this->assertDatabaseHas('posts', [
        'title' => 'My First Post',
        'user_id' => $user->id,
    ]);
});

test('post can be updated', function () {
    $user = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user->id]);

    actingAs($user)
        ->put("/posts/{$post->id}", [
            'title' => 'Updated Title',
            'content' => 'Updated content',
        ])
        ->assertStatus(302);

    expect($post->fresh())
        ->title->toBe('Updated Title')
        ->content->toBe('Updated content');
});

test('post can be deleted', function () {
    $user = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user->id]);

    actingAs($user)
        ->delete("/posts/{$post->id}")
        ->assertStatus(302);

    $this->assertDatabaseMissing('posts', ['id' => $post->id]);
});

test('user can only update their own posts', function () {
    $user1 = User::factory()->create();
    $user2 = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user1->id]);

    actingAs($user2)
        ->put("/posts/{$post->id}", ['title' => 'Hacked'])
        ->assertStatus(403);

    expect($post->fresh()->title)->not->toBe('Hacked');
});
```

### Pattern 2: Database Testing with Factories

```php
<?php

// database/factories/UserFactory.php
namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;

class UserFactory extends Factory
{
    public function definition(): array
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => Hash::make('password'),
            'remember_token' => Str::random(10),
        ];
    }

    public function unverified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => null,
        ]);
    }

    public function admin(): static
    {
        return $this->state(fn (array $attributes) => [
            'is_admin' => true,
        ]);
    }
}

// database/factories/PostFactory.php
class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'title' => fake()->sentence(),
            'slug' => fake()->slug(),
            'content' => fake()->paragraphs(3, true),
            'status' => 'draft',
            'published_at' => null,
        ];
    }

    public function published(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'published',
            'published_at' => now(),
        ]);
    }

    public function withComments(int $count = 3): static
    {
        return $this->has(Comment::factory()->count($count));
    }
}

// Using factories in tests
test('can create user with factory', function () {
    $user = User::factory()->create();

    expect($user)->toBeInstanceOf(User::class)
        ->and($user->email)->toContain('@');

    $this->assertDatabaseHas('users', ['email' => $user->email]);
});

test('can create multiple users', function () {
    $users = User::factory()->count(10)->create();

    expect($users)->toHaveCount(10);
    $this->assertDatabaseCount('users', 10);
});

test('can use factory states', function () {
    $admin = User::factory()->admin()->create();
    $unverified = User::factory()->unverified()->create();

    expect($admin->is_admin)->toBeTrue()
        ->and($unverified->email_verified_at)->toBeNull();
});

test('can create related models', function () {
    $post = Post::factory()
        ->published()
        ->withComments(5)
        ->create();

    expect($post->status)->toBe('published')
        ->and($post->comments)->toHaveCount(5);
});

test('can use factory relationships', function () {
    $user = User::factory()
        ->has(Post::factory()->count(3)->published())
        ->create();

    expect($user->posts)->toHaveCount(3)
        ->and($user->posts->first()->status)->toBe('published');
});
```

### Pattern 3: HTTP Testing with Assertions

```php
<?php

use function Pest\Laravel\{get, post, put, delete};

test('index page returns success', function () {
    get('/posts')
        ->assertStatus(200)
        ->assertViewIs('posts.index')
        ->assertViewHas('posts');
});

test('post creation validation', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->post('/posts', [])
        ->assertStatus(302)
        ->assertSessionHasErrors(['title', 'content']);
});

test('api returns json', function () {
    $posts = Post::factory()->count(3)->create();

    get('/api/posts')
        ->assertStatus(200)
        ->assertJsonCount(3)
        ->assertJsonStructure([
            '*' => ['id', 'title', 'content', 'created_at']
        ]);
});

test('api returns specific post', function () {
    $post = Post::factory()->create([
        'title' => 'Specific Title',
    ]);

    get("/api/posts/{$post->id}")
        ->assertStatus(200)
        ->assertJson([
            'data' => [
                'id' => $post->id,
                'title' => 'Specific Title',
            ]
        ])
        ->assertJsonPath('data.title', 'Specific Title');
});

test('post creation returns created resource', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->post('/api/posts', [
            'title' => 'New Post',
            'content' => 'Content here',
        ])
        ->assertStatus(201)
        ->assertJsonFragment(['title' => 'New Post']);
});

test('redirects after successful action', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->post('/posts', [
            'title' => 'Test',
            'content' => 'Content',
        ])
        ->assertRedirect('/posts')
        ->assertSessionHas('success', 'Post created successfully');
});

test('forbidden for unauthorized users', function () {
    $user = User::factory()->create();
    $post = Post::factory()->create();

    actingAs($user)
        ->delete("/posts/{$post->id}")
        ->assertStatus(403);
});
```

### Pattern 4: Authentication Testing

```php
<?php

test('user can login with valid credentials', function () {
    $user = User::factory()->create([
        'email' => 'john@example.com',
        'password' => bcrypt('password'),
    ]);

    post('/login', [
        'email' => 'john@example.com',
        'password' => 'password',
    ])
        ->assertStatus(302)
        ->assertRedirect('/dashboard');

    $this->assertAuthenticatedAs($user);
});

test('user cannot login with invalid credentials', function () {
    $user = User::factory()->create([
        'email' => 'john@example.com',
        'password' => bcrypt('password'),
    ]);

    post('/login', [
        'email' => 'john@example.com',
        'password' => 'wrong-password',
    ])
        ->assertStatus(302)
        ->assertSessionHasErrors();

    $this->assertGuest();
});

test('user can logout', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->post('/logout')
        ->assertStatus(302)
        ->assertRedirect('/');

    $this->assertGuest();
});

test('authenticated user redirected from login', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->get('/login')
        ->assertRedirect('/dashboard');
});

test('guest redirected to login', function () {
    get('/dashboard')
        ->assertRedirect('/login');
});

test('email verification required', function () {
    $user = User::factory()->unverified()->create();

    actingAs($user)
        ->get('/dashboard')
        ->assertRedirect('/verify-email');
});

test('verified user can access dashboard', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->get('/dashboard')
        ->assertStatus(200);
});
```

### Pattern 5: Database Assertions

```php
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('database has record', function () {
    $user = User::factory()->create(['email' => 'test@example.com']);

    $this->assertDatabaseHas('users', ['email' => 'test@example.com']);
});

test('database missing record', function () {
    $this->assertDatabaseMissing('users', ['email' => 'nonexistent@example.com']);
});

test('database count', function () {
    User::factory()->count(5)->create();

    $this->assertDatabaseCount('users', 5);
});

test('soft delete works', function () {
    $post = Post::factory()->create();

    $post->delete();

    $this->assertSoftDeleted($post);
    $this->assertDatabaseHas('posts', ['id' => $post->id]);
    $this->assertDatabaseMissing('posts', [
        'id' => $post->id,
        'deleted_at' => null,
    ]);
});

test('model exists in database', function () {
    $user = User::factory()->create();

    $this->assertModelExists($user);
});

test('model missing from database', function () {
    $user = User::factory()->make(['id' => 999]);

    $this->assertModelMissing($user);
});

test('transaction rollback', function () {
    DB::beginTransaction();

    User::factory()->create(['email' => 'test@example.com']);

    $this->assertDatabaseHas('users', ['email' => 'test@example.com']);

    DB::rollBack();

    $this->assertDatabaseMissing('users', ['email' => 'test@example.com']);
});
```

### Pattern 6: Mocking External Services

```php
<?php

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Storage;

test('http requests can be faked', function () {
    Http::fake([
        'api.example.com/*' => Http::response([
            'id' => 1,
            'name' => 'Test',
        ], 200),
    ]);

    $service = new ExternalApiService();
    $result = $service->fetchData();

    expect($result['name'])->toBe('Test');

    Http::assertSent(function ($request) {
        return $request->url() === 'https://api.example.com/data';
    });
});

test('http failures can be simulated', function () {
    Http::fake([
        'api.example.com/*' => Http::response([], 500),
    ]);

    $service = new ExternalApiService();

    expect(fn () => $service->fetchData())
        ->toThrow(\Exception::class);
});

test('mail can be faked', function () {
    Mail::fake();

    $user = User::factory()->create();

    $user->sendWelcomeEmail();

    Mail::assertSent(WelcomeEmail::class, function ($mail) use ($user) {
        return $mail->hasTo($user->email);
    });
});

test('queued jobs can be faked', function () {
    Queue::fake();

    $user = User::factory()->create();

    ProcessUser::dispatch($user);

    Queue::assertPushed(ProcessUser::class);
    Queue::assertPushed(ProcessUser::class, fn ($job) => $job->user->id === $user->id);
});

test('storage can be faked', function () {
    Storage::fake('public');

    $response = $this->post('/upload', [
        'file' => UploadedFile::fake()->image('photo.jpg'),
    ]);

    Storage::disk('public')->assertExists('photos/photo.jpg');
});

test('events can be faked', function () {
    Event::fake([UserRegistered::class]);

    $user = User::factory()->create();

    Event::assertDispatched(UserRegistered::class);
    Event::assertDispatched(
        UserRegistered::class,
        fn ($event) => $event->user->id === $user->id
    );
});
```

### Pattern 7: Testing Eloquent Models

```php
<?php

// tests/Unit/PostTest.php

test('post has title', function () {
    $post = new Post(['title' => 'Test Title']);

    expect($post->title)->toBe('Test Title');
});

test('post belongs to user', function () {
    $user = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user->id]);

    expect($post->user)->toBeInstanceOf(User::class)
        ->and($post->user->id)->toBe($user->id);
});

test('user has many posts', function () {
    $user = User::factory()
        ->has(Post::factory()->count(3))
        ->create();

    expect($user->posts)->toHaveCount(3)
        ->and($user->posts->first())->toBeInstanceOf(Post::class);
});

test('post has slug generated from title', function () {
    $post = Post::factory()->create(['title' => 'Hello World']);

    expect($post->slug)->toBe('hello-world');
});

test('post scopes work correctly', function () {
    Post::factory()->count(5)->published()->create();
    Post::factory()->count(3)->create(['status' => 'draft']);

    $published = Post::published()->get();

    expect($published)->toHaveCount(5);
});

test('post can be published', function () {
    $post = Post::factory()->create(['status' => 'draft']);

    $post->publish();

    expect($post->status)->toBe('published')
        ->and($post->published_at)->not->toBeNull();
});

test('post casts work correctly', function () {
    $post = Post::factory()->create([
        'published_at' => '2024-01-01 12:00:00',
        'metadata' => ['views' => 100],
    ]);

    expect($post->published_at)->toBeInstanceOf(Carbon::class)
        ->and($post->metadata)->toBeArray()
        ->and($post->metadata['views'])->toBe(100);
});
```

### Pattern 8: Testing Services and Business Logic

```php
<?php

// tests/Unit/OrderServiceTest.php

use App\Services\OrderService;
use App\Models\Order;
use App\Models\Product;

beforeEach(function () {
    $this->orderService = new OrderService();
});

test('can calculate order total', function () {
    $order = Order::factory()->create();
    $order->items()->create([
        'product_id' => Product::factory()->create(['price' => 10])->id,
        'quantity' => 2,
    ]);

    $total = $this->orderService->calculateTotal($order);

    expect($total)->toBe(20.0);
});

test('applies discount correctly', function () {
    $order = Order::factory()->create();
    $order->items()->create([
        'product_id' => Product::factory()->create(['price' => 100])->id,
        'quantity' => 1,
    ]);

    $total = $this->orderService->calculateTotal($order, discountPercent: 10);

    expect($total)->toBe(90.0);
});

test('throws exception for invalid order', function () {
    $order = Order::factory()->create();

    expect(fn () => $this->orderService->process($order))
        ->toThrow(\InvalidArgumentException::class, 'Order has no items');
});

test('processes order successfully', function () {
    $order = Order::factory()->create();
    $order->items()->create([
        'product_id' => Product::factory()->create()->id,
        'quantity' => 1,
    ]);

    $result = $this->orderService->process($order);

    expect($result)->toBeTrue()
        ->and($order->fresh()->status)->toBe('processed');
});
```

### Pattern 9: Testing Validation Rules

```php
<?php

use App\Http\Requests\CreatePostRequest;
use Illuminate\Support\Facades\Validator;

test('post validation requires title', function () {
    $validator = Validator::make([], (new CreatePostRequest())->rules());

    expect($validator->fails())->toBeTrue()
        ->and($validator->errors()->has('title'))->toBeTrue();
});

test('post validation requires minimum title length', function () {
    $validator = Validator::make(
        ['title' => 'ab'],
        (new CreatePostRequest())->rules()
    );

    expect($validator->fails())->toBeTrue()
        ->and($validator->errors()->first('title'))
        ->toContain('at least 3 characters');
});

test('post validation accepts valid data', function () {
    $validator = Validator::make(
        [
            'title' => 'Valid Title',
            'content' => 'Valid content',
            'status' => 'draft',
        ],
        (new CreatePostRequest())->rules()
    );

    expect($validator->passes())->toBeTrue();
});

test('custom validation rule works', function () {
    $validator = Validator::make(
        ['slug' => 'invalid slug!'],
        ['slug' => 'alpha_dash']
    );

    expect($validator->fails())->toBeTrue();
});
```

### Pattern 10: Testing API Resources

```php
<?php

use App\Http\Resources\PostResource;
use App\Models\Post;

test('post resource transforms correctly', function () {
    $post = Post::factory()->create([
        'title' => 'Test Post',
        'content' => 'Test content',
    ]);

    $resource = new PostResource($post);
    $response = $resource->toArray(request());

    expect($response)
        ->toHaveKey('id', $post->id)
        ->toHaveKey('title', 'Test Post')
        ->toHaveKey('content', 'Test content')
        ->toHaveKey('created_at');
});

test('post collection resource', function () {
    $posts = Post::factory()->count(3)->create();

    get('/api/posts')
        ->assertStatus(200)
        ->assertJsonCount(3, 'data')
        ->assertJsonStructure([
            'data' => [
                '*' => ['id', 'title', 'content', 'created_at']
            ]
        ]);
});

test('resource includes relationships', function () {
    $user = User::factory()->create(['name' => 'John Doe']);
    $post = Post::factory()->create(['user_id' => $user->id]);

    get("/api/posts/{$post->id}")
        ->assertStatus(200)
        ->assertJsonPath('data.author.name', 'John Doe');
});
```

## Advanced Patterns

### Pattern 11: Parallel Testing

```php
<?php

// tests/Pest.php
uses(Tests\TestCase::class)->in('Feature');

// Run tests in parallel
php artisan test --parallel

// Or with Pest directly
./vendor/bin/pest --parallel

// Configure parallel testing in phpunit.xml
<testsuites>
    <testsuite name="Feature">
        <directory suffix="Test.php">./tests/Feature</directory>
    </testsuite>
</testsuites>

// tests/ParallelTest.php
test('runs in parallel', function () {
    expect(true)->toBeTrue();
})->group('parallel');

// Run specific groups in parallel
php artisan test --parallel --group=parallel
```

### Pattern 12: Database Transactions

```php
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Foundation\Testing\DatabaseTransactions;

// Option 1: Refresh entire database
uses(RefreshDatabase::class)->in('Feature');

// Option 2: Use transactions (faster)
uses(DatabaseTransactions::class)->in('Feature');

test('creates user in transaction', function () {
    $user = User::factory()->create();

    $this->assertDatabaseHas('users', ['id' => $user->id]);
});

// After test completes, transaction is rolled back

test('with specific database connection', function () {
    $this->beforeApplicationDestroyed(function () {
        DB::connection('mysql')->rollBack();
    });

    // Test code
});
```

### Pattern 13: Testing Middleware

```php
<?php

test('middleware blocks unauthenticated users', function () {
    get('/dashboard')
        ->assertStatus(302)
        ->assertRedirect('/login');
});

test('middleware allows authenticated users', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->get('/dashboard')
        ->assertStatus(200);
});

test('admin middleware restricts access', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->get('/admin/dashboard')
        ->assertStatus(403);
});

test('admin middleware allows admins', function () {
    $admin = User::factory()->admin()->create();

    actingAs($admin)
        ->get('/admin/dashboard')
        ->assertStatus(200);
});

test('rate limiting middleware works', function () {
    for ($i = 0; $i < 60; $i++) {
        get('/api/posts')->assertStatus(200);
    }

    get('/api/posts')->assertStatus(429); // Too Many Requests
});
```

### Pattern 14: Testing File Uploads

```php
<?php

use Illuminate\Http\UploadedFile;

test('can upload image', function () {
    Storage::fake('public');

    $file = UploadedFile::fake()->image('avatar.jpg');

    $user = User::factory()->create();

    actingAs($user)
        ->post('/profile/avatar', ['avatar' => $file])
        ->assertStatus(200);

    Storage::disk('public')->assertExists('avatars/' . $file->hashName());
});

test('validates file type', function () {
    Storage::fake('public');

    $file = UploadedFile::fake()->create('document.pdf');

    $user = User::factory()->create();

    actingAs($user)
        ->post('/profile/avatar', ['avatar' => $file])
        ->assertStatus(302)
        ->assertSessionHasErrors(['avatar']);

    Storage::disk('public')->assertMissing('avatars/' . $file->hashName());
});

test('validates file size', function () {
    Storage::fake('public');

    $file = UploadedFile::fake()->image('large.jpg')->size(10240); // 10MB

    $user = User::factory()->create();

    actingAs($user)
        ->post('/profile/avatar', ['avatar' => $file])
        ->assertStatus(302)
        ->assertSessionHasErrors(['avatar']);
});
```

### Pattern 15: Testing Console Commands

```php
<?php

test('command executes successfully', function () {
    $this->artisan('users:cleanup')
        ->expectsOutput('Cleaning up users...')
        ->expectsOutput('Cleanup complete!')
        ->assertExitCode(0);
});

test('command prompts for confirmation', function () {
    $this->artisan('posts:delete-all')
        ->expectsQuestion('Are you sure?', 'yes')
        ->expectsOutput('All posts deleted')
        ->assertExitCode(0);
});

test('command handles arguments', function () {
    $this->artisan('user:delete', ['id' => 1])
        ->expectsOutput('User 1 deleted')
        ->assertExitCode(0);
});

test('command with table output', function () {
    User::factory()->count(3)->create();

    $this->artisan('users:list')
        ->expectsTable(['ID', 'Name', 'Email'], [
            [1, 'User 1', 'user1@example.com'],
            [2, 'User 2', 'user2@example.com'],
            [3, 'User 3', 'user3@example.com'],
        ])
        ->assertExitCode(0);
});
```

## Real-World Applications

### Application 1: Complete Feature Test Suite

```php
<?php

// tests/Feature/BlogFeatureTest.php

use function Pest\Laravel\{get, post, put, delete, actingAs};

describe('Blog Feature', function () {
    beforeEach(function () {
        $this->user = User::factory()->create();
        $this->admin = User::factory()->admin()->create();
    });

    describe('Post Management', function () {
        test('guest can view published posts', function () {
            $posts = Post::factory()->count(5)->published()->create();

            get('/posts')
                ->assertStatus(200)
                ->assertViewHas('posts', fn ($viewPosts) => $viewPosts->count() === 5);
        });

        test('guest cannot view draft posts', function () {
            $post = Post::factory()->create(['status' => 'draft']);

            get("/posts/{$post->slug}")
                ->assertStatus(404);
        });

        test('author can create post', function () {
            actingAs($this->user)
                ->post('/posts', [
                    'title' => 'New Post',
                    'content' => 'Post content',
                ])
                ->assertRedirect('/posts')
                ->assertSessionHas('success');

            $this->assertDatabaseHas('posts', [
                'title' => 'New Post',
                'user_id' => $this->user->id,
            ]);
        });

        test('author can update own post', function () {
            $post = Post::factory()->create(['user_id' => $this->user->id]);

            actingAs($this->user)
                ->put("/posts/{$post->id}", [
                    'title' => 'Updated',
                    'content' => 'Updated content',
                ])
                ->assertRedirect();

            expect($post->fresh()->title)->toBe('Updated');
        });

        test('author cannot update others posts', function () {
            $post = Post::factory()->create();

            actingAs($this->user)
                ->put("/posts/{$post->id}", ['title' => 'Hacked'])
                ->assertStatus(403);
        });

        test('admin can update any post', function () {
            $post = Post::factory()->create();

            actingAs($this->admin)
                ->put("/posts/{$post->id}", ['title' => 'Admin Update'])
                ->assertRedirect();

            expect($post->fresh()->title)->toBe('Admin Update');
        });
    });

    describe('Comments', function () {
        test('user can comment on post', function () {
            $post = Post::factory()->published()->create();

            actingAs($this->user)
                ->post("/posts/{$post->id}/comments", [
                    'content' => 'Great post!',
                ])
                ->assertRedirect();

            $this->assertDatabaseHas('comments', [
                'post_id' => $post->id,
                'user_id' => $this->user->id,
                'content' => 'Great post!',
            ]);
        });

        test('comment requires approval', function () {
            $post = Post::factory()->published()->create();

            actingAs($this->user)
                ->post("/posts/{$post->id}/comments", [
                    'content' => 'Great post!',
                ]);

            $this->assertDatabaseHas('comments', [
                'approved' => false,
            ]);

            get("/posts/{$post->slug}")
                ->assertDontSee('Great post!');
        });
    });
});
```

### Application 2: API Testing Suite

```php
<?php

// tests/Feature/Api/PostApiTest.php

describe('Posts API', function () {
    beforeEach(function () {
        $this->user = User::factory()->create();
        Sanctum::actingAs($this->user);
    });

    test('can list posts', function () {
        Post::factory()->count(15)->published()->create();

        get('/api/posts')
            ->assertStatus(200)
            ->assertJsonCount(15, 'data')
            ->assertJsonStructure([
                'data' => [
                    '*' => ['id', 'title', 'content', 'author', 'created_at']
                ],
                'links',
                'meta'
            ]);
    });

    test('can filter posts by status', function () {
        Post::factory()->count(5)->published()->create();
        Post::factory()->count(3)->create(['status' => 'draft']);

        get('/api/posts?status=published')
            ->assertStatus(200)
            ->assertJsonCount(5, 'data');
    });

    test('can search posts', function () {
        Post::factory()->create(['title' => 'Laravel Testing']);
        Post::factory()->create(['title' => 'PHP Basics']);

        get('/api/posts?search=Laravel')
            ->assertStatus(200)
            ->assertJsonCount(1, 'data')
            ->assertJsonPath('data.0.title', 'Laravel Testing');
    });

    test('can create post', function () {
        post('/api/posts', [
            'title' => 'API Post',
            'content' => 'Created via API',
        ])
            ->assertStatus(201)
            ->assertJsonPath('data.title', 'API Post');
    });

    test('validates required fields', function () {
        post('/api/posts', [])
            ->assertStatus(422)
            ->assertJsonValidationErrors(['title', 'content']);
    });

    test('can update post', function () {
        $post = Post::factory()->create(['user_id' => $this->user->id]);

        put("/api/posts/{$post->id}", [
            'title' => 'Updated Title',
        ])
            ->assertStatus(200)
            ->assertJsonPath('data.title', 'Updated Title');
    });

    test('can delete post', function () {
        $post = Post::factory()->create(['user_id' => $this->user->id]);

        delete("/api/posts/{$post->id}")
            ->assertStatus(204);

        $this->assertDatabaseMissing('posts', ['id' => $post->id]);
    });
});
```

## Performance Best Practices

### Practice 1: Use In-Memory SQLite

```php
<?php

// config/database.php
'connections' => [
    'sqlite_testing' => [
        'driver' => 'sqlite',
        'database' => ':memory:',
        'prefix' => '',
    ],
],

// phpunit.xml
<php>
    <env name="DB_CONNECTION" value="sqlite_testing"/>
</php>
```

### Practice 2: Minimize Database Queries

```php
<?php

// BAD: Creates multiple database records unnecessarily
test('slow test', function () {
    $users = User::factory()->count(100)->create();
    // ...
});

// GOOD: Create only what's needed
test('fast test', function () {
    $user = User::factory()->create();
    // ...
});
```

### Practice 3: Use Faker Efficiently

```php
<?php

// BAD: Regenerate faker for each attribute
$user = User::factory()->create([
    'name' => fake()->name(),
    'email' => fake()->email(),
]);

// GOOD: Let factory handle it
$user = User::factory()->create();
```

## Common Pitfalls

### Pitfall 1: Not Using RefreshDatabase

```php
// WRONG: Tests pollute database
test('creates user', function () {
    User::create(['name' => 'John', 'email' => 'john@example.com']);
    // ...
});

// CORRECT: Use RefreshDatabase
uses(RefreshDatabase::class);

test('creates user', function () {
    User::create(['name' => 'John', 'email' => 'john@example.com']);
    // Database is reset after test
});
```

### Pitfall 2: Testing Implementation Instead of Behavior

```php
// WRONG: Testing implementation details
test('uses specific query', function () {
    DB::shouldReceive('select')->once();
    // ...
});

// CORRECT: Test behavior
test('returns correct results', function () {
    $results = $service->getData();
    expect($results)->toHaveCount(5);
});
```

### Pitfall 3: Shared State Between Tests

```php
// WRONG: Shared state
beforeEach(function () {
    $this->user = User::find(1); // Assumes user exists
});

// CORRECT: Create fresh state
beforeEach(function () {
    $this->user = User::factory()->create();
});
```

### Pitfall 4: Not Asserting Anything

```php
// WRONG: No assertions
test('creates user', function () {
    User::factory()->create();
});

// CORRECT: Assert expectations
test('creates user', function () {
    $user = User::factory()->create();

    $this->assertDatabaseHas('users', ['id' => $user->id]);
});
```

### Pitfall 5: Over-Mocking

```php
// WRONG: Mock everything
test('processes order', function () {
    $orderService = Mockery::mock(OrderService::class);
    $paymentGateway = Mockery::mock(PaymentGateway::class);
    // Testing mocks, not real code
});

// CORRECT: Only mock external dependencies
test('processes order', function () {
    Http::fake();
    $order = Order::factory()->create();
    $service = new OrderService();
    $service->process($order);
    // Test real code with faked external calls
});
```

## Testing

```php
<?php

// Run all tests
php artisan test

// Run specific test file
php artisan test tests/Feature/PostTest.php

// Run specific test
php artisan test --filter test_user_can_create_post

// Run with coverage
php artisan test --coverage

// Run in parallel
php artisan test --parallel

// Run with Pest
./vendor/bin/pest

// Run specific group
./vendor/bin/pest --group=feature

// Watch mode
./vendor/bin/pest --watch
```

## Resources

- **Laravel Testing Documentation**: https://laravel.com/docs/testing
- **Pest PHP**: https://pestphp.com/
- **Laravel Dusk**: https://laravel.com/docs/dusk
- **PHPUnit**: https://phpunit.de/
- **Mockery**: http://docs.mockery.io/
- **Laravel Factories**: https://laravel.com/docs/eloquent-factories

## Best Practices Summary

1. **Use Pest for cleaner syntax** and better readability
2. **Always use RefreshDatabase** to isolate tests
3. **Use factories** for all test data generation
4. **Test behavior, not implementation** details
5. **Mock external services** but test your code
6. **Keep tests fast** with in-memory databases
7. **Write descriptive test names** that explain expectations
8. **Use feature tests** for HTTP flows, unit tests for logic
9. **Run tests in parallel** for faster feedback
10. **Maintain test coverage** but focus on critical paths
