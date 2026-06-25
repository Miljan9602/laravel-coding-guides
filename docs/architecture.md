# Application layers & where logic goes

The single most consequential decision in a Laravel codebase is *where each piece of logic lives*. This page is the canonical map: it names every layer, gives it one job, and tells you exactly what it may and must never do — so that any engineer can place new code without guessing.

## Rules at a glance

- **Routes** declare URLs and bind them to a controller method. No logic.
- **Controllers** are HTTP glue only: resolve input, call one Action or Service, return a response. No business logic, no queries.
- **Form Requests** own validation *and* authorization for an HTTP request. Nothing else.
- **Actions** perform exactly one write or one orchestrated workflow. They are the verbs of the app.
- **Services** provide a reusable domain capability or wrap an external integration (Slack, GitHub). Stateless.
- **Repositories** are the *only* place that touches the database (or other persistence). Everything else asks them.
- **Models** describe schema, relations, casts, and scopes. No business logic, no external calls.
- **DTOs / Value Objects** carry immutable, typed data between layers. No behaviour beyond derived accessors.
- **Enums** are the canonical set of named states and the small logic that belongs to a state.
- **Livewire** holds transient UI state and *delegates* every real operation to an Action or Service.
- **Jobs** run work in the background; they carry **ids, not models**, and delegate.
- **Commands** are thin CLI entry points; they parse options and delegate.
- **Filament** is admin UI bound to models; keep domain rules in Actions/Services, not in resources.
- **Golden rule:** dependencies point *downward* and cross boundaries *through interfaces*. A Controller may depend on an Action; an Action never depends on a Controller.

---

## The golden rule: dependencies point downward, through interfaces

Layers form a one-way stack. Upper layers (HTTP, Livewire, Console, Jobs) depend on the layers below (Actions → Services → Repositories → Models/Database). A lower layer must **never** reach upward — a Repository knows nothing about HTTP; an Action knows nothing about Livewire.

Where a boundary is crossed, depend on an **interface (Contract)**, not a concrete class. This keeps the dependency graph acyclic, makes every collaborator fakeable in tests, and lets you swap an implementation (a REST GitHub client for a GraphQL one) without touching callers.

```
HTTP / Livewire / Console / Jobs   ← entry points (thin)
            │  delegates to
            ▼
         Actions                    ← one write / one workflow
            │  composes
            ▼
         Services                   ← reusable capability, integrations
            │  reads & writes via
            ▼
        Repositories  (Contracts)   ← all persistence access
            │
            ▼
     Models / Database / External APIs
```

```php
// ✅ Do: the entry point depends on an Action; the Action depends on a Contract.
final readonly class PublishReportController
{
    public function __construct(
        private PublishReport $publishReport,
    ) {}

    public function __invoke(PublishReportRequest $request, Workspace $workspace): RedirectResponse
    {
        $this->publishReport->handle($workspace, $request->toData());

        return to_route('reports.index', $workspace);
    }
}
```

```php
// ❌ Avoid: a Repository reaching back into HTTP/session — an upward dependency.
final class ReportRepository
{
    public function forCurrentUser(): Collection
    {
        // A persistence layer must never know about auth(), request(), or session().
        return Report::query()->where('user_id', auth()->id())->get();
    }
}
```

---

## Routes

**One job:** map a URL + HTTP verb to exactly one controller action, with the right middleware.

**MAY:** group by middleware/prefix; name routes; bind route-model parameters; reference a single-action controller via `__invoke`.

**NEVER:** contain closures with logic, run queries, or transform data. A route file is a table of contents, not a chapter.

```php
// ✅ Do: routes/web.php — declarative, points at controllers.
Route::middleware(['auth', 'verified'])->group(function (): void {
    Route::get('/workspaces/{workspace}/reports', ListReportsController::class)
        ->name('reports.index');

    Route::post('/workspaces/{workspace}/reports', PublishReportController::class)
        ->name('reports.store');
});
```

```php
// ❌ Avoid: business logic stuffed into a route closure.
Route::post('/reports', function (Request $request) {
    $report = Report::create($request->all());      // validation? authorization? persistence here?
    Http::post(config('digest.slack.webhook'), [...]); // and an external call too?
    return redirect()->back();
});
```

---

## Controllers — HTTP glue only

**One job:** translate an HTTP request into a domain call, and a domain result into an HTTP response.

**MAY:** type-hint a Form Request for validation/authorization; resolve route-model bindings; call **one** Action or Service; choose the response (view, redirect, JSON, status code).

