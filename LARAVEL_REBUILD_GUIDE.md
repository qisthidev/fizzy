# 🚀 Laravel Rebuild Guide for Fizzy

> **A comprehensive documentation for rebuilding Fizzy as a Laravel application**  
> This guide serves as a learning resource for building enterprise-level SaaS applications.

---

## 📋 Table of Contents

1. [Application Overview](#-application-overview)
2. [Architecture Overview](#-architecture-overview)
3. [Database Schema & Models](#-database-schema--models)
4. [Routes / API Structure](#-routes--api-structure)
5. [Background Jobs](#-background-jobs)
6. [Mailers](#-mailers)
7. [Key Features to Implement](#-key-features-to-implement)
8. [Suggested Laravel Project Structure](#-suggested-laravel-project-structure)
9. [Getting Started](#-getting-started)
10. [Recommended Laravel Packages](#-recommended-laravel-packages)
11. [Learning Outcomes](#-learning-outcomes)

---

## 📖 Application Overview

**Fizzy** is a collaborative project management and issue tracking application built as a Kanban-style tool. It enables teams to create and manage cards (tasks/issues) across boards, organize work into columns representing workflow stages, and collaborate via comments, mentions, and assignments.

### Core Features

| Feature | Description |
|---------|-------------|
| 🗂️ **Kanban Boards** | Customizable boards with drag-and-drop columns for workflow stages |
| 📝 **Cards (Issues/Tasks)** | Rich text descriptions, attachments, checklists, and due dates |
| 🏢 **Multi-tenant Architecture** | Accounts with multiple users, URL-based tenant isolation |
| ⚡ **Real-time Updates** | WebSocket broadcasting for live collaboration |
| 🔐 **Magic Link Authentication** | Passwordless sign-in via email magic links |
| 🔔 **Web Push Notifications** | Browser push notifications for updates |
| 🔗 **Webhooks** | Integration webhooks for Slack, Campfire, Basecamp |
| 🔍 **Full-text Search** | Partitioned MySQL FULLTEXT search across cards and comments |
| 📊 **Activity Timeline/Events** | Comprehensive activity logging and timeline view |
| 📎 **File Attachments** | Cloud storage (S3) for images and file attachments |
| 📱 **PWA Support** | Progressive Web App with service worker |
| 📤 **Data Export** | Full account data export functionality |
| 🏷️ **Tags & Taggings** | Polymorphic tagging system for cards |
| 👥 **Assignments** | Card-to-user assignments with notifications |
| ✅ **Checklists (Steps)** | Multi-step checklists on cards |
| ⏰ **Entropy/Auto-postpone** | Automatic postponement of stale cards |
| 🌍 **Public Boards** | Shareable public board URLs |

---

## 🏗️ Architecture Overview

### Technology Mapping: Rails → Laravel

| Rails Component | Laravel Equivalent | Notes |
|----------------|-------------------|-------|
| Ruby 3.x | PHP 8.2+ | Modern PHP with typed properties |
| Rails 8.x | Laravel 11.x | Full-stack framework |
| Solid Queue | Laravel Horizon + Redis | Job queue with dashboard |
| Solid Cache | Laravel Cache (Redis) | Caching layer |
| Solid Cable | Laravel Broadcasting (Pusher/Soketi) | WebSocket server |
| Action Cable | Laravel Echo + WebSockets | Real-time client |
| Active Storage | Laravel Storage + Spatie Media Library | File attachments |
| Action Text | TipTap/Trix integration | Rich text editing |
| Turbo/Stimulus | Livewire or Inertia.js | Frontend reactivity |
| SQLite/MySQL | MySQL/PostgreSQL | Primary database |
| Importmap | Vite | Asset bundling |
| ActiveRecord | Eloquent ORM | Database abstraction |
| RSpec/Minitest | PHPUnit/Pest | Testing framework |
| Rubocop | Laravel Pint/PHP CS Fixer | Code style |
| Brakeman | Larastan/PHPStan | Static analysis |

### Multi-Tenancy Architecture

Fizzy uses **URL path-based multi-tenancy**:

```
/{account_id}/boards/...
/{account_id}/cards/...
/{account_id}/users/...
```

**Key Concepts:**
- Each Account (tenant) has a unique `external_account_id` (7+ digits)
- Middleware extracts account ID from URL and sets current context
- All models include `account_id` for data isolation
- Background jobs automatically serialize and restore account context

---

## 🗄️ Database Schema & Models

### Core Entities

#### 1. Accounts (Multi-tenancy)

The root tenant model. All data belongs to an account.

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('accounts', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->string('name');
            $table->bigInteger('external_account_id')->unique();
            $table->bigInteger('cards_count')->default(0);
            $table->timestamps();
            
            $table->index('external_account_id');
        });
    }
};
```

**Model Traits:** `Entropic`, `Seedeable`

**Relationships:**
- `hasOne` JoinCode
- `hasMany` Users, Boards, Cards, Webhooks, Tags, Columns, Exports

---

#### 2. Users

Account membership with roles.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('identity_id')->nullable()->constrained()->nullOnDelete();
            $table->string('name');
            $table->string('role')->default('member'); // owner, admin, member, system
            $table->boolean('active')->default(true);
            $table->timestamps();
            
            $table->unique(['account_id', 'identity_id']);
            $table->index(['account_id', 'role']);
        });
    }
};
```

**Model Traits:**
- `Accessor` - Board access management
- `Assignee` - Card assignments
- `Attachable` - Avatar attachments
- `Configurable` - User settings
- `EmailAddressChangeable` - Email management
- `Mentionable` - @mention support
- `Named` - Name utilities
- `Notifiable` - Notification handling
- `Role` - Role management
- `Searcher` - Search functionality
- `Watcher` - Card watching
- `Timelined` - Activity timeline

---

#### 3. Identities

Global user identity (email-based), separate from account membership.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('identities', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->string('email_address')->unique();
            $table->boolean('staff')->default(false);
            $table->timestamps();
        });
    }
};
```

**Model Traits:** `Joinable`, `Transferable`

**Key Concept:** An Identity can have Users in multiple Accounts, enabling cross-account authentication.

---

#### 4. Sessions

User session management with device tracking.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('sessions', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('identity_id')->constrained()->cascadeOnDelete();
            $table->string('ip_address')->nullable();
            $table->string('user_agent', 4096)->nullable();
            $table->timestamps();
            
            $table->index('identity_id');
        });
    }
};
```

---

#### 5. Magic Links

Passwordless authentication tokens.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('magic_links', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('identity_id')->nullable()->constrained()->cascadeOnDelete();
            $table->string('code', 6)->unique();
            $table->integer('purpose'); // sign_in, sign_up
            $table->timestamp('expires_at');
            $table->timestamps();
            
            $table->index('expires_at');
        });
    }
};
```

**Constants:**
- `CODE_LENGTH = 6`
- `EXPIRATION_TIME = 15 minutes`

---

#### 6. Boards

Primary organizational unit for cards.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('boards', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('creator_id')->constrained('users');
            $table->string('name');
            $table->boolean('all_access')->default(false);
            $table->timestamps();
            
            $table->index('account_id');
            $table->index('creator_id');
        });
    }
};
```

**Model Traits:**
- `Accessible` - Access control
- `AutoPostponing` - Stale card detection
- `Broadcastable` - WebSocket events
- `Cards` - Card relationships
- `Entropic` - Entropy configuration
- `Filterable` - Filter support
- `Publishable` - Public board sharing
- `Triageable` - Triage column

---

#### 7. Columns

Kanban columns with positioning and colors.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('columns', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('board_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->string('color');
            $table->integer('position')->default(0);
            $table->timestamps();
            
            $table->index(['board_id', 'position']);
        });
    }
};
```

**Model Traits:** `Colored`, `Positioned`

---

#### 8. Cards

The core work item entity.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('cards', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('board_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('column_id')->nullable()->constrained()->nullOnDelete();
            $table->foreignUuid('creator_id')->constrained('users');
            $table->string('title')->nullable();
            $table->string('status')->default('drafted'); // drafted, published
            $table->bigInteger('number');
            $table->timestamp('last_active_at');
            $table->date('due_on')->nullable();
            $table->timestamps();
            
            $table->unique(['account_id', 'number']);
            $table->index(['account_id', 'last_active_at', 'status']);
            $table->index('board_id');
            $table->index('column_id');
        });
    }
};
```

