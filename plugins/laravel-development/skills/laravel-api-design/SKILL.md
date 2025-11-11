---
name: laravel-api-design
description: Master RESTful API design with Laravel including API resources, Sanctum authentication, rate limiting, pagination, versioning, error handling, and comprehensive API development patterns. Use when building REST APIs, mobile backends, or third-party integrations.
---

# Laravel API Design

Comprehensive guide to designing and building RESTful APIs in Laravel with API resources, authentication, authorization, rate limiting, versioning, pagination, error handling, and best practices for scalable API development.

## When to Use This Skill

- Building RESTful APIs for mobile or web applications
- Creating backend services for SPA frameworks
- Developing third-party API integrations
- Implementing microservices architectures
- Building public or internal API endpoints
- Versioning APIs for backward compatibility
- Implementing API authentication and authorization
- Designing API documentation and contracts
- Rate limiting and throttling API requests
- Handling API errors and validation consistently

## Core Concepts

### 1. RESTful Principles
- **Resources**: API endpoints represent resources (users, posts, etc.)
- **HTTP Methods**: GET, POST, PUT/PATCH, DELETE for CRUD operations
- **Status Codes**: Proper HTTP status codes for responses
- **Stateless**: Each request contains all needed information

### 2. API Resources
- Transform Eloquent models to JSON responses
- Control what data is exposed
- Include relationships conditionally
- Format dates and complex data types

### 3. Authentication
- **Token-based**: Sanctum for SPA and mobile apps
- **OAuth2**: For third-party integrations
- **API Keys**: For simple service-to-service auth

### 4. Rate Limiting
- Protect API from abuse
- Throttle requests per user/IP
- Different limits for different endpoints

### 5. Versioning
- Maintain backward compatibility
- Allow clients to migrate gradually
- Support multiple API versions simultaneously

## Quick Start

```php
<?php

// Install Sanctum
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate

// Create API resource
php artisan make:resource UserResource

// Define API routes - routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('posts', PostController::class);
});

// API Controller
class PostController extends Controller
{
    public function index()
    {
        return PostResource::collection(Post::paginate());
    }

    public function store(Request $request)
    {
        $post = Post::create($request->validated());
        return new PostResource($post);
    }
}

// Test API
curl -X GET http://api.example.com/api/posts \
  -H "Accept: application/json" \
  -H "Authorization: Bearer {token}"
```

## Fundamental Patterns

### Pattern 1: RESTful API Controllers

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Http\Resources\PostResource;
use App\Models\Post;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class PostController extends Controller
{
    /**
     * Display a listing of posts.
     */
    public function index(): AnonymousResourceCollection
    {
        $posts = Post::with(['author', 'category'])
            ->latest()
            ->paginate(20);

        return PostResource::collection($posts);
    }

    /**
     * Store a newly created post.
     */
    public function store(StorePostRequest $request): JsonResponse
    {
        $post = Post::create([
            'user_id' => $request->user()->id,
            'title' => $request->title,
            'content' => $request->content,
            'status' => 'draft',
        ]);

        return (new PostResource($post))
            ->response()
            ->setStatusCode(201);
    }

    /**
     * Display the specified post.
     */
    public function show(Post $post): PostResource
    {
        $post->load(['author', 'category', 'tags', 'comments']);

        return new PostResource($post);
    }

    /**
     * Update the specified post.
     */
    public function update(UpdatePostRequest $request, Post $post): PostResource
    {
        $this->authorize('update', $post);

        $post->update($request->validated());

        return new PostResource($post);
    }

    /**
     * Remove the specified post.
     */
    public function destroy(Post $post): JsonResponse
    {
        $this->authorize('delete', $post);

        $post->delete();

        return response()->json(null, 204);
    }
}

// routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('posts', PostController::class);

    // Additional custom endpoints
    Route::post('posts/{post}/publish', [PostController::class, 'publish']);
    Route::get('posts/{post}/related', [PostController::class, 'related']);
});
```

### Pattern 2: API Resources for Data Transformation

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    /**
     * Transform the resource into an array.
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            'excerpt' => $this->excerpt,
            'content' => $this->when($request->routeIs('posts.show'), $this->content),
            'status' => $this->status,
            'published_at' => $this->published_at?->toIso8601String(),
            'created_at' => $this->created_at->toIso8601String(),
            'updated_at' => $this->updated_at->toIso8601String(),

            // Relationships
            'author' => new UserResource($this->whenLoaded('author')),
            'category' => new CategoryResource($this->whenLoaded('category')),
            'tags' => TagResource::collection($this->whenLoaded('tags')),
            'comments' => CommentResource::collection($this->whenLoaded('comments')),

            // Conditional attributes
            'can_edit' => $this->when(
                $request->user()?->can('update', $this->resource),
                true
            ),
            'can_delete' => $this->when(
                $request->user()?->can('delete', $this->resource),
                true
            ),

            // Counts
            'comments_count' => $this->whenCounted('comments'),
            'likes_count' => $this->whenCounted('likes'),

            // Aggregates
            'average_rating' => $this->whenAggregated('reviews', 'rating', 'avg'),

            // Pivot data for many-to-many
            'pivot' => $this->when(
                $this->pivot,
                fn () => [
                    'assigned_at' => $this->pivot->created_at,
                    'assigned_by' => $this->pivot->assigned_by,
                ]
            ),

            // Links
            'links' => [
                'self' => route('posts.show', $this->id),
            ],
        ];
    }

    /**
     * Get additional data that should be returned with the resource array.
     */
    public function with(Request $request): array
    {
        return [
            'meta' => [
                'version' => '1.0',
            ],
        ];
    }
}

// Collection Resource
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class PostCollection extends ResourceCollection
{
    /**
     * Transform the resource collection into an array.
     */
    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'links' => [
                'self' => route('posts.index'),
            ],
            'meta' => [
                'total' => $this->total(),
                'count' => $this->count(),
                'per_page' => $this->perPage(),
                'current_page' => $this->currentPage(),
                'total_pages' => $this->lastPage(),
            ],
        ];
    }
}

// Usage in controller
public function index()
{
    $posts = Post::with('author')->paginate(20);

    return new PostCollection($posts);
}
```

### Pattern 3: Sanctum Authentication

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\LoginRequest;
use App\Http\Requests\RegisterRequest;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    /**
     * Register a new user.
     */
    public function register(RegisterRequest $request): JsonResponse
    {
        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);

        $token = $user->createToken('auth_token')->plainTextToken;

        return response()->json([
            'access_token' => $token,
            'token_type' => 'Bearer',
            'user' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
            ],
        ], 201);
    }

    /**
     * Authenticate user and return token.
     */
    public function login(LoginRequest $request): JsonResponse
    {
        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        // Revoke all existing tokens
        $user->tokens()->delete();

        // Create new token
        $token = $user->createToken('auth_token')->plainTextToken;

        return response()->json([
            'access_token' => $token,
            'token_type' => 'Bearer',
            'user' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
            ],
        ]);
    }

    /**
     * Logout user (revoke token).
     */
    public function logout(Request $request): JsonResponse
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logged out successfully']);
    }

    /**
     * Get authenticated user.
     */
    public function me(Request $request): JsonResponse
    {
        return response()->json($request->user());
    }

    /**
     * Create token with specific abilities.
     */
    public function createToken(Request $request): JsonResponse
    {
        $request->validate([
            'name' => 'required|string|max:255',
            'abilities' => 'array',
        ]);

        $token = $request->user()->createToken(
            $request->name,
            $request->abilities ?? ['*']
        );

        return response()->json([
            'access_token' => $token->plainTextToken,
            'token_type' => 'Bearer',
        ]);
    }

    /**
     * List user's tokens.
     */
    public function tokens(Request $request): JsonResponse
    {
        $tokens = $request->user()->tokens()->get(['id', 'name', 'abilities', 'created_at']);

        return response()->json(['tokens' => $tokens]);
    }

    /**
     * Revoke specific token.
     */
    public function revokeToken(Request $request, string $tokenId): JsonResponse
    {
        $request->user()->tokens()->where('id', $tokenId)->delete();

        return response()->json(['message' => 'Token revoked']);
    }
}

// routes/api.php
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::get('/me', [AuthController::class, 'me']);
    Route::post('/tokens', [AuthController::class, 'createToken']);
    Route::get('/tokens', [AuthController::class, 'tokens']);
    Route::delete('/tokens/{tokenId}', [AuthController::class, 'revokeToken']);
});

