---
name: eloquent-relationships
description: Master Eloquent ORM relationships including one-to-one, one-to-many, many-to-many, polymorphic, and advanced relationship patterns with eager loading and N+1 prevention. Use when designing database relationships, optimizing queries, or implementing complex data models in Laravel.
---

# Eloquent Relationships

Comprehensive guide to implementing and optimizing Eloquent ORM relationships in Laravel, covering all relationship types, eager loading strategies, query optimization, and advanced patterns for building efficient data models.

## When to Use This Skill

- Designing database relationships and model structures
- Implementing one-to-one, one-to-many, or many-to-many relationships
- Building polymorphic relationships for flexible data models
- Optimizing queries with eager loading to prevent N+1 problems
- Creating has-one-through and has-many-through relationships
- Implementing dynamic relationships with conditions
- Setting up relationship constraints and cascading operations
- Building complex queries across multiple related models
- Managing inverse relationships and bidirectional data access
- Implementing relationship caching and performance optimization

## Core Concepts

### 1. Relationship Types
- **One-to-One**: Single record relates to single record (User has one Profile)
- **One-to-Many**: Single record relates to multiple records (Post has many Comments)
- **Many-to-Many**: Multiple records relate to multiple records (Users have many Roles)
- **Polymorphic**: Model can belong to multiple other models (Comments on Posts or Videos)

### 2. Eager Loading
- Load related models in a single query to prevent N+1 issues
- Reduce database queries from hundreds to just a few
- Critical for performance in production applications

### 3. Lazy vs Eager Loading
- **Lazy Loading**: Related models loaded on access (can cause N+1)
- **Eager Loading**: Related models loaded upfront with `with()`
- **Lazy Eager Loading**: Load relationships after initial query with `load()`

### 4. Relationship Constraints
- Add where clauses to relationship queries
- Filter related models based on conditions
- Order and limit related results

### 5. Inverse Relationships
- Define relationships in both directions
- Access parent from child and vice versa
- Maintain bidirectional data consistency

## Quick Start

```php
<?php

// Define a basic relationship
class User extends Model
{
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}

class Post extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

// Use the relationship
$user = User::find(1);
$posts = $user->posts; // Lazy load
$users = User::with('posts')->get(); // Eager load
```

## Fundamental Patterns

### Pattern 1: One-to-One Relationships

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasOne;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class User extends Model
{
    /**
     * Get the profile associated with the user.
     */
    public function profile(): HasOne
    {
        return $this->hasOne(Profile::class);
    }

    /**
     * Get the user's address.
     */
    public function address(): HasOne
    {
        return $this->hasOne(Address::class);
    }
}

class Profile extends Model
{
    protected $fillable = ['user_id', 'bio', 'avatar', 'website'];

    /**
     * Get the user that owns the profile.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}

class Address extends Model
{
    protected $fillable = ['user_id', 'street', 'city', 'state', 'zip'];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}

// Usage examples
$user = User::find(1);
$profile = $user->profile; // Access profile
$bio = $user->profile->bio; // Access profile attributes

// Create related model
$user->profile()->create([
    'bio' => 'Software developer',
    'avatar' => 'avatar.jpg',
    'website' => 'https://example.com'
]);

// Update or create (upsert)
$user->profile()->updateOrCreate(
    ['user_id' => $user->id],
    ['bio' => 'Updated bio']
);

// Access inverse relationship
$profile = Profile::find(1);
$owner = $profile->user;
```

### Pattern 2: One-to-Many Relationships

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Post extends Model
{
    protected $fillable = ['user_id', 'title', 'content', 'published_at'];

    /**
     * Get the comments for the post.
     */
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }

    /**
     * Get approved comments only.
     */
    public function approvedComments(): HasMany
    {
        return $this->hasMany(Comment::class)
            ->where('approved', true)
            ->orderBy('created_at', 'desc');
    }

    /**
     * Get the author of the post.
     */
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}

class Comment extends Model
{
    protected $fillable = ['post_id', 'user_id', 'content', 'approved'];

    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class);
    }

    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}