**Model Traits (17+):**
- `Assignable` - User assignments
- `Attachments` - File attachments
- `Broadcastable` - WebSocket events
- `Closeable` - Close/reopen
- `Colored` - Color from column
- `Entropic` - Auto-postpone detection
- `Eventable` - Activity tracking
- `Exportable` - Data export
- `Golden` - Golden card status
- `Mentions` - @mention parsing
- `Multistep` - Checklist steps
- `Pinnable` - Pin to dashboard
- `Postponable` - Not now/postpone
- `Promptable` - AI prompts
- `Readable` - Read tracking
- `Searchable` - Full-text search
- `Stallable` - Stale detection
- `Statuses` - Status management
- `Taggable` - Tags
- `Triageable` - Triage workflow
- `Watchable` - Watch/unwatch

---

#### 9. Comments

Card discussions with rich text and reactions.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('comments', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('card_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('creator_id')->constrained('users');
            $table->timestamps();
            
            $table->index('card_id');
        });

        // Rich text body stored in action_text_rich_texts table
    }
};
```

**Model Traits:** `Attachments`, `Eventable`, `Mentions`, `Promptable`, `Searchable`

---

#### 10. Assignments

Card-to-user assignments.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('assignments', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('card_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('assignee_id')->constrained('users');
            $table->foreignUuid('assigner_id')->constrained('users');
            $table->timestamps();
            
            $table->unique(['assignee_id', 'card_id']);
            $table->index('card_id');
        });
    }
};
```