// Protect routes with abilities
Route::middleware(['auth:sanctum', 'ability:post:create'])->group(function () {
    Route::post('/posts', [PostController::class, 'store']);
});
```

### Pattern 4: Request Validation

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Contracts\Validation\Validator;
use Illuminate\Http\Exceptions\HttpResponseException;

class StorePostRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     */
    public function authorize(): bool
    {
        return true;
    }

    /**
     * Get the validation rules that apply to the request.
     */
    public function rules(): array
    {
        return [
            'title' => 'required|string|max:255',
            'content' => 'required|string',
            'category_id' => 'required|exists:categories,id',
            'tags' => 'array',
            'tags.*' => 'exists:tags,id',
            'status' => 'in:draft,published',
            'published_at' => 'nullable|date|after:now',
            'featured_image' => 'nullable|image|max:2048',
        ];
    }

    /**
     * Get custom messages for validator errors.
     */
    public function messages(): array
    {
        return [
            'title.required' => 'A post title is required',
            'content.required' => 'Post content cannot be empty',
            'category_id.exists' => 'The selected category does not exist',
        ];
    }

    /**
     * Get custom attributes for validator errors.
     */
    public function attributes(): array
    {
        return [
            'category_id' => 'category',
            'featured_image' => 'image',
        ];
    }

    /**
     * Handle a failed validation attempt (for API).
     */
    protected function failedValidation(Validator $validator)
    {
        throw new HttpResponseException(
            response()->json([
                'message' => 'Validation failed',
                'errors' => $validator->errors(),
            ], 422)
        );
    }

    /**
     * Prepare the data for validation.
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => Str::slug($this->title),
        ]);
    }
}

// Complex validation rules
class UpdateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email' => [
                'required',
                'email',
                Rule::unique('users')->ignore($this->user()),
            ],
            'username' => [
                'required',
                'alpha_dash',
                Rule::unique('users')->ignore($this->user()),
            ],
            'password' => [
                'nullable',
                'min:8',
                Password::min(8)
                    ->mixedCase()
                    ->numbers()
                    ->symbols()
                    ->uncompromised(),
            ],
        ];
    }
}
```

### Pattern 5: Error Handling and Responses

```php
<?php

namespace App\Exceptions;

use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Throwable;

class Handler extends ExceptionHandler
{
    /**
     * Register the exception handling callbacks for the application.
     */
    public function register(): void
    {
        $this->renderable(function (NotFoundHttpException $e, $request) {
            if ($request->is('api/*')) {
                return response()->json([
                    'message' => 'Resource not found',
                ], 404);
            }
        });

        $this->renderable(function (ModelNotFoundException $e, $request) {
            if ($request->is('api/*')) {
                return response()->json([
                    'message' => 'Record not found',
                ], 404);
            }
        });

        $this->renderable(function (AuthenticationException $e, $request) {
            if ($request->is('api/*')) {
                return response()->json([
                    'message' => 'Unauthenticated',
                ], 401);
            }
        });

        $this->renderable(function (ValidationException $e, $request) {
            if ($request->is('api/*')) {
                return response()->json([
                    'message' => 'Validation failed',
                    'errors' => $e->errors(),
                ], 422);
            }
        });

        $this->renderable(function (Throwable $e, $request) {
            if ($request->is('api/*') && config('app.debug') === false) {
                return response()->json([
                    'message' => 'Server error',
                ], 500);
            }
        });
    }
}

// Custom exception classes
namespace App\Exceptions;

use Exception;

class ApiException extends Exception
{
    protected int $statusCode;
    protected array $errors;

    public function __construct(
        string $message,
        int $statusCode = 400,
        array $errors = []
    ) {
        parent::__construct($message);
        $this->statusCode = $statusCode;
        $this->errors = $errors;
    }

    public function render($request)
    {
        $response = [
            'message' => $this->message,
        ];

        if (!empty($this->errors)) {
            $response['errors'] = $this->errors;
        }

        return response()->json($response, $this->statusCode);
    }
}

// Usage in controller
public function store(Request $request)
{
    if ($request->user()->posts()->count() >= 100) {
        throw new ApiException('Post limit reached', 403);
    }

    // ...
}

// Standardized API response helper
namespace App\Traits;

trait ApiResponses
{
    protected function success($data = null, string $message = null, int $code = 200)
    {
        return response()->json([
            'success' => true,
            'message' => $message,
            'data' => $data,
        ], $code);
    }

    protected function error(string $message, int $code = 400, $errors = null)
    {
        $response = [
            'success' => false,
            'message' => $message,
        ];

        if ($errors) {
            $response['errors'] = $errors;
        }

        return response()->json($response, $code);
    }

    protected function created($data, string $message = 'Resource created')
    {
        return $this->success($data, $message, 201);
    }

    protected function noContent()
    {
        return response()->json(null, 204);
    }
}

// Use trait in controller
class PostController extends Controller
{
    use ApiResponses;

    public function store(Request $request)
    {
        $post = Post::create($request->validated());

        return $this->created(new PostResource($post), 'Post created successfully');
    }

    public function destroy(Post $post)
    {
        $post->delete();

        return $this->noContent();
    }
}
```