**NEVER:** validate manually, authorize manually, run queries, build emails, call third-party APIs, or contain conditional business rules. If a controller method is longer than a handful of lines, logic has leaked in.

Prefer **single-action controllers** (`__invoke`) — they make the one-route-one-job rule physical.

```php
// ✅ Do: thin, single responsibility, delegates once.
declare(strict_types=1);

namespace App\Http\Controllers\Reports;

final readonly class PublishReportController
{
    public function __construct(
        private PublishReport $publishReport,
    ) {}

    public function __invoke(PublishReportRequest $request, Workspace $workspace): RedirectResponse
    {
        $report = $this->publishReport->handle($workspace, $request->toData());

        return to_route('reports.show', [$workspace, $report])
            ->with('status', 'Report published.');
    }
}
```

```php
// ❌ Avoid: the controller doing validation, queries, formatting and integration itself.
final class PublishReportController extends Controller
{
    public function store(Request $request, Workspace $workspace): RedirectResponse
    {
        $request->validate(['title' => 'required']);                 // belongs in a Form Request

        $report = $workspace->reports()->create($request->only('title')); // belongs in a Repository

        Http::post(config('digest.slack.webhook'), [                  // belongs in a Service
            'text' => "Published {$report->title}",
        ]);

        return back();
    }
}
```

---

## Form Requests — validation + authorization

**One job:** decide whether an incoming HTTP request is *allowed* and *well-formed*, then hand the controller clean data.

**MAY:** declare `rules()`, `authorize()`, custom messages, `prepareForValidation()` normalisation, and a typed `toData()` accessor that returns a DTO of validated input.

**NEVER:** persist anything, dispatch jobs, call services, or contain workflow. A Form Request is a gate, not a worker.

Authorization belongs here (not in the controller) because it is part of *accepting the request*. Use Policies for model-level rules and call them from `authorize()`.

```php
// ✅ Do: validation + authorization + a typed handoff to the domain.
declare(strict_types=1);

namespace App\Http\Requests\Reports;

final class PublishReportRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('publish', $this->route('workspace'));
    }

    /** @return array<string, list<string>> */
    public function rules(): array
    {
        return [
            'title'        => ['required', 'string', 'max:160'],
            'period_days'  => ['required', 'integer', 'between:1,30'],
            'channel'      => ['required', 'string', 'starts_with:#'],
        ];
    }

    public function toData(): PublishReportData
    {
        return new PublishReportData(
            title: $this->string('title')->value(),
            periodDays: $this->integer('period_days'),
            channel: $this->string('channel')->value(),
        );
    }
}
```

```php
// ❌ Avoid: a Form Request that does work.
final class PublishReportRequest extends FormRequest
{
    public function rules(): array
    {
        // Side effects during validation: forbidden.
        Report::create([...]);
        SlackNotifier::send('...');

        return ['title' => ['required']];
    }
}
```

---

## Actions — one write or one orchestrated workflow

**One job:** perform a single business operation end-to-end. An Action is a *verb*: `PublishReport`, `InviteMember`, `SyncWorkspaceRepositories`.

**MAY:** open a database transaction; call several Repositories and Services; dispatch Jobs and events; enforce invariants that span multiple models; return a DTO or a Model.

**NEVER:** know about HTTP (no `Request`, no `redirect()`), render views, or read superglobals. An Action is callable identically from a controller, a command, a job, or a test.

An Action is the natural transaction boundary. If an operation must be atomic, the `DB::transaction()` lives here — not in a controller, not in a repository.

```php
// ✅ Do: a stateless action that orchestrates one workflow atomically.
declare(strict_types=1);

namespace App\Actions\Reports;

final readonly class PublishReport
{
    public function __construct(
        private ReportRepository $reports,
        private SlackNotifier $slack,
        private GitHubActivityService $activity,
    ) {}

    public function handle(Workspace $workspace, PublishReportData $data): Report
    {
        return DB::transaction(function () use ($workspace, $data): Report {
            $summary = $this->activity->summariseLastDays($workspace, $data->periodDays);

            $report = $this->reports->create($workspace, [
                'title'   => $data->title,
                'summary' => $summary,
                'status'  => ReportStatus::Published,
            ]);

            $this->slack->postReport($data->channel, $report);

            ReportPublished::dispatch($report);

            return $report;
        });
    }
}
```

