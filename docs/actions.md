# Actions (single-purpose write operations)

An **Action** is a class that encapsulates exactly one write or orchestration in your domain — connect a Slack channel, create a team, request a report. It is the smallest reusable unit of intent: one verb, one effect, one entry point.

## Rules at a glance

- One Action = one write or orchestration. If it does two unrelated things, split it.
- Exactly **one public method**, named `handle()`. Everything else is `private`.
- Name the class as an **imperative verb phrase**: `ConnectSlackChannel`, `CreateTeam`, `RequestReport`, `CaptureInstallation`.
- Live in `app/Actions/<Domain>/` — group by domain, not by layer.
- Declare `final readonly class` with `declare(strict_types=1)`.
- Inject repositories and services via the constructor (`private readonly`). **Never touch an Eloquent model directly** — go through a repository.
- Return the created/updated model, or `void` if there is nothing meaningful to return.
- Wrap multiple writes in `DB::transaction()` so a partial failure rolls back.
- Actions may compose other actions and services; keep the orchestration shallow and readable.
- An Action does **one thing once**. A **Service** is a reusable capability. Choose accordingly.

## Why Actions exist

Controllers, commands, jobs, and Livewire components all need to *do* things — and they all tend to accumulate the same business logic copy-pasted across entry points. An Action gives that logic exactly one home. The result:

- **Single call site to reason about.** "What happens when we connect a Slack channel?" has one answer: `ConnectSlackChannel::handle()`.
- **Trivially testable.** Construct the Action with fake repositories, call `handle()`, assert on the result. No HTTP layer, no console kernel.
- **Reusable across entry points.** The same Action runs from a controller, an artisan command, a queued job, and a Livewire action without duplication.
- **Composable.** Bigger workflows are built by calling smaller Actions, so each layer stays small and single-purpose.

## The shape of an Action

### One public method named `handle()`

A consistent entry point means every caller — and every reader — knows where to look. The method name carries no information that the class name doesn't already carry, so we fix it as `handle()` and let the *class* name describe the intent.

```php
// app/Actions/Slack/ConnectSlackChannel.php
declare(strict_types=1);

namespace App\Actions\Slack;

final readonly class ConnectSlackChannel
{
    public function __construct(
        private SlackChannelRepository $channels,
    ) {}

    public function handle(Workspace $workspace, string $channelId): SlackChannel
    {
        return $this->channels->connect($workspace, $channelId);
    }
}
```

```php
// ✅ Do: invoke via the fixed entry point
$channel = $connectSlackChannel->handle($workspace, 'C0123ABCD');
```

```php
// ❌ Avoid: multiple public methods turn an Action back into a grab-bag service
final readonly class SlackChannelManager
{
    public function connect(Workspace $workspace, string $channelId): SlackChannel { /* ... */ }
    public function disconnect(SlackChannel $channel): void { /* ... */ }
    public function rename(SlackChannel $channel, string $name): SlackChannel { /* ... */ }
}
```

Those three operations are three Actions: `ConnectSlackChannel`, `DisconnectSlackChannel`, `RenameSlackChannel`.

> Some teams prefer `__invoke()` so the object is callable. That is a valid variant, but pick one convention and apply it everywhere. This guide standardises on `handle()` because it reads clearly at the call site (`$action->handle(...)`) and never collides with framework auto-invocation.

### Imperative class names

A name is a contract. `CreateTeam` promises to create a team; `TeamService` promises nothing in particular. Verb-first names make call sites read like sentences and make the `app/Actions/` directory a browsable list of everything the system *does*.

```php
// ✅ Do — imperative, specific, reads as an instruction
final readonly class CaptureInstallation { /* ... */ }
final readonly class RequestReport { /* ... */ }
final readonly class ConnectSlackChannel { /* ... */ }

// ❌ Avoid — nouns and vague managers say nothing about the effect
final readonly class InstallationHandler { /* ... */ }
final readonly class ReportService { /* ... */ }   // is this a write? a query? unclear
final readonly class SlackManager { /* ... */ }
```

