---
name: laravel-security-patterns
description: Master Laravel security with CSRF protection, XSS prevention, SQL injection defense, mass assignment protection, rate limiting, authentication, authorization, input validation, and comprehensive security best practices. Use when securing applications, preventing attacks, or implementing defense-in-depth strategies.
---

# Laravel Security Patterns

Comprehensive guide to securing Laravel applications with CSRF protection, XSS prevention, SQL injection defense, mass assignment protection, authentication, authorization, rate limiting, input validation, and building secure, production-ready applications.

## When to Use This Skill

- Protecting applications from common web vulnerabilities
- Implementing secure authentication and authorization
- Preventing CSRF, XSS, and SQL injection attacks
- Validating and sanitizing user input
- Setting up rate limiting and throttling
- Securing API endpoints and routes
- Managing sensitive data and secrets
- Implementing two-factor authentication
- Auditing security logs and events
- Building compliance-ready applications

## Core Concepts

### 1. OWASP Top 10 in Laravel
- **Injection**: Prevented by Eloquent ORM and prepared statements
- **Broken Authentication**: Laravel's built-in auth system
- **XSS**: Blade automatic escaping
- **CSRF**: Built-in token verification
- **Security Misconfiguration**: Environment-based configs

### 2. Defense in Depth
- Multiple layers of security
- Input validation and output encoding
- Secure defaults
- Principle of least privilege

### 3. Authentication vs Authorization
- **Authentication**: Who are you? (Login)
- **Authorization**: What can you do? (Permissions)

### 4. Input Validation
- Server-side validation always required
- Whitelist over blacklist
- Type checking and sanitization

### 5. Secure Configuration
- Environment variables for secrets
- HTTPS in production
- Secure session configuration
- Proper error handling

## Quick Start

```php
<?php

// CSRF Protection (automatic in forms)
<form method="POST" action="/profile">
    @csrf
    <!-- form fields -->
</form>

// Input Validation
$request->validate([
    'email' => 'required|email',
    'password' => 'required|min:8',
]);

// XSS Prevention (Blade auto-escapes)
{{ $userInput }} // Escaped
{!! $trustedHtml !!} // Unescaped

// SQL Injection Prevention (use Eloquent)
User::where('email', $email)->first(); // Safe

// Mass Assignment Protection
protected $fillable = ['name', 'email'];
protected $guarded = ['role', 'is_admin'];

// Authorization
$this->authorize('update', $post);
Gate::allows('delete', $post);

// Rate Limiting
Route::middleware('throttle:60,1')->group(function () {
    // Routes
});
```

## Fundamental Patterns

### Pattern 1: CSRF Protection

```php
<?php

// CSRF tokens automatically included in forms
// app/Http/Middleware/VerifyCsrfToken.php
namespace App\Http\Middleware;

use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken as Middleware;

class VerifyCsrfToken extends Middleware
{
    /**
     * URIs that should be excluded from CSRF verification.
     */
    protected $except = [
        'api/*',           // API routes typically use tokens
        'webhooks/*',      // Webhook endpoints
        'stripe/webhook',  // Third-party webhooks
    ];
}

// Blade forms automatically include CSRF token
<form method="POST" action="/posts">
    @csrf
    <!-- Form fields -->
</form>

// AJAX requests with CSRF token
<meta name="csrf-token" content="{{ csrf_token() }}">

<script>
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});

// Or with Axios
axios.defaults.headers.common['X-CSRF-TOKEN'] = document.querySelector('meta[name="csrf-token"]').getAttribute('content');

// Make AJAX request
$.post('/api/posts', data);
</script>

// Manual CSRF token validation in controller
public function store(Request $request)
{
    if (!$request->session()->token() === $request->input('_token')) {
        abort(403, 'CSRF token mismatch');
    }

    // Process request
}

// CSRF token in API responses for forms
public function create()
{
    return response()->json([
        'csrf_token' => csrf_token(),
        'form_data' => // ...
    ]);
}

// Regenerate CSRF token after authentication
Auth::login($user);
$request->session()->regenerateToken();
```