```php
// ❌ Avoid: an action that depends on HTTP and renders a response.
final class PublishReport
{
    public function handle(Request $request): View   // HTTP leaking into the domain
    {
        $report = Report::create($request->all());
        return view('reports.show', compact('report')); // an action must not render
    }
}
```

> **One Action vs. a Service?** Reach for an **Action** when the thing happens *once* as a discrete write/workflow ("publish this report"). Reach for a **Service** when the capability is *reused* across many actions ("talk to GitHub", "summarise activity"). Actions tend to be unique; Services tend to be shared.

---

## Services — reusable domain capability / external integration

**One job:** encapsulate a capability that is reused across the app, or wrap a single external system behind a clean, mockable surface.

**MAY:** make HTTP calls to third parties (GitHub, Slack, an AI provider); perform pure domain calculations; expose methods through a **Contract** so callers depend on the interface; hold injected configuration.

**NEVER:** carry mutable per-request state (services are stateless and resolved from the container), depend on HTTP request objects, or own persistence — services that need data ask a Repository.

External integrations especially must sit behind an interface so tests fake them and you can swap providers.

```php
// ✅ Do: an integration behind a Contract, stateless, config injected — not env().
declare(strict_types=1);

namespace App\Services\GitHub;

final readonly class RestGitHubClient implements GitHubClient
{
    public function __construct(
        private PendingRequest $http,
    ) {}

    /** @return Collection<int, PullRequestData> */
    public function recentPullRequests(string $repository, CarbonImmutable $since): Collection
    {
        $response = $this->http
            ->withToken(config('digest.github.token'))
            ->get("repos/{$repository}/pulls", [
                'state' => 'closed',
                'sort'  => 'updated',
            ])
            ->throw();

        return collect($response->json())
            ->filter(fn (array $pr): bool => CarbonImmutable::parse($pr['merged_at'] ?? null)?->gte($since) ?? false)
            ->map(PullRequestData::fromApi(...))
            ->values();
    }
}
```

```php
// ❌ Avoid: a stateful service reading env() and persisting directly.
final class GitHubService
{
    private array $cache = [];                 // per-request mutable state in a singleton: a footgun

    public function pulls(string $repo): array
    {
        $token = env('GITHUB_TOKEN');          // env() outside config: returns null once cached
        $rows = Http::withToken($token)->get(...)->json();
        PullRequest::insert($rows);            // a service must not own persistence
        return $rows;
    }
}
```

---

## Repositories — all data access

**One job:** be the *only* layer that reads from and writes to persistence. Every query lives behind a Repository method with a meaningful name.

**MAY:** build Eloquent queries, apply scopes, eager-load, paginate, write/update/delete, and return Models or Collections of Models. Implement a **Contract** in `Contracts/Repositories`.

**NEVER:** call third-party APIs, read the request/session/auth, format output for the UI, or contain multi-step business workflow. A Repository answers "give me / store this data", nothing more.

Centralising queries means a tuned index or a soft-delete rule changes in one place, and tests can bind a fake repository instead of hitting a database.

```php
// ✅ Do: a focused repository implementing a Contract.
declare(strict_types=1);

namespace App\Repositories;

final readonly class EloquentReportRepository implements ReportRepository
{
    /** @return Collection<int, Report> */
    public function publishedFor(Workspace $workspace, int $limit = 20): Collection
    {
        return Report::query()
            ->where('workspace_id', $workspace->id)
            ->where('status', ReportStatus::Published)
            ->latest('published_at')
            ->limit($limit)
            ->get();
    }

    /** @param array<string, mixed> $attributes */
    public function create(Workspace $workspace, array $attributes): Report
    {
        return $workspace->reports()->create($attributes);
    }
}
```

```php
// ❌ Avoid: a repository that strays out of its lane.
final class ReportRepository
{
    public function publishAndAnnounce(array $data): Report
    {
        $report = Report::create($data);
        Http::post(config('digest.slack.webhook'), [...]); // external call: belongs in a Service
        return $report->load('author')->append('formatted_summary'); // formatting: belongs in the view/DTO
    }
}
```

The Contract lives apart from the implementation so callers bind to the abstraction:

```php
// app/Contracts/Repositories/ReportRepository.php
declare(strict_types=1);

namespace App\Contracts\Repositories;

interface ReportRepository
{
    /** @return Collection<int, Report> */
    public function publishedFor(Workspace $workspace, int $limit = 20): Collection;

    /** @param array<string, mixed> $attributes */
    public function create(Workspace $workspace, array $attributes): Report;
}
```

