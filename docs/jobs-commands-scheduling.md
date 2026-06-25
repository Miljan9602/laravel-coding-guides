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
- [Testing](testing.md) — `Queue::fake()`, fake repositories, and exercising `handle()` through the container.