### Pattern 6: Rate Limiting and Throttling

```php
<?php

// app/Providers/RouteServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    // Default API rate limit
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });

    // Custom rate limit for specific endpoints
    RateLimiter::for('uploads', function (Request $request) {
        return Limit::perMinute(10)->by($request->user()->id);
    });

    // Different limits for authenticated vs guests
    RateLimiter::for('posts', function (Request $request) {
        return $request->user()
            ? Limit::perMinute(100)->by($request->user()->id)
            : Limit::perMinute(10)->by($request->ip());
    });

    // Multiple limits (rate limit per minute and per day)
    RateLimiter::for('strict', function (Request $request) {
        return [
            Limit::perMinute(10)->by($request->user()->id),
            Limit::perDay(1000)->by($request->user()->id),
        ];
    });

    // Custom response on rate limit exceeded
    RateLimiter::for('custom', function (Request $request) {
        return Limit::perMinute(60)
            ->by($request->user()->id)
            ->response(function (Request $request, array $headers) {
                return response()->json([
                    'message' => 'Too many requests. Please try again later.',
                ], 429, $headers);
            });
    });
}

// Apply to routes
Route::middleware(['throttle:api'])->group(function () {
    Route::apiResource('posts', PostController::class);
});

Route::middleware(['auth:sanctum', 'throttle:uploads'])->group(function () {
    Route::post('files/upload', [FileController::class, 'upload']);
});

// Custom middleware for rate limiting
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

class ThrottleRequests
{
    public function handle(Request $request, Closure $next, int $maxAttempts = 60)
    {
        $key = $request->user()?->id ?: $request->ip();

        if (RateLimiter::tooManyAttempts($key, $maxAttempts)) {
            $seconds = RateLimiter::availableIn($key);

            return response()->json([
                'message' => "Too many requests. Retry after {$seconds} seconds.",
            ], 429, [
                'Retry-After' => $seconds,
                'X-RateLimit-Limit' => $maxAttempts,
                'X-RateLimit-Remaining' => 0,
            ]);
        }

        RateLimiter::hit($key);

        $response = $next($request);

        $response->headers->add([
            'X-RateLimit-Limit' => $maxAttempts,
            'X-RateLimit-Remaining' => RateLimiter::remaining($key, $maxAttempts),
        ]);

        return $response;
    }
}
```

### Pattern 7: API Versioning

```php
<?php

// routes/api.php

// Version 1 routes
Route::prefix('v1')->group(function () {
    Route::apiResource('posts', Api\V1\PostController::class);
    Route::apiResource('users', Api\V1\UserController::class);
});

// Version 2 routes
Route::prefix('v2')->group(function () {
    Route::apiResource('posts', Api\V2\PostController::class);
    Route::apiResource('users', Api\V2\UserController::class);
});

// Header-based versioning middleware
namespace App\Http\Middleware;

class ApiVersion
{
    public function handle(Request $request, Closure $next, string $version)
    {
        if ($request->header('Accept') !== "application/vnd.api.{$version}+json") {
            return response()->json([
                'message' => 'Invalid API version',
            ], 406);
        }

        return $next($request);
    }
}

// Version-specific controllers
namespace App\Http\Controllers\Api\V1;

class PostController extends Controller
{
    public function index()
    {
        return PostResource::collection(Post::paginate());
    }
}

namespace App\Http\Controllers\Api\V2;

class PostController extends Controller
{
    public function index()
    {
        // V2 with additional features
        return PostResourceV2::collection(
            Post::with('author', 'comments')->paginate()
        );
    }
}

// Version-specific resources
namespace App\Http\Resources\V1;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'content' => $this->content,
        ];
    }
}

namespace App\Http\Resources\V2;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'content' => $this->content,
            'author' => new UserResource($this->whenLoaded('author')),
            'comments_count' => $this->whenCounted('comments'),
            // New fields in v2
            'metadata' => $this->metadata,
            'tags' => TagResource::collection($this->whenLoaded('tags')),
        ];
    }
}
```

### Pattern 8: Pagination and Filtering

