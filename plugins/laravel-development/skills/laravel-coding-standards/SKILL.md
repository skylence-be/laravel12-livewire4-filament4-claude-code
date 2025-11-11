---
name: laravel-coding-standards
description: Comprehensive Laravel and PHP coding standards derived from Spatie guidelines. Use when writing Laravel code, refactoring, or reviewing code quality. Ensures PSR compliance, Laravel conventions, and modern PHP patterns.
---

# Laravel & PHP Guidelines for AI Code Assistants

This file contains Laravel and PHP coding standards optimized for AI code assistants like Claude Code, GitHub Copilot, and Cursor. These guidelines are derived from Spatie's comprehensive Laravel & PHP standards.

## When to Use This Skill

Invoke this skill when:

- **Writing new Laravel code** - Ensure new features follow established conventions
- **Refactoring existing code** - Improve code quality and maintainability
- **Code review and quality checks** - Validate adherence to standards
- **Setting up project standards** - Establish team coding guidelines
- **Training team on Laravel best practices** - Reference for developers
- **Ensuring PSR compliance** - Follow PHP-FIG standards (PSR-1, PSR-12)
- **Implementing modern PHP 8+ features** - Use enums, attributes, typed properties
- **Following Laravel 11-12 conventions** - Stay current with framework patterns

## Core Laravel Principle

**Follow Laravel conventions first.** If Laravel has a documented way to do something, use it. Only deviate when you have a clear justification.

## Code Style Standards

### PSR Compliance

Follow PSR-1 (Basic Coding Standard) and PSR-12 (Extended Coding Style) with exceptions noted below.

### Spacing and Braces

```php
// Good: Opening brace on same line for methods/functions
class User extends Model
{
    public function getName(): string
    {
        return $this->name;
    }
}

// Good: Opening brace on same line for control structures
if ($condition) {
    // code
} elseif ($otherCondition) {
    // code
} else {
    // code
}

// Bad: Opening brace on new line
if ($condition)
{
    // code
}
```

### Blank Lines

```php
// Good: One blank line between methods
class UserController extends Controller
{
    public function index()
    {
        return view('users.index');
    }

    public function show(User $user)
    {
        return view('users.show', compact('user'));
    }
}

// Bad: No blank lines between methods
class UserController extends Controller
{
    public function index()
    {
        return view('users.index');
    }
    public function show(User $user)
    {
        return view('users.show', compact('user'));
    }
}
```

### Arrays

Use short array syntax and consistent formatting:

```php
// Good: Short syntax, trailing comma
$array = [
    'item1',
    'item2',
    'item3',
];

// Good: Single line for short arrays
$colors = ['red', 'green', 'blue'];

// Bad: Old array syntax
$array = array('item1', 'item2');

// Bad: No trailing comma on multiline
$array = [
    'item1',
    'item2',
    'item3'
];
```

### Docblocks

Use docblocks only when they add value. Don't state the obvious:

```php
// Good: Docblock adds value by documenting exceptions
/**
 * @throws \App\Exceptions\InsufficientFundsException
 */
public function withdraw(int $amount): void
{
    if ($this->balance < $amount) {
        throw new InsufficientFundsException();
    }
}

// Bad: Docblock states the obvious
/**
 * Get the user's name
 *
 * @return string
 */
public function getName(): string
{
    return $this->name;
}

// Good: Type hints are sufficient
public function getName(): string
{
    return $this->name;
}
```

### Comments

Write code that explains itself. Use comments sparingly:

```php
// Good: Self-explanatory code
public function isEligibleForDiscount(): bool
{
    return $this->age >= 65 || $this->isMember;
}

// Bad: Comment explains what code does
public function check(): bool
{
    // Check if user is over 65 or is a member
    return $this->age >= 65 || $this->isMember;
}

// Good: Comment explains WHY, not WHAT
// We use bcrypt instead of argon2 for compatibility with legacy systems
Hash::make($password, ['driver' => 'bcrypt']);
```

### Ternary Operators

Keep them simple and readable:

```php
// Good: Simple ternary
$status = $user->isActive() ? 'active' : 'inactive';

// Good: Null coalescing
$name = $user->name ?? 'Guest';

// Good: Null coalescing assignment (PHP 7.4+)
$this->name ??= 'Guest';

// Bad: Nested ternaries
$result = $a ? $b ? $c : $d : $e;

// Good: Use if/else for complex logic
if ($a) {
    $result = $b ? $c : $d;
} else {
    $result = $e;
}
```

### If Statements

