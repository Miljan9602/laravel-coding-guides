# Jobs, commands & scheduling

How background work and the command-line interface are built: queued jobs that carry identifiers and re-load their own state, thin commands that delegate to actions, and a data-driven scheduler that fires per-tenant work from one host cron entry.

## Rules at a glance

- A job carries **identifiers** (`workspaceId`, `reportId`, `teamId`) — never serialized request state, session data, authenticated user, or a "current tenant" resolved from a request.
- Inside `handle()`, **re-load aggregates** through repositories from those identifiers. A worker has no request and processes many tenants in sequence; nothing request-scoped is in scope.
- Inject repositories and services into `handle()` via **container method injection**, not into the constructor (the constructor only holds the serialized identifiers).
- Jobs are **idempotent** and **mark failure explicitly**: set `status = failed`, then rethrow so the queue records the failure and retries/backoff apply.
- **Binding-order gotcha**: if a job swaps in a per-tenant dependency (`app()->bind(...)`), it must resolve the orchestrating action/service from the container *after* that bind — otherwise the orchestrator is built with the default dependency.
- **Commands stay thin**: parse input, resolve a workspace, delegate to an action/service, map the result to an exit code. Inject dependencies into `handle()`.
- **Schedule by data, not code**: a `reports:dispatch` command runs every minute, reads each workspace's schedule, and fires the ones due *now in their own timezone* by dispatching a job. Cadence lives in the database, not in `routes/console.php`.
- Guard the dispatcher with `withoutOverlapping()`. The host has exactly one cron line: `* * * * * php artisan schedule:run`.
- **`$timeout` MUST be shorter than the queue's `retry_after`** — otherwise a slow job is still running when the queue releases it again, and `RunWorkspaceReport` runs *twice concurrently* for one workspace.
- **Job middleware (`WithoutOverlapping`, `RateLimited`) `release()` the job and consume an attempt.** With a low `$tries`, a job that merely lost a lock is marked *failed* though nothing went wrong. Raise `$tries` and bound real failures with `$maxExceptions` or `retryUntil()`.
- **Put `onOneServer()` on scheduled tasks and `ShouldBeUnique`/unique jobs, backed by a shared cache driver** (`redis`/`database`, never `file`/`array`). `withoutOverlapping()` alone does **not** stop two HA app servers from each firing every workspace report.
- **`ShouldBeUnique` (`uniqueId()` + `uniqueFor`) is the declarative dedupe** against double-dispatch — a per-workspace, per-kind, per-period key means one queued `RunWorkspaceReport` per workspace per cycle, even if the dispatcher fires twice.
- Test jobs with `Queue::fake()` (dispatch assertions) and by invoking `handle()` through the container — `dispatchSync()` or `app()->call([$job, 'handle'])` — so method injection still resolves.

## Jobs carry identifiers, not state

### Why

A job is a serialized message. When it is dispatched, its constructor arguments are written to the queue store (database, Redis) and rehydrated — possibly minutes later, on a different host, inside a worker process that has **no HTTP request, no session, and no authenticated user**. A single long-lived worker dequeues jobs for many workspaces back to back. Anything you stash on the job that depended on "the current request" — the logged-in user, a request-bound tenant, a freshly-loaded Eloquent model with stale relations — is either meaningless or actively wrong by the time `handle()` runs.

The discipline is simple: a job's payload is the **minimum set of identifiers** needed to re-derive everything else. Pass a `workspaceId`, not a `Workspace`. Pass a `reportId`, not a hydrated `Report` with eager-loaded contributors. Inside `handle()`, load fresh through repositories. This keeps the serialized payload tiny, avoids stale-snapshot bugs, and means the job always operates on the current truth.

### How

✅ Do — promote identifiers, re-load inside `handle()`:

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Actions\RunReport;
use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Contracts\Repositories\WorkspaceRepositoryInterface;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