```php
<?php

namespace App\Http\Controllers\Api;

class PostController extends Controller
{
    /**
     * List posts with pagination and filtering.
     */
    public function index(Request $request)
    {
        $query = Post::query();

        // Filter by status
        if ($request->has('status')) {
            $query->where('status', $request->status);
        }

        // Filter by category
        if ($request->has('category_id')) {
            $query->where('category_id', $request->category_id);
        }

        // Search by title or content
        if ($request->has('search')) {
            $query->where(function ($q) use ($request) {
                $q->where('title', 'like', "%{$request->search}%")
                    ->orWhere('content', 'like', "%{$request->search}%");
            });
        }

        // Filter by date range
        if ($request->has('from_date')) {
            $query->where('created_at', '>=', $request->from_date);
        }

        if ($request->has('to_date')) {
            $query->where('created_at', '<=', $request->to_date);
        }

        // Sort
        $sortBy = $request->get('sort_by', 'created_at');
        $sortOrder = $request->get('sort_order', 'desc');
        $query->orderBy($sortBy, $sortOrder);

        // Include relationships
        if ($request->has('include')) {
            $includes = explode(',', $request->include);
            $query->with($includes);
        }

        // Pagination
        $perPage = $request->get('per_page', 15);
        $posts = $query->paginate($perPage);

        return PostResource::collection($posts);
    }

    /**
     * Cursor-based pagination for better performance.
     */
    public function indexCursor(Request $request)
    {
        $posts = Post::orderBy('id')
            ->cursorPaginate($request->get('per_page', 15));

        return PostResource::collection($posts);
    }
}

// Query string filters trait
namespace App\Traits;

trait Filterable
{
    public function scopeFilter($query, array $filters)
    {
        foreach ($filters as $key => $value) {
            if (method_exists($this, $method = 'filter' . Str::studly($key))) {
                $this->$method($query, $value);
            } else {
                $query->where($key, $value);
            }
        }

        return $query;
    }

    protected function filterSearch($query, $value)
    {
        return $query->where(function ($q) use ($value) {
            $q->where('title', 'like', "%{$value}%")
                ->orWhere('content', 'like', "%{$value}%");
        });
    }

    protected function filterStatus($query, $value)
    {
        return $query->where('status', $value);
    }
}

// Usage
class Post extends Model
{
    use Filterable;
}

// In controller
$posts = Post::filter($request->all())->paginate();
```

### Pattern 9: API Documentation with Annotations

```php
<?php

namespace App\Http\Controllers\Api;

/**
 * @OA\Info(
 *     title="Blog API",
 *     version="1.0.0",
 *     description="RESTful API for blog management"
 * )
 *
 * @OA\Server(
 *     url="https://api.example.com",
 *     description="Production API Server"
 * )
 *
 * @OA\SecurityScheme(
 *     securityScheme="bearerAuth",
 *     type="http",
 *     scheme="bearer",
 *     bearerFormat="JWT"
 * )
 */
class PostController extends Controller
{
    /**
     * @OA\Get(
     *     path="/api/posts",
     *     summary="List all posts",
     *     tags={"Posts"},
     *     @OA\Parameter(
     *         name="page",
     *         in="query",
     *         description="Page number",
     *         required=false,
     *         @OA\Schema(type="integer")
     *     ),
     *     @OA\Parameter(
     *         name="per_page",
     *         in="query",
     *         description="Items per page",
     *         required=false,
     *         @OA\Schema(type="integer", default=15)
     *     ),
     *     @OA\Response(
     *         response=200,
     *         description="Successful operation",
     *         @OA\JsonContent(
     *             @OA\Property(property="data", type="array",
     *                 @OA\Items(ref="#/components/schemas/Post")
     *             ),
     *             @OA\Property(property="links", type="object"),
     *             @OA\Property(property="meta", type="object")
     *         )
     *     )
     * )
     */
    public function index()
    {
        return PostResource::collection(Post::paginate());
    }

    /**
     * @OA\Post(
     *     path="/api/posts",
     *     summary="Create a new post",
     *     tags={"Posts"},
     *     security={{"bearerAuth":{}}},
     *     @OA\RequestBody(
     *         required=true,
     *         @OA\JsonContent(
     *             required={"title","content"},
     *             @OA\Property(property="title", type="string", example="My Post"),
     *             @OA\Property(property="content", type="string", example="Post content"),
     *             @OA\Property(property="category_id", type="integer", example=1)
     *         )
     *     ),
     *     @OA\Response(
     *         response=201,
     *         description="Post created",
     *         @OA\JsonContent(ref="#/components/schemas/Post")
     *     ),
     *     @OA\Response(response=422, description="Validation error")
     * )
     */
    public function store(Request $request)
    {
        // Implementation
    }
}

/**
 * @OA\Schema(
 *     schema="Post",
 *     type="object",
 *     @OA\Property(property="id", type="integer", example=1),
 *     @OA\Property(property="title", type="string", example="Post Title"),
 *     @OA\Property(property="content", type="string", example="Post content"),
 *     @OA\Property(property="status", type="string", enum={"draft", "published"}),
 *     @OA\Property(property="created_at", type="string", format="date-time"),
 *     @OA\Property(property="updated_at", type="string", format="date-time")
 * )
 */

// Generate documentation
php artisan l5-swagger:generate
```