Prefer positive conditions and avoid `else` when possible:

```php
// Good: Early return, no else needed
public function process(Order $order): void
{
    if (! $order->isPaid()) {
        throw new UnpaidOrderException();
    }

    $order->ship();
}

// Bad: Unnecessary else
public function process(Order $order): void
{
    if ($order->isPaid()) {
        $order->ship();
    } else {
        throw new UnpaidOrderException();
    }
}

// Good: Positive condition is clearer
if ($user->isActive()) {
    // handle active user
}

// Acceptable but less clear
if (! $user->isInactive()) {
    // handle active user
}
```

## Naming Conventions

### Classes

Use PascalCase for class names:

```php
// Good
class UserController
class OrderRepository
class PaymentService

// Bad
class user_controller
class orderrepository
class payment_service
```

### Methods and Variables

Use camelCase:

```php
// Good
public function getUserName(): string
{
    $firstName = $this->first_name;
    return $firstName;
}

// Bad
public function get_user_name(): string
{
    $first_name = $this->first_name;
    return $first_name;
}
```

### Database Columns

Use snake_case:

```php
// Good
Schema::create('users', function (Blueprint $table) {
    $table->string('first_name');
    $table->string('last_name');
    $table->timestamp('created_at');
});

// Bad
Schema::create('users', function (Blueprint $table) {
    $table->string('firstName');
    $table->string('lastName');
    $table->timestamp('createdAt');
});
```

### Routes

Use kebab-case for URLs:

```php
// Good
Route::get('user-profile', [UserController::class, 'show']);
Route::get('api/v1/user-settings', [SettingsController::class, 'index']);

// Bad
Route::get('user_profile', [UserController::class, 'show']);
Route::get('api/v1/userSettings', [SettingsController::class, 'index']);
```

### Boolean Methods

Prefix with `is`, `has`, `can`, or `should`:

```php
// Good
public function isActive(): bool
public function hasPermission(string $permission): bool
public function canEdit(User $user): bool
public function shouldSendNotification(): bool

// Bad
public function active(): bool
public function permission(string $permission): bool
public function editable(User $user): bool
```

## Type Declarations

### Always Use Type Hints

Use type hints for parameters and return types:

```php
// Good: Full type coverage
public function createUser(string $name, string $email, int $age): User
{
    return User::create([
        'name' => $name,
        'email' => $email,
        'age' => $age,
    ]);
}

// Bad: No type hints
public function createUser($name, $email, $age)
{
    return User::create([
        'name' => $name,
        'email' => $email,
        'age' => $age,
    ]);
}

// Good: Nullable types
public function findUser(?int $id): ?User
{
    return $id ? User::find($id) : null;
}

// Good: Union types (PHP 8+)
public function process(int|string $id): User
{
    return User::findOrFail($id);
}
```

### Void Return Type

Use `void` when a method doesn't return a value:

```php
// Good
public function sendNotification(User $user): void
{
    Mail::to($user)->send(new WelcomeEmail());
}

// Bad
public function sendNotification(User $user)
{
    Mail::to($user)->send(new WelcomeEmail());
}
```

### Typed Properties

Always declare property types (PHP 7.4+):

```php
// Good
class User extends Model
{
    protected string $name;
    protected ?string $nickname = null;
    protected array $permissions = [];
}

// Bad
class User extends Model
{
    protected $name;
    protected $nickname = null;
    protected $permissions = [];
}
```

## Modern Laravel (11-12) Patterns

### Enums

Use backed enums for fixed sets of values:

```php
// Good: Backed enum with PascalCase cases
namespace App\Enums;

enum OrderStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Completed = 'completed';
    case Cancelled = 'cancelled';

    public function label(): string
    {
        return match($this) {
            self::Pending => 'Pending',
            self::Processing => 'Processing',
            self::Completed => 'Completed',
            self::Cancelled => 'Cancelled',
        };
    }

    public function color(): string
    {
        return match($this) {
            self::Pending => 'gray',
            self::Processing => 'blue',
            self::Completed => 'green',
            self::Cancelled => 'red',
        };
    }
}

// Usage in models
class Order extends Model
{
    protected function casts(): array
    {
        return [
            'status' => OrderStatus::class,
        ];
    }
}

// Usage in code
$order->status = OrderStatus::Processing;
if ($order->status === OrderStatus::Completed) {
    // process completed order
}
```

### PHP 8 Attributes

Use attributes for routes, validation, and metadata:

