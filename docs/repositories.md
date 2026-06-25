# Repositories — the only place models are touched

A repository is the single seam through which the application reads and writes persisted data. Application code (actions, services, commands, jobs, Livewire components, controllers) never speaks Eloquent directly — it asks a repository, by intent, for exactly the data it needs.

## Rules at a glance

- Application code MUST NOT touch Eloquent directly: no `Model::query()`, `::where()`, `::find()`, `::create()`, `::updateOrCreate()`, `::firstOrCreate()`, and no relationship-query traversal like `$workspace->reports()->where(...)`.
- Every persisted model has an interface in `app/Contracts/Repositories/<Model>RepositoryInterface` and an Eloquent implementation in `app/Repositories/<Model>Repository`.
- Bind interface → implementation in a dedicated `RepositoryServiceProvider`. Consumers type-hint the interface only.
- Repository methods are typed and intent-revealing. They return models, collections, or scalars — NEVER an Eloquent query builder. Don't leak Eloquent past the boundary.
- Eloquent is allowed ONLY inside: repository implementations, model relationship/cast/attribute definitions, Filament resource/table/infolist declarations, factories/seeders, and tests.
- Pure static helpers on a model (e.g. `DeliveryChannel::hashTarget()`) are fine — they aren't data access.
- Add only the methods the app actually calls. Don't pre-build a generic CRUD surface "just in case."
- Tests bind a fake implementation of the interface; no test reaches the database to exercise application logic.

## The central rule: application code never touches a model directly

### Why

Eloquent is a powerful, sprawling API. Left unrestricted, query logic metastasises: the same `where('status', 'active')->whereNotNull('activated_at')` predicate gets re-typed in a controller, a job, a Blade view, and three tests, each slightly different. There is no single place to fix an index hint, add soft-delete awareness, or swap the persistence engine. You cannot unit-test a consumer without a database, because the query is welded into it.

A repository fixes this by establishing exactly one seam between the application and persistence. Every read and write of a given model funnels through one class. That gives you:

- **A single source of truth** for each query — define `activeForWorkspace` once.
- **Testability** — consumers depend on an interface; tests swap in a fake with zero database.
- **Swappable persistence** — the interface says nothing about Eloquent; an implementation could use raw SQL, an external API, or a cache, without touching callers.
- **Enforceable boundaries** — "no stray queries" becomes a grep-able, lint-able invariant instead of a code-review hope.

### How

❌ Avoid — Eloquent leaking into an action:

```php
<?php

declare(strict_types=1);

namespace App\Actions\Reports;

use App\Models\Report;
use App\Models\Workspace;

final readonly class PublishLatestStandup
{
    public function handle(Workspace $workspace): void
    {
        // ❌ static query on the model
        $report = Report::query()
            ->where('workspace_id', $workspace->id)
            ->where('type', 'standup')
            ->latest('generated_at')
            ->firstOrFail();

        // ❌ relationship-query traversal
        $stale = $workspace->reports()
            ->where('published_at', '<', now()->subWeek())
            ->get();

        $report->update(['published_at' => now()]); // ❌ write through the model
    }
}
```

✅ Do — depend on a repository, by intent:

```php
<?php

declare(strict_types=1);

namespace App\Actions\Reports;

use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Models\Workspace;

final readonly class PublishLatestStandup
{
    public function __construct(
        private ReportRepositoryInterface $reports,
    ) {}

    public function handle(Workspace $workspace): void
    {
        $report = $this->reports->latestStandupForWorkspace($workspace);

        $this->reports->markPublished($report);
    }
}
```

The action now reads as a sentence. It does not know — and must not know — that `Report` is an Eloquent model, that `workspace_id` is the foreign key, or that `generated_at` is the sort column. All of that lives behind the seam.

## Structure: interface, implementation, binding

### The interface lives in `app/Contracts/Repositories`

The interface is the contract the rest of the application sees. It is pure: typed methods returning domain types (models, collections, scalars), with no hint of how persistence works.