---

#### 11. Tags & Taggings

Polymorphic tagging system.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('tags', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->string('title');
            $table->timestamps();
            
            $table->unique(['account_id', 'title']);
        });

        Schema::create('taggings', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('card_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('tag_id')->constrained()->cascadeOnDelete();
            $table->timestamps();
            
            $table->unique(['card_id', 'tag_id']);
        });
    }
};
```

---

#### 12. Steps (Checklists)

Checklist items on cards.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('steps', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('card_id')->constrained()->cascadeOnDelete();
            $table->text('content');
            $table->boolean('completed')->default(false);
            $table->timestamps();
            
            $table->index(['card_id', 'completed']);
        });
    }
};
```

---

#### 13. Events

Activity log with polymorphic eventable.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('events', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('board_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('creator_id')->constrained('users');
            $table->string('action');
            $table->uuidMorphs('eventable');
            $table->json('particulars')->default('{}');
            $table->timestamps();
            
            $table->index(['account_id', 'action']);
            $table->index(['board_id', 'action', 'created_at']);
        });
    }
};
```

**Model Traits:** `Notifiable`, `Particulars`, `Promptable`

**Event Actions:**
- `card_assigned`, `card_unassigned`
- `card_closed`, `card_reopened`
- `card_postponed`, `card_auto_postponed`
- `card_board_changed`
- `card_published`
- `card_sent_back_to_triage`, `card_triaged`
- `comment_created`

---

#### 14. Notifications

In-app notifications with broadcasting.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('notifications', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('user_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('creator_id')->nullable()->constrained('users');
            $table->uuidMorphs('source');
            $table->timestamp('read_at')->nullable();
            $table->timestamps();
            
            $table->index(['user_id', 'read_at', 'created_at']);
        });
    }
};
```

**Model Traits:** `PushNotifiable`

---

#### 15. Notification Bundles

Email digest bundling.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('notification_bundles', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('user_id')->constrained()->cascadeOnDelete();
            $table->timestamp('starts_at');
            $table->timestamp('ends_at');
            $table->integer('status')->default(0); // pending, delivered
            $table->timestamps();
            
            $table->index(['ends_at', 'status']);
            $table->index(['user_id', 'status']);
        });
    }
};
```

---

#### 16. Accesses

Board access control.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('accesses', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('board_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('user_id')->constrained()->cascadeOnDelete();
            $table->string('involvement')->default('access_only'); // access_only, watching
            $table->timestamp('accessed_at')->nullable();
            $table->timestamps();
            
            $table->unique(['board_id', 'user_id']);
            $table->index(['account_id', 'accessed_at']);
        });
    }
};
```

---

#### 17. Webhooks & Webhook Deliveries

Integration webhooks for external services.

```php
<?php

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('webhooks', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('board_id')->constrained()->cascadeOnDelete();
            $table->string('name')->nullable();
            $table->text('url');
            $table->string('signing_secret');
            $table->text('subscribed_actions')->nullable(); // JSON array
            $table->boolean('active')->default(true);
            $table->timestamps();
        });

        Schema::create('webhook_deliveries', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('webhook_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('event_id')->constrained()->cascadeOnDelete();
            $table->string('state'); // pending, delivered, failed
            $table->text('request')->nullable();
            $table->text('response')->nullable();
            $table->timestamps();
        });

        Schema::create('webhook_delinquency_trackers', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('account_id')->constrained()->cascadeOnDelete();
            $table->foreignUuid('webhook_id')->constrained()->cascadeOnDelete();
            $table->integer('consecutive_failures_count')->default(0);
            $table->timestamp('first_failure_at')->nullable();
            $table->timestamps();
        });
    }
};
```

---

### Additional Tables

| Table | Purpose |
|-------|---------|
| `pins` | User pinned cards |
| `watches` | Card watching preferences |
| `closures` | Card closure records |
| `reactions` | Comment emoji reactions |
| `mentions` | @mention tracking |
| `filters` | Saved filter configurations |
| `board_publications` | Public board sharing keys |
| `card_goldnesses` | Golden card status |
| `card_not_nows` | Postponed card tracking |
| `card_activity_spikes` | Activity spike detection |
| `card_engagements` | Engagement tracking |
| `entropies` | Auto-postpone configuration |
| `push_subscriptions` | Web push endpoints |
| `user_settings` | User preferences |
| `account_join_codes` | Invite codes |
| `account_exports` | Data export jobs |
| `search_records_*` | Partitioned full-text search (16 shards) |