```php
// Good: Route attributes (Laravel 11+)
use Illuminate\Support\Facades\Route;

#[Route('/users', name: 'users.')]
class UserController extends Controller
{
    #[Route('/', methods: ['GET'], name: 'index')]
    public function index()
    {
        return view('users.index');
    }

    #[Route('/{user}', methods: ['GET'], name: 'show')]
    public function show(User $user)
    {
        return view('users.show', compact('user'));
    }
}

// Good: Custom attributes for validation
namespace App\Attributes;

use Attribute;

#[Attribute(Attribute::TARGET_PROPERTY)]
class Uppercase
{
    public function __construct(
        public bool $enabled = true
    ) {}
}

// Usage in DTOs
class CreateUserData
{
    public function __construct(
        #[Uppercase]
        public string $name,
        public string $email,
    ) {}
}
```

### Invokable Controllers

Use single-action controllers for focused endpoints:

```php
// Good: Invokable controller
namespace App\Http\Controllers;

class ShowDashboard extends Controller
{
    public function __invoke()
    {
        $stats = [
            'users' => User::count(),
            'orders' => Order::count(),
            'revenue' => Order::sum('total'),
        ];

        return view('dashboard', compact('stats'));
    }
}

// Route registration
Route::get('/dashboard', ShowDashboard::class)->name('dashboard');

// Good: Invokable controller with dependencies
class GenerateInvoice extends Controller
{
    public function __construct(
        private InvoiceGenerator $generator
    ) {}

    public function __invoke(Order $order)
    {
        $pdf = $this->generator->generate($order);

        return response()->streamDownload(
            fn () => print($pdf),
            "invoice-{$order->id}.pdf"
        );
    }
}
```

### Model Casts

Use modern cast types for better data handling:

```php
// Good: Modern casts with AsArrayObject, AsCollection
namespace App\Models;

use Illuminate\Database\Eloquent\Casts\AsArrayObject;
use Illuminate\Database\Eloquent\Casts\AsCollection;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'settings' => AsArrayObject::class,
            'permissions' => AsCollection::class,
            'metadata' => 'array',
            'is_active' => 'boolean',
            'birth_date' => 'date',
            'status' => UserStatus::class, // Enum cast
        ];
    }
}

// Usage
$user->settings->theme = 'dark'; // AsArrayObject allows object access
$user->permissions->push('edit-posts'); // AsCollection provides collection methods
```

### Service Container Best Practices

Use constructor injection and method injection properly:

```php
// Good: Constructor injection for dependencies used in multiple methods
class OrderService
{
    public function __construct(
        private PaymentGateway $gateway,
        private EmailService $emailService,
        private LoggerInterface $logger
    ) {}

    public function createOrder(array $data): Order
    {
        $order = Order::create($data);

        $this->logger->info('Order created', ['order_id' => $order->id]);

        return $order;
    }

    public function processPayment(Order $order): void
    {
        $this->gateway->charge($order->total);
        $this->emailService->sendReceipt($order);
    }
}

// Good: Method injection for dependencies used in single method
class UserController extends Controller
{
    public function store(StoreUserRequest $request, UserRepository $repository): User
    {
        return $repository->create($request->validated());
    }
}

// Good: Binding interfaces to implementations
// In AppServiceProvider
public function register(): void
{
    $this->app->bind(PaymentGateway::class, StripeGateway::class);
    $this->app->singleton(CacheService::class);
}
```

## Eloquent Models

### Model Organization

Organize model contents in a consistent order:

```php
class User extends Model
{
    // 1. Traits
    use HasFactory, Notifiable, SoftDeletes;

    // 2. Constants
    public const ROLE_ADMIN = 'admin';
    public const ROLE_USER = 'user';

    // 3. Properties
    protected $table = 'users';
    protected $fillable = ['name', 'email', 'password'];
    protected $hidden = ['password', 'remember_token'];

    // 4. Casts
    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'is_active' => 'boolean',
        ];
    }

    // 5. Boot method
    protected static function booted(): void
    {
        static::creating(function ($user) {
            $user->uuid = Str::uuid();
        });
    }

    // 6. Relationships
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }

    // 7. Scopes
    public function scopeActive(Builder $query): void
    {
        $query->where('is_active', true);
    }

    // 8. Accessors & Mutators
    protected function name(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => strtolower($value),
        );
    }

    // 9. Custom methods
    public function isAdmin(): bool
    {
        return $this->role === self::ROLE_ADMIN;
    }
}
```

### Relationship Type Hints

Always type-hint relationships:

```php
// Good
public function posts(): HasMany
{
    return $this->hasMany(Post::class);
}

public function author(): BelongsTo
{
    return $this->belongsTo(User::class);
}

public function tags(): BelongsToMany
{
    return $this->belongsToMany(Tag::class);
}

// Bad
public function posts()
{
    return $this->hasMany(Post::class);
}
```

### Accessors and Mutators

Use modern accessor/mutator syntax (Laravel 9+):

```php
// Good: Modern syntax
protected function firstName(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => ucfirst($value),
        set: fn (string $value) => strtolower($value),
    );
}

// Good: Get-only accessor
protected function fullName(): Attribute
{
    return Attribute::make(
        get: fn () => "{$this->first_name} {$this->last_name}",
    );
}

// Bad: Old accessor syntax
public function getFirstNameAttribute($value)
{
    return ucfirst($value);
}
```

### Query Scopes

Use scopes for reusable query logic:

```php
// Good: Query scopes
class Post extends Model
{
    public function scopePublished(Builder $query): void
    {
        $query->where('published_at', '<=', now());
    }

    public function scopeByAuthor(Builder $query, User $author): void
    {
        $query->where('user_id', $author->id);
    }

    public function scopePopular(Builder $query, int $threshold = 100): void
    {
        $query->where('views', '>=', $threshold);
    }
}

// Usage
$posts = Post::published()->popular()->get();
$authorPosts = Post::byAuthor($user)->published()->get();
```

## Controllers

### Thin Controllers

Keep controllers thin by moving business logic to services or actions:

```php
// Good: Thin controller using service
class OrderController extends Controller
{
    public function __construct(
        private OrderService $orderService
    ) {}

    public function store(StoreOrderRequest $request): RedirectResponse
    {
        $order = $this->orderService->create($request->validated());

        return redirect()->route('orders.show', $order)
            ->with('success', 'Order created successfully');
    }
}

// Bad: Fat controller with business logic
class OrderController extends Controller
{
    public function store(Request $request): RedirectResponse
    {
        $validated = $request->validate([
            'product_id' => 'required|exists:products,id',
            'quantity' => 'required|integer|min:1',
        ]);

        $product = Product::findOrFail($validated['product_id']);

        if ($product->stock < $validated['quantity']) {
            throw new InsufficientStockException();
        }

        $order = Order::create([
            'user_id' => auth()->id(),
            'product_id' => $validated['product_id'],
            'quantity' => $validated['quantity'],
            'total' => $product->price * $validated['quantity'],
        ]);

        $product->decrement('stock', $validated['quantity']);

        Mail::to(auth()->user())->send(new OrderConfirmation($order));

        return redirect()->route('orders.show', $order)
            ->with('success', 'Order created successfully');
    }
}
```

### Resource Controllers

Use standard RESTful naming:

```php
// Good: Standard resource methods
class PostController extends Controller
{
    public function index(): View
    {
        $posts = Post::paginate();
        return view('posts.index', compact('posts'));
    }

    public function create(): View
    {
        return view('posts.create');
    }

    public function store(StorePostRequest $request): RedirectResponse
    {
        $post = Post::create($request->validated());
        return redirect()->route('posts.show', $post);
    }

    public function show(Post $post): View
    {
        return view('posts.show', compact('post'));
    }

    public function edit(Post $post): View
    {
        return view('posts.edit', compact('post'));
    }

    public function update(UpdatePostRequest $request, Post $post): RedirectResponse
    {
        $post->update($request->validated());
        return redirect()->route('posts.show', $post);
    }

    public function destroy(Post $post): RedirectResponse
    {
        $post->delete();
        return redirect()->route('posts.index');
    }
}
```

## Validation

### Form Requests

Use Form Requests for complex validation:

```php
// Good: Form Request with custom logic
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create-users');
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'confirmed', 'min:8'],
            'role' => ['required', 'in:admin,user,moderator'],
        ];
    }

    public function messages(): array
    {
        return [
            'email.unique' => 'This email address is already registered.',
            'role.in' => 'The selected role is invalid.',
        ];
    }

    public function prepareForValidation(): void
    {
        $this->merge([
            'email' => strtolower($this->email),
        ]);
    }
}

// Usage in controller
public function store(StoreUserRequest $request): User
{
    return User::create($request->validated());
}
```

### Validation Rules

Use array syntax and custom rules:

```php
// Good: Array syntax with custom rules
$request->validate([
    'email' => ['required', 'email', 'unique:users,email'],
    'age' => ['required', 'integer', 'min:18', 'max:100'],
    'url' => ['required', 'url', 'active_url'],
    'tags' => ['required', 'array', 'min:1', 'max:5'],
    'tags.*' => ['string', 'max:20'],
]);

// Bad: String syntax
$request->validate([
    'email' => 'required|email|unique:users,email',
]);
```