### Location: group by domain

Actions live in `app/Actions/<Domain>/`. Grouping by domain (not by `app/Actions/Create/`, `app/Actions/Update/`) keeps everything about Reports — creating, requesting, archiving — in one folder, which is how engineers actually navigate.

```
app/Actions/
├── Slack/
│   ├── ConnectSlackChannel.php
│   └── PostReportToSlack.php
├── Team/
│   ├── CreateTeam.php
│   └── CaptureInstallation.php
└── Report/
    ├── RequestReport.php
    └── ArchiveReport.php
```

## Construction and dependencies

### `final readonly class` with constructor DI

An Action holds no mutable state — it takes inputs to `handle()` and produces an effect. Marking it `final readonly` makes that explicit and lets the container resolve it cleanly. Dependencies are promoted, `private`, and `readonly`.

```php
declare(strict_types=1);

namespace App\Actions\Report;

final readonly class ArchiveReport
{
    public function __construct(
        private ReportRepository $reports,
        private AuditLogger $audit,
    ) {}

    public function handle(Report $report): Report
    {
        $archived = $this->reports->markArchived($report);
        $this->audit->record('report.archived', $archived->id);

        return $archived;
    }
}
```

Because the container builds Actions, you never `new` them in production code — type-hint the Action and Laravel injects it with its dependencies already wired.

```php
// ✅ Do: let the container resolve the Action and its dependencies
final class ReportController
{
    public function store(StoreReportRequest $request, RequestReport $requestReport): RedirectResponse
    {
        $report = $requestReport->handle(
            $request->user()->currentWorkspace(),
            $request->validated('repository'),
        );

        return to_route('reports.show', $report);
    }
}
```

```php
// ❌ Avoid: manual construction defeats DI and makes wiring brittle
$requestReport = new RequestReport(
    new ReportRepository(),
    new GitHubClient(config('services.github.token')), // and on, and on...
);
```

### Never touch a model directly — go through a repository

This is the rule that keeps Actions thin and persistence swappable. The Action expresses *intent*; the repository owns *how* that intent hits the database. Calling `Report::create(...)` or `$report->save()` inside an Action leaks Eloquent into your business logic and makes the Action impossible to unit-test without a database.

```php
// ✅ Do: delegate persistence to a repository
final readonly class CreateTeam
{
    public function __construct(
        private TeamRepository $teams,
    ) {}

    public function handle(User $owner, string $name): Team
    {
        return $this->teams->create($owner, $name);
    }
}
```

```php
// ❌ Avoid: Eloquent calls inside the Action
final readonly class CreateTeam
{
    public function handle(User $owner, string $name): Team
    {
        return Team::create([            // model touched directly
            'owner_id' => $owner->id,     // now you can't fake persistence in a test
            'name'     => $name,
            'slug'     => Str::slug($name),
        ]);
    }
}
```

### `config()`, not `env()`

Outside of `config/` files, read configuration through `config()`. `env()` returns `null` once the config is cached in production, which silently breaks the Action.

```php
// ✅ Do
$maxRepos = config('digest.contributors.max');

// ❌ Avoid — null after `php artisan config:cache`
$maxRepos = env('DIGEST_MAX_CONTRIBUTORS');
```

## Return values

Return the model the Action created or updated, so callers can redirect to it, serialise it, or chain further work. Return `void` only when there is genuinely nothing to hand back (a fire-and-forget side effect). Never return an array of loosely-typed data — return a typed model or DTO.

```php
// ✅ Do: return the entity so the caller can use it
public function handle(Workspace $workspace, string $channelId): SlackChannel
{
    return $this->channels->connect($workspace, $channelId);
}

// ✅ Also fine: void when there is no meaningful result
public function handle(SlackChannel $channel): void
{
    $this->channels->disconnect($channel);
}

// ❌ Avoid: untyped bags of data
public function handle(Workspace $workspace, string $channelId): array
{
    return ['ok' => true, 'channel' => $channelId];
}
```

