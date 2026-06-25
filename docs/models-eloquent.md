# Models & Eloquent

Eloquent models are the schema-facing edge of the application: they declare table shape, relationships, casts, and mass-assignment rules — and nothing else. Business logic, orchestration, and cross-aggregate decisions live in Actions and Services; persistence access funnels through repositories.

## Rules at a glance

- Every model is a `final class` with `declare(strict_types=1);`.
- Enable `Model::shouldBeStrict(! app()->isProduction())` in `AppServiceProvider::boot()` — the ORM analogue of `declare(strict_types=1)`; throw in dev/CI, degrade in prod.
- Models hold **schema, relationships, casts, and scopes** — never business logic.
- Type every relationship method explicitly (`: HasMany`, `: BelongsTo`, `: BelongsToMany`, `: MorphMany`).
- Add a generic docblock to every relation (`/** @return HasMany<Contributor, $this> */`) so `phpstan level max` types related records.
- Declare casts in a `protected function casts(): array` method, not a `$casts` property.
- Bridge a column to a value object with a `CastsAttributes` cast (or an inline `Attribute`) — never reconstruct it by hand at call sites.
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

## Eloquent strict mode: fail loudly at the ORM layer

### Rule

Enable Eloquent strict mode in `AppServiceProvider::boot()` so the ORM throws — instead of silently degrading — on the three classic Eloquent footguns. Gate the throwing behaviour to non-production: assert-in-dev, degrade-in-prod.

### Why

We open every file with `declare(strict_types=1);` and we preach "fail loudly, no silent fallback" everywhere — yet by default Eloquent does the exact opposite of that philosophy at runtime. Three operations that are almost always bugs succeed silently:

- **Lazy-loading an unloaded relation.** `$report->contributors` without a prior `with('contributors')` quietly fires one query *per report* — the N+1 that only surfaces as a latency regression in production, never in a passing test.
- **Mass-assigning a non-`$fillable` key.** `Report::create(['is_admin' => true])` where `is_admin` isn't fillable doesn't error — it *drops the key* and persists the row without it. The write you thought you made never happened.
- **Reading an unloaded / unselected column.** `$report->title` after `select('id')`, or any attribute that isn't on the row, returns `null` instead of signalling that you never loaded it — a `null` that then propagates into summaries, Slack blocks, and emails.

Strict mode is the ORM-runtime analogue of `declare(strict_types=1)`: it converts these three silent miscompiles into immediate exceptions, where a test or a local request catches them before they ship. We enforce strict typing in the language; this enforces it at the persistence boundary, the one layer where we previously left the safety off.

Gate it to non-production. A missed `with()` that reaches production should **degrade** (an extra query, a slower request) rather than throw a 500 at a user. In development, CI, and staging it must throw, so the missing eager-load is a failing test — not a silent N+1 you discover from a latency graph. This pairs directly with the eager-loading discipline in [Repositories](repositories.md): repositories decide the load strategy, and strict mode is the tripwire that proves they did.

### How

✅ Do — one call in the app service provider, gated to non-production:

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\ServiceProvider;