### Pattern 2: XSS Prevention

```php
<?php

// Blade templates automatically escape output
// Safe by default
<div>{{ $userInput }}</div>
<div>{{ $request->input('comment') }}</div>

// Display HTML safely (use with caution)
<div>{!! Purifier::clean($userHtml) !!}</div>

// Install HTML Purifier
composer require mews/purifier

// Configure purifier - config/purifier.php
return [
    'encoding' => 'UTF-8',
    'finalize' => true,
    'cachePath' => storage_path('app/purifier'),
    'settings' => [
        'default' => [
            'HTML.Doctype' => 'HTML 4.01 Transitional',
            'HTML.Allowed' => 'p,b,i,strong,em,a[href],ul,ol,li',
            'AutoFormat.RemoveEmpty' => true,
        ],
    ],
];

// Use Purifier in controller
use Mews\Purifier\Facades\Purifier;

public function store(Request $request)
{
    $request->validate([
        'content' => 'required|string',
    ]);

    Post::create([
        'content' => Purifier::clean($request->content),
    ]);
}

// Create custom sanitization helper
namespace App\Helpers;

class SecurityHelper
{
    /**
     * Sanitize user input for XSS prevention.
     */
    public static function sanitize(string $input): string
    {
        return htmlspecialchars($input, ENT_QUOTES, 'UTF-8');
    }

    /**
     * Strip all HTML tags.
     */
    public static function stripTags(string $input): string
    {
        return strip_tags($input);
    }

    /**
     * Allow only safe HTML tags.
     */
    public static function sanitizeHtml(string $html): string
    {
        return Purifier::clean($html, [
            'HTML.Allowed' => 'p,b,i,strong,em,a[href],ul,ol,li,br',
        ]);
    }
}

// Use in views
<div>{{ SecurityHelper::sanitize($userInput) }}</div>

// Content Security Policy (CSP)
// Add to middleware or header
public function handle($request, Closure $next)
{
    $response = $next($request);

    $response->headers->set('Content-Security-Policy',
        "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';"
    );

    return $response;
}

// Escape JSON for embedding in HTML
<script>
    var data = @json($data); // Automatically escaped
</script>

// NEVER do this (vulnerable to XSS)
<script>
    var data = {!! json_encode($data) !!}; // Unescaped
</script>
```

### Pattern 3: SQL Injection Prevention

```php
<?php

// ALWAYS use Eloquent or Query Builder (parameterized queries)

// Safe: Eloquent
$user = User::where('email', $email)->first();
$posts = Post::where('user_id', $userId)->get();

// Safe: Query Builder with bindings
$users = DB::table('users')
    ->where('email', $email)
    ->where('active', true)
    ->get();

// Safe: Named bindings
$users = DB::select('select * from users where email = :email', [
    'email' => $email
]);

// Safe: Positional bindings
$users = DB::select('select * from users where email = ?', [$email]);

// DANGEROUS: Raw queries (NEVER do this)
// $users = DB::select("select * from users where email = '$email'"); // VULNERABLE!

// If you MUST use raw SQL, use bindings
$users = DB::select(
    DB::raw('select * from users where email = ?'),
    [$email]
);

// Safe: whereRaw with bindings
$posts = Post::whereRaw('views > ? AND created_at > ?', [100, $date])->get();

// Safe: Dynamic where clauses
$query = User::query();

if ($request->has('email')) {
    $query->where('email', $request->email);
}

if ($request->has('status')) {
    $query->where('status', $request->status);
}

$users = $query->get();

// Validate input before queries
$request->validate([
    'email' => 'required|email',
    'status' => 'required|in:active,inactive,pending',
]);

// Use enum for constants
enum UserStatus: string
{
    case Active = 'active';
    case Inactive = 'inactive';
    case Pending = 'pending';
}

$users = User::where('status', UserStatus::Active->value)->get();
```

