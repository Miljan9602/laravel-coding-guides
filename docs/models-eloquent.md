# Models & Eloquent

Eloquent models are the schema-facing edge of the application: they declare table shape, relationships, casts, and mass-assignment rules — and nothing else. Business logic, orchestration, and cross-aggregate decisions live in Actions and Services; persistence access funnels through repositories.

## Rules at a glance

- Every model is a `final class` with `declare(strict_types=1);`.
- Models hold **schema, relationships, casts, and scopes** — never business logic.
- Type every relationship method explicitly (`: HasMany`, `: BelongsTo`, `: BelongsToMany`, `: MorphMany`).
- Declare casts in a `protected function casts(): array` method, not a `$casts` property.
- Use enum casts, `'array'`/`AsArrayObject` for JSON/jsonb, `'immutable_datetime'` for dates, and `'encrypted'` for secrets.
- Use `$fillable` (explicit allowlist). **Never** `$guarded = []` — it opens mass assignment to every column.
- Define accessors/mutators with the `Attribute` API, not legacy `getXAttribute`/`setXAttribute`.
- Put reusable query constraints in **query scopes**, not in repository ad-hoc `where()` chains.
- Relationship definitions live **only** on the model; callers reach relations through repositories.
- Prefer Eloquent + Collections over raw SQL and the query builder; drop down only for measured hot paths.
- For multi-tenancy, use an **explicit `forWorkspace()` scope + relationship traversal**, not a global scope.
- Encrypt sensitive columns with the `'encrypted'` cast, and add a separate **hash column** for lookups.

## Models are schema, not behaviour

### Rule

A model declares what a row *is* — its columns' types, its relations, and its mass-assignment surface. It does not decide *what should happen*. Sending a Slack message, scoring a report, calling the GitHub API, or enforcing a business invariant belongs in an Action or Service.

### Why

Fat models become god objects. When `Report::publish()` also notifies Slack, recalculates contributor stats, and writes an audit log, the model can no longer be reasoned about, mocked, or reused. Console commands, queued jobs, and HTTP requests all instantiate models, so any side effect baked into a model fires in contexts that never expected it. Keeping models thin makes them trivially testable (construct, assert casts/relations) and keeps behaviour in classes you can inject and fake.

### How

✅ Do — model declares shape; behaviour lives in an Action:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use App\Enums\ReportStatus;
use Illuminate\Database\Eloquent\Casts\AsArrayObject;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

final class Report extends Model
{
    protected $fillable = [
        'workspace_id',
        'title',
        'status',
        'period_start',
        'period_end',
        'metrics',
    ];

    public function workspace(): BelongsTo
    {
        return $this->belongsTo(Workspace::class);
    }

    public function contributors(): HasMany
    {
        return $this->hasMany(Contributor::class);
    }