final class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Throw on lazy-loading, silent attribute discards, and missing
        // attribute access everywhere except production. In production these
        // degrade instead of 500-ing a user.
        Model::shouldBeStrict(! $this->app->isProduction());
    }
}
```

`shouldBeStrict()` is shorthand for the three independent toggles. Reach for them individually when you want a different policy per check — for example, keep the two write/read guards on in production (they catch genuine data bugs) but never throw for lazy loading there:

```php
public function boot(): void
{
    $strict = ! $this->app->isProduction();

    Model::preventLazyLoading($strict);
    Model::preventSilentlyDiscardingAttributes($strict);
    Model::preventAccessingMissingAttributes($strict);
}
```

To degrade rather than crash in production, log the violation instead of throwing — turn the silent bug into an observable signal without taking the request down:

```php
public function boot(): void
{
    Model::shouldBeStrict(! $this->app->isProduction());

    // In production: don't throw on a missed eager-load — record it so the
    // N+1 is visible in logs and can be fixed, while the request still serves.
    Model::handleLazyLoadingViolationUsing(function (Model $model, string $relation): void {
        if ($this->app->isProduction()) {
            logger()->warning('Lazy load in production', [
                'model' => $model::class,
                'relation' => $relation,
            ]);

            return;
        }

        throw new \Illuminate\Database\LazyLoadingViolationException($model, $relation);
    });
}
```

❌ Avoid — leaving strict mode off and absorbing the bugs:

```php
public function boot(): void
{
    // ❌ Default Eloquent: the N+1, the dropped mass-assignment, and the
    //    null-from-an-unloaded-column all pass silently. The philosophy we
    //    enforce in the type system is switched off at the ORM.
}
```

### Edge cases

- **Tests must run with strict mode on.** That is the point — a feature test exercising a Livewire screen or a `GeneratePreviewReport` job will fail the instant a relation is lazy-loaded, forcing the `with()` into the repository where it belongs. Do not disable strict mode for the test environment.
- **`preventAccessingMissingAttributes` and `select()`.** If a query deliberately selects a column subset, accessing an unselected attribute now throws. That is correct: either select the column or don't read it. It does not affect appended accessors or `$attributes` defaults you set explicitly.
- **Factories and seeders.** These run in non-production and so run strict. Ensure factories set every non-nullable, non-defaulted attribute; a missing one surfaces immediately rather than as a confusing `null` downstream.
- **Polymorphic and pivot access.** `preventLazyLoading` covers pivot and morph relations too — eager-load `members` *with* its pivot, and morph relations, the same as any other.

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

## Type relationships with generics

### Rule

Add a generic docblock to every relationship method declaring the related model (and `$this`) — `/** @return HasMany<Contributor, $this> */`. The native return type states *cardinality*; the generic states *what's inside*.

### Why

A bare `: HasMany` tells PHPStan a collection-shaped relation, but not of *what*. Without the generic, `$report->contributors->first()` is typed `Model` — so `$contributor->login` is unknown, every downstream property access is unchecked, and the "typed everything" claim we make for the rest of the codebase quietly stops at the relationship boundary. The generic argument is what propagates the real `Contributor` type through `->get()`, `->first()`, and eager-loaded access, so static analysis verifies the property and method calls you make on related records.

At `phpstan level max` the [larastan](tooling-and-ci.md) extension turns this on with `checkGenericClassInNonGenericObjectType`: an un-parameterised `HasMany` becomes a reported error, not a silent gap. Typing the generic is what makes our typed-everything guarantee real for relations — the same way the native return type made it real for cardinality.

### How

✅ Do — native return type *and* a generic docblock on every relation:

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

    /** @return BelongsTo<User, $this> */
    public function owner(): BelongsTo
    {
        return $this->belongsTo(User::class, 'owner_id');
    }

    /** @return HasMany<Report, $this> */
    public function reports(): HasMany
    {
        return $this->hasMany(Report::class);
    }

    /** @return BelongsToMany<User, $this> */
    public function members(): BelongsToMany
    {
        return $this->belongsToMany(User::class)
            ->withPivot('role')
            ->withTimestamps();
    }

    /** @return MorphMany<AuditEvent, $this> */
    public function auditEvents(): MorphMany
    {
        return $this->morphMany(AuditEvent::class, 'subject');
    }
}
```

With the generic in place, the chain is fully typed end to end:

```php
// $report->contributors is Collection<int, Contributor>, so $c is a Contributor.
$logins = $report->contributors
    ->map(fn (Contributor $c): string => $c->login);
```

❌ Avoid — a typed return with no generic argument:

```php
// ❌ Cardinality is typed, contents are not. PHPStan max flags the missing
//    generic; $report->contributors->first() is Model, so ->login is unchecked.
public function contributors(): HasMany
{
    return $this->hasMany(Contributor::class);
}
```

### Edge cases

- **Two type parameters.** Most relations take `<TRelatedModel, $this>`. `$this` (not the class name) is the convention so the generic stays correct on the model and any class that uses it.
- **`HasManyThrough` / `MorphTo`.** Through-relations take an extra intermediate parameter (`<TRelated, TThrough, $this>`); `MorphTo` is parameterless because the target is dynamic — leave it as `: MorphTo` with no generic.
- **`hasOne` returning a related model.** `/** @return HasOne<ContributorSummary, $this> */` — the same shape; cardinality differs, the generic discipline does not.
- **Don't restate the native type in the docblock when there's nothing to add** — the generic is the reason the docblock exists; a `@return HasMany` with no parameters earns nothing over the signature.

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

## Custom casts for value objects

### Rule

When a column should materialise as one of our [value objects](dtos-value-objects.md) rather than a scalar, write a `CastsAttributes` implementation and reference it from `casts()`. A custom cast is the canonical bridge between a value object and a database column.