final class RunWorkspaceReport implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    public int $tries = 3;

    public int $backoff = 30;

    /**
     * @param  'standup'|'digest'  $kind
     */
    public function __construct(
        public readonly int $workspaceId,
        public readonly string $kind,
        public readonly int $reportId,
    ) {}

    public function handle(
        WorkspaceRepositoryInterface $workspaces,
        ReportRepositoryInterface $reports,
        RunReport $action,
    ): void {
        $workspace = $workspaces->findOrFail($this->workspaceId);
        $report    = $reports->findOrFail($this->reportId);

        $result = $action->handle($workspace, $report, $this->kind);

        $reports->markGenerated($report, $result);
    }
}
```

❌ Avoid — serializing request state and hydrated aggregates:

```php
final class RunWorkspaceReport implements ShouldQueue
{
    public function __construct(
        public readonly Workspace $workspace,        // stale snapshot on the wire
        public readonly Report $report,              // eager-loaded relations go stale
        public readonly User $requestedBy,           // there is no "request" in a worker
        public readonly string $currentTenantSlug,   // request-scoped — meaningless here
    ) {}

    public function handle(): void
    {
        // Operates on whatever was true at dispatch time, not now.
        // Relations may be missing; the tenant may be the wrong one.
    }
}
```

> `SerializesModels` will reduce a passed Eloquent model to its key and re-query it on unserialize, so passing a model is not *catastrophic* — but it re-loads only that one row with no relations and hides the identifier you actually want to reason about. Pass the id explicitly. It is honest about the contract and keeps the payload obvious.

## Never rely on a request-scoped "current tenant"

### Why

In a web request it is common to resolve "the current workspace" once (middleware, a scoped singleton) and let everything downstream read it ambiently. That pattern is a trap for jobs. A worker boots once and stays alive; there is no per-request reset. If a job reaches for an ambient `CurrentWorkspace` singleton, it gets whatever the *previous* job left behind — a different tenant entirely. This is the classic cross-tenant data leak.

A job must be **explicit and self-contained** about which tenant it serves. The tenant identity arrives as a constructor identifier; the job loads that workspace and threads it explicitly through every call. Nothing is read from a global.

### How

✅ Do — the workspace is loaded from the payload id and passed explicitly:

```php
public function handle(WorkspaceRepositoryInterface $workspaces, RunReport $action): void
{
    $workspace = $workspaces->findOrFail($this->workspaceId);

    // Everything downstream receives $workspace as an argument.
    $action->handle($workspace, $this->reportId, $this->kind);
}
```

❌ Avoid — reading the tenant from ambient request-scoped state:

```php
public function handle(RunReport $action): void
{
    // Set by web middleware. In a worker this is the *previous* job's tenant.
    $workspace = app(CurrentWorkspace::class)->get();

    $action->handle($workspace, $this->reportId, $this->kind);
}
```

## Idempotency and explicit failure

### Why

Queues retry. A job can run twice because of a worker crash after the work completed but before the ack, a manual `queue:retry`, or a timeout that fired while the job was still finishing. So `handle()` must be safe to run more than once: prefer upserts keyed by a stable id, guard on a status column, and avoid "append a row every run" side effects.

When something does go wrong, fail **loudly and recordably**. Mark the aggregate as `failed` so the UI and other readers see an accurate state, then **rethrow** so the queue marks the attempt failed, applies backoff, retries up to `$tries`, and eventually routes to `failed_jobs` / `failed()`. Swallowing the exception leaves the job marked "succeeded" with broken data — the worst outcome.

### How

✅ Do — guard for idempotency, set `failed`, rethrow, and clean up in `failed()`:

```php
public function handle(ReportRepositoryInterface $reports, RunReport $action): void
{
    $report = $reports->findOrFail($this->reportId);

    if ($report->status === 'generated') {
        return; // Already done by a prior attempt — safe no-op.
    }

    try {
        $result = $action->handle(/* ... */);
        $reports->markGenerated($report, $result);
    } catch (\Throwable $e) {
        $reports->markFailed($report, $e->getMessage());

        throw $e; // Let the queue record the failure and apply retry/backoff.
    }
}

/**
 * Runs after the final attempt is exhausted.
 */