```php
<?php

declare(strict_types=1);

namespace App\Contracts\Repositories;

use App\Models\Report;
use App\Models\Workspace;
use Illuminate\Support\Collection;

interface ReportRepositoryInterface
{
    public function find(int $id): ?Report;

    public function findOrFail(int $id): Report;

    /**
     * @return Collection<int, Report>
     */
    public function publishedForWorkspace(Workspace $workspace): Collection;

    public function latestStandupForWorkspace(Workspace $workspace): Report;

    /**
     * @param  array{type: string, generated_at: \DateTimeInterface, summary: string}  $attributes
     */
    public function createForWorkspace(Workspace $workspace, array $attributes): Report;

    public function markPublished(Report $report): Report;

    public function countActivated(Workspace $workspace): int;
}
```

Note the generic annotation `Collection<int, Report>` so static analysis and the IDE know what the collection holds. Return `?Report` for "may be absent," and a non-nullable `Report` for `findOrFail`/`latest*` methods that throw when nothing matches.

### The implementation lives in `app/Repositories`

This is the ONLY place Eloquent for `Report` is permitted. It is `final readonly` — a repository is a stateless service that maps intent to queries; it holds no mutable state.

```php
<?php

declare(strict_types=1);

namespace App\Repositories;

use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Models\Report;
use App\Models\Workspace;
use Illuminate\Support\Collection;

final readonly class ReportRepository implements ReportRepositoryInterface
{
    public function find(int $id): ?Report
    {
        return Report::query()->find($id);
    }

    public function findOrFail(int $id): Report
    {
        return Report::query()->findOrFail($id);
    }

    /**
     * @return Collection<int, Report>
     */
    public function publishedForWorkspace(Workspace $workspace): Collection
    {
        return Report::query()
            ->where('workspace_id', $workspace->id)
            ->whereNotNull('published_at')
            ->latest('published_at')
            ->get();
    }

    public function latestStandupForWorkspace(Workspace $workspace): Report
    {
        return Report::query()
            ->where('workspace_id', $workspace->id)
            ->where('type', 'standup')
            ->latest('generated_at')
            ->firstOrFail();
    }

    /**
     * @param  array{type: string, generated_at: \DateTimeInterface, summary: string}  $attributes
     */
    public function createForWorkspace(Workspace $workspace, array $attributes): Report
    {
        return $workspace->reports()->create($attributes);
    }

    public function markPublished(Report $report): Report
    {
        $report->forceFill(['published_at' => now()])->save();

        return $report->refresh();
    }

    public function countActivated(Workspace $workspace): int
    {
        return Report::query()
            ->where('workspace_id', $workspace->id)
            ->whereNotNull('activated_at')
            ->count();
    }
}
```

Inside the implementation, every Eloquent idiom is fair game: `Report::query()`, relationship writes via `$workspace->reports()->create(...)`, aggregates, eager loading. The boundary is the class itself — Eloquent goes in, domain types come out.

### Bind interface → implementation in a dedicated provider

Do not scatter `bind` calls across `AppServiceProvider`. Give repositories their own provider so the full persistence wiring is visible in one file. Use `singleton` — repositories are stateless, so there is no reason to reconstruct them per resolution.

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use App\Contracts\Repositories\DeliveryChannelRepositoryInterface;
use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Contracts\Repositories\WorkspaceRepositoryInterface;
use App\Repositories\DeliveryChannelRepository;
use App\Repositories\ReportRepository;
use App\Repositories\WorkspaceRepository;
use Illuminate\Support\ServiceProvider;

final class RepositoryServiceProvider extends ServiceProvider
{
    /**
     * @var array<class-string, class-string>
     */
    private const array BINDINGS = [
        WorkspaceRepositoryInterface::class => WorkspaceRepository::class,
        ReportRepositoryInterface::class => ReportRepository::class,
        DeliveryChannelRepositoryInterface::class => DeliveryChannelRepository::class,
    ];

    public function register(): void
    {
        foreach (self::BINDINGS as $contract => $implementation) {
            $this->app->singleton($contract, $implementation);
        }
    }
}
```

Register the provider in `bootstrap/providers.php` (Laravel 11+/13):

```php
<?php