### Pattern 4: Mass Assignment Protection

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Whitelist: Only these fields can be mass assigned.
     */
    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    /**
     * Blacklist: These fields cannot be mass assigned.
     */
    protected $guarded = [
        'id',
        'is_admin',
        'role',
        'remember_token',
    ];

    /**
     * Completely disable mass assignment protection (DANGEROUS).
     */
    // protected $guarded = []; // Don't use unless you know what you're doing
}

// Safe usage in controller
public function store(Request $request)
{
    $request->validate([
        'name' => 'required|string|max:255',
        'email' => 'required|email|unique:users',
        'password' => 'required|min:8',
    ]);

    // Only validated fields are assigned
    $user = User::create([
        'name' => $request->name,
        'email' => $request->email,
        'password' => Hash::make($request->password),
    ]);

    return response()->json($user, 201);
}

// DANGEROUS: Don't use request->all()
// $user = User::create($request->all()); // Can set is_admin!

// Safe: Use only()
$user = User::create($request->only(['name', 'email']));

// Safe: Use validated()
$user = User::create($request->validated());

// Force fill (bypass mass assignment protection)
$user = new User();
$user->forceFill([
    'name' => $name,
    'is_admin' => true, // Would normally be guarded
])->save();

// Make attribute mass assignable temporarily
User::reguard(); // Enable mass assignment protection
User::unguard(); // Disable (use cautiously)

// Protect sensitive attributes
class User extends Model
{
    protected $hidden = [
        'password',
        'remember_token',
        'two_factor_secret',
    ];

    protected $visible = [
        'id',
        'name',
        'email',
        'created_at',
    ];
}
```

### Pattern 5: Authentication Security

```php
<?php

// Strong password requirements
use Illuminate\Validation\Rules\Password;

public function register(Request $request)
{
    $request->validate([
        'email' => 'required|email|unique:users',
        'password' => [
            'required',
            'confirmed',
            Password::min(8)
                ->mixedCase()
                ->numbers()
                ->symbols()
                ->uncompromised(), // Check against data breaches
        ],
    ]);

    $user = User::create([
        'email' => $request->email,
        'password' => Hash::make($request->password),
    ]);

    return response()->json($user, 201);
}

// Check password strength
use Illuminate\Support\Facades\Hash;

public function updatePassword(Request $request)
{
    $request->validate([
        'current_password' => 'required',
        'password' => ['required', 'confirmed', Password::defaults()],
    ]);

    $user = $request->user();

    // Verify current password
    if (!Hash::check($request->current_password, $user->password)) {
        return back()->withErrors(['current_password' => 'Incorrect password']);
    }

    // Update password
    $user->update([
        'password' => Hash::make($request->password),
    ]);

    // Logout other devices
    Auth::logoutOtherDevices($request->password);

    return back()->with('status', 'Password updated successfully');
}

// Session security
// config/session.php
return [
    'lifetime' => 120, // Minutes
    'expire_on_close' => true,
    'secure' => env('SESSION_SECURE_COOKIE', true), // HTTPS only
    'http_only' => true, // Prevent JavaScript access
    'same_site' => 'strict', // CSRF protection
];

// Regenerate session on login
public function login(Request $request)
{
    $credentials = $request->only('email', 'password');

    if (Auth::attempt($credentials)) {
        $request->session()->regenerate(); // Prevent session fixation

        return redirect()->intended('dashboard');
    }

    return back()->withErrors(['email' => 'Invalid credentials']);
}

// Failed login attempts tracking
use Illuminate\Support\Facades\RateLimiter;

public function login(Request $request)
{
    $key = 'login.' . $request->ip();

    if (RateLimiter::tooManyAttempts($key, 5)) {
        $seconds = RateLimiter::availableIn($key);

        return back()->withErrors([
            'email' => "Too many login attempts. Please try again in {$seconds} seconds."
        ]);
    }

    if (Auth::attempt($request->only('email', 'password'))) {
        RateLimiter::clear($key);
        $request->session()->regenerate();

        return redirect()->intended('dashboard');
    }

    RateLimiter::hit($key, 60); // Block for 60 seconds

    return back()->withErrors(['email' => 'Invalid credentials']);
}