---

## Models — schema, relations, casts (no business logic)

**One job:** describe a database table to Eloquent — its fillable columns, relationships, casts, and reusable query scopes.

**MAY:** declare `$fillable`/`$guarded`, `casts()`, relationship methods, query scopes, accessors/mutators for *attribute shaping*, and factory hooks.

**NEVER:** orchestrate workflows, call Services or external APIs, dispatch jobs from arbitrary methods, or hold branching business rules. A "fat model" that publishes reports and posts to Slack has swallowed the Action and Service layers.

Keep models declarative. Behaviour that *coordinates* belongs in an Action; behaviour that is *reused capability* belongs in a Service.

```php
// ✅ Do: a declarative model — relations, casts, scopes, no orchestration.
declare(strict_types=1);

namespace App\Models;

final class Report extends Model
{
    /** @var list<string> */
    protected $fillable = ['title', 'summary', 'status', 'published_at'];

    /** @return array<string, string> */
    protected function casts(): array
    {
        return [
            'status'       => ReportStatus::class,
            'published_at' => 'immutable_datetime',
        ];
    }

    public function workspace(): BelongsTo
    {
        return $this->belongsTo(Workspace::class);
    }

    #[Scope]
    protected function published(Builder $query): void
    {
        $query->where('status', ReportStatus::Published);
    }
}
```

```php
// ❌ Avoid: a model that runs the business.
final class Report extends Model
{
    public function publish(string $channel): void
    {
        $this->update(['status' => 'published']);
        app(SlackNotifier::class)->postReport($channel, $this); // orchestration in the model
        ReportPublished::dispatch($this);                       // workflow leaking into ActiveRecord
    }
}
```

---

## DTOs / Value Objects — immutable data between layers

**One job:** move *typed, immutable* data across a boundary (request → action, collector → aggregator) without leaking framework objects.

**MAY:** be a `final readonly class` with promoted properties; expose named constructors (`fromApi`, `fromRequest`); provide derived, side-effect-free accessors. A Value Object additionally enforces its own validity in the constructor (a `Channel` that rejects a name without `#`).

**NEVER:** be mutable, run queries, perform I/O, or depend on the container. A DTO is data with a shape, not a worker.

DTOs make signatures honest: `handle(PublishReportData $data)` tells you exactly what is required, and the compiler enforces it.

```php
// ✅ Do: an immutable DTO with a typed named constructor.
declare(strict_types=1);

namespace App\Data;

final readonly class PullRequestData
{
    public function __construct(
        public int $number,
        public string $title,
        public string $author,
        public CarbonImmutable $mergedAt,
    ) {}

    /** @param array<string, mixed> $payload */
    public static function fromApi(array $payload): self
    {
        return new self(
            number: (int) $payload['number'],
            title: (string) $payload['title'],
            author: (string) $payload['user']['login'],
            mergedAt: CarbonImmutable::parse($payload['merged_at']),
        );
    }
}
```

```php
// ✅ Do: a self-validating Value Object.
final readonly class SlackChannel
{
    public function __construct(public string $value)
    {
        if (! str_starts_with($value, '#')) {
            throw new InvalidArgumentException("Slack channel must start with '#', got [{$value}].");
        }
    }
}
```

```php
// ❌ Avoid: a mutable "DTO" that fetches its own data.
final class ReportData
{
    public string $title;                       // public mutable state
    public function load(int $id): void
    {
        $this->title = Report::find($id)->title; // a data carrier performing a query
    }
}
```

---

## Enums

**One job:** be the canonical, type-safe set of named states or kinds (`ReportStatus`, `ContributionType`), plus the *small* logic intrinsic to a value.

**MAY:** be a backed enum; add helper methods (`label()`, `color()`, `isTerminal()`); implement interfaces Laravel/Filament expect (`HasLabel`, `HasColor`). Cast model columns to them.

**NEVER:** carry per-instance mutable state, perform I/O, or grow into a service. Logic that *uses many enum values to make a decision* is workflow — that lives in an Action.

```php
// ✅ Do: a backed enum that owns its own presentation logic.
declare(strict_types=1);

namespace App\Enums;

enum ReportStatus: string
{
    case Draft     = 'draft';
    case Published = 'published';
    case Archived  = 'archived';

    public function label(): string
    {
        return ucfirst($this->value);
    }

    public function isTerminal(): bool
    {
        return $this === self::Archived;
    }
}
```