## Wrap multiple writes in a transaction

When `handle()` performs more than one write, wrap them in `DB::transaction()`. A closure-based transaction commits on success and rolls back on any thrown exception, so you never leave half-created state — a `Team` without its owning membership, an installation row without its repositories.

```php
declare(strict_types=1);

namespace App\Actions\Team;

use Illuminate\Support\Facades\DB;

final readonly class CreateTeam
{
    public function __construct(
        private TeamRepository $teams,
        private MembershipRepository $memberships,
    ) {}

    public function handle(User $owner, string $name): Team
    {
        return DB::transaction(function () use ($owner, $name): Team {
            $team = $this->teams->create($owner, $name);

            $this->memberships->add($team, $owner, role: 'owner');

            return $team;
        });
    }
}
```

```php
// ❌ Avoid: two writes with no transaction — a failure on the second
// leaves an orphaned team with no owner membership
public function handle(User $owner, string $name): Team
{
    $team = $this->teams->create($owner, $name);
    $this->memberships->add($team, $owner, role: 'owner'); // throws? team already committed

    return $team;
}
```

If your repositories run raw queries or call out to non-transactional systems (e.g. an external API) between writes, prefer the manual form so you can place the external call *after* the commit:

```php
public function handle(User $owner, string $name): Team
{
    $team = DB::transaction(function () use ($owner, $name): Team {
        $team = $this->teams->create($owner, $name);
        $this->memberships->add($team, $owner, role: 'owner');

        return $team;
    });

    // Side effects that must not run inside the transaction go here.
    $this->slack->announce("Team {$team->name} created");

    return $team;
}
```

## Complete example: create a record, then dispatch a job

`RequestReport` records the request through a repository and then queues the slow work (collecting GitHub activity, calling the AI summariser) for a background job. The write and the dispatch are wrapped together so we never queue a job for a report that failed to persist — `DB::transaction` only fires the queued job after the surrounding transaction commits when you use `dispatch()` on a job that implements `ShouldQueue`, but to be explicit and safe we dispatch inside the transaction-aware boundary.

```php
declare(strict_types=1);

namespace App\Actions\Report;

use App\Jobs\GenerateReport;
use App\Models\Report;
use App\Models\Workspace;
use App\Repositories\ReportRepository;
use Illuminate\Support\Facades\DB;

final readonly class RequestReport
{
    public function __construct(
        private ReportRepository $reports,
    ) {}

    public function handle(Workspace $workspace, string $repository, int $days = 7): Report
    {
        $report = DB::transaction(
            fn (): Report => $this->reports->createPending($workspace, $repository, $days),
        );

        GenerateReport::dispatch($report)->afterCommit();

        return $report;
    }
}
```

Notes on this example:

- The Action returns the **pending** `Report` immediately, so a controller can redirect the user to a "your report is being generated" page.
- `GenerateReport::dispatch(...)->afterCommit()` ensures the job is only enqueued once the database transaction has committed — a worker can never pick up a `Report` row that hasn't been written yet. (Set `after_commit` globally on the queue connection to make this the default; calling `->afterCommit()` is explicit and safe regardless.)
- All the heavy lifting lives in the job, which itself can call further Actions and services. The Action's job is to *kick it off*, durably.

## Complete example: call an external service, then persist

`CaptureInstallation` handles a GitHub App installation webhook. It calls the external GitHub client to expand the installation into its repositories, then persists everything through a repository inside one transaction. This is the canonical "talk to the outside world, then write" shape.

```php
declare(strict_types=1);

namespace App\Actions\Team;

use App\Contracts\GitHubClient;
use App\Data\InstallationPayload;
use App\Models\Installation;
use App\Repositories\InstallationRepository;
use Illuminate\Support\Facades\DB;

final readonly class CaptureInstallation
{
    public function __construct(
        private GitHubClient $github,
        private InstallationRepository $installations,
    ) {}

    public function handle(InstallationPayload $payload): Installation
    {
        // 1. External call FIRST — outside any transaction. If GitHub is
        //    unreachable we fail before opening a database transaction.
        $repositories = $this->github->installationRepositories($payload->installationId);

        // 2. Persist atomically. The installation and its repositories are
        //    written together or not at all.
        return DB::transaction(
            fn (): Installation => $this->installations->store($payload, $repositories),
        );
    }
}
```