---

## 🛤️ Routes / API Structure

### Authentication Routes

```php
<?php

use Illuminate\Support\Facades\Route;

// Session Management
Route::resource('session', SessionController::class)->only(['create', 'store', 'destroy']);
Route::prefix('session')->group(function () {
    Route::resource('transfers', Session\TransferController::class);
    Route::resource('magic-link', Session\MagicLinkController::class)->only(['create', 'store']);
    Route::get('menu', [Session\MenuController::class, 'show']);
});

// Signup
Route::get('signup', fn() => redirect('/signup/new'));
Route::resource('signup', SignupController::class)->only(['create', 'store']);
Route::prefix('signup')->group(function () {
    Route::get('completion/new', [Signup\CompletionController::class, 'create']);
    Route::post('completion', [Signup\CompletionController::class, 'store']);
});

// Join via invite code
Route::get('join/{code}', [JoinCodeController::class, 'create'])->name('join');
Route::post('join/{code}', [JoinCodeController::class, 'store']);
```

### Account Management Routes

```php
<?php

Route::prefix('account')->name('account.')->group(function () {
    Route::resource('join-code', Account\JoinCodeController::class)->only(['show', 'update', 'destroy']);
    Route::resource('settings', Account\SettingsController::class)->only(['show', 'update']);
    Route::resource('entropy', Account\EntropyController::class)->only(['show', 'update']);
    Route::resource('exports', Account\ExportController::class)->only(['create', 'show']);
});
```

### Users Routes

```php
<?php

Route::resource('users', UserController::class);
Route::prefix('users/{user}')->group(function () {
    Route::resource('avatar', User\AvatarController::class)->only(['show', 'update', 'destroy']);
    Route::resource('role', User\RoleController::class)->only(['update']);
    Route::resource('events', User\EventController::class)->only(['show']);
    Route::resource('push-subscriptions', User\PushSubscriptionController::class);
    Route::resource('email-addresses', User\EmailAddressController::class)->parameters(['email-addresses' => 'token']);
    Route::post('email-addresses/{token}/confirmation', [User\EmailAddress\ConfirmationController::class, 'store']);
});
```

### Boards Routes

```php
<?php

Route::resource('boards', BoardController::class);
Route::prefix('boards/{board}')->group(function () {
    Route::resource('subscriptions', Board\SubscriptionController::class)->only(['show', 'update']);
    Route::resource('involvement', Board\InvolvementController::class)->only(['show', 'update']);
    Route::resource('publication', Board\PublicationController::class)->only(['show', 'store', 'destroy']);
    Route::resource('entropy', Board\EntropyController::class)->only(['show', 'update']);
    
    // Special columns
    Route::prefix('columns')->name('columns.')->group(function () {
        Route::get('not-now', [Board\Column\NotNowController::class, 'show']);
        Route::get('stream', [Board\Column\StreamController::class, 'show']);
        Route::get('closed', [Board\Column\ClosedController::class, 'show']);
    });
    
    Route::resource('columns', Board\ColumnController::class);
    Route::resource('cards', CardController::class)->only(['store']);
    
    // Webhooks
    Route::resource('webhooks', WebhookController::class);
    Route::post('webhooks/{webhook}/activation', [Webhook\ActivationController::class, 'store']);
});

// Column positioning
Route::prefix('columns/{column}')->group(function () {
    Route::resource('left-position', Column\LeftPositionController::class)->only(['update']);
    Route::resource('right-position', Column\RightPositionController::class)->only(['update']);
});
```

### Cards Routes

```php
<?php

Route::resource('cards', CardController::class);
Route::prefix('cards/{card}')->group(function () {
    Route::resource('board', Card\BoardController::class)->only(['show', 'update']);
    Route::resource('closure', Card\ClosureController::class)->only(['store', 'destroy']);
    Route::resource('column', Card\ColumnController::class)->only(['update']);
    Route::resource('goldness', Card\GoldnessController::class)->only(['store', 'destroy']);
    Route::resource('image', Card\ImageController::class)->only(['store', 'destroy']);
    Route::resource('not-now', Card\NotNowController::class)->only(['store', 'destroy']);
    Route::resource('pin', Card\PinController::class)->only(['store', 'destroy']);
    Route::resource('publish', Card\PublishController::class)->only(['store']);
    Route::resource('reading', Card\ReadingController::class)->only(['store']);
    Route::resource('triage', Card\TriageController::class)->only(['store']);
    Route::resource('watch', Card\WatchController::class)->only(['show', 'store', 'destroy']);
    
    Route::resource('assignments', Card\AssignmentController::class);
    Route::resource('steps', Card\StepController::class);
    Route::resource('taggings', Card\TaggingController::class);
    
    Route::resource('comments', Card\CommentController::class);
    Route::resource('comments/{comment}/reactions', Card\Comment\ReactionController::class);
});

// Card drops (drag-and-drop targets)
Route::prefix('columns/cards/{card}/drops')->group(function () {
    Route::post('not-now', [Card\Drop\NotNowController::class, 'store']);
    Route::post('stream', [Card\Drop\StreamController::class, 'store']);
    Route::post('closure', [Card\Drop\ClosureController::class, 'store']);
    Route::post('column', [Card\Drop\ColumnController::class, 'store']);
});

// Card previews
Route::resource('cards/previews', Card\PreviewController::class);
```