## Migrations

### Migration Structure

Only use `up()` methods, no `down()` methods:

```php
// Good: Only up() method
class CreateUsersTable extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->rememberToken();
            $table->timestamps();
        });
    }
}

// Bad: Including down() method
class CreateUsersTable extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            // ...
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
    }
}
```

### Naming Conventions

Use descriptive names that explain the change:

```php
// Good
2024_01_01_000001_create_users_table.php
2024_01_01_000002_create_posts_table.php
2024_01_01_000003_add_status_to_orders_table.php
2024_01_01_000004_create_post_tag_pivot_table.php

// Bad
2024_01_01_000001_users.php
2024_01_01_000002_update_orders.php
```

### Ordering

Create tables first, then alter them:

```php
// Good: Create table first
// Migration: 2024_01_01_000001_create_users_table.php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->timestamps();
});

// Good: Alter table later
// Migration: 2024_01_02_000001_add_email_to_users_table.php
Schema::table('users', function (Blueprint $table) {
    $table->string('email')->unique()->after('name');
});
```

### Foreign Keys and Indexes

Always index foreign keys:

```php
// Good: Foreign key with automatic index
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('title');
    $table->text('content');
    $table->timestamps();

    // Additional indexes
    $table->index('created_at');
    $table->index(['user_id', 'created_at']);
});

// Good: Custom foreign key
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->foreignId('post_id')
        ->constrained()
        ->cascadeOnDelete();
    $table->foreignId('author_id')
        ->constrained('users')
        ->nullOnDelete();
    $table->text('content');
    $table->timestamps();
});
```

### Column Positioning

Use `after()` for clarity:

```php
// Good: Specify position with after()
Schema::table('users', function (Blueprint $table) {
    $table->string('phone')->nullable()->after('email');
    $table->string('address')->nullable()->after('phone');
});
```

## Security Best Practices

### Input Validation

Always validate user input:

```php
// Good: Comprehensive validation
public function store(Request $request): User
{
    $validated = $request->validate([
        'email' => ['required', 'email', 'unique:users,email'],
        'password' => ['required', 'min:8', 'confirmed'],
        'age' => ['required', 'integer', 'min:18', 'max:120'],
    ]);

    return User::create($validated);
}

// Bad: No validation
public function store(Request $request): User
{
    return User::create($request->all());
}
```

### SQL Injection Prevention

Use query builder and Eloquent, never raw queries with user input:

```php
// Good: Query builder with bindings
$users = DB::table('users')
    ->where('email', $email)
    ->where('is_active', true)
    ->get();

// Good: Eloquent
$users = User::where('email', $email)->active()->get();

// Good: Named bindings in raw queries when necessary
$users = DB::select('SELECT * FROM users WHERE email = :email', [
    'email' => $email,
]);

// DANGEROUS: Never concatenate user input
$users = DB::select("SELECT * FROM users WHERE email = '{$email}'");
```

### XSS Prevention

Use Blade's automatic escaping:

```blade
{{-- Good: Automatic escaping --}}
<h1>{{ $user->name }}</h1>
<p>{{ $post->content }}</p>

{{-- DANGEROUS: Unescaped output --}}
<div>{!! $userInput !!}</div>

{{-- Good: Unescaped only for trusted content --}}
<div>{!! $sanitized->content !!}</div>
```

### CSRF Protection

Always use CSRF protection for state-changing requests:

```blade
{{-- Good: CSRF token in forms --}}
<form method="POST" action="{{ route('posts.store') }}">
    @csrf
    <input type="text" name="title">
    <button type="submit">Submit</button>
</form>

{{-- Good: Method spoofing with CSRF --}}
<form method="POST" action="{{ route('posts.destroy', $post) }}">
    @csrf
    @method('DELETE')
    <button type="submit">Delete</button>
</form>
```

### Mass Assignment Protection

Use `$fillable` or `$guarded`:

```php
// Good: Explicit fillable properties
class User extends Model
{
    protected $fillable = [
        'name',
        'email',
        'password',
    ];
}

// Good: Using guarded for inverse protection
class User extends Model
{
    protected $guarded = [
        'id',
        'is_admin',
        'remember_token',
    ];
}

// DANGEROUS: No protection
class User extends Model
{
    protected $guarded = [];
}

// Safe usage with validated data
User::create($request->validated());

// DANGEROUS: Creating with all input
User::create($request->all());
```