public function failed(\Throwable $e): void
{
    app(ReportRepositoryInterface::class)
        ->markFailed($this->reportId, $e->getMessage());
}
```

❌ Avoid — silently swallowing, and side effects that duplicate on retry:

```php
public function handle(ReportRepositoryInterface $reports): void
{
    try {
        $reports->appendDeliveryLog($this->reportId, 'started'); // grows every retry
        // ... work ...
    } catch (\Throwable $e) {
        report($e);  // swallowed: job is marked "succeeded", status stays "pending"
    }
}
```

## Inject dependencies into `handle()`

### Why

The constructor of a queued job is for serialized data only — its arguments go on the wire. Services and repositories are not serializable (and must not be: you want a fresh, container-built instance at execution time, with the right per-environment bindings). Laravel resolves the parameters of `handle()` out of the container when the job runs. This is **container method injection**: declare what you need, typed, and the framework provides it. It also keeps jobs trivially testable — a test binds fakes for those interfaces and the same `handle()` signature picks them up.

### How

✅ Do — identifiers in the constructor, collaborators in `handle()`:

```php
public function __construct(
    public readonly int $workspaceId,
    public readonly int $reportId,
) {}

public function handle(
    WorkspaceRepositoryInterface $workspaces,
    ReportRepositoryInterface $reports,
    SlackNotifier $slack,
): void {
    // $workspaces, $reports, $slack are container-resolved at run time.
}
```

❌ Avoid — collaborators in the constructor (unserializable, snapshots bindings):

```php
public function __construct(
    public readonly int $workspaceId,
    private readonly SlackNotifier $slack, // serialized? — no. resolved too early? — yes.
) {}
```

## The container-binding-order gotcha

### Why

This is the subtle one. A job often needs a **per-tenant** version of a shared dependency — for example a `GitHubClient` authenticated with *this* workspace's installation token rather than the default app token. The natural move is to rebind it in the container at the start of `handle()`:

```php
app()->bind(GitHubClient::class, fn () => $this->tenantGitHubClient($workspace));
```

The trap: the orchestrating action (`RunReport`) declares `GitHubClient` as a **constructor** dependency. If that action was *already resolved* — because you injected it into `handle()`'s signature, or resolved it before the bind — then it was constructed with the **default** `GitHubClient`. Your rebind happens too late to matter. The action quietly talks to GitHub as the wrong identity.

The container builds an object's constructor dependencies at the moment the object is resolved. Therefore: **rebind first, resolve the orchestrator after.** Anything that depends on the swapped binding must be pulled from the container *after* the `bind()` call, not injected into `handle()` (which resolves before your method body executes).

### How

✅ Do — bind the per-tenant dependency, then resolve the orchestrator from the container:

```php
public function handle(WorkspaceRepositoryInterface $workspaces): void
{
    $workspace = $workspaces->findOrFail($this->workspaceId);

    // 1. Swap in this tenant's GitHub client for the duration of this job.
    app()->bind(
        GitHubClient::class,
        fn (): GitHubClient => new RestGitHubClient($workspace->githubInstallationToken()),
    );

    // 2. Resolve the orchestrator AFTER the bind, so its constructor
    //    receives the tenant-scoped GitHubClient.
    $action = app(RunReport::class);

    $action->handle($workspace, $this->reportId, $this->kind);
}
```

❌ Avoid — injecting the orchestrator into `handle()`, so it is built before the rebind:

```php
public function handle(
    WorkspaceRepositoryInterface $workspaces,
    RunReport $action,            // resolved by the container BEFORE your body runs...
): void {
    $workspace = $workspaces->findOrFail($this->workspaceId);

    app()->bind(GitHubClient::class, fn () => /* tenant client */); // ...too late.

    $action->handle($workspace, $this->reportId, $this->kind); // uses the DEFAULT client
}
```

> If you find yourself doing container surgery often, consider passing the tenant client *explicitly* down the call chain instead of rebinding — explicit arguments have no ordering hazard. Rebinding is justified when a deep, hard-to-thread dependency must change for one tenant; when it is, the order rule is non-negotiable.

## Commands stay thin

### Why

A console command is an *entry point*, the CLI sibling of a controller. Its job is to translate input (arguments, options) into a call on an action or service and translate the result back into an exit code and console output. Business logic does not belong in `handle()` — if it lives there it cannot be reused by a job, a route, or a test without booting the whole console kernel. Keep commands thin and delegate, exactly as controllers delegate to actions.

Inject dependencies into the command's `handle()` (Laravel applies method injection to commands too). Resolve the per-tenant workspace from the option, hand off, and map the outcome to `self::SUCCESS` / `self::FAILURE`.

### How

✅ Do — parse, delegate, map to an exit code:

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use App\Actions\RequestReport;
use App\Contracts\Repositories\WorkspaceRepositoryInterface;
use Illuminate\Console\Command;

final class SendStandupCommand extends Command
{
    protected $signature = 'standup:send {--workspace=} {--dry-run}';

    protected $description = 'Generate and deliver the daily standup for a workspace.';

    public function handle(
        WorkspaceRepositoryInterface $workspaces,
        RequestReport $action,
    ): int {
        $workspace = $workspaces->findBySlug((string) $this->option('workspace'));

        if ($workspace === null) {
            $this->error('Unknown workspace.');

            return self::FAILURE;
        }

        $report = $action->handle($workspace, 'standup', dryRun: (bool) $this->option('dry-run'));

        $this->info("Standup {$report->status} for {$workspace->slug}.");

        return self::SUCCESS;
    }
}
```