### Notifications Routes

```php
<?php

Route::resource('notifications', NotificationController::class);
Route::prefix('notifications')->group(function () {
    Route::get('tray', [Notification\TrayController::class, 'show']);
    Route::resource('settings', Notification\SettingsController::class)->only(['show', 'update']);
    Route::resource('unsubscribe', Notification\UnsubscribeController::class)->only(['show', 'store']);
    Route::post('bulk-reading', [Notification\BulkReadingController::class, 'store']);
});
Route::post('notifications/{notification}/reading', [Notification\ReadingController::class, 'store']);
```

### Search & Filters Routes

```php
<?php

Route::resource('search', SearchController::class)->only(['show', 'store']);
Route::resource('searches/queries', Search\QueryController::class);
Route::resource('filters', FilterController::class);
Route::post('filters/settings-refresh', [Filter\SettingsRefreshController::class, 'store']);
```

### Events/Timeline Routes

```php
<?php

Route::resource('events', EventController::class)->only(['index']);
Route::prefix('events')->group(function () {
    Route::resource('days', Event\DayController::class);
    Route::get('day-timeline/columns/{column}', [Event\DayTimeline\ColumnController::class, 'show']);
});
```

### Public Boards Routes

```php
<?php

Route::prefix('public')->name('public.')->group(function () {
    Route::resource('boards', Public\BoardController::class)->only(['show']);
    Route::prefix('boards/{board}')->group(function () {
        Route::get('columns/not-now', [Public\Board\Column\NotNowController::class, 'show']);
        Route::get('columns/stream', [Public\Board\Column\StreamController::class, 'show']);
        Route::get('columns/closed', [Public\Board\Column\ClosedController::class, 'show']);
        Route::get('columns/{column}', [Public\Board\ColumnController::class, 'show']);
        Route::resource('cards', Public\Board\CardController::class)->only(['show']);
    });
});
```

### Personal Dashboard Routes

```php
<?php

Route::prefix('my')->name('my.')->group(function () {
    Route::resource('pins', My\PinController::class);
    Route::resource('timezone', My\TimezoneController::class)->only(['show', 'update']);
    Route::get('menu', [My\MenuController::class, 'show']);
});
```

### PWA & Admin Routes

```php
<?php

// PWA
Route::get('manifest', [PwaController::class, 'manifest'])->name('pwa.manifest');
Route::get('service-worker', [PwaController::class, 'serviceWorker']);

// Health check
Route::get('up', fn() => response('OK'));

// Admin
Route::prefix('admin')->name('admin.')->middleware(['auth', 'admin'])->group(function () {
    Route::get('stats', [Admin\StatsController::class, 'show']);
    // Mount Horizon dashboard here
});
```

### Prompts Routes (Autocomplete)

```php
<?php

Route::prefix('prompts')->name('prompts.')->group(function () {
    Route::resource('cards', Prompt\CardController::class);
    Route::resource('tags', Prompt\TagController::class);
    Route::resource('users', Prompt\UserController::class);
    Route::resource('boards', Prompt\BoardController::class);
    Route::resource('boards/{board}/users', Prompt\Board\UserController::class);
});
```

---

## ⚙️ Background Jobs

### Jobs to Implement

| Job Class | Purpose | Schedule |
|-----------|---------|----------|
| `Board\CleanInaccessibleDataJob` | Clean up data when access is revoked | On access deletion |
| `Card\RemoveInaccessibleNotificationsJob` | Remove notifications for inaccessible cards | On board change |
| `Event\WebhookDispatchJob` | Dispatch webhooks for events | After event creation |
| `Mention\CreateJob` | Create mention records and notify | After content save |
| `Notification\Bundle\DeliverAllJob` | Deliver all pending notification bundles | Every 30 minutes |
| `Notification\Bundle\DeliverJob` | Deliver single notification bundle email | Queued |
| `Webhook\DeliveryJob` | Deliver webhook payload | Queued |
| `DeleteUnusedTagsJob` | Clean up orphan tags | Daily at 04:02 |
| `ExportAccountDataJob` | Generate account data export | On demand |
| `NotifyRecipientsJob` | Notify recipients of events | After event |
| `PushNotificationJob` | Send web push notification | On notification create |