### Pattern 10: HATEOAS and Hypermedia Links

```php
<?php

namespace App\Http\Resources;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'content' => $this->content,
            'status' => $this->status,

            // HATEOAS links
            'links' => [
                'self' => [
                    'href' => route('posts.show', $this->id),
                    'method' => 'GET',
                ],
                'update' => $this->when(
                    $request->user()?->can('update', $this->resource),
                    [
                        'href' => route('posts.update', $this->id),
                        'method' => 'PUT',
                    ]
                ),
                'delete' => $this->when(
                    $request->user()?->can('delete', $this->resource),
                    [
                        'href' => route('posts.destroy', $this->id),
                        'method' => 'DELETE',
                    ]
                ),
                'publish' => $this->when(
                    $this->status === 'draft',
                    [
                        'href' => route('posts.publish', $this->id),
                        'method' => 'POST',
                    ]
                ),
                'comments' => [
                    'href' => route('posts.comments.index', $this->id),
                    'method' => 'GET',
                ],
                'author' => [
                    'href' => route('users.show', $this->user_id),
                    'method' => 'GET',
                ],
            ],

            // Related resources
            'relationships' => [
                'author' => [
                    'links' => [
                        'self' => route('posts.relationships.author', $this->id),
                        'related' => route('users.show', $this->user_id),
                    ],
                ],
                'comments' => [
                    'links' => [
                        'self' => route('posts.relationships.comments', $this->id),
                        'related' => route('posts.comments.index', $this->id),
                    ],
                    'meta' => [
                        'count' => $this->comments_count,
                    ],
                ],
            ],
        ];
    }
}
```

## Advanced Patterns

### Pattern 11: API Middleware Stack

```php
<?php

// routes/api.php
Route::middleware([
    'api',
    'throttle:api',
    'auth:sanctum',
    'verified',
])->group(function () {
    Route::apiResource('posts', PostController::class);
});

// Custom API middleware
namespace App\Http\Middleware;

class EnsureApiKeyIsValid
{
    public function handle(Request $request, Closure $next)
    {
        $apiKey = $request->header('X-API-Key');

        if (!$apiKey || !ApiKey::where('key', $apiKey)->active()->exists()) {
            return response()->json(['message' => 'Invalid API key'], 401);
        }

        return $next($request);
    }
}

class ForceJsonResponse
{
    public function handle(Request $request, Closure $next)
    {
        $request->headers->set('Accept', 'application/json');

        return $next($request);
    }
}

class LogApiRequest
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        ApiLog::create([
            'user_id' => $request->user()?->id,
            'method' => $request->method(),
            'url' => $request->fullUrl(),
            'ip' => $request->ip(),
            'status_code' => $response->status(),
            'duration' => microtime(true) - LARAVEL_START,
        ]);

        return $response;
    }
}
```

### Pattern 12: Complex Querying and Relationships

```php
<?php

namespace App\Http\Controllers\Api;

class PostController extends Controller
{
    /**
     * Get posts with complex filtering and relationships.
     */
    public function index(Request $request)
    {
        $query = Post::query();

        // Eager load relationships based on request
        $includes = [];
        if ($request->has('include')) {
            $allowedIncludes = ['author', 'category', 'tags', 'comments'];
            $includes = array_intersect(
                explode(',', $request->include),
                $allowedIncludes
            );
        }

        if (!empty($includes)) {
            $query->with($includes);
        }

        // Count relationships
        if ($request->has('count')) {
            $counts = explode(',', $request->count);
            $query->withCount($counts);
        }

        // Filter by relationships
        if ($request->has('author_id')) {
            $query->where('user_id', $request->author_id);
        }

        if ($request->has('tag')) {
            $query->whereHas('tags', function ($q) use ($request) {
                $q->where('slug', $request->tag);
            });
        }

        // Sparse fieldsets (only return specific fields)
        if ($request->has('fields')) {
            $fields = explode(',', $request->fields);
            $query->select(array_merge(['id'], $fields));
        }

        return PostResource::collection($query->paginate());
    }
}
```

### Pattern 13: Batch Operations