// Account lockout after failed attempts
class User extends Model
{
    protected $fillable = [
        'name', 'email', 'password',
        'failed_login_attempts', 'locked_until',
    ];

    protected $casts = [
        'locked_until' => 'datetime',
    ];

    public function isLocked(): bool
    {
        return $this->locked_until && $this->locked_until->isFuture();
    }

    public function incrementFailedLogins(): void
    {
        $this->increment('failed_login_attempts');

        if ($this->failed_login_attempts >= 5) {
            $this->update(['locked_until' => now()->addMinutes(30)]);
        }
    }

    public function resetFailedLogins(): void
    {
        $this->update([
            'failed_login_attempts' => 0,
            'locked_until' => null,
        ]);
    }
}
```

### Pattern 6: Authorization with Gates and Policies

```php
<?php

// Define Gates in AuthServiceProvider
use Illuminate\Support\Facades\Gate;

class AuthServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Gate::define('update-post', function (User $user, Post $post) {
            return $user->id === $post->user_id;
        });

        Gate::define('delete-post', function (User $user, Post $post) {
            return $user->id === $post->user_id || $user->isAdmin();
        });

        Gate::define('view-admin', function (User $user) {
            return $user->isAdmin();
        });
    }
}

// Use Gates in controllers
public function update(Request $request, Post $post)
{
    if (Gate::denies('update-post', $post)) {
        abort(403, 'Unauthorized action');
    }

    // Or use authorize helper
    $this->authorize('update-post', $post);

    $post->update($request->validated());

    return redirect()->back();
}

// Create Policy
php artisan make:policy PostPolicy --model=Post

// app/Policies/PostPolicy.php
namespace App\Policies;

class PostPolicy
{
    /**
     * Determine if user can view post.
     */
    public function view(User $user, Post $post): bool
    {
        return $post->published || $user->id === $post->user_id;
    }

    /**
     * Determine if user can create posts.
     */
    public function create(User $user): bool
    {
        return $user->email_verified_at !== null;
    }

    /**
     * Determine if user can update post.
     */
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    /**
     * Determine if user can delete post.
     */
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->isAdmin();
    }

    /**
     * Admin can do anything.
     */
    public function before(User $user, string $ability): ?bool
    {
        if ($user->isAdmin()) {
            return true;
        }

        return null;
    }
}

// Register policy
protected $policies = [
    Post::class => PostPolicy::class,
];

// Use in controllers
public function update(Request $request, Post $post)
{
    $this->authorize('update', $post);

    $post->update($request->validated());

    return redirect()->back();
}

// Use in Blade
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan

@cannot('delete', $post)
    <p>You cannot delete this post</p>
@endcannot

// Use in routes
Route::middleware('can:update,post')->group(function () {
    Route::put('/posts/{post}', [PostController::class, 'update']);
});

// Check multiple abilities
if ($user->can('update', $post) && $user->can('publish', $post)) {
    // User can update and publish
}

// Check any ability
if ($user->canAny(['update', 'delete'], $post)) {
    // User can update OR delete
}
```

### Pattern 7: Rate Limiting and Throttling

```php
<?php

// Basic route throttling
Route::middleware('throttle:60,1')->group(function () {
    Route::post('/posts', [PostController::class, 'store']);
});

// Different limits for authenticated users
Route::middleware('throttle:api')->group(function () {
    // Defined in RouteServiceProvider
});

// Custom rate limiters in RouteServiceProvider
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    RateLimiter::for('api', function (Request $request) {
        return $request->user()
            ? Limit::perMinute(100)->by($request->user()->id)
            : Limit::perMinute(10)->by($request->ip());
    });

    RateLimiter::for('uploads', function (Request $request) {
        return Limit::perMinute(5)->by($request->user()->id);
    });

    RateLimiter::for('login', function (Request $request) {
        return Limit::perMinute(5)->by($request->email . $request->ip());
    });

    // Multiple limits
    RateLimiter::for('strict', function (Request $request) {
        return [
            Limit::perMinute(10)->by($request->user()->id),
            Limit::perHour(100)->by($request->user()->id),
            Limit::perDay(1000)->by($request->user()->id),
        ];
    });
}