Why this ordering matters: a long external HTTP call should **never** happen while a database transaction is open, or you hold row locks for the duration of a network round-trip. Do the I/O first, then write quickly and atomically.

## Composing Actions

Actions may call other Actions and services. Keep the composition shallow — an Action that orchestrates a workflow reads as a short, linear list of steps. When a step is itself a reusable write, extract it into its own Action.

```php
declare(strict_types=1);

namespace App\Actions\Report;

use App\Actions\Slack\PostReportToSlack;
use App\Models\Report;
use App\Repositories\ReportRepository;

final readonly class PublishReport
{
    public function __construct(
        private ReportRepository $reports,
        private PostReportToSlack $postToSlack,
    ) {}

    public function handle(Report $report): Report
    {
        $published = $this->reports->markPublished($report);

        $this->postToSlack->handle($published);

        return $published;
    }
}
```

```php
// ❌ Avoid: inlining what is already a reusable Action.
// The Slack-posting logic belongs in PostReportToSlack, not copy-pasted here.
public function handle(Report $report): Report
{
    $published = $this->reports->markPublished($report);

    $blocks = (new SlackBlockBuilder())->forReport($published)->build();
    Http::post(config('digest.slack.webhook'), ['blocks' => $blocks]); // duplication

    return $published;
}
```

## Actions vs. Services

These two patterns are complementary, not interchangeable. The distinction is **once vs. reusable capability**.

| | Action | Service |
|---|---|---|
| Does | One thing, once, on demand | Provides a reusable capability |
| Public surface | One method, `handle()` | Whatever the capability needs |
| Name | Imperative verb (`CreateTeam`) | Noun / capability (`GitHubClient`, `SlackNotifier`) |
| Lifetime of intent | Per invocation | Long-lived collaborator |
| State | Stateless, `final readonly` | Often stateless, may hold config/client |
| Example | `RequestReport` | `AiClient`, `RestGitHubClient` |

A helpful test: if you can phrase it as a sentence the user could trigger ("connect this Slack channel", "create a team"), it is an **Action**. If it is a tool other code reaches for ("a client that talks to GitHub", "something that sends Slack messages"), it is a **Service**. Actions *use* services; services do not use Actions.

```php
// Action: a single user-triggered write.
final readonly class ConnectSlackChannel
{
    public function handle(Workspace $workspace, string $channelId): SlackChannel { /* ... */ }
}

// Service: a reusable capability the Action depends on.
final readonly class SlackNotifier
{
    public function send(string $channelId, array $blocks): void { /* ... */ }
}
```

## Testing an Action

Because an Action only depends on injected repositories and services, you test it by constructing it with fakes and asserting on the result and the recorded effects. No HTTP, no console kernel, no real database needed for the orchestration logic.

```php
declare(strict_types=1);

use App\Actions\Report\RequestReport;
use App\Jobs\GenerateReport;
use Illuminate\Support\Facades\Queue;

it('creates a pending report and queues generation', function (): void {
    Queue::fake();

    $reports = new FakeReportRepository();
    $action  = new RequestReport($reports);

    $workspace = Workspace::factory()->make();

    $report = $action->handle($workspace, 'NIMA-Labs/digest', days: 7);

    expect($report->status)->toBe('pending')
        ->and($report->days)->toBe(7);

    Queue::assertPushed(GenerateReport::class, fn (GenerateReport $job): bool =>
        $job->report->is($report),
    );
});
```

## See also

- [Services](services.md) — reusable capabilities and stateless collaborators that Actions depend on.
- [Repositories](repositories.md) — the persistence boundary every Action writes through instead of touching Eloquent directly.