❌ Avoid — collectors, HTTP, and aggregation inlined in the command:

```php
final class SendStandupCommand extends Command
{
    public function handle(): int
    {
        $commits = Http::withToken(config('digest.github_token'))
            ->get('https://api.github.com/...')->json(); // collection logic in a command

        $report = /* aggregate, summarise, build Slack blocks, send... all here */;

        return self::SUCCESS;
    }
}
```

## Scheduling: data-driven, per-tenant dispatch

### Why

A single hard-coded `Schedule::command('standup:send')->weekdays()->at('09:00')` works for exactly one tenant in one timezone. The moment you have many workspaces — each with its own send time, timezone, and cadence — baking that into `routes/console.php` means a code change and deploy every time a customer wants their standup at 08:30 instead of 09:00. **Cadence is data.**

The durable pattern is a thin **dispatcher command** that the scheduler runs *every minute*. It reads each enabled workspace schedule, asks "is this due right now, in this workspace's own timezone?", and for the ones that are, dispatches a queued job. `routes/console.php` then contains a single, stable line. Adding, removing, or retiming a workspace is a database write — no deploy.

Guard the dispatcher with `withoutOverlapping()` so a slow minute never starts a second overlapping pass. The host needs exactly one crontab entry: `* * * * * php artisan schedule:run`.

### How

`routes/console.php` — one entry, forever:

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Schedule;

// reports:dispatch fires each workspace's standup/digest in its own timezone.
// Host crontab: * * * * * php artisan schedule:run
Schedule::command('reports:dispatch')
    ->everyMinute()
    ->withoutOverlapping();
```

The dispatcher command — reads schedules, fires what is due *now*:

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use App\Contracts\Repositories\ScheduleRepositoryInterface;
use App\Jobs\RunWorkspaceReport;
use App\Models\Schedule as WorkspaceSchedule;
use Carbon\CarbonImmutable;
use Illuminate\Console\Command;

final class DispatchScheduledReports extends Command
{
    protected $signature = 'reports:dispatch';

    protected $description = 'Dispatch per-workspace scheduled standups/digests due now.';

    public function handle(ScheduleRepositoryInterface $schedules): int
    {
        $dispatched = 0;

        foreach ($schedules->allEnabled() as $schedule) {
            if ($this->isDue($schedule)) {
                RunWorkspaceReport::dispatch(
                    $schedule->workspace_id,
                    $schedule->kind,
                    $schedule->report_id,
                );

                $dispatched++;
            }
        }

        $this->info("Dispatched {$dispatched} scheduled report(s).");

        return self::SUCCESS;
    }

    private function isDue(WorkspaceSchedule $schedule): bool
    {
        // Evaluate the clock in the WORKSPACE's timezone, not the server's.
        $now = CarbonImmutable::now($schedule->timezone);

        if ($now->format('H:i') !== $schedule->time) {
            return false;
        }

        return match ($schedule->kind) {
            'standup' => $now->isWeekday(),
            'digest'  => $now->isSaturday(),
            default   => false,
        };
    }
}
```

The minute-granularity comparison (`$now->format('H:i') === $schedule->time`) pairs with the every-minute cron: each schedule fires once, in the minute it matches. Store `time` as `HH:MM` and `timezone` as an IANA name (`Europe/Belgrade`) so a workspace's 09:00 means *their* 09:00.