// Apply to routes
Route::middleware('throttle:uploads')->post('/files', [FileController::class, 'upload']);
Route::middleware('throttle:login')->post('/login', [AuthController::class, 'login']);

// Custom middleware for rate limiting
namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\RateLimiter;

class ThrottleRequests
{
    public function handle(Request $request, Closure $next, int $maxAttempts = 60)
    {
        $key = $this->resolveRequestSignature($request);

        if (RateLimiter::tooManyAttempts($key, $maxAttempts)) {
            return response()->json([
                'message' => 'Too many requests',
                'retry_after' => RateLimiter::availableIn($key),
            ], 429);
        }

        RateLimiter::hit($key, 60);

        $response = $next($request);

        return $this->addHeaders($response, $maxAttempts, $key);
    }

    protected function resolveRequestSignature(Request $request): string
    {
        if ($user = $request->user()) {
            return sha1($user->id);
        }

        return sha1($request->ip());
    }

    protected function addHeaders($response, $maxAttempts, $key)
    {
        $response->headers->add([
            'X-RateLimit-Limit' => $maxAttempts,
            'X-RateLimit-Remaining' => RateLimiter::remaining($key, $maxAttempts),
        ]);

        return $response;
    }
}

// IP-based blocking
class IpBlockingService
{
    public function block(string $ip, int $minutes = 60): void
    {
        Cache::put("blocked_ip:{$ip}", true, now()->addMinutes($minutes));
    }

    public function isBlocked(string $ip): bool
    {
        return Cache::has("blocked_ip:{$ip}");
    }

    public function unblock(string $ip): void
    {
        Cache::forget("blocked_ip:{$ip}");
    }
}

// Middleware to check blocked IPs
class BlockedIpMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $ipService = app(IpBlockingService::class);

        if ($ipService->isBlocked($request->ip())) {
            abort(403, 'Your IP has been blocked');
        }

        return $next($request);
    }
}
```

### Pattern 8: Input Validation and Sanitization

```php
<?php

// Form Request validation
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'title' => 'required|string|max:255',
            'slug' => 'required|alpha_dash|unique:posts',
            'content' => 'required|string',
            'category_id' => 'required|exists:categories,id',
            'tags' => 'array',
            'tags.*' => 'exists:tags,id',
            'status' => 'in:draft,published',
            'published_at' => 'nullable|date|after:now',
            'featured_image' => 'nullable|image|max:2048|dimensions:min_width=800,min_height=600',
        ];
    }

    /**
     * Prepare data for validation.
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => Str::slug($this->title),
        ]);
    }

    /**
     * Get validated and sanitized data.
     */
    public function sanitized(): array
    {
        $data = $this->validated();

        // Strip tags from content
        $data['content'] = strip_tags($data['content'], '<p><br><strong><em><a>');

        // Sanitize HTML
        $data['content'] = Purifier::clean($data['content']);

        return $data;
    }
}

// Custom validation rules
php artisan make:rule NoScriptTag

namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;

class NoScriptTag implements Rule
{
    public function passes($attribute, $value): bool
    {
        return !preg_match('/<script\b[^>]*>(.*?)<\/script>/is', $value);
    }

    public function message(): string
    {
        return 'The :attribute contains prohibited script tags.';
    }
}

// Use custom rule
$request->validate([
    'content' => ['required', new NoScriptTag],
]);

// Validate file uploads
$request->validate([
    'avatar' => 'required|image|mimes:jpeg,png,jpg|max:2048',
    'document' => 'required|mimetypes:application/pdf|max:10240',
]);

// Validate URLs
$request->validate([
    'website' => 'required|url|active_url',
]);

// Validate JSON
$request->validate([
    'metadata' => 'required|json',
]);

// Sanitize helper functions
class InputSanitizer
{
    public static function sanitizeString(string $input): string
    {
        return htmlspecialchars(trim($input), ENT_QUOTES, 'UTF-8');
    }

    public static function sanitizeEmail(string $email): string
    {
        return filter_var($email, FILTER_SANITIZE_EMAIL);
    }