// Usage examples
$post = Post::find(1);

// Get all comments
$comments = $post->comments;

// Get count without loading models
$commentCount = $post->comments()->count();

// Add new comment
$post->comments()->create([
    'user_id' => auth()->id(),
    'content' => 'Great post!',
    'approved' => false
]);

// Create multiple comments
$post->comments()->createMany([
    ['user_id' => 1, 'content' => 'Comment 1'],
    ['user_id' => 2, 'content' => 'Comment 2'],
]);

// Query relationship
$recentComments = $post->comments()
    ->where('created_at', '>', now()->subDays(7))
    ->get();

// Check if relationship exists
if ($post->comments()->exists()) {
    // Post has comments
}
```

### Pattern 3: Many-to-Many Relationships

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class User extends Model
{
    /**
     * The roles that belong to the user.
     */
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class)
            ->withTimestamps()
            ->withPivot('assigned_at', 'assigned_by');
    }

    /**
     * The teams that the user belongs to.
     */
    public function teams(): BelongsToMany
    {
        return $this->belongsToMany(Team::class, 'team_user')
            ->using(TeamUser::class)
            ->withPivot('role', 'joined_at')
            ->withTimestamps();
    }
}

class Role extends Model
{
    protected $fillable = ['name', 'description'];

    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class)
            ->withTimestamps();
    }

    public function permissions(): BelongsToMany
    {
        return $this->belongsToMany(Permission::class);
    }
}

class Team extends Model
{
    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class, 'team_user')
            ->using(TeamUser::class)
            ->withPivot('role', 'joined_at')
            ->withTimestamps();
    }
}

// Custom pivot model
class TeamUser extends Pivot
{
    protected $table = 'team_user';

    protected $casts = [
        'joined_at' => 'datetime',
    ];
}

// Usage examples
$user = User::find(1);

// Attach roles
$user->roles()->attach(1);
$user->roles()->attach([2, 3]);
$user->roles()->attach(1, ['assigned_at' => now(), 'assigned_by' => auth()->id()]);

// Detach roles
$user->roles()->detach(1);
$user->roles()->detach([1, 2, 3]);
$user->roles()->detach(); // Remove all

// Sync roles (attach/detach to match exactly)
$user->roles()->sync([1, 2, 3]);
$user->roles()->syncWithoutDetaching([4, 5]); // Only attach new

// Toggle roles
$user->roles()->toggle([1, 2, 3]);

// Access pivot data
foreach ($user->roles as $role) {
    echo $role->pivot->assigned_at;
    echo $role->pivot->assigned_by;
}

// Query with pivot conditions
$admins = User::whereHas('roles', function ($query) {
    $query->where('name', 'admin');
})->get();

// Update pivot
$user->roles()->updateExistingPivot($roleId, [
    'assigned_by' => auth()->id()
]);
```

### Pattern 4: Has-One-Through Relationships

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasOneThrough;

class Mechanic extends Model
{
    /**
     * Get the car's owner (through Car).
     */
    public function carOwner(): HasOneThrough
    {
        return $this->hasOneThrough(
            Owner::class,     // Final model
            Car::class,       // Intermediate model
            'mechanic_id',    // Foreign key on cars table
            'car_id',         // Foreign key on owners table
            'id',             // Local key on mechanics table
            'id'              // Local key on cars table
        );
    }
}

class Car extends Model
{
    protected $fillable = ['mechanic_id', 'make', 'model'];

    public function mechanic()
    {
        return $this->belongsTo(Mechanic::class);
    }

    public function owner()
    {
        return $this->hasOne(Owner::class);
    }
}

class Owner extends Model
{
    protected $fillable = ['car_id', 'name', 'email'];

    public function car()
    {
        return $this->belongsTo(Car::class);
    }
}