return [
    App\Providers\AppServiceProvider::class,
    App\Providers\RepositoryServiceProvider::class,
];
```

Consumers never call `app()->make(...)`; they type-hint the interface in their constructor and let the container inject the bound implementation.

## Repository methods are intent-revealing — and never return a query builder

### Why

The method name is the API. `latestStandupForWorkspace(Workspace $workspace): Report` tells the caller precisely what comes back and under what condition it throws. A bag of generic helpers (`all()`, `where()`, `query()`) just relocates Eloquent one layer down without removing it — the caller still has to know the schema and assemble the query. Worse, returning a query builder from a repository method re-opens the boundary you just closed: the caller can now chain `->where()->get()`, and you are back to scattered query logic.

### How

✅ Do — names describe intent; return concrete types:

```php
public function firstForUser(User $user): ?DeliveryChannel;

public function latestStandupForWorkspace(Workspace $workspace): Report;

public function countActivated(Workspace $workspace): int;

public function updateAuthoredDraft(Report $report, string $summary): Report;
```

❌ Avoid — leaking the builder or forcing the caller to query:

```php
// ❌ caller has to know the schema and finish the query
public function query(): Builder;

// ❌ a thin passthrough that pushes Eloquent onto the caller
public function reportsFor(Workspace $workspace): Builder
{
    return $workspace->reports();
}
```

When a screen needs pagination, return a paginator from the repository rather than a builder — the paginator is a finished result, not an open query.

```php
use Illuminate\Contracts\Pagination\LengthAwarePaginator;

/**
 * @return LengthAwarePaginator<int, Report>
 */
public function paginatePublishedForWorkspace(Workspace $workspace, int $perPage = 25): LengthAwarePaginator
{
    return Report::query()
        ->where('workspace_id', $workspace->id)
        ->whereNotNull('published_at')
        ->latest('published_at')
        ->paginate($perPage);
}
```

If you genuinely need flexible filtering, accept a typed criteria object as a parameter and build the query inside the repository — keep the assembly behind the seam, not in the caller.

## The Eloquent boundary, defined precisely

Eloquent — meaning `Model::query()`, `where()`, `find*`, `create`, `update`, `save`, `delete`, relationship-query traversal, and the rest of the query/persistence API — is permitted ONLY in these five places:

1. **Repository implementations** (`app/Repositories/*`) — the sanctioned data-access seam.
2. **Model definitions** (`app/Models/*`) — relationship methods (`hasMany`, `belongsTo`), `casts()`, accessors/mutators, and scopes. A model declaring `reports(): HasMany` is defining structure, not performing application data access.
3. **Filament declarations** — `Resource`, `Table`, `Infolist`, and `Form` classes are framework-bound to Eloquent by design. A Filament `Table` configures `Report::query()` because that is the framework's contract; do not contort it through a repository.
4. **Factories and seeders** (`database/factories/*`, `database/seeders/*`) — these exist to manufacture model rows; Eloquent is their whole job.
5. **Tests** — integration/feature tests may hit the database directly to arrange state and assert persistence (`Report::factory()->create()`, `assertDatabaseHas`).

Everywhere else — actions, services, jobs, commands, listeners, Livewire components, controllers, mailables, view composers — Eloquent is forbidden. Those layers receive models as already-fetched objects (passed in, or returned from a repository) and ask repositories for anything else.

### Pure static helpers on a model are fine

A static method that performs no query is not data access and does not belong behind a repository. It is just a function that happens to live on the model for cohesion.

✅ Do — call a pure helper directly:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

final class DeliveryChannel extends Model
{
    /**
     * Hash a delivery target (Slack webhook URL, email) for safe logging.
     * Pure: no I/O, no query — fine to call anywhere.
     */
    public static function hashTarget(string $target): string
    {
        return hash('sha256', $target);
    }
}
```

```php
// In any service — this is not data access, so no repository needed:
$fingerprint = DeliveryChannel::hashTarget($webhookUrl);
```

The litmus test: does the method touch the database or build a query? If no, it can live as a static helper and be called anywhere. If yes, it belongs in a repository.

## A complete consumer example

A service that depends only on interfaces, with no Eloquent in sight:

```php
<?php

declare(strict_types=1);

namespace App\Services\Reports;

use App\Contracts\Repositories\DeliveryChannelRepositoryInterface;
use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Models\Workspace;
use App\Services\Slack\SlackNotifier;