❌ Avoid — per-tenant cadence hard-coded in `routes/console.php`:

```php
Schedule::command('standup:send --workspace=nima')->weekdays()->at('09:00');
Schedule::command('standup:send --workspace=acme')->weekdays()->at('08:30');
Schedule::command('digest:send  --workspace=nima')->saturdays()->at('10:00');
// Every new customer or time change is a code edit + deploy. Does not scale.
```

### Why dispatch a job rather than do the work inline

The dispatcher must finish *fast* — it runs every minute and is `withoutOverlapping()`. If it generated reports synchronously, one slow GitHub call would stall the whole pass and skip other due workspaces. So the dispatcher only *decides and dispatches*; the heavy lifting (collect → aggregate → summarise → deliver) happens in `RunWorkspaceReport` on the queue, where retries and backoff protect it.

## Reliability: timeouts, retries, overlap, and uniqueness

Our two reliability hazards are structural, not accidental. (1) `reports:dispatch` runs **every minute**, so the same workspace can be evaluated as "due" twice in adjacent minutes or by two concurrent passes — a *duplicate-dispatch* risk. (2) `RunWorkspaceReport` is **slow and external** (it calls GitHub, then the `SummaryChain` AI fallback, then Slack/email), so it can outlive a timeout or a lease — a *double-run* and *false-failure* risk. The four controls below close those gaps. They compose; none is redundant.

### 1. `$timeout` must be shorter than the queue's `retry_after`

#### Why

A queue worker leases a job for `retry_after` seconds (set per connection in `config/queue.php`). If the job has not been acknowledged within that window, the queue assumes the worker died and **releases the job back for another worker** — even though the original worker is still grinding through a slow `SummaryChain` call. Now two workers run `RunWorkspaceReport` for the same workspace at once: two GitHub crawls, two AI bills, two Slack posts. The job's own `$timeout` is the only thing that kills the first attempt *before* the lease expires. So the invariant is non-negotiable:

```
$timeout  <  retry_after
```

Earlier sections name double-runs as a thing to design against (idempotency, `ShouldBeUnique`); this is the mechanism that prevents the most common cause in the first place. Idempotency limits the *damage* of a double-run; `$timeout < retry_after` prevents the *concurrency*.

#### How

✅ Do — set a `$timeout` comfortably under `retry_after`, and pick a connection whose `retry_after` exceeds your worst-case run:

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

final class RunWorkspaceReport implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    // The whole pipeline (GitHub crawl + SummaryChain + delivery) must finish
    // inside this. It MUST be < the connection's retry_after.
    public int $timeout = 240;

    public int $tries = 3;

    public int $backoff = 30;

    public function __construct(
        public readonly int $workspaceId,
        public readonly string $kind,
        public readonly int $reportId,
    ) {}
}
```

```php
<?php

declare(strict_types=1);