### Recurring Tasks (config/recurring.yml equivalent)

```php
<?php
// app/Console/Kernel.php

protected function schedule(Schedule $schedule): void
{
    // Notifications
    $schedule->call(fn() => NotificationBundle::deliverAllLater())
        ->everyThirtyMinutes();
    
    // Auto-postpone stale cards
    $schedule->call(fn() => Card::autoPostponeAllDue())
        ->hourlyAt(50);
    
    // Cleanup
    $schedule->job(new DeleteUnusedTagsJob)
        ->dailyAt('04:02');
    
    $schedule->call(fn() => WebhookDelivery::cleanup())
        ->everyFourHours();
    
    $schedule->call(fn() => MagicLink::cleanup())
        ->everyFourHours();
    
    $schedule->call(fn() => AccountExport::cleanup())
        ->hourlyAt(20);
}
```

---

## 📧 Mailers

### Mail Classes to Implement

| Mailer Class | Purpose |
|--------------|---------|
| `MagicLinkMail` | Send magic link for passwordless authentication |
| `ExportCompletedMail` | Notify user when data export is ready |
| `UserInvitationMail` | Invite new users to account |
| `NotificationBundleMail` | Bundled notification digest email |

### Example Implementation

```php
<?php

namespace App\Mail;

use App\Models\MagicLink;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;

class MagicLinkMail extends Mailable
{
    use Queueable, SerializesModels;

    public function __construct(
        public MagicLink $magicLink
    ) {}

    public function build(): self
    {
        return $this
            ->subject('Sign in to Fizzy')
            ->markdown('emails.magic-link', [
                'code' => $this->magicLink->code,
                'expiresAt' => $this->magicLink->expires_at,
            ]);
    }
}
```

---

## 🔑 Key Features to Implement

### 1. Multi-tenancy

**Concept:** Account-scoped data isolation with URL-based routing.

**Implementation:**
- Middleware to extract account from URL path
- Global scope on all tenant models
- Context preservation in background jobs

```php
<?php

namespace App\Http\Middleware;

use App\Models\Account;
use Closure;
use Illuminate\Http\Request;

class SetCurrentAccount
{
    public function handle(Request $request, Closure $next)
    {
        $accountId = $request->route('account');
        $account = Account::where('external_account_id', $accountId)->firstOrFail();
        
        app()->instance('current.account', $account);
        
        return $next($request);
    }
}
```

---

### 2. Passwordless Authentication

**Concept:** Magic link email authentication with 6-digit codes.

**Flow:**
1. User enters email
2. System generates 6-digit code, sends email
3. User enters code or clicks link
4. Session created, user redirected

**Features:**
- 15-minute expiration
- Session transfer between devices (QR code)
- Multiple account support per identity

---

### 3. Real-time Updates

**Concept:** WebSocket broadcasting for live collaboration.

**Channels:**
- `boards.{boardId}` - Board updates
- `cards.{cardId}` - Card updates
- `users.{userId}.notifications` - Personal notifications

```php
<?php

namespace App\Events;

use App\Models\Card;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

class CardUpdated implements ShouldBroadcast
{
    use InteractsWithSockets;

    public function __construct(
        public Card $card
    ) {}

    public function broadcastOn(): Channel
    {
        return new Channel('boards.' . $this->card->board_id);
    }
}
```

---

### 4. Kanban Board

**Concept:** Drag-and-drop columns for workflow stages.

**Special Columns:**
- **Stream** - Uncategorized cards (triage)
- **Columns** - User-defined workflow stages
- **Not Now** - Postponed cards
- **Closed** - Completed cards

**Features:**
- Drag-drop between columns
- Column reordering
- Color customization
- Card positioning within columns

---

### 5. Activity Events

**Concept:** Comprehensive activity logging for audit trails.

**Event Types:**
- Card lifecycle (created, published, closed, reopened)
- Card movement (triaged, column changed, board changed)
- Assignments (assigned, unassigned)
- Comments (created)
- Auto-actions (auto-postponed)

**Features:**
- Timeline view by day
- Webhook delivery per event
- User attribution

---

### 6. Full-text Search

**Concept:** Full-text search across cards and comments.

**Options:**

**Option A: MySQL FULLTEXT (Rails approach)**
- 16 search_records_* tables (sharded by account ID hash)
- Indexes on title and content
- Recent search tracking

```php
<?php

// Determine shard from account ID
$shard = crc32($accountId) % 16;
$tableName = "search_records_{$shard}";
```

**Option B: Laravel Scout + Meilisearch (Recommended for Laravel)**
- Simpler implementation with built-in model trait
- Better search relevance and typo tolerance
- Easier scaling and maintenance