final readonly class StandupDeliveryService
{
    public function __construct(
        private ReportRepositoryInterface $reports,
        private DeliveryChannelRepositoryInterface $channels,
        private SlackNotifier $slack,
    ) {}

    public function deliverLatest(Workspace $workspace): void
    {
        $report = $this->reports->latestStandupForWorkspace($workspace);
        $channel = $this->channels->firstForUser($workspace->owner);

        if ($channel === null) {
            return;
        }

        $this->slack->post($channel, $report->summary);

        $this->reports->markPublished($report);
    }
}
```

Every collaborator is an injected interface or an already-materialised model. The service is pure orchestration; it can be unit-tested without a database.

## How tests bind a fake

Because consumers depend on the interface, tests provide an in-memory fake and assert behaviour without any database round-trip. This is the payoff for the boilerplate.

```php
<?php

declare(strict_types=1);

use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Models\Report;
use App\Models\Workspace;
use App\Services\Reports\StandupDeliveryService;
use Illuminate\Support\Collection;

/**
 * Minimal in-memory fake implementing the full interface.
 */
final class FakeReportRepository implements ReportRepositoryInterface
{
    /** @var Collection<int, Report> */
    public Collection $published;

    public function __construct(private readonly Report $latest)
    {
        $this->published = collect();
    }

    public function find(int $id): ?Report
    {
        return $this->latest->id === $id ? $this->latest : null;
    }

    public function findOrFail(int $id): Report
    {
        return $this->latest;
    }

    /** @return Collection<int, Report> */
    public function publishedForWorkspace(Workspace $workspace): Collection
    {
        return $this->published;
    }

    public function latestStandupForWorkspace(Workspace $workspace): Report
    {
        return $this->latest;
    }

    /** @param array{type: string, generated_at: \DateTimeInterface, summary: string} $attributes */
    public function createForWorkspace(Workspace $workspace, array $attributes): Report
    {
        return $this->latest;
    }

    public function markPublished(Report $report): Report
    {
        $this->published->push($report);

        return $report;
    }

    public function countActivated(Workspace $workspace): int
    {
        return $this->published->count();
    }
}

it('publishes the latest standup', function (): void {
    $report = Report::factory()->make(['type' => 'standup']);
    $fake = new FakeReportRepository($report);

    $this->app->instance(ReportRepositoryInterface::class, $fake);

    app(StandupDeliveryService::class)->deliverLatest(
        Workspace::factory()->make(),
    );

    expect($fake->published)->toHaveCount(1);
});
```

For one-off expectations you can bind a Mockery double instead of a hand-written fake:

```php
use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Models\Report;

it('marks the report published exactly once', function (): void {
    $report = Report::factory()->make();

    $repo = Mockery::mock(ReportRepositoryInterface::class);
    $repo->shouldReceive('latestStandupForWorkspace')->once()->andReturn($report);
    $repo->shouldReceive('markPublished')->once()->with($report)->andReturn($report);

    $this->app->instance(ReportRepositoryInterface::class, $repo);

    // ... resolve the consumer and invoke it ...
});
```

Either way, the consumer's logic is exercised with zero schema, zero migrations, and zero query tuning — because it never knew about any of that.

## The trade-off, and how to keep it lean

The cost is real: every model gains an interface, an implementation, and a binding line. Accept it deliberately, and keep it from sprawling:

- **Add methods on demand.** Do not pre-build `all()`, `update()`, `delete()`, `paginate()` for a model that needs none of them. Each method should answer a question the application actually asks.
- **Name by intent, not by SQL.** `activeForWorkspace`, not `whereStatusActive`. The name documents the use case.
- **One repository per model.** Cross-model reads belong to whichever model's repository owns the result type; if a query truly spans models with no clear owner, a small read-model/query service is preferable to a god-repository.
- **Keep implementations `final readonly`.** They are stateless; a repository holding mutable state is a smell.
- **Don't wrap pure helpers.** Static, query-free methods stay on the model.

The boilerplate buys a single, enforceable seam — and on a project of any longevity, that seam is what lets you reason about, test, and evolve data access at all.

## See also

- [Models & Eloquent](models-eloquent.md)
- [Services & Actions](services.md)
- [Testing](testing.md)