// config/queue.php — retry_after gives the job MORE wall-clock than its $timeout,
// so $timeout fires first and there is no concurrent re-lease.
return [
    'connections' => [
        'redis' => [
            'driver' => 'redis',
            'queue' => 'reports',
            'retry_after' => 300, // > RunWorkspaceReport::$timeout (240)
        ],
    ],
];
```

❌ Avoid — a timeout longer than (or equal to) `retry_after`, so the lease expires mid-run:

```php
final class RunWorkspaceReport implements ShouldQueue
{
    public int $timeout = 600; // 'retry_after' is 300 → job re-leased at 300s while
                               // attempt #1 is still running. Two concurrent runs.
}
```

Edge cases:

- **`$timeout` needs the `pcntl` extension** to actually interrupt the running job; without it the timeout is best-effort. CI and production workers must have it.
- **A long, legitimately slow job** (large org, slow AI) means you raise *both* numbers together, keeping the `<` gap — not just bump `$timeout`.
- **`SummaryChain` HTTP calls should carry their own client timeouts** so the job times out cleanly rather than blocking the worker beyond `$timeout` in un-interruptible C calls.

### 2. Middleware `release()` consumes an attempt — don't let a lock look like a failure

#### Why

`WithoutOverlapping` and `RateLimited` do not pause a job in place. When they can't acquire the lock (or the rate budget is spent), they **`release()` the job back onto the queue** — and *a release still counts as a used attempt*. So a `RunWorkspaceReport` with `$tries = 3` that loses the overlap lock three times in a row is moved to `failed_jobs` having done **no work and hit no error**. The `failed()` hook runs, the report is marked `failed`, and an operator is paged for a non-event.

The fix is to stop counting "couldn't get the lock yet" as a failure. Two levers:

- **`$maxExceptions`** caps how many attempts may end in a *thrown exception* — independent of total `$tries`. Set `$tries` high (so lock-driven releases have room to retry) and `$maxExceptions` low (so a handful of real errors still fails fast).
- **`retryUntil()`** replaces the attempt *count* with a wall-clock *deadline*. The job retries — through releases and errors — until the deadline, which suits a report that is worthless once its cycle has passed.

#### How

✅ Do — high `$tries` so released jobs can re-run, bounded by `$maxExceptions` and a deadline:

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Models\Workspace;
use Carbon\CarbonImmutable;
use DateTimeInterface;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\Middleware\WithoutOverlapping;
use Illuminate\Queue\SerializesModels;

final class RunWorkspaceReport implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    // Generous attempts so overlap/rate-limit RELEASES have room to retry...
    public int $tries = 25;

    // ...but only 3 attempts may end in a real thrown exception before we give up.
    public int $maxExceptions = 3;

    public int $timeout = 240;

    public function __construct(
        public readonly int $workspaceId,
        public readonly string $kind,
        public readonly int $reportId,
    ) {}

    /**
     * @return array<int, object>
     */
    public function middleware(): array
    {
        // One in-flight report per workspace; a contender is RELEASED, not failed.
        return [
            (new WithoutOverlapping((string) $this->workspaceId))
                ->releaseAfter(30)
                ->expireAfter(300),
        ];
    }

    /**
     * Stop retrying once the report's cycle has passed — releases are pointless then.
     */
    public function retryUntil(): DateTimeInterface
    {
        return CarbonImmutable::now()->addMinutes(30);
    }
}
```

❌ Avoid — a low `$tries` with overlap middleware, so a lost lock burns the budget and "fails":

```php
final class RunWorkspaceReport implements ShouldQueue
{
    public int $tries = 3; // three lock RELEASES exhaust this — job is marked FAILED
                           // though it never ran and never threw.

    public function middleware(): array
    {
        return [new WithoutOverlapping((string) $this->workspaceId)];
    }

    public function failed(\Throwable $e): void
    {
        // Fires for a job that only ever lost a lock. False alarm, false `failed` row.
    }
}
```

Edge cases:

- **`releaseAfter()` vs `$backoff`**: `WithoutOverlapping::releaseAfter()` sets the delay for a *lock-loss* release; `$backoff` governs the delay after a *thrown* error. Set both deliberately — a lock contender should retry sooner than an erroring one.
- **Always `expireAfter()` on `WithoutOverlapping`.** If the lock holder dies mid-run without releasing, the lock is otherwise held forever and every later attempt for that workspace is released into a deadlock.
- **`retryUntil()` is evaluated once at first dispatch** and stored on the payload, not recomputed each attempt — so base it on "now + window", not on an absolute time you expect to recompute.

### 3. `onOneServer()` + a shared cache lock for the HA / multi-server case

#### Why

`withoutOverlapping()` (the scheduler guard) and `WithoutOverlapping` (the job middleware) both take a lock in the **cache**. With the default `file` (or, worse, `array`) cache driver, that lock lives on *one box's local disk*. The instant the app runs on two HA servers, each server has its **own** scheduler and its **own** file lock — so both run `reports:dispatch` in the same minute, both find every workspace due, and every workspace gets **two** `RunWorkspaceReport` jobs. The overlap guard saw no conflict because the two locks never met.

Two things fix this together:

- **`onOneServer()`** on the scheduled task. Before running, the scheduler claims a cache lock that all servers contend for; only the winner runs the command that minute. This *requires* a shared, atomic-lock cache driver — `redis`, `database`, `memcached`, or `dynamodb` — **never** `file`/`array`.
- **A shared cache driver for every lock**, so `withoutOverlapping()` and `WithoutOverlapping`/`ShouldBeUnique` locks are visible across servers in the first place.