```php
<?php

namespace App\Http\Controllers\Api;

class PostBatchController extends Controller
{
    /**
     * Batch create posts.
     */
    public function store(Request $request)
    {
        $request->validate([
            'posts' => 'required|array|min:1|max:100',
            'posts.*.title' => 'required|string',
            'posts.*.content' => 'required|string',
        ]);

        $posts = collect($request->posts)->map(function ($data) use ($request) {
            return Post::create([
                'user_id' => $request->user()->id,
                ...$data,
            ]);
        });

        return response()->json([
            'message' => 'Posts created successfully',
            'data' => PostResource::collection($posts),
        ], 201);
    }

    /**
     * Batch update posts.
     */
    public function update(Request $request)
    {
        $request->validate([
            'posts' => 'required|array',
            'posts.*.id' => 'required|exists:posts,id',
        ]);

        DB::transaction(function () use ($request) {
            foreach ($request->posts as $data) {
                $post = Post::findOrFail($data['id']);
                $this->authorize('update', $post);
                $post->update($data);
            }
        });

        return response()->json(['message' => 'Posts updated successfully']);
    }

    /**
     * Batch delete posts.
     */
    public function destroy(Request $request)
    {
        $request->validate([
            'ids' => 'required|array',
            'ids.*' => 'exists:posts,id',
        ]);

        $posts = Post::whereIn('id', $request->ids)->get();

        foreach ($posts as $post) {
            $this->authorize('delete', $post);
        }

        Post::whereIn('id', $request->ids)->delete();

        return response()->json(['message' => 'Posts deleted successfully']);
    }
}
```

### Pattern 14: Webhook System

```php
<?php

namespace App\Http\Controllers\Api;

class WebhookController extends Controller
{
    /**
     * Register a webhook.
     */
    public function store(Request $request)
    {
        $request->validate([
            'url' => 'required|url',
            'events' => 'required|array',
            'events.*' => 'in:post.created,post.updated,post.deleted',
        ]);

        $webhook = $request->user()->webhooks()->create([
            'url' => $request->url,
            'events' => $request->events,
            'secret' => Str::random(40),
        ]);

        return response()->json($webhook, 201);
    }

    /**
     * Test webhook.
     */
    public function test(Webhook $webhook)
    {
        $this->authorize('update', $webhook);

        dispatch(new TriggerWebhook($webhook, 'test', ['message' => 'Test webhook']));

        return response()->json(['message' => 'Webhook triggered']);
    }
}

// Webhook trigger job
namespace App\Jobs;

class TriggerWebhook implements ShouldQueue
{
    public function __construct(
        public Webhook $webhook,
        public string $event,
        public array $payload
    ) {
    }

    public function handle()
    {
        $data = [
            'event' => $this->event,
            'payload' => $this->payload,
            'timestamp' => now()->timestamp,
        ];

        $signature = hash_hmac('sha256', json_encode($data), $this->webhook->secret);

        Http::withHeaders([
            'X-Webhook-Signature' => $signature,
        ])->post($this->webhook->url, $data);
    }
}
```

### Pattern 15: API Testing with Pest

```php
<?php

// tests/Feature/Api/PostTest.php

use function Pest\Laravel\{getJson, postJson, putJson, deleteJson};

beforeEach(function () {
    $this->user = User::factory()->create();
    Sanctum::actingAs($this->user);
});

test('can list posts', function () {
    Post::factory()->count(15)->create();

    getJson('/api/posts')
        ->assertStatus(200)
        ->assertJsonCount(15, 'data')
        ->assertJsonStructure([
            'data' => [
                '*' => ['id', 'title', 'content']
            ]
        ]);
});

test('can create post', function () {
    postJson('/api/posts', [
        'title' => 'Test Post',
        'content' => 'Test content',
    ])
        ->assertStatus(201)
        ->assertJsonPath('data.title', 'Test Post');

    expect(Post::count())->toBe(1);
});

test('validates post creation', function () {
    postJson('/api/posts', [])
        ->assertStatus(422)
        ->assertJsonValidationErrors(['title', 'content']);
});

test('can update own post', function () {
    $post = Post::factory()->create(['user_id' => $this->user->id]);

    putJson("/api/posts/{$post->id}", [
        'title' => 'Updated',
    ])
        ->assertStatus(200);

    expect($post->fresh()->title)->toBe('Updated');
});

test('cannot update others post', function () {
    $post = Post::factory()->create();

    putJson("/api/posts/{$post->id}", ['title' => 'Hacked'])
        ->assertStatus(403);
});

test('can delete post', function () {
    $post = Post::factory()->create(['user_id' => $this->user->id]);

    deleteJson("/api/posts/{$post->id}")
        ->assertStatus(204);

    expect(Post::count())->toBe(0);
});

test('respects rate limiting', function () {
    for ($i = 0; $i < 60; $i++) {
        getJson('/api/posts')->assertStatus(200);
    }

    getJson('/api/posts')
        ->assertStatus(429);
});
```