### Authorization

Always check permissions:

```php
// Good: Policy authorization
public function update(Request $request, Post $post): RedirectResponse
{
    $this->authorize('update', $post);

    $post->update($request->validated());

    return redirect()->route('posts.show', $post);
}

// Good: Gate authorization
public function destroy(User $user): RedirectResponse
{
    if (Gate::denies('delete-users')) {
        abort(403);
    }

    $user->delete();

    return redirect()->route('users.index');
}

// Bad: No authorization check
public function update(Request $request, Post $post): RedirectResponse
{
    $post->update($request->validated());
    return redirect()->route('posts.show', $post);
}
```

## Performance Patterns

### N+1 Query Prevention

Always eager load relationships:

```php
// Bad: N+1 problem (1 + N queries)
$posts = Post::all(); // 1 query
foreach ($posts as $post) {
    echo $post->author->name; // N queries
}

// Good: Eager loading (2 queries)
$posts = Post::with('author')->get();
foreach ($posts as $post) {
    echo $post->author->name;
}

// Good: Multiple relationships
$posts = Post::with(['author', 'comments', 'tags'])->get();

// Good: Nested relationships
$posts = Post::with(['author.profile', 'comments.user'])->get();

// Good: Conditional eager loading
$posts = Post::query()
    ->when($includeAuthor, fn ($q) => $q->with('author'))
    ->when($includeComments, fn ($q) => $q->with('comments'))
    ->get();

// Good: Eager load specific columns
$posts = Post::with(['author:id,name,email'])->get();
```

### Query Optimization

Use efficient queries:

```php
// Bad: Loading unnecessary data
$users = User::all();
$count = $users->count();

// Good: Count in database
$count = User::count();

// Bad: exists() is faster than count() for existence checks
if (User::where('email', $email)->count() > 0) {
    // ...
}

// Good: Use exists()
if (User::where('email', $email)->exists()) {
    // ...
}

// Bad: Loading all records to check for first
$user = User::where('email', $email)->get()->first();

// Good: Use first() directly
$user = User::where('email', $email)->first();

// Good: Select only needed columns
$users = User::select(['id', 'name', 'email'])->get();

// Good: Chunk large datasets
User::chunk(200, function ($users) {
    foreach ($users as $user) {
        // Process user
    }
});

// Good: Lazy loading for memory efficiency
User::lazy()->each(function ($user) {
    // Process user
});
```

### Caching Patterns

Use caching strategically:

```php
// Good: Cache expensive queries
$stats = Cache::remember('dashboard.stats', now()->addHour(), function () {
    return [
        'users' => User::count(),
        'posts' => Post::count(),
        'comments' => Comment::count(),
    ];
});

// Good: Cache with tags for grouped invalidation
$posts = Cache::tags(['posts', 'blog'])->remember('posts.recent', 3600, function () {
    return Post::with('author')->latest()->take(10)->get();
});

// Invalidate tagged cache
Cache::tags(['posts', 'blog'])->flush();

// Good: Model cache pattern
class Post extends Model
{
    protected static function booted(): void
    {
        static::saved(function ($post) {
            Cache::forget("post.{$post->id}");
        });

        static::deleted(function ($post) {
            Cache::forget("post.{$post->id}");
        });
    }

    public static function findCached(int $id): ?self
    {
        return Cache::remember("post.{$id}", 3600, function () use ($id) {
            return self::find($id);
        });
    }
}
```

### Database Indexing

Add indexes for frequently queried columns:

```php
// Good: Index frequently searched columns
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();
    $table->string('slug')->unique();
    $table->string('title');
    $table->text('content');
    $table->timestamp('published_at')->nullable();
    $table->timestamps();

    // Indexes for common queries
    $table->index('published_at');
    $table->index(['user_id', 'published_at']);
    $table->index('slug'); // Redundant if unique, but showing pattern
});
```

## Testing Patterns (Pest 4)

### Test Structure

Use Pest's modern syntax:

```php
// Good: Pest test structure
use App\Models\User;
use App\Models\Post;

beforeEach(function () {
    $this->user = User::factory()->create();
});

it('can create a post', function () {
    $post = Post::factory()->create([
        'user_id' => $this->user->id,
        'title' => 'Test Post',
    ]);

    expect($post)
        ->title->toBe('Test Post')
        ->user_id->toBe($this->user->id);
});

it('requires authentication to create posts', function () {
    $response = $this->post(route('posts.store'), [
        'title' => 'Test Post',
        'content' => 'Test content',
    ]);

    $response->assertRedirect(route('login'));
});

it('validates required fields', function () {
    $this->actingAs($this->user)
        ->post(route('posts.store'), [])
        ->assertSessionHasErrors(['title', 'content']);
});
```