    public static function sanitizeUrl(string $url): string
    {
        return filter_var($url, FILTER_SANITIZE_URL);
    }

    public static function sanitizeFilename(string $filename): string
    {
        return preg_replace('/[^a-zA-Z0-9._-]/', '', $filename);
    }
}
```

### Pattern 9: Secure File Uploads

```php
<?php

class FileUploadService
{
    /**
     * Validate and store uploaded file.
     */
    public function upload(UploadedFile $file, string $path = 'uploads'): string
    {
        // Validate file type
        $allowedMimes = ['image/jpeg', 'image/png', 'image/gif'];

        if (!in_array($file->getMimeType(), $allowedMimes)) {
            throw new \Exception('Invalid file type');
        }

        // Validate file size (max 5MB)
        if ($file->getSize() > 5 * 1024 * 1024) {
            throw new \Exception('File too large');
        }

        // Generate safe filename
        $filename = Str::random(40) . '.' . $file->getClientOriginalExtension();

        // Store outside public directory
        $filePath = $file->storeAs($path, $filename, 'private');

        return $filePath;
    }

    /**
     * Serve private file.
     */
    public function serve(string $path)
    {
        if (!Storage::disk('private')->exists($path)) {
            abort(404);
        }

        return Storage::disk('private')->response($path);
    }

    /**
     * Validate image dimensions.
     */
    public function validateImage(UploadedFile $file): bool
    {
        [$width, $height] = getimagesize($file->getRealPath());

        return $width >= 800 && $height >= 600;
    }

    /**
     * Scan file for malware (using ClamAV).
     */
    public function scanFile(UploadedFile $file): bool
    {
        $clam = new \Xenolope\Quahog\Client(
            new \Socket\Raw\Factory(),
            'unix:///var/run/clamav/clamd.sock'
        );

        $result = $clam->scanFile($file->getRealPath());

        return $result['status'] === 'OK';
    }
}

// Controller usage
public function upload(Request $request)
{
    $request->validate([
        'file' => 'required|file|mimes:jpg,png,pdf|max:5120',
    ]);

    $service = new FileUploadService();

    // Scan for malware
    if (!$service->scanFile($request->file('file'))) {
        return back()->withErrors(['file' => 'File contains malware']);
    }

    $path = $service->upload($request->file('file'));

    return back()->with('success', 'File uploaded successfully');
}

// Serve protected file
Route::middleware('auth')->get('/files/{file}', function ($file) {
    $user = auth()->user();

    // Check authorization
    $fileRecord = File::where('filename', $file)->firstOrFail();

    if ($fileRecord->user_id !== $user->id) {
        abort(403);
    }

    return Storage::disk('private')->download($fileRecord->path);
});
```

### Pattern 10: Protecting Sensitive Data

```php
<?php

// Encrypt sensitive data
use Illuminate\Support\Facades\Crypt;

class User extends Model
{
    protected $fillable = [
        'name', 'email', 'password',
        'phone', 'ssn', 'credit_card',
    ];

    // Cast attributes to encrypted
    protected $casts = [
        'ssn' => 'encrypted',
        'credit_card' => 'encrypted',
    ];

    // Manual encryption
    public function setPhoneAttribute($value): void
    {
        $this->attributes['phone'] = Crypt::encryptString($value);
    }

    public function getPhoneAttribute($value): ?string
    {
        return $value ? Crypt::decryptString($value) : null;
    }
}

// Store API keys securely
// Use environment variables
API_KEY=your-secret-key
AWS_SECRET_KEY=your-aws-secret

// Access in code
$apiKey = config('services.api.key');

// Never commit .env file
// Add to .gitignore
.env
.env.*

// Hash sensitive tokens
use Illuminate\Support\Facades\Hash;

$token = Hash::make(Str::random(60));

// Verify token
if (Hash::check($inputToken, $storedToken)) {
    // Token valid
}

// Use database encryption for specific columns
// In migration
$table->string('credit_card')->nullable();