```php
<?php

use Laravel\Scout\Searchable;

class Card extends Model
{
    use Searchable;

    public function toSearchableArray(): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'description' => $this->description?->toPlainText(),
            'account_id' => $this->account_id,
        ];
    }
}
```

---

### 7. Notifications

**Concept:** Multi-channel notification system.

**Channels:**
- In-app (real-time via WebSocket)
- Email bundles (configurable frequency)
- Web push (browser notifications)

**Features:**
- Read/unread states
- Bulk mark as read
- Notification preferences
- Email digest bundling (30-minute windows)

---

### 8. Access Control

**Concept:** Board-level access permissions.

**Access Types:**
- **All Access** - Everyone in account can access
- **Invite Only** - Explicit access grants required

**Involvement Levels:**
- `access_only` - Can view and edit
- `watching` - Receives notifications

---

### 9. Entropy/Auto-postpone

**Concept:** Automatic card management to prevent stale backlogs.

**Configuration:**
- Account-level default period
- Board-level override
- Default: 30 days

**Features:**
- Cards auto-postpone after inactivity period
- Activity spikes reset timer
- Configurable per board

---

### 10. Rich Content

**Concept:** Rich text editing with mentions and attachments.

**Features:**
- TipTap/Trix rich text editor
- @mention autocomplete
- Image attachments (direct paste)
- File attachments
- Markdown support

---

## 📁 Suggested Laravel Project Structure

```
app/
├── Actions/                    # Single-purpose action classes
│   ├── Cards/
│   │   ├── CreateCard.php
│   │   ├── CloseCard.php
│   │   └── PostponeCard.php
│   ├── Boards/
│   └── Users/
│
├── Broadcasting/               # WebSocket channels
│   ├── BoardChannel.php
│   └── NotificationChannel.php
│
├── Console/
│   └── Commands/
│
├── Events/                     # Domain events
│   ├── CardCreated.php
│   ├── CardClosed.php
│   └── CommentCreated.php
│
├── Http/
│   ├── Controllers/
│   │   ├── Account/
│   │   ├── Board/
│   │   ├── Card/
│   │   ├── Notification/
│   │   ├── Public/
│   │   └── Session/
│   ├── Middleware/
│   │   ├── SetCurrentAccount.php
│   │   └── EnsureBoardAccess.php
│   └── Requests/
│       ├── Card/
│       └── Board/
│
├── Jobs/                       # Background jobs
│   ├── Board/
│   ├── Card/
│   ├── Event/
│   ├── Notification/
│   └── Webhook/
│
├── Listeners/                  # Event listeners
│   ├── NotifyAssignee.php
│   └── DispatchWebhooks.php
│
├── Mail/                       # Mailable classes
│   ├── MagicLinkMail.php
│   ├── ExportCompletedMail.php
│   └── NotificationBundleMail.php
│
├── Models/
│   ├── Traits/                 # Model traits (concerns)
│   │   ├── Assignable.php
│   │   ├── Broadcastable.php
│   │   ├── Closeable.php
│   │   ├── Eventable.php
│   │   ├── Mentionable.php
│   │   ├── Searchable.php
│   │   └── Watchable.php
│   ├── Account.php
│   ├── Board.php
│   ├── Card.php
│   ├── User.php
│   └── ...
│
├── Notifications/              # Laravel notifications
│   ├── CardAssigned.php
│   └── MentionNotification.php
│
├── Observers/                  # Model observers
│   ├── CardObserver.php
│   └── CommentObserver.php
│
├── Policies/                   # Authorization policies
│   ├── BoardPolicy.php
│   ├── CardPolicy.php
│   └── UserPolicy.php
│
├── Providers/
│   ├── AppServiceProvider.php
│   └── EventServiceProvider.php
│
└── Services/                   # Business logic services
    ├── Search/
    │   └── SearchService.php
    ├── Webhook/
    │   └── WebhookDeliveryService.php
    └── Export/
        └── AccountExportService.php

config/
├── broadcasting.php
├── queue.php
└── services.php

database/
├── migrations/
└── seeders/

resources/
├── js/
│   ├── echo.js                 # Laravel Echo setup
│   └── components/
├── views/
│   ├── boards/
│   ├── cards/
│   ├── components/
│   ├── emails/
│   └── layouts/
└── css/

routes/
├── web.php
├── channels.php
└── console.php

tests/
├── Feature/
│   ├── Cards/
│   ├── Boards/
│   └── Authentication/
└── Unit/
    ├── Models/
    └── Services/
```

---

## 🚦 Getting Started

### Phase 1: Foundation (Weeks 1-2)

- [ ] Set up Laravel 11 with multi-tenancy package
- [ ] Implement URL-based tenant routing middleware
- [ ] Create Account, Identity, User, Session models
- [ ] Build magic link authentication flow
- [ ] Set up basic layouts and views