### Database Testing

Use RefreshDatabase trait:

```php
// Good: Database testing
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('stores user in database', function () {
    $user = User::factory()->create([
        'name' => 'John Doe',
        'email' => 'john@example.com',
    ]);

    $this->assertDatabaseHas('users', [
        'email' => 'john@example.com',
    ]);

    expect(User::count())->toBe(1);
});

it('soft deletes posts', function () {
    $post = Post::factory()->create();

    $post->delete();

    $this->assertSoftDeleted('posts', ['id' => $post->id]);
    expect(Post::count())->toBe(0);
    expect(Post::withTrashed()->count())->toBe(1);
});
```

### Factory Usage

Create comprehensive factories:

```php
// Good: Detailed factory
namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'title' => fake()->sentence(),
            'slug' => fake()->slug(),
            'content' => fake()->paragraphs(3, true),
            'published_at' => fake()->boolean(70) ? now() : null,
        ];
    }

    public function published(): static
    {
        return $this->state(fn (array $attributes) => [
            'published_at' => now(),
        ]);
    }

    public function draft(): static
    {
        return $this->state(fn (array $attributes) => [
            'published_at' => null,
        ]);
    }
}

// Usage in tests
$publishedPost = Post::factory()->published()->create();
$draftPost = Post::factory()->draft()->create();
$posts = Post::factory()->count(5)->published()->create();
```

### API Testing

Test API endpoints thoroughly:

```php
// Good: API endpoint testing
it('returns paginated posts', function () {
    Post::factory()->count(15)->create();

    $response = $this->getJson(route('api.posts.index'));

    $response
        ->assertOk()
        ->assertJsonStructure([
            'data' => [
                '*' => ['id', 'title', 'content', 'created_at'],
            ],
            'links',
            'meta',
        ])
        ->assertJsonCount(15, 'data');
});

it('creates post via API', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->postJson(route('api.posts.store'), [
            'title' => 'New Post',
            'content' => 'Post content',
        ]);

    $response
        ->assertCreated()
        ->assertJsonPath('data.title', 'New Post');

    expect(Post::count())->toBe(1);
});

it('validates API input', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->postJson(route('api.posts.store'), []);

    $response
        ->assertUnprocessable()
        ->assertJsonValidationErrors(['title', 'content']);
});
```

### Feature and Unit Tests

Distinguish between feature and unit tests:

```php
// Good: Feature test - tests user-facing functionality
// tests/Feature/PostManagementTest.php
it('allows authenticated users to create posts', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('posts.store'), [
            'title' => 'My Post',
            'content' => 'Post content',
        ])
        ->assertRedirect(route('posts.index'))
        ->assertSessionHas('success');

    expect(Post::count())->toBe(1);
});

// Good: Unit test - tests isolated logic
// tests/Unit/PostTest.php
it('generates slug from title', function () {
    $post = new Post();
    $post->title = 'My Awesome Post';

    expect($post->generateSlug())
        ->toBe('my-awesome-post');
});

it('checks if post is published', function () {
    $published = Post::factory()->make(['published_at' => now()]);
    $draft = Post::factory()->make(['published_at' => null]);

    expect($published->isPublished())->toBeTrue();
    expect($draft->isPublished())->toBeFalse();
});
```

## Common Pitfalls

### Pitfall 1: Not Using Eager Loading

```php
// Wrong: N+1 queries
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->user->name; // Executes query for each post
}

// Right: Eager loading
$posts = Post::with('user')->get();
foreach ($posts as $post) {
    echo $post->user->name; // No additional queries
}
```

### Pitfall 2: Using `all()` Instead of `get()`

```php
// Wrong: Can't chain after all()
$users = User::all()->where('active', true); // Won't work as expected

// Right: Use get() for query chaining
$users = User::where('active', true)->get();
```

### Pitfall 3: Not Validating Input

```php
// Wrong: No validation
public function store(Request $request)
{
    User::create($request->all());
}

// Right: Always validate
public function store(Request $request)
{
    $validated = $request->validate([
        'name' => 'required|max:255',
        'email' => 'required|email|unique:users',
    ]);

    User::create($validated);
}
```

### Pitfall 4: Exposing Mass Assignment Vulnerabilities