// In model with accessor/mutator
protected function creditCard(): Attribute
{
    return Attribute::make(
        get: fn ($value) => $value ? Crypt::decryptString($value) : null,
        set: fn ($value) => $value ? Crypt::encryptString($value) : null,
    );
}

// Secure session data
config(['session.encrypt' => true]);

// Hide sensitive attributes from JSON
class User extends Model
{
    protected $hidden = [
        'password',
        'remember_token',
        'two_factor_secret',
        'two_factor_recovery_codes',
    ];
}

// Redact sensitive data in logs
Log::info('User registered', [
    'user_id' => $user->id,
    'email' => Str::mask($user->email, '*', 3, -10),
]);

// Secure password reset tokens
// config/auth.php
'passwords' => [
    'users' => [
        'provider' => 'users',
        'table' => 'password_reset_tokens',
        'expire' => 60, // Minutes
        'throttle' => 60, // Seconds between attempts
    ],
],
```

## Advanced Patterns

### Pattern 11: Two-Factor Authentication

```php
<?php

// Install Laravel Fortify
composer require laravel/fortify

// Enable 2FA in fortify config
'features' => [
    Features::twoFactorAuthentication([
        'confirm' => true,
        'confirmPassword' => true,
    ]),
],

// Custom 2FA implementation
class TwoFactorService
{
    public function enable(User $user): array
    {
        $secret = $this->generateSecret();

        $user->forceFill([
            'two_factor_secret' => encrypt($secret),
            'two_factor_recovery_codes' => encrypt(json_encode($this->generateRecoveryCodes())),
        ])->save();

        return [
            'secret' => $secret,
            'qr_code' => $this->getQrCodeUrl($user, $secret),
        ];
    }

    public function verify(User $user, string $code): bool
    {
        $secret = decrypt($user->two_factor_secret);

        $google2fa = app(\PragmaRX\Google2FA\Google2FA::class);

        return $google2fa->verifyKey($secret, $code);
    }

    protected function generateSecret(): string
    {
        $google2fa = app(\PragmaRX\Google2FA\Google2FA::class);

        return $google2fa->generateSecretKey();
    }

    protected function generateRecoveryCodes(): array
    {
        return collect(range(1, 8))->map(function () {
            return Str::random(10) . '-' . Str::random(10);
        })->all();
    }

    protected function getQrCodeUrl(User $user, string $secret): string
    {
        $google2fa = app(\PragmaRX\Google2FA\Google2FA::class);

        return $google2fa->getQRCodeUrl(
            config('app.name'),
            $user->email,
            $secret
        );
    }
}
```

### Pattern 12: Security Headers

```php
<?php

// Security headers middleware
namespace App\Http\Middleware;

class SecurityHeaders
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-Frame-Options', 'SAMEORIGIN');
        $response->headers->set('X-XSS-Protection', '1; mode=block');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->headers->set('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');

        // HSTS (only in production with HTTPS)
        if (config('app.env') === 'production') {
            $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
        }

        // Content Security Policy
        $response->headers->set('Content-Security-Policy', $this->getCsp());

        return $response;
    }

    protected function getCsp(): string
    {
        return implode('; ', [
            "default-src 'self'",
            "script-src 'self' 'unsafe-inline' 'unsafe-eval'",
            "style-src 'self' 'unsafe-inline'",
            "img-src 'self' data: https:",
            "font-src 'self' data:",
            "connect-src 'self'",
            "frame-ancestors 'none'",
        ]);
    }
}

// Register middleware
protected $middlewareGroups = [
    'web' => [
        \App\Http\Middleware\SecurityHeaders::class,
        // ...
    ],
];
```

### Pattern 13: Security Auditing

```php
<?php

// Audit log model
class AuditLog extends Model
{
    protected $fillable = [
        'user_id',
        'action',
        'model_type',
        'model_id',
        'ip_address',
        'user_agent',
        'changes',
    ];

    protected $casts = [
        'changes' => 'array',
    ];
}