```php
// ❌ Avoid: an enum doing I/O or holding state.
enum ReportStatus: string
{
    case Published = 'published';

    public function notifySlack(Report $report): void   // I/O does not belong on an enum
    {
        app(SlackNotifier::class)->postReport('#general', $report);
    }
}
```

---

## Parallel surfaces

These are alternative *entry points* into the same domain. They live beside controllers, not below them, and they all obey the same rule: **stay thin, delegate down.**

### Livewire — UI state + delegate

**One job:** hold the transient state of a piece of UI and react to user events by delegating to an Action or Service.

**MAY:** keep public properties for bound inputs; validate form input (Livewire validation mirrors Form Request rules); call an Action on a method; emit events; manage loading/UI flags.

**NEVER:** contain the business workflow itself or query the database directly for writes. A Livewire `save()` should read like a controller action: validate, call one Action, reflect the result.

```php
// ✅ Do: component holds UI state, delegates the write to an Action.
declare(strict_types=1);

namespace App\Livewire\Reports;

final class PublishReportForm extends Component
{
    public string $title = '';
    public int $periodDays = 7;
    public string $channel = '#standup';

    public function save(PublishReport $publishReport, Workspace $workspace): void
    {
        $this->validate([
            'title'      => ['required', 'string', 'max:160'],
            'periodDays' => ['required', 'integer', 'between:1,30'],
            'channel'    => ['required', 'string', 'starts_with:#'],
        ]);

        $publishReport->handle($workspace, new PublishReportData(
            title: $this->title,
            periodDays: $this->periodDays,
            channel: $this->channel,
        ));

        $this->dispatch('report-published');
    }

    public function render(): View
    {
        return view('livewire.reports.publish-report-form');
    }
}
```

### Jobs — background work, carry ids not models

**One job:** run an operation off the request cycle, then delegate to the same Action/Service the synchronous path would have used.

**MAY:** be queued; carry **scalar identifiers** in their constructor; re-resolve the model and the Action inside `handle()`; define `tries`/`backoff`/`uniqueId`.

**NEVER:** serialise a whole Eloquent model with loaded relations into the payload (it goes stale and bloats the queue), or duplicate workflow that already exists in an Action.

```php
// ✅ Do: carry an id, re-resolve, delegate.
declare(strict_types=1);

namespace App\Jobs;

final class GenerateScheduledReport implements ShouldQueue
{
    use Queueable;

    public function __construct(
        private readonly int $workspaceId,
        private readonly int $periodDays,
    ) {}

    public function handle(PublishReport $publishReport, WorkspaceRepository $workspaces): void
    {
        $workspace = $workspaces->findOrFail($this->workspaceId);

        $publishReport->handle($workspace, new PublishReportData(
            title: "Weekly digest — {$workspace->name}",
            periodDays: $this->periodDays,
            channel: $workspace->defaultChannel,
        ));
    }
}
```

```php
// ❌ Avoid: passing a hydrated model and re-implementing the workflow in the job.
final class GenerateScheduledReport implements ShouldQueue
{
    public function __construct(private readonly Report $report) {} // goes stale on the queue

    public function handle(): void
    {
        $this->report->update(['status' => 'published']);          // duplicated workflow
        Http::post(config('digest.slack.webhook'), [...]);
    }
}
```

### Commands — CLI entry points, thin

**One job:** parse CLI arguments/options and hand off to an Action or Service. A console command is a controller for the terminal.

**MAY:** declare `signature`/`description`; read options; support `--dry-run`; print progress; return an exit code.

**NEVER:** hold the workflow. The body of `handle()` should mostly be option parsing plus one delegated call.

```php
// ✅ Do: thin command that delegates.
declare(strict_types=1);

namespace App\Console\Commands;

final class SendDigestCommand extends Command
{
    protected $signature = 'digest:send {--days=7} {--dry-run}';
    protected $description = 'Generate and deliver the weekly digest.';

    public function handle(RunReport $runReport): int
    {
        $report = $runReport->handle(
            type: ReportType::Digest,
            periodDays: (int) $this->option('days'),
            dryRun: (bool) $this->option('dry-run'),
        );

        $this->info("Digest covered {$report->contributors->count()} contributors.");

        return self::SUCCESS;
    }
}
```

### Filament — admin UI, framework-bound to models

**One job:** provide CRUD/admin screens over your models. Filament Resources are intentionally coupled to Eloquent.

**MAY:** define forms, tables, infolists, filters, and Actions on Resources; reuse your Enums (`HasLabel`/`HasColor`) and Form Request-equivalent validation rules.