### Why

We keep meaning in value objects — a `WorkspaceReportContext`, a parsed `DeliveryChannel` target, a validated period — instead of passing raw scalars around. But Eloquent only knows scalars, JSON, dates, and enums out of the box. A custom cast is the one seam that makes the model speak value objects directly: every read of the attribute returns the rich object (already validated, with its behaviour), every write accepts that object and serialises it back to the column. The conversion lives in *one* class, so there is no place left for an ad-hoc `new ValueObject($model->raw_column)` to drift. It is the model-layer counterpart to the enum cast: enums bridge a scalar to a typed enum; a custom cast bridges a column to any value object.

This is the same mechanism that powers the encrypted `DeliveryChannel` target — there the value-object boundary is expressed with an `Attribute` (a `get`/`set` pair that decrypts to and encrypts from the value object), which is the inline form of the same idea. Reach for a full `CastsAttributes` class when the conversion is reused across models or warrants its own unit test; reach for an `Attribute` when it's a single column on a single model.

### How

✅ Do — a `CastsAttributes` class converting a column to and from a value object:

```php
<?php

declare(strict_types=1);

namespace App\Casts;

use App\ValueObjects\Period;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\CarbonImmutable;

/**
 * Casts two stored timestamp columns to and from a single Period value object.
 *
 * @implements CastsAttributes<Period, Period>
 */
final class PeriodCast implements CastsAttributes
{
    /**
     * @param  array<string, mixed>  $attributes
     */
    public function get(Model $model, string $key, mixed $value, array $attributes): ?Period
    {
        if ($attributes['period_start'] === null || $attributes['period_end'] === null) {
            return null;
        }

        return new Period(
            CarbonImmutable::parse($attributes['period_start']),
            CarbonImmutable::parse($attributes['period_end']),
        );
    }

    /**
     * @param  array<string, mixed>  $attributes
     * @return array<string, mixed>
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): array
    {
        if (! $value instanceof Period) {
            throw new \InvalidArgumentException('The period attribute must be a '.Period::class.'.');
        }

        return [
            'period_start' => $value->start,
            'period_end' => $value->end,
        ];
    }
}
```

Reference it from `casts()` exactly like a built-in cast:

```php
protected function casts(): array
{
    return [
        'period' => PeriodCast::class,
    ];
}
```

```php
// Reads and writes speak the value object; the two columns are an implementation detail.
$report->period = new Period(now()->subWeek(), now());
$report->save();

$report->period->start; // CarbonImmutable, validated by Period's constructor
```

The encrypted `DeliveryChannel` target is the same boundary done inline with an `Attribute` — decrypt-to-object on read, encrypt-from-object on write:

```php
/**
 * Stored encrypted; surfaced as a validated DeliveryTarget value object.
 */
protected function target(): Attribute
{
    return Attribute::make(
        get: fn (string $value): DeliveryTarget => DeliveryTarget::fromString(Crypt::decryptString($value)),
        set: fn (DeliveryTarget $value): string => Crypt::encryptString($value->value()),
    );
}
```

❌ Avoid — reconstructing the value object by hand at every call site:

```php
// ❌ The column→object mapping lives nowhere; every caller re-parses the raw
//    value, and a change to Period's shape must be hunted across the codebase.
$period = new Period(
    CarbonImmutable::parse($report->period_start),
    CarbonImmutable::parse($report->period_end),
);
```

### Edge cases

- **Null columns.** Decide explicitly whether `get` returns `null` or a null-object; a custom cast is the right place to make that choice once. The example returns `null` only when both backing columns are null.
- **Multi-column casts.** `set` may return an array of attributes (above, two timestamp columns from one `Period`) — the same multi-attribute return shape used by the encrypted-token mutator below.
- **Validate on the way in.** A `set` that rejects a non-value-object argument keeps invalid state out of the row; the value object's own constructor enforces the rest. Don't duplicate that validation in the cast.
- **Querying.** A custom-cast column is still stored as its underlying scalar(s), so query against those columns (or their [scopes](#query-scopes-for-reusable-constraints)) — not the value object.

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
- [Migrations & schema conventions](migrations.md)
- [DTOs & Value Objects](dtos-value-objects.md)
- [Enums](enums.md)
- [Philosophy](philosophy.md)
- [Tooling & CI](tooling-and-ci.md)