// Usage
$mechanic = Mechanic::find(1);
$owner = $mechanic->carOwner; // Get owner through car
```

### Pattern 5: Has-Many-Through Relationships

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasManyThrough;

class Country extends Model
{
    /**
     * Get all posts for the country (through users).
     */
    public function posts(): HasManyThrough
    {
        return $this->hasManyThrough(
            Post::class,      // Final model
            User::class,      // Intermediate model
            'country_id',     // Foreign key on users table
            'user_id',        // Foreign key on posts table
            'id',             // Local key on countries table
            'id'              // Local key on users table
        );
    }

    /**
     * Get all comments through posts and users.
     */
    public function comments(): HasManyThrough
    {
        return $this->hasManyThrough(Comment::class, Post::class);
    }
}

class User extends Model
{
    public function country()
    {
        return $this->belongsTo(Country::class);
    }

    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}

// Usage
$country = Country::find(1);
$posts = $country->posts; // All posts by users in this country
$recentPosts = $country->posts()
    ->where('created_at', '>', now()->subDays(7))
    ->get();
```

### Pattern 6: Polymorphic One-to-Many

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;
use Illuminate\Database\Eloquent\Relations\MorphMany;

class Comment extends Model
{
    protected $fillable = ['content', 'user_id'];

    /**
     * Get the parent commentable model (post or video).
     */
    public function commentable(): MorphTo
    {
        return $this->morphTo();
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

class Post extends Model
{
    /**
     * Get all comments for the post.
     */
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Video extends Model
{
    /**
     * Get all comments for the video.
     */
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Usage examples
$post = Post::find(1);
$post->comments()->create([
    'content' => 'Great post!',
    'user_id' => auth()->id()
]);

$video = Video::find(1);
$video->comments()->create([
    'content' => 'Nice video!',
    'user_id' => auth()->id()
]);

// Retrieve parent model
$comment = Comment::find(1);
$commentable = $comment->commentable; // Returns Post or Video

// Query by type
$postComments = Comment::where('commentable_type', Post::class)->get();

// Eager load polymorphic relationships
$comments = Comment::with('commentable')->get();
```

### Pattern 7: Polymorphic Many-to-Many

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphToMany;
use Illuminate\Database\Eloquent\Relations\MorphedByMany;

class Tag extends Model
{
    protected $fillable = ['name'];

    /**
     * Get all posts with this tag.
     */
    public function posts(): MorphedByMany
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }

    /**
     * Get all videos with this tag.
     */
    public function videos(): MorphedByMany
    {
        return $this->morphedByMany(Video::class, 'taggable');
    }
}

class Post extends Model
{
    /**
     * Get all tags for the post.
     */
    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable')
            ->withTimestamps();
    }
}

class Video extends Model
{
    /**
     * Get all tags for the video.
     */
    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable')
            ->withTimestamps();
    }
}

// Migration for taggables table
// Schema::create('taggables', function (Blueprint $table) {
//     $table->foreignId('tag_id')->constrained()->cascadeOnDelete();
//     $table->morphs('taggable'); // Creates taggable_id and taggable_type
//     $table->timestamps();
// });

// Usage
$post = Post::find(1);
$post->tags()->attach([1, 2, 3]);

$video = Video::find(1);
$video->tags()->sync([2, 3, 4]);

// Query posts by tag
$tag = Tag::find(1);
$posts = $tag->posts;
$videos = $tag->videos;

// Get all taggable models
$allTagged = $tag->posts->merge($tag->videos);
```

### Pattern 8: Eager Loading and N+1 Prevention

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use App\Models\Post;
use Illuminate\Support\Facades\DB;

class PostController extends Controller
{
    /**
     * BAD: N+1 Problem - Causes 101 queries for 100 posts
     */
    public function indexBad()
    {
        $posts = Post::all(); // 1 query

        foreach ($posts as $post) {
            echo $post->author->name; // 100 additional queries (1 per post)
        }
    }

    /**
     * GOOD: Eager loading - Only 2 queries
     */
    public function indexGood()
    {
        // Load posts with authors in 2 queries total
        $posts = Post::with('author')->get();

        foreach ($posts as $post) {
            echo $post->author->name; // No additional queries
        }
    }

    /**
     * Multiple relationships
     */
    public function showWithRelationships($id)
    {
        return Post::with(['author', 'comments', 'tags'])
            ->findOrFail($id);
    }

    /**
     * Nested eager loading
     */
    public function indexWithNested()
    {
        return Post::with([
            'author.profile',           // Load author and their profile
            'comments.author',          // Load comments and comment authors
            'comments.replies.author'   // Load comment replies and their authors
        ])->get();
    }

    /**
     * Constrained eager loading
     */
    public function indexWithConstraints()
    {
        return Post::with([
            'comments' => function ($query) {
                $query->where('approved', true)
                    ->orderBy('created_at', 'desc')
                    ->limit(10);
            },
            'author' => function ($query) {
                $query->select('id', 'name', 'email'); // Only load specific columns
            }
        ])->get();
    }

    /**
     * Lazy eager loading (load after initial query)
     */
    public function lazyEagerLoading()
    {
        $posts = Post::all();

        // Later, decide to load relationships
        if (someCondition()) {
            $posts->load('author', 'comments');
        }

        return $posts;
    }

    /**
     * Count relationships without loading
     */
    public function withCounts()
    {
        return Post::withCount('comments')
            ->withCount(['comments as approved_comments_count' => function ($query) {
                $query->where('approved', true);
            }])
            ->get();
    }

    /**
     * Check relationship existence
     */
    public function postsWithComments()
    {
        // Only posts that have comments
        return Post::has('comments')->get();

        // Posts with at least 10 comments
        return Post::has('comments', '>=', 10)->get();

        // Posts with approved comments
        return Post::whereHas('comments', function ($query) {
            $query->where('approved', true);
        })->get();
    }
}
```

### Pattern 9: Relationship Query Methods

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    public function posts()
    {
        return $this->hasMany(Post::class);
    }

    public function publishedPosts()
    {
        return $this->hasMany(Post::class)
            ->where('status', 'published')
            ->orderBy('published_at', 'desc');
    }

    public function comments()
    {
        return $this->hasMany(Comment::class);
    }

    public function roles()
    {
        return $this->belongsToMany(Role::class);
    }
}

// Query relationship methods
$user = User::find(1);

// Check if relationship exists
$hasComments = $user->comments()->exists();
$doesntHaveComments = $user->comments()->doesntExist();

// Get first or create
$comment = $user->comments()->firstOrCreate(
    ['content' => 'First!'],
    ['approved' => false]
);

// First or new (doesn't save)
$comment = $user->comments()->firstOrNew(['content' => 'Draft']);

// Update or create
$post = $user->posts()->updateOrCreate(
    ['slug' => 'my-first-post'],
    ['title' => 'My First Post', 'content' => '...']
);

// Find or create
$post = $user->posts()->findOr(1, function () use ($user) {
    return $user->posts()->create([
        'title' => 'Default Post',
        'content' => 'Default content'
    ]);
});

// Paginate relationships
$comments = $user->comments()->paginate(20);

// Cursor pagination
$comments = $user->comments()->cursorPaginate(20);

// Chunk relationships
$user->posts()->chunk(100, function ($posts) {
    foreach ($posts as $post) {
        // Process post
    }
});

// Delete related models
$user->posts()->delete();

// Force delete (if using soft deletes)
$user->posts()->forceDelete();
```

### Pattern 10: Custom Relationship Methods

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class User extends Model
{
    /**
     * Get the user's posts.
     */
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    /**
     * Get published posts with custom query.
     */
    public function publishedPosts(): HasMany
    {
        return $this->posts()
            ->where('status', 'published')
            ->where('published_at', '<=', now());
    }

    /**
     * Get popular posts (custom scope in relationship).
     */
    public function popularPosts(): HasMany
    {
        return $this->hasMany(Post::class)
            ->where('views', '>=', 1000)
            ->orderBy('views', 'desc');
    }

    /**
     * Get posts from last month.
     */
    public function recentPosts(): HasMany
    {
        return $this->hasMany(Post::class)
            ->whereBetween('created_at', [
                now()->subMonth(),
                now()
            ]);
    }

    /**
     * Helper method to get post count efficiently.
     */
    public function getPostCountAttribute(): int
    {
        return $this->posts()->count();
    }

    /**
     * Check if user has role.
     */
    public function hasRole(string $roleName): bool
    {
        return $this->roles()
            ->where('name', $roleName)
            ->exists();
    }

    /**
     * Assign role to user.
     */
    public function assignRole(string $roleName): void
    {
        $role = Role::where('name', $roleName)->firstOrFail();
        $this->roles()->attach($role->id);
    }
}

class Post extends Model
{
    /**
     * Scope for published posts.
     */
    public function scopePublished($query)
    {
        return $query->where('status', 'published')
            ->where('published_at', '<=', now());
    }

    /**
     * Scope for popular posts.
     */
    public function scopePopular($query, $threshold = 1000)
    {
        return $query->where('views', '>=', $threshold);
    }
}

// Usage
$user = User::find(1);
$publishedPosts = $user->publishedPosts;
$popularPosts = $user->popularPosts;
$recentPosts = $user->recentPosts;

// Use with scopes
$posts = $user->posts()->published()->popular()->get();
```

## Advanced Patterns

### Pattern 11: Relationship Exists Queries

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use App\Models\Post;

class UserController extends Controller
{
    /**
     * Users who have written posts.
     */
    public function usersWithPosts()
    {
        return User::has('posts')->get();
    }

    /**
     * Users with at least 5 posts.
     */
    public function prolificAuthors()
    {
        return User::has('posts', '>=', 5)->get();
    }

    /**
     * Users with published posts.
     */
    public function authorsWithPublishedPosts()
    {
        return User::whereHas('posts', function ($query) {
            $query->where('status', 'published');
        })->get();
    }

    /**
     * Users without any posts.
     */
    public function usersWithoutPosts()
    {
        return User::doesntHave('posts')->get();
    }

    /**
     * Users without published posts.
     */
    public function usersWithoutPublishedPosts()
    {
        return User::whereDoesntHave('posts', function ($query) {
            $query->where('status', 'published');
        })->get();
    }

    /**
     * Complex nested relationship queries.
     */
    public function usersWithApprovedComments()
    {
        return User::whereHas('posts.comments', function ($query) {
            $query->where('approved', true);
        })->get();
    }

    /**
     * Multiple relationship existence checks.
     */
    public function activeUsers()
    {
        return User::whereHas('posts')
            ->whereHas('comments')
            ->whereHas('roles', function ($query) {
                $query->where('name', '!=', 'banned');
            })
            ->get();
    }

    /**
     * Relationship counts with constraints.
     */
    public function usersWithCounts()
    {
        return User::withCount([
            'posts',
            'posts as published_posts_count' => function ($query) {
                $query->where('status', 'published');
            },
            'comments',
            'comments as approved_comments_count' => function ($query) {
                $query->where('approved', true);
            }
        ])->get();
    }
}
```

### Pattern 12: Touching Parent Timestamps

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Comment extends Model
{
    /**
     * All relationships to touch when comment is saved.
     */
    protected $touches = ['post', 'author'];

    public function post()
    {
        return $this->belongsTo(Post::class);
    }

    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}

class Post extends Model
{
    /**
     * Touch parent when post is updated.
     */
    protected $touches = ['author'];

    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    public function comments()
    {
        return $this->hasMany(Comment::class);
    }
}

// When a comment is created or updated, both the post
// and the author's updated_at timestamps are automatically updated
$comment = new Comment(['content' => 'Great post!']);
$comment->post_id = 1;
$comment->user_id = 2;
$comment->save(); // Updates comment, post, and user timestamps
```

### Pattern 13: Default Models for Relationships

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    /**
     * Get the author (with default if none exists).
     */
    public function author()
    {
        return $this->belongsTo(User::class, 'user_id')
            ->withDefault([
                'name' => 'Guest Author',
                'email' => 'guest@example.com'
            ]);
    }

    /**
     * Alternative: Use callback for default.
     */
    public function category()
    {
        return $this->belongsTo(Category::class)
            ->withDefault(function ($category, $post) {
                $category->name = 'Uncategorized';
                $category->slug = 'uncategorized';
            });
    }
}

// Usage - No need to check if author exists
$post = Post::find(1);
echo $post->author->name; // Returns 'Guest Author' if no author
```

### Pattern 14: Querying Relationship Aggregate Functions

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use App\Models\Post;
use Illuminate\Support\Facades\DB;

class AnalyticsController extends Controller
{
    /**
     * Get users with aggregate data.
     */
    public function userStats()
    {
        return User::withCount('posts')
            ->withSum('posts', 'views')
            ->withAvg('posts', 'rating')
            ->withMax('posts', 'created_at')
            ->withMin('posts', 'created_at')
            ->get();
    }

    /**
     * Custom aggregate queries.
     */
    public function customAggregates()
    {
        return User::withCount([
            'posts as published_posts_count' => function ($query) {
                $query->where('status', 'published');
            },
            'posts as draft_posts_count' => function ($query) {
                $query->where('status', 'draft');
            }
        ])
        ->withSum([
            'posts as total_views' => function ($query) {
                $query->where('status', 'published');
            }
        ], 'views')
        ->get();
    }

    /**
     * Load aggregates on existing collections.
     */
    public function loadAggregates()
    {
        $users = User::all();

        $users->loadCount('posts');
        $users->loadSum('posts', 'views');
        $users->loadAvg('posts', 'rating');

        return $users;
    }
}
```

### Pattern 15: Relationship Caching

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Cache;

class User extends Model
{
    /**
     * Get cached posts.
     */
    public function getCachedPostsAttribute()
    {
        return Cache::remember(
            "user.{$this->id}.posts",
            now()->addHour(),
            fn () => $this->posts()->get()
        );
    }

    /**
     * Clear posts cache when relationship changes.
     */
    protected static function booted()
    {
        static::saved(function ($user) {
            Cache::forget("user.{$user->id}.posts");
        });
    }

    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}

class Post extends Model
{
    protected static function booted()
    {
        // Clear cache when post is created/updated/deleted
        static::saved(function ($post) {
            Cache::forget("user.{$post->user_id}.posts");
        });

        static::deleted(function ($post) {
            Cache::forget("user.{$post->user_id}.posts");
        });
    }

    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}
```

## Real-World Applications

### Application 1: Blog with Categories and Tags

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;

    protected $fillable = ['title', 'slug', 'content', 'status', 'published_at'];

    protected $casts = [
        'published_at' => 'datetime',
    ];

    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    public function category()
    {
        return $this->belongsTo(Category::class);
    }

    public function tags()
    {
        return $this->belongsToMany(Tag::class)
            ->withTimestamps();
    }

    public function comments()
    {
        return $this->hasMany(Comment::class);
    }

    public function approvedComments()
    {
        return $this->hasMany(Comment::class)
            ->where('approved', true)
            ->orderBy('created_at', 'desc');
    }

    public function meta()
    {
        return $this->hasOne(PostMeta::class);
    }

    // Scopes
    public function scopePublished($query)
    {
        return $query->where('status', 'published')
            ->where('published_at', '<=', now());
    }

    public function scopeWithRelations($query)
    {
        return $query->with([
            'author:id,name,email',
            'category:id,name,slug',
            'tags:id,name,slug',
            'meta'
        ])->withCount(['comments', 'approvedComments']);
    }
}

// Controller usage
class PostController extends Controller
{
    public function index()
    {
        return Post::published()
            ->withRelations()
            ->orderBy('published_at', 'desc')
            ->paginate(20);
    }

    public function show($slug)
    {
        return Post::where('slug', $slug)
            ->published()
            ->with([
                'author.profile',
                'category',
                'tags',
                'approvedComments.author',
                'meta'
            ])
            ->firstOrFail();
    }
}
```

### Application 2: E-commerce with Products and Orders

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    protected $fillable = ['name', 'slug', 'price', 'description'];

    public function category()
    {
        return $this->belongsTo(Category::class);
    }

    public function images()
    {
        return $this->morphMany(Image::class, 'imageable')
            ->orderBy('order');
    }

    public function reviews()
    {
        return $this->hasMany(Review::class);
    }

    public function orders()
    {
        return $this->belongsToMany(Order::class, 'order_items')
            ->withPivot('quantity', 'price', 'subtotal')
            ->withTimestamps();
    }
}

class Order extends Model
{
    protected $fillable = ['user_id', 'status', 'total', 'notes'];

    protected $casts = [
        'total' => 'decimal:2',
    ];

    public function customer()
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    public function products()
    {
        return $this->belongsToMany(Product::class, 'order_items')
            ->withPivot('quantity', 'price', 'subtotal')
            ->withTimestamps();
    }

    public function items()
    {
        return $this->hasMany(OrderItem::class);
    }

    public function shippingAddress()
    {
        return $this->hasOne(OrderAddress::class)
            ->where('type', 'shipping');
    }

    public function billingAddress()
    {
        return $this->hasOne(OrderAddress::class)
            ->where('type', 'billing');
    }

    public function payment()
    {
        return $this->hasOne(Payment::class);
    }
}

// Usage in service class
class OrderService
{
    public function getOrderWithDetails($orderId)
    {
        return Order::with([
            'customer:id,name,email',
            'products.category',
            'products.images',
            'items.product',
            'shippingAddress',
            'billingAddress',
            'payment'
        ])->findOrFail($orderId);
    }

    public function getUserOrders($userId)
    {
        return Order::where('user_id', $userId)
            ->with(['items.product', 'payment'])
            ->withCount('items')
            ->withSum('items', 'subtotal')
            ->orderBy('created_at', 'desc')
            ->paginate(10);
    }
}
```

## Performance Best Practices

### Practice 1: Always Eager Load in Loops

```php
<?php

// BAD: N+1 queries
$users = User::all();
foreach ($users as $user) {
    echo $user->profile->bio; // Query per user
}

// GOOD: Eager load
$users = User::with('profile')->get();
foreach ($users as $user) {
    echo $user->profile->bio; // No additional queries
}

// GOOD: Selective columns
$users = User::with('profile:user_id,bio')->get();
```

### Practice 2: Use Exists Instead of Count

```php
<?php

// BAD: Loads all comments to check existence
if (count($post->comments) > 0) {
    // Has comments
}

// GOOD: Uses EXISTS query
if ($post->comments()->exists()) {
    // Has comments
}

// GOOD: Direct count query
if ($post->comments()->count() > 0) {
    // Has comments
}
```

### Practice 3: Chunk Large Datasets

```php
<?php

// BAD: Loads all users into memory
$users = User::with('posts')->get();
foreach ($users as $user) {
    // Process user
}

// GOOD: Process in chunks
User::with('posts')->chunk(100, function ($users) {
    foreach ($users as $user) {
        // Process user
    }
});

// GOOD: Cursor for memory efficiency
foreach (User::with('posts')->cursor() as $user) {
    // Process user
}
```

### Practice 4: Load Only Required Columns

```php
<?php

// BAD: Loads all columns
$posts = Post::with('author')->get();

// GOOD: Select specific columns
$posts = Post::select('id', 'title', 'user_id', 'created_at')
    ->with('author:id,name,email')
    ->get();

// GOOD: Use relationship column selection
$posts = Post::with([
    'author' => function ($query) {
        $query->select('id', 'name', 'email');
    }
])->get();
```

### Practice 5: Use Relationship Counts

```php
<?php

// BAD: Loads all comments
$posts = Post::with('comments')->get();
foreach ($posts as $post) {
    echo count($post->comments); // All comments loaded
}

// GOOD: Only count, don't load
$posts = Post::withCount('comments')->get();
foreach ($posts as $post) {
    echo $post->comments_count; // No comments loaded
}
```

## Common Pitfalls

### Pitfall 1: N+1 Query Problem

```php
<?php

// WRONG: N+1 queries
$posts = Post::all(); // 1 query
foreach ($posts as $post) {
    echo $post->author->name; // N queries
}

// CORRECT: Eager loading
$posts = Post::with('author')->get(); // 2 queries total
foreach ($posts as $post) {
    echo $post->author->name;
}
```

### Pitfall 2: Loading Too Much Data

```php
<?php

// WRONG: Load all columns and relationships
$users = User::with(['posts', 'comments', 'profile', 'roles'])->get();

// CORRECT: Load only what you need
$users = User::select('id', 'name', 'email')
    ->with('profile:user_id,bio')
    ->get();
```

### Pitfall 3: Incorrect Relationship Keys

```php
<?php

// WRONG: Incorrect foreign key assumption
class Post extends Model
{
    public function author()
    {
        // Assumes 'author_id' but table uses 'user_id'
        return $this->belongsTo(User::class);
    }
}

// CORRECT: Specify correct foreign key
class Post extends Model
{
    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}
```

### Pitfall 4: Forgetting Inverse Relationships

```php
<?php

// WRONG: Only one direction
class User extends Model
{
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}
// Post model has no relationship back to User

// CORRECT: Both directions
class User extends Model
{
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}

class Post extends Model
{
    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}
```

### Pitfall 5: Not Using withDefault

```php
<?php

// WRONG: Can cause null pointer errors
$post = Post::find(1);
echo $post->author->name; // Error if author is null

// CORRECT: Use withDefault
class Post extends Model
{
    public function author()
    {
        return $this->belongsTo(User::class)
            ->withDefault(['name' => 'Unknown Author']);
    }
}

echo $post->author->name; // Safe, returns 'Unknown Author' if null
```

## Testing

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use App\Models\Post;
use App\Models\Comment;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class RelationshipTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_has_many_posts()
    {
        $user = User::factory()->create();
        $posts = Post::factory()->count(3)->create(['user_id' => $user->id]);

        $this->assertCount(3, $user->posts);
        $this->assertTrue($user->posts->contains($posts->first()));
    }

    public function test_post_belongs_to_user()
    {
        $user = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $user->id]);

        $this->assertInstanceOf(User::class, $post->author);
        $this->assertEquals($user->id, $post->author->id);
    }