**NEVER:** become the home of domain workflow. When a Filament action does more than a simple write, have it call your **Action** class, exactly like a controller would — so the admin path and the app path run identical logic.

```php
// ✅ Do: a Filament action delegating to the domain Action.
Action::make('publish')
    ->requiresConfirmation()
    ->action(function (Report $record, PublishReport $publishReport): void {
        $publishReport->handle($record->workspace, PublishReportData::fromModel($record));
    });
```

---

## Where does this logic go?

| Concern | Belongs in | Never in |
| --- | --- | --- |
| Validating request shape & rules | **Form Request** (or Livewire `validate()`) | Controller, Action, Model |
| Authorizing the request/actor | **Form Request `authorize()` / Policy** | Repository, Model |
| A single DB write or read | **Repository** | Controller, Service, Job |
| An HTTP call to a third party (Slack, GitHub, AI) | **Service** (behind a Contract) | Controller, Repository, Model |
| A multi-step workflow / transaction | **Action** | Controller, Model, Job body |
| Coordinating several Services + Repositories | **Action** | Controller, Livewire |
| Reusable domain calculation used in many places | **Service** | Model, Controller |
| Formatting data for display | **DTO accessor / Blade view / View Model** | Repository, Action |
| Moving typed data between layers | **DTO / Value Object** | Arrays passed around |
| A fixed set of named states + tiny per-state logic | **Enum** | Strings/consts, Service |
| Choosing the HTTP response (view/redirect/JSON) | **Controller** | Action, Service |
| Transient form/UI state | **Livewire component** | Action, Model |
| Deferring work to the background | **Job** (carrying ids) → delegates to Action | Controller inline |
| Parsing CLI options | **Command** → delegates to Action | Action, Service |

When in doubt, ask: *"Is this a single write/workflow (Action), a reusable capability/integration (Service), or a query (Repository)?"* Almost everything resolves to one of those three.

---

## Recommended `app/` directory layout

Group by **domain inside each layer** so a feature's pieces are easy to find and the structure scales past a handful of models.

```
app/
├── Actions/
│   ├── Reports/
│   │   ├── PublishReport.php
│   │   └── ArchiveReport.php
│   └── Workspaces/
│       └── SyncWorkspaceRepositories.php
├── Services/
│   ├── GitHub/
│   │   └── RestGitHubClient.php
│   ├── Slack/
│   │   └── SlackNotifier.php
│   └── Ai/
│       └── AiClient.php
├── Repositories/
│   ├── EloquentReportRepository.php
│   └── EloquentWorkspaceRepository.php
├── Contracts/
│   ├── Repositories/
│   │   ├── ReportRepository.php
│   │   └── WorkspaceRepository.php
│   └── GitHubClient.php
├── Http/
│   ├── Controllers/
│   │   └── Reports/
│   │       └── PublishReportController.php
│   └── Requests/
│       └── Reports/
│           └── PublishReportRequest.php
├── Data/
│   ├── PublishReportData.php
│   └── PullRequestData.php
├── Enums/
│   ├── ReportStatus.php
│   └── ContributionType.php
├── Livewire/
│   └── Reports/
│       └── PublishReportForm.php
├── Jobs/
│   └── GenerateScheduledReport.php
├── Models/
│   ├── Report.php
│   └── Workspace.php
└── Console/
    └── Commands/
        └── SendDigestCommand.php
```

Bind every Contract to its implementation in a service provider, so the entire app depends on interfaces and tests swap fakes in one place:

```php
// app/Providers/AppServiceProvider.php
public function register(): void
{
    $this->app->bind(ReportRepository::class, EloquentReportRepository::class);
    $this->app->bind(WorkspaceRepository::class, EloquentWorkspaceRepository::class);
    $this->app->bind(GitHubClient::class, RestGitHubClient::class);
}
```

---

## See also

- [Controllers](controllers.md)
- [Form Requests](form-requests.md)
- [Actions](actions.md)
- [Services](services.md)
- [Repositories](repositories.md)
- [Models](models-eloquent.md)
- [DTOs & Value Objects](dtos-value-objects.md)
- [Enums](enums.md)
- [Livewire](livewire.md)
- [Jobs & Queues](jobs-commands-scheduling.md)
- [Console Commands](jobs-commands-scheduling.md)
- [Filament](filament.md)
- [Dependency Injection & Service Container](services.md)
- [Testing](testing.md)