    protected function casts(): array
    {
        return [
            'status' => ReportStatus::class,
            'period_start' => 'immutable_datetime',
            'period_end' => 'immutable_datetime',
            'metrics' => AsArrayObject::class,
        ];
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Actions\Reports;

use App\Enums\ReportStatus;
use App\Models\Report;
use App\Services\Slack\SlackNotifier;

final readonly class PublishReport
{
    public function __construct(
        private SlackNotifier $slack,
    ) {}

    public function handle(Report $report): Report
    {
        $report->status = ReportStatus::Published;
        $report->save();

        $this->slack->announcePublished($report);

        return $report;
    }
}
```

❌ Avoid — side effects and invariants smuggled into the model:

```php
final class Report extends Model
{
    // ❌ Notifies Slack on every save context, including jobs and seeders.
    // ❌ Hardcodes a business rule the rest of the app can't see or test.
    public function publish(): void
    {
        if ($this->contributors()->count() === 0) {
            throw new \RuntimeException('Cannot publish empty report');
        }

        $this->update(['status' => 'published']);

        app(SlackNotifier::class)->announcePublished($this);
    }
}
```

## Final classes and strict types

### Rule

Models are `final` and every file opens with `declare(strict_types=1);`.

### Why

`final` blocks the inheritance-based extension that tempts teams into deep model hierarchies (`PublishedReport extends Report`) — share behaviour through traits, scopes, and composition instead. `strict_types` turns silent string→int coercions in IDs, casts, and scope arguments into immediate errors rather than corrupt rows.

### How

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

final class Workspace extends Model
{
    // ...
}
```

## Typed relationships

### Rule

Every relationship method declares its concrete return type (`HasMany`, `BelongsTo`, `BelongsToMany`, `HasOne`, `MorphMany`, etc.).

### Why

The return type documents cardinality at a glance and lets static analysis and the IDE know whether `->get()`, `->first()`, or `->save()` is valid on the relation. Untyped relationship methods erase that signal and let mistakes (treating a `BelongsTo` like a collection) survive to runtime.

### How

✅ Do:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\MorphMany;

final class Workspace extends Model
{
    protected $fillable = ['name', 'slug', 'github_org'];

    public function owner(): BelongsTo
    {
        return $this->belongsTo(User::class, 'owner_id');
    }

    public function reports(): HasMany
    {
        return $this->hasMany(Report::class);
    }

    public function members(): BelongsToMany
    {
        return $this->belongsToMany(User::class)
            ->withPivot('role')
            ->withTimestamps();
    }

    public function auditEvents(): MorphMany
    {
        return $this->morphMany(AuditEvent::class, 'subject');
    }
}
```

❌ Avoid — untyped relations:

```php
// ❌ No return type: callers can't tell a single record from a collection.
public function reports()
{
    return $this->hasMany(Report::class);
}
```

## The `casts()` method

### Rule

Declare casts in `protected function casts(): array`, the method form introduced for modern Laravel, rather than the legacy `protected $casts` property.

### Why

The method form is typed (`: array`), can reference class-string casts and enums without quoting gymnastics, and can compose values at runtime when needed. It keeps all type coercion in one obvious place so reading a model tells you exactly how each column materialises in PHP.

### How

```php
<?php

declare(strict_types=1);

namespace App\Models;

use App\Enums\DeliveryChannel;
use App\Enums\ReportStatus;
use Illuminate\Database\Eloquent\Casts\AsArrayObject;
use Illuminate\Database\Eloquent\Model;

final class Report extends Model
{
    protected function casts(): array
    {
        return [
            // Enum cast — column stores the backing value, PHP gets the enum.
            'status' => ReportStatus::class,
            'default_channel' => DeliveryChannel::class,

            // JSON / jsonb column → mutable object you can index and reassign.
            'metrics' => AsArrayObject::class,

            // Plain array cast when you only ever read/replace wholesale.
            'tags' => 'array',

            // Immutable dates: arithmetic returns new instances, never mutates.
            'period_start' => 'immutable_datetime',
            'period_end' => 'immutable_datetime',

            // Encrypted at rest; transparently decrypted on access.
            'github_token' => 'encrypted',

            'is_archived' => 'boolean',
            'contributor_count' => 'integer',
        ];
    }
}
```

### Cast cheatsheet

- **Enums** — cast to the enum class-string (`ReportStatus::class`). The DB stores the backing scalar; PHP always sees a typed enum. See [Enums](enums.md).
- **JSON / jsonb** — use `'array'` for read/replace-whole-value access, or `AsArrayObject::class` when you mutate nested keys. On Postgres, `jsonb` columns work with the same casts and gain index/operator support.
- **Dates** — prefer `'immutable_datetime'` (backed by `CarbonImmutable`) so a stray `->addDay()` can't silently mutate a shared instance. Reserve mutable `'datetime'` for cases that genuinely need in-place mutation.
- **Encrypted** — `'encrypted'` (or `'encrypted:array'`, `'encrypted:object'`) encrypts on write and decrypts on read using the app key. The stored ciphertext is **not** queryable — see the hashed-lookup section below.

## Mass assignment: `$fillable`, never `$guarded = []`

### Rule

List the columns safe for mass assignment in `$fillable`. Never set `protected $guarded = []`.

### Why

`$guarded = []` disables mass-assignment protection entirely: any key present in `$request->all()` (or any array handed to `create()`/`update()`) can write any column — `is_admin`, `workspace_id`, `owner_id`, `status`. That is exactly the privilege-escalation and tenant-crossing class of bug mass assignment exists to prevent. An explicit `$fillable` allowlist is self-documenting and fails closed: a new column is *not* writable until you deliberately add it.

### How

✅ Do:

```php
final class Report extends Model
{
    protected $fillable = [
        'workspace_id',
        'title',
        'status',
        'period_start',
        'period_end',
        'metrics',
    ];
}
```

❌ Avoid:

```php
final class Report extends Model
{
    // ❌ Every column — including workspace_id — is now mass-assignable.
    protected $guarded = [];
}
```

When a value must never come from user input (a tenant key, an owner), set it explicitly after construction rather than relaxing the allowlist:

```php
$report = new Report($request->validated());
$report->workspace_id = $workspace->id; // authoritative, server-set
$report->save();
```

## Accessors & mutators via the `Attribute` API

### Rule

Define derived and transformed attributes with the `Attribute` API. Do not use the legacy `getFooAttribute()` / `setFooAttribute()` methods.

### Why

`Attribute` pairs get and set logic in a single, discoverable method with real type hints, supports caching for object-valued attributes, and reads cleanly next to relationships and casts. The legacy magic-method form scatters logic across `get`/`set` pairs and offers no typing.

### How

✅ Do:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

final class Contributor extends Model
{
    protected $fillable = ['report_id', 'login', 'display_name'];

    /**
     * Normalise the GitHub login on write; never null on read.
     */
    protected function login(): Attribute
    {
        return Attribute::make(
            get: fn (string $value): string => strtolower($value),
            set: fn (string $value): string => strtolower(trim($value)),
        );
    }

    /**
     * Read-only derived attribute.
     */
    protected function profileUrl(): Attribute
    {
        return Attribute::make(
            get: fn (mixed $value, array $attributes): string =>
                'https://github.com/'.Str::lower($attributes['login']),
        )->shouldCache();
    }
}
```

❌ Avoid — legacy accessor/mutator pair:

```php
// ❌ Untyped, split across two methods, no caching.
public function getProfileUrlAttribute(): string
{
    return 'https://github.com/'.$this->login;
}

public function setLoginAttribute($value): void
{
    $this->attributes['login'] = strtolower(trim($value));
}
```

Keep accessors to pure presentation/normalisation. Anything that touches another aggregate, performs IO, or encodes a business rule belongs in an Action — not an accessor.

## Query scopes for reusable constraints

### Rule

Express recurring query constraints as scopes on the model. Repositories then compose scopes instead of hand-rolling the same `where()` clauses.

### Why

A scope names a constraint once (`->published()`, `->forWorkspace($w)`, `->withinPeriod(...)`) and gives every caller the same definition. Scattering `where('status', 'published')` across repositories means a change to the meaning of "published" must be hunted down in many places — and one missed spot is a bug. Scopes also chain, so repositories build precise queries from small, tested pieces.

### How

✅ Do — typed scopes with the modern `Scope` attribute (or `scopeXxx` convention):

```php
<?php

declare(strict_types=1);

namespace App\Models;

use App\Enums\ReportStatus;
use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Carbon;

final class Report extends Model
{
    #[Scope]
    protected function published(Builder $query): void
    {
        $query->where('status', ReportStatus::Published);
    }

    #[Scope]
    protected function withinPeriod(Builder $query, Carbon $from, Carbon $to): void
    {
        $query->where('period_start', '>=', $from)
            ->where('period_end', '<=', $to);
    }
}
```

The repository composes them — see [Repositories](repositories.md):

```php
$reports = Report::query()
    ->forWorkspace($workspace)
    ->published()
    ->withinPeriod($from, $to)
    ->latest('period_end')
    ->get();
```

❌ Avoid — duplicated raw constraints in every call site:

```php
// ❌ The definition of "published within a period" is copy-pasted and drifts.
Report::where('workspace_id', $workspace->id)
    ->where('status', 'published')
    ->where('period_start', '>=', $from)
    ->where('period_end', '<=', $to)
    ->get();
```

## Relations live on the model; callers go through repositories

### Rule

Relationship *definitions* are the single source of truth and belong only on the model. Application code does not query relations directly from controllers/actions — it calls repository methods that internally traverse those relations.

### Why

Centralising relations on the model keeps cardinality and foreign keys in one place. Routing reads through repositories means eager-loading strategy, scoping, and pagination are decided once and consistently, and the rest of the app depends on an interface it can fake in tests rather than on Eloquent's global query surface.

### How

```php
// Model: defines the relation.
public function contributors(): HasMany
{
    return $this->hasMany(Contributor::class);
}
```

```php
// Repository: the only place that reaches through the relation.
final readonly class ReportRepository
{
    public function topContributors(Report $report, int $limit): Collection
    {
        return $report->contributors()
            ->orderByDesc('commits')
            ->limit($limit)
            ->get();
    }
}
```

## Prefer Eloquent and Collections over raw SQL

### Rule

Reach for Eloquent relations, scopes, and Collection methods first. Use `DB::raw`/raw SQL only for a measured, documented hot path that Eloquent genuinely cannot express efficiently.

### Why

Eloquent gives casts, scopes, tenant scoping, and the same model objects everywhere; Collections give expressive, typed, in-memory transforms. Raw SQL bypasses casts and scopes (so a tenant filter or enum cast silently doesn't apply), is database-specific, and is invisible to model events. When data is already loaded, transforming it with a Collection is clearer and avoids extra round trips than re-querying.

### How

✅ Do — Eloquent + Collection pipeline:

```php
$summary = $report->contributors
    ->groupBy(fn (Contributor $c): string => $c->team)
    ->map(fn (Collection $group): int => $group->sum('commits'))
    ->sortDesc();
```

❌ Avoid — raw SQL that skips casts and tenant scope:

```php
// ❌ Bypasses ReportStatus cast, workspace scoping, and model events.
$rows = DB::select(
    'select * from reports where status = ? and workspace_id = ?',
    ['published', $workspaceId],
);
```

## Multi-tenant scoping without a global scope

### Rule

Scope tenant data with an **explicit** `forWorkspace()` query scope plus relationship traversal. Do **not** register a global scope that injects the current workspace automatically.

### Why

A global scope reads tenant context from somewhere ambient — typically the authenticated request. Queued jobs, scheduled commands, the digest CLI, seeders, and tests have **no request context**. In those contexts a request-derived global scope either:

- resolves to *no* workspace and silently returns/affects **every** tenant's rows (a cross-tenant leak, or — for an `update`/`delete` — a cross-tenant erasure), or
- throws deep inside unrelated queries because there is no current workspace.

Both failure modes are silent and catastrophic. An explicit scope makes the tenant boundary visible at every call site: if a query forgets `forWorkspace()`, that omission is reviewable in the diff rather than hidden behind a global default. Traversing from the workspace's own relations (`$workspace->reports()`) is inherently scoped — the foreign key does the work.

### How

✅ Do — a trait that adds an explicit scope and a hard requirement to pass a workspace:

```php
<?php

declare(strict_types=1);

namespace App\Models\Concerns;

use App\Models\Workspace;
use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;

trait BelongsToWorkspace
{
    #[Scope]
    protected function forWorkspace(Builder $query, Workspace $workspace): void
    {
        $query->where(
            $this->qualifyColumn('workspace_id'),
            $workspace->id,
        );
    }
}
```

```php
final class Report extends Model
{
    use BelongsToWorkspace;

    protected $fillable = ['workspace_id', 'title', 'status'];

    public function workspace(): BelongsTo
    {
        return $this->belongsTo(Workspace::class);
    }
}
```

Every read and write states the tenant explicitly:

```php
// Explicit scope — visible, reviewable, works in jobs and CLI.
$reports = Report::query()->forWorkspace($workspace)->published()->get();

// Or traverse from the tenant aggregate, which is inherently scoped.
$reports = $workspace->reports()->published()->get();

// In a queued job there is no request — the workspace is passed in.
final readonly class GenerateDigest
{
    public function handle(Workspace $workspace): void
    {
        $reports = $workspace->reports()
            ->forWorkspace($workspace)
            ->withinPeriod(now()->subWeek(), now())
            ->get();
        // ...
    }
}
```

❌ Avoid — a request-coupled global scope:

```php
// ❌ In a job/CLI run, auth()->user() is null → either every workspace's
//    reports are returned, or a delete wipes rows across all tenants.
protected static function booted(): void
{
    static::addGlobalScope('workspace', function (Builder $query): void {
        $query->where('workspace_id', auth()->user()?->current_workspace_id);
    });
}
```

If you must enforce tenancy at the model layer, fail loud instead of defaulting: throw when no workspace is supplied rather than letting a null silently widen the query.

## Encrypted columns with a hash for lookups

### Rule

Encrypt sensitive columns with the `'encrypted'` cast. Because ciphertext is non-deterministic and unsearchable, store a **separate deterministic hash column** of the same value to support equality lookups.

### Why

Laravel's encryption is authenticated and randomised: encrypting the same plaintext twice yields different ciphertext, so you cannot `where('github_token', $value)` against an encrypted column. A dedicated hash column (a deterministic HMAC keyed with the app key) gives you a stable, indexable value to match on, while the real secret stays encrypted at rest. Never reverse this by storing the secret in plaintext "just for lookups."

### How

Migration — one encrypted column, one indexed hash column:

```php
Schema::table('workspaces', function (Blueprint $table): void {
    $table->text('github_token');      // holds ciphertext
    $table->string('github_token_hash')->index();
});
```

Model — cast the secret, derive the hash in a mutator:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Hash;

final class Workspace extends Model
{
    protected $fillable = ['name', 'slug'];

    protected function casts(): array
    {
        return [
            'github_token' => 'encrypted',
        ];
    }

    /**
     * Setting the token also refreshes its deterministic lookup hash.
     */
    protected function githubToken(): Attribute
    {
        return Attribute::make(
            set: fn (string $value): array => [
                'github_token' => $value,
                'github_token_hash' => hash_hmac(
                    'sha256',
                    $value,
                    (string) config('app.key'),
                ),
            ],
        );
    }
}
```

> Note: the `set` callback returns an **array** of attributes, so assigning `github_token` writes both the encrypted column and its hash in one operation. The `'encrypted'` cast still applies to the `github_token` key on the way to the database.

Lookup by the hash, never by the ciphertext:

```php
$hash = hash_hmac('sha256', $candidateToken, (string) config('app.key'));

$workspace = Workspace::query()
    ->where('github_token_hash', $hash)
    ->first();
```

For a value object that wraps and validates such a secret before it reaches the model, see [DTOs & Value Objects](dtos-value-objects.md).

## See also

- [Repositories](repositories.md)
- [DTOs & Value Objects](dtos-value-objects.md)
- [Enums](enums.md)