```php
// Wrong: No fillable/guarded protection
class User extends Model
{
    protected $guarded = [];
}

// Right: Define fillable fields
class User extends Model
{
    protected $fillable = ['name', 'email', 'password'];
}
```

### Pitfall 5: Not Using Transactions

```php
// Wrong: No transaction for multiple operations
public function transferFunds(User $from, User $to, int $amount): void
{
    $from->decrement('balance', $amount);
    $to->increment('balance', $amount);
    // If second operation fails, data is inconsistent
}

// Right: Use transactions
public function transferFunds(User $from, User $to, int $amount): void
{
    DB::transaction(function () use ($from, $to, $amount) {
        $from->decrement('balance', $amount);
        $to->increment('balance', $amount);
    });
}
```

### Pitfall 6: Using `env()` Outside Config Files

```php
// Wrong: Using env() in application code
if (env('APP_DEBUG')) {
    // This won't work after config caching
}

// Right: Use config() helper
if (config('app.debug')) {
    // This works after config caching
}
```

### Pitfall 7: Not Type Hinting

```php
// Wrong: No type hints
public function createUser($name, $email)
{
    return User::create(['name' => $name, 'email' => $email]);
}

// Right: Full type coverage
public function createUser(string $name, string $email): User
{
    return User::create(['name' => $name, 'email' => $email]);
}
```

### Pitfall 8: Fat Controllers

```php
// Wrong: Business logic in controller
class OrderController extends Controller
{
    public function store(Request $request)
    {
        $order = Order::create([...]);
        $order->items()->createMany([...]);
        $this->processPayment($order);
        $this->sendEmails($order);
        $this->updateInventory($order);
        return redirect()->route('orders.show', $order);
    }
}

// Right: Delegate to service
class OrderController extends Controller
{
    public function store(Request $request, OrderService $service)
    {
        $order = $service->create($request->validated());
        return redirect()->route('orders.show', $order);
    }
}
```

### Pitfall 9: Not Using Route Model Binding

```php
// Wrong: Manual model loading
Route::get('/posts/{id}', function ($id) {
    $post = Post::findOrFail($id);
    return view('posts.show', compact('post'));
});

// Right: Route model binding
Route::get('/posts/{post}', function (Post $post) {
    return view('posts.show', compact('post'));
});
```

### Pitfall 10: Ignoring Database Indexes

```php
// Wrong: No index on frequently queried column
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('slug'); // Frequently queried, but not indexed
    $table->timestamp('published_at'); // Frequently queried, but not indexed
});

// Right: Add indexes
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('slug')->unique();
    $table->timestamp('published_at')->index();
});
```

## Best Practices Summary

1. **Follow Laravel Conventions** - Use framework patterns before creating custom solutions
2. **Type Everything** - Use type hints for parameters, return types, and properties
3. **Keep Controllers Thin** - Move business logic to services or actions
4. **Use Form Requests** - Validate complex input with Form Request classes
5. **Eager Load Relationships** - Prevent N+1 queries with `with()`
6. **Use Enums** - Replace string/int constants with typed enums
7. **Always Validate Input** - Never trust user input, validate everything
8. **Write Readable Code** - Self-documenting code over comments
9. **Use Modern Syntax** - Leverage PHP 8+ features and Laravel 11-12 patterns
10. **Test Thoroughly** - Feature tests for functionality, unit tests for logic
11. **Secure by Default** - Use CSRF protection, validate input, check permissions
12. **Optimize Queries** - Select only needed columns, use indexes, cache expensive queries
13. **Use Transactions** - Wrap multiple database operations in transactions
14. **Index Strategically** - Add indexes for foreign keys and frequently queried columns
15. **Consistent Naming** - camelCase methods, snake_case database, kebab-case routes
16. **Avoid Pitfalls** - Don't use `env()` outside config, don't skip eager loading
17. **Use Route Model Binding** - Leverage automatic model resolution
18. **Cache Smartly** - Cache expensive operations, invalidate on changes
19. **Organize Models** - Follow consistent property and method ordering
20. **Embrace Attributes** - Use PHP 8 attributes for routes and metadata

## Reference Links

- [Laravel Documentation](https://laravel.com/docs)
- [Spatie Guidelines](https://spatie.be/guidelines/laravel-php)
- [PSR-12 Standard](https://www.php-fig.org/psr/psr-12/)
- [Pest PHP](https://pestphp.com)

---

**Version:** 1.0.0
**Last Updated:** 2025-11-11
**Applies to:** Laravel 11-12, PHP 8.2+, Livewire 4, Pest 4