// Trait for auditable models
trait Auditable
{
    protected static function bootAuditable(): void
    {
        static::created(function ($model) {
            $model->audit('created');
        });

        static::updated(function ($model) {
            $model->audit('updated');
        });

        static::deleted(function ($model) {
            $model->audit('deleted');
        });
    }

    public function audit(string $action): void
    {
        AuditLog::create([
            'user_id' => auth()->id(),
            'action' => $action,
            'model_type' => get_class($this),
            'model_id' => $this->getKey(),
            'ip_address' => request()->ip(),
            'user_agent' => request()->userAgent(),
            'changes' => $this->getDirty(),
        ]);
    }
}

// Use on models
class Post extends Model
{
    use Auditable;
}
```

## Real-World Applications

### Application 1: Complete Secure API

```php
<?php

// Secure API endpoint
Route::middleware([
    'auth:sanctum',
    'throttle:api',
    'verified',
])->group(function () {
    Route::apiResource('posts', PostController::class);
});

class PostController extends Controller
{
    public function store(StorePostRequest $request)
    {
        $this->authorize('create', Post::class);

        $post = Post::create([
            'user_id' => auth()->id(),
            'title' => $request->title,
            'content' => Purifier::clean($request->content),
            'status' => 'draft',
        ]);

        AuditLog::create([
            'user_id' => auth()->id(),
            'action' => 'post.created',
            'model_type' => Post::class,
            'model_id' => $post->id,
        ]);

        return new PostResource($post);
    }
}
```

## Performance Best Practices

### Practice 1: Use Prepared Statements

```php
// Always use Eloquent or Query Builder
User::where('email', $email)->first();
```

### Practice 2: Hash Passwords

```php
// Always hash passwords
Hash::make($password);
```

### Practice 3: Validate Input

```php
// Always validate
$request->validate(['email' => 'required|email']);
```

## Common Pitfalls

### Pitfall 1: Not Validating Input

```php
// WRONG
User::create($request->all());

// CORRECT
User::create($request->validated());
```

### Pitfall 2: Storing Plain Passwords

```php
// WRONG
$user->password = $request->password;

// CORRECT
$user->password = Hash::make($request->password);
```

### Pitfall 3: Exposing Sensitive Data

```php
// WRONG
return $user;

// CORRECT
return new UserResource($user);
```

### Pitfall 4: No Rate Limiting

```php
// WRONG
Route::post('/login', [AuthController::class, 'login']);

// CORRECT
Route::middleware('throttle:login')->post('/login', [AuthController::class, 'login']);
```

### Pitfall 5: Trusting User Input

```php
// WRONG
{!! $request->input('comment') !!}

// CORRECT
{{ $request->input('comment') }}
```

## Testing

```php
<?php

test('csrf protection works', function () {
    $this->post('/posts')->assertStatus(419);

    $this->post('/posts', [], ['X-CSRF-TOKEN' => csrf_token()])
        ->assertStatus(302);
});

test('rate limiting works', function () {
    for ($i = 0; $i < 60; $i++) {
        $this->get('/api/posts')->assertStatus(200);
    }

    $this->get('/api/posts')->assertStatus(429);
});

test('authorization works', function () {
    $user = User::factory()->create();
    $post = Post::factory()->create();

    actingAs($user)
        ->put("/posts/{$post->id}")
        ->assertStatus(403);
});
```

## Resources

- **Laravel Security Documentation**: https://laravel.com/docs/security
- **OWASP Top 10**: https://owasp.org/www-project-top-ten/
- **Laravel Fortify**: https://laravel.com/docs/fortify
- **Laravel Sanctum**: https://laravel.com/docs/sanctum
- **Security Headers**: https://securityheaders.com/

## Best Practices Summary

1. **Always validate input** on the server side
2. **Use Eloquent/Query Builder** to prevent SQL injection
3. **Hash all passwords** with bcrypt or argon2
4. **Enable CSRF protection** for all forms
5. **Escape output** in Blade templates
6. **Implement rate limiting** on sensitive endpoints
7. **Use authorization** with gates and policies
8. **Store secrets** in environment variables
9. **Enable security headers** in production
10. **Audit security events** and monitor logs