### Phase 2: Core Features (Weeks 3-4)

- [ ] Create Board, Column, Card models with relationships
- [ ] Implement Kanban board UI with drag-drop
- [ ] Add Comment model with rich text
- [ ] Build tag and tagging system
- [ ] Implement card assignments

### Phase 3: Real-time & Notifications (Weeks 5-6)

- [ ] Set up Laravel Broadcasting with Soketi/Pusher
- [ ] Implement WebSocket channels for boards/cards
- [ ] Create notification system with preferences
- [ ] Build notification bundles for email digests
- [ ] Add web push notifications

### Phase 4: Advanced Features (Weeks 7-8)

- [ ] Implement full-text search with partitioned tables
- [ ] Build activity event system with timeline
- [ ] Create webhook integration with Slack/Campfire support
- [ ] Add entropy/auto-postpone functionality
- [ ] Implement board access control

### Phase 5: Polish (Weeks 9-10)

- [ ] Build PWA with service worker
- [ ] Implement account data export
- [ ] Create admin dashboard with stats
- [ ] Performance optimization (caching, eager loading)
- [ ] Testing and documentation

---

## 📦 Recommended Laravel Packages

| Package | Purpose | Install Command |
|---------|---------|-----------------|
| [spatie/laravel-multitenancy](https://github.com/spatie/laravel-multitenancy) | Multi-tenant architecture | `composer require spatie/laravel-multitenancy` |
| [spatie/laravel-medialibrary](https://github.com/spatie/laravel-medialibrary) | File attachments | `composer require spatie/laravel-medialibrary` |
| [spatie/laravel-permission](https://github.com/spatie/laravel-permission) | Roles & permissions | `composer require spatie/laravel-permission` |
| [spatie/laravel-activitylog](https://github.com/spatie/laravel-activitylog) | Activity logging | `composer require spatie/laravel-activitylog` |
| [laravel/scout](https://github.com/laravel/scout) + Meilisearch | Full-text search | `composer require laravel/scout meilisearch/meilisearch-php` |
| [beyondcode/laravel-websockets](https://github.com/beyondcode/laravel-websockets) or Soketi | WebSocket server | `composer require beyondcode/laravel-websockets` |
| [laravel/horizon](https://github.com/laravel/horizon) | Job queue dashboard | `composer require laravel/horizon` |
| [graham-campbell/markdown](https://github.com/GrahamCampbell/Laravel-Markdown) | Markdown rendering | `composer require graham-campbell/markdown` |
| [simplesoftwareio/simple-qrcode](https://github.com/SimpleSoftwareIO/simple-qrcode) | QR code generation | `composer require simplesoftwareio/simple-qrcode` |
| [laravel-notification-channels/webpush](https://github.com/laravel-notification-channels/webpush) | Web push notifications | `composer require laravel-notification-channels/webpush` |
| [livewire/livewire](https://github.com/livewire/livewire) | Reactive UI components | `composer require livewire/livewire` |
| [ueberdosis/tiptap](https://github.com/ueberdosis/tiptap) | Rich text editor (JS) | `npm install @tiptap/core @tiptap/starter-kit` |

---

## 🎓 Learning Outcomes

By rebuilding Fizzy in Laravel, you will learn:

### Architecture & Design Patterns
- ✅ Multi-tenant SaaS architecture patterns
- ✅ Domain-driven design with Actions/Services
- ✅ Event-driven architecture
- ✅ Repository and Observer patterns
- ✅ Trait-based model composition

### Backend Development
- ✅ Advanced Eloquent relationships and scopes
- ✅ Background job processing with Horizon
- ✅ Real-time WebSocket broadcasting
- ✅ Email queueing and bundling
- ✅ Webhook implementation with signing

### Frontend Development
- ✅ Livewire/Inertia.js reactive components
- ✅ Drag-and-drop interfaces
- ✅ Rich text editing integration
- ✅ PWA implementation
- ✅ Real-time UI updates with Echo

### Security & Performance
- ✅ Passwordless authentication
- ✅ Row-level multi-tenant security
- ✅ Rate limiting and API protection
- ✅ Database query optimization
- ✅ Caching strategies

### DevOps & Infrastructure
- ✅ Queue worker management
- ✅ Scheduled task configuration
- ✅ WebSocket server deployment
- ✅ Cloud storage integration
- ✅ Database partitioning strategies

---

## 📚 Additional Resources

- [Laravel Documentation](https://laravel.com/docs)
- [Spatie Packages](https://spatie.be/open-source)
- [Laravel Daily](https://laraveldaily.com/)
- [Laravel News](https://laravel-news.com/)

---

*This guide is based on the Fizzy Rails application architecture and is designed to help developers learn enterprise SaaS development patterns while building a feature-rich Laravel application.*