## Real-World Applications

### Application 1: Complete Blog API

```php
<?php

// Complete API implementation
Route::prefix('v1')->middleware('api')->group(function () {
    // Public routes
    Route::get('posts', [PostController::class, 'index']);
    Route::get('posts/{post}', [PostController::class, 'show']);

    // Authentication
    Route::post('register', [AuthController::class, 'register']);
    Route::post('login', [AuthController::class, 'login']);

    // Protected routes
    Route::middleware('auth:sanctum')->group(function () {
        Route::post('logout', [AuthController::class, 'logout']);

        // Posts
        Route::post('posts', [PostController::class, 'store']);
        Route::put('posts/{post}', [PostController::class, 'update']);
        Route::delete('posts/{post}', [PostController::class, 'destroy']);

        // Comments
        Route::post('posts/{post}/comments', [CommentController::class, 'store']);
        Route::delete('comments/{comment}', [CommentController::class, 'destroy']);

        // User profile
        Route::get('me', [UserController::class, 'me']);
        Route::put('me', [UserController::class, 'update']);
    });
});
```

## Performance Best Practices

### Practice 1: Cache API Responses

```php
<?php

public function index(Request $request)
{
    $cacheKey = 'posts:' . md5(json_encode($request->all()));

    return Cache::remember($cacheKey, 300, function () {
        return PostResource::collection(Post::paginate());
    });
}
```

### Practice 2: Use API Resources

```php
<?php

// Always use resources for consistent output
return PostResource::collection($posts);

// Not raw models
return $posts; // Don't do this
```

### Practice 3: Eager Load Relationships

```php
<?php

// Good: Prevent N+1
$posts = Post::with(['author', 'category'])->paginate();

// Bad: N+1 queries
$posts = Post::paginate();
```

## Common Pitfalls

### Pitfall 1: Not Validating Input

```php
// WRONG
public function store(Request $request)
{
    Post::create($request->all());
}

// CORRECT
public function store(StorePostRequest $request)
{
    Post::create($request->validated());
}
```

### Pitfall 2: Exposing Sensitive Data

```php
// WRONG
return $user; // Exposes password hash, tokens, etc.

// CORRECT
return new UserResource($user);
```

### Pitfall 3: Not Using Status Codes

```php
// WRONG
return response()->json(['error' => 'Not found']);

// CORRECT
return response()->json(['error' => 'Not found'], 404);
```

### Pitfall 4: Missing Rate Limiting

```php
// WRONG
Route::post('/api/posts', [PostController::class, 'store']);

// CORRECT
Route::middleware('throttle:60,1')->group(function () {
    Route::post('/api/posts', [PostController::class, 'store']);
});
```

### Pitfall 5: No API Documentation

```php
// WRONG: Undocumented API

// CORRECT: Use OpenAPI annotations or tools like Scribe
/**
 * @OA\Get(path="/api/posts", ...)
 */
```

## Testing

```php
<?php

test('api authentication works', function () {
    $user = User::factory()->create();
    $token = $user->createToken('test')->plainTextToken;

    getJson('/api/posts', [
        'Authorization' => "Bearer {$token}"
    ])->assertStatus(200);
});

test('api returns proper error format', function () {
    postJson('/api/posts', [])
        ->assertStatus(422)
        ->assertJsonStructure([
            'message',
            'errors' => []
        ]);
});
```

## Resources

- **Laravel API Documentation**: https://laravel.com/docs/eloquent-resources
- **Laravel Sanctum**: https://laravel.com/docs/sanctum
- **RESTful API Design**: https://restfulapi.net/
- **OpenAPI Specification**: https://swagger.io/specification/
- **Postman**: API testing tool
- **Insomnia**: API client and testing tool

## Best Practices Summary

1. **Use API resources** for consistent data transformation
2. **Implement proper authentication** with Sanctum or Passport
3. **Apply rate limiting** to prevent abuse
4. **Version your APIs** for backward compatibility
5. **Validate all input** with Form Requests
6. **Use proper HTTP status codes** for all responses
7. **Document your API** with OpenAPI/Swagger
8. **Implement pagination** for large datasets
9. **Handle errors consistently** across all endpoints
10. **Test your API** thoroughly with feature tests