`onOneServer()` solves cross-server *scheduler* duplication; `ShouldBeUnique` (next section) solves cross-server *dispatch* duplication. They are complementary, not alternatives.

#### How

✅ Do — `onOneServer()` on the per-minute dispatcher, on a shared cache driver:

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Schedule;

// Host crontab on EVERY app server: * * * * * php artisan schedule:run
// onOneServer() ensures only ONE server actually runs reports:dispatch each minute.
// Requires config('cache.default') to be a shared driver (redis/database), not file.
Schedule::command('reports:dispatch')
    ->everyMinute()
    ->onOneServer()        // cross-server: one winner per minute
    ->withoutOverlapping(); // same-server: no overlapping pass
```

```php
<?php

declare(strict_types=1);

// config/cache.php — locks live in a store all servers share, so onOneServer(),
// withoutOverlapping(), WithoutOverlapping, and ShouldBeUnique actually coordinate.
return [
    'default' => config('CACHE_STORE', 'redis'), // NOT 'file' / 'array' in production
];
```

❌ Avoid — overlap guard only, on a per-box cache, across two servers:

```php
// File-cache lock = one lock per server. Both servers run this minute.
Schedule::command('reports:dispatch')
    ->everyMinute()
    ->withoutOverlapping(); // stops a single box overlapping itself — and nothing more
// Result on 2 servers: every workspace report dispatched twice, every minute.
```

Edge cases:

- **`onOneServer()` needs distinct task identity.** Two scheduled entries that look identical (same command, same frequency) can confuse the one-server lock; differentiate with `->name('reports:dispatch')`.
- **The cache driver must support atomic locks.** `redis`, `database`, `memcached`, `dynamodb`, `array` do; the *file* driver does not support cross-process locks reliably for this — use `redis`/`database` in production.
- **Queue workers run on yet more servers.** `onOneServer()` only dedupes the *scheduler*; once a job is queued, any worker may pick it up — which is exactly why `RunWorkspaceReport` is still idempotent and `ShouldBeUnique`.

### 4. `ShouldBeUnique` as the declarative dedupe for double-dispatch

#### Why

Even with `onOneServer()`, dispatch can double up: a redeploy that overlaps the minute boundary, a manual `reports:dispatch` while cron also fires, a retry of the dispatcher itself. Rather than reason about every race, **declare the invariant on the job**: *at most one queued `RunWorkspaceReport` per workspace, per kind, per cycle.* `ShouldBeUnique` enforces it — a second dispatch with the same `uniqueId()` is silently dropped while a matching job is enqueued or running.

This is the declarative twin of the imperative idempotency guard in `handle()`. The status check inside `handle()` stops *duplicate work*; `ShouldBeUnique` stops the *duplicate job ever entering the queue*, which also saves the wasted dequeue, the GitHub round-trip, and the lock churn.

#### How

✅ Do — `ShouldBeUnique` keyed by workspace + kind + period, bounded by `$uniqueFor`:

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use Carbon\CarbonImmutable;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

final class RunWorkspaceReport implements ShouldQueue, ShouldBeUnique
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    // Auto-release the unique lock after this many seconds, in case a worker dies
    // holding it — keep it >= $timeout so it never frees mid-run.
    public int $uniqueFor = 300;

    public function __construct(
        public readonly int $workspaceId,
        public readonly string $kind,
        public readonly int $reportId,
    ) {}

    /**
     * One queued report per workspace, per kind, per calendar day — a second
     * dispatch this cycle is dropped, not enqueued.
     */
    public function uniqueId(): string
    {
        $day = CarbonImmutable::now()->toDateString();

        return "report:{$this->workspaceId}:{$this->kind}:{$day}";
    }
}
```

The unique lock uses the cache, so the **shared-driver rule from §3 applies here too** — on the `file` driver, `ShouldBeUnique` is per-box and two servers each enqueue their "unique" job.

❌ Avoid — leaning on `handle()`'s status check alone to absorb double-dispatch:

```php
final class RunWorkspaceReport implements ShouldQueue // NOT ShouldBeUnique
{
    public function handle(/* ... */): void
    {
        // The duplicate job already paid: dequeued, re-loaded the workspace, hit GitHub,
        // took the overlap lock — only to discover status === 'generated' and no-op.
        if ($this->report->status === 'generated') {
            return;
        }
    }
}
```

Edge cases:

- **`ShouldBeUnique` vs `ShouldBeUniqueUntilProcessing`.** The default holds the lock until the job *finishes* (or `$uniqueFor` elapses); `ShouldBeUniqueUntilProcessing` frees it the moment processing *starts*, allowing the next cycle's job to queue while this one runs. For a per-cycle report, the default is correct — you do not want a re-dispatch sneaking in while one is still generating.
- **Pick the period granularity to match the cadence.** Keying by `toDateString()` is right for a once-daily standup; a digest keyed by ISO week (`now()->format('o-W')`) so one digest per week survives a retry across a day boundary.
- **`$uniqueFor` is a safety valve, not the dedupe window.** Keep it `>= $timeout` so a crashed worker can't free the lock and admit a concurrent run, but small enough that a genuinely new cycle isn't blocked by a stale lock.

## Testing jobs and commands

### Why

Two distinct things need testing, with two distinct tools. **"Was the job dispatched, with the right identifiers?"** is answered with `Queue::fake()` — assert on the queue without running the job. **"Does the job do the right thing when it runs?"** is answered by executing `handle()` *through the container*, so method injection still resolves your fakes. Use `dispatchSync()` or `app()->call([$job, 'handle'])`; never call `$job->handle()` with no arguments — that bypasses method injection and the typed parameters will be unresolved.

### How

Assert dispatch with `Queue::fake()`:

```php
use App\Jobs\RunWorkspaceReport;
use Illuminate\Support\Facades\Queue;

it('dispatches a report job for each due workspace', function (): void {
    Queue::fake();

    $schedules = new FakeScheduleRepository([
        makeSchedule(workspaceId: 1, kind: 'standup', time: now()->format('H:i')),
    ]);

    $this->artisan('reports:dispatch', [], )->assertSuccessful();

    Queue::assertPushed(
        RunWorkspaceReport::class,
        fn (RunWorkspaceReport $job): bool => $job->workspaceId === 1 && $job->kind === 'standup',
    );
});
```

Exercise the job body through the container so `handle()` gets its fakes:

```php
use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Jobs\RunWorkspaceReport;

it('marks the report generated when it runs', function (): void {
    $reports = new FakeReportRepository([makeReport(id: 42, status: 'pending')]);
    app()->instance(ReportRepositoryInterface::class, $reports);

    // dispatchSync runs handle() now, with container method injection.
    RunWorkspaceReport::dispatchSync(workspaceId: 1, kind: 'standup', reportId: 42);

    expect($reports->find(42)->status)->toBe('generated');
});
```

Equivalently, `app()->call([new RunWorkspaceReport(1, 'standup', 42), 'handle'])` runs the same body with injected dependencies — useful when you want to construct the job by hand and skip the queue machinery entirely.

❌ Avoid — calling `handle()` with no arguments:

```php
(new RunWorkspaceReport(1, 'standup', 42))->handle();
// TypeError: $reports, $action, ... were never resolved. Method injection skipped.
```

For commands, `$this->artisan('standup:send', ['--workspace' => 'nima', '--dry-run' => true])` boots the command through the kernel (so its `handle()` dependencies are injected) and lets you assert on exit code and output with `assertSuccessful()`, `expectsOutput()`, and friends.

## See also

- [Actions](actions.md) — the single-purpose orchestrators that jobs and commands delegate to.
- [Repositories](repositories.md) — how `handle()` re-loads aggregates from identifiers instead of carrying hydrated models.
- [Config & secrets](config-and-secrets.md) — `config('queue.*')` `retry_after` and `config('cache.default')`: the shared-driver, timeout-vs-lease settings the reliability rules depend on.
- [Error handling](error-handling.md) — failing loudly, `$maxExceptions`/`retryUntil()`, and not letting a released-lock job masquerade as a real failure.
- [Testing](testing.md) — `Queue::fake()`, fake repositories, and exercising `handle()` through the container.