    public function test_eager_loading_prevents_n_plus_one()
    {
        $users = User::factory()->count(10)->create();
        $users->each(fn ($user) => Post::factory()->count(5)->create(['user_id' => $user->id]));

        // Enable query log
        \DB::enableQueryLog();

        // Load with eager loading
        $posts = Post::with('author')->get();
        $posts->each(fn ($post) => $post->author->name);

        // Should be 2 queries: 1 for posts, 1 for users
        $this->assertCount(2, \DB::getQueryLog());
    }

    public function test_many_to_many_relationship()
    {
        $user = User::factory()->create();
        $roles = Role::factory()->count(3)->create();

        $user->roles()->attach($roles->pluck('id'));

        $this->assertCount(3, $user->roles);
        $this->assertTrue($user->roles->contains($roles->first()));
    }
}
```

## Resources

- **Laravel Eloquent Relationships**: https://laravel.com/docs/eloquent-relationships
- **Eager Loading**: https://laravel.com/docs/eloquent-relationships#eager-loading
- **Polymorphic Relationships**: https://laravel.com/docs/eloquent-relationships#polymorphic-relationships
- **Laravel Debugbar**: Debug N+1 queries
- **Laravel Telescope**: Monitor database queries
- **Clockwork**: Browser extension for debugging

## Best Practices Summary

1. **Always eager load** relationships when accessing them in loops
2. **Use withCount()** instead of loading full relationships for counts
3. **Define inverse relationships** for bidirectional access
4. **Specify foreign keys** explicitly when they don't follow conventions
5. **Use relationship constraints** to filter related data
6. **Implement withDefault()** to prevent null errors on belongsTo
7. **Leverage relationship methods** like exists(), create(), updateOrCreate()
8. **Monitor queries** with Debugbar or Telescope to catch N+1 problems
9. **Load only required columns** from relationships
10. **Use cursor()** or chunk() for large datasets with relationships
