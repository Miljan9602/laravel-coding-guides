# Philosophy & how to use this guide

This is the canonical Laravel coding-standards reference for our projects (PHP 8.4, Laravel 13+, Pest, Pint, Livewire 3/4, Filament). It exists so that any engineer, on any project, makes the same structural and stylistic decisions without re-litigating them — code that looks like one person wrote it, even across years and teams.

## Rules at a glance

- **Standards are not taste.** They exist to make code readable, consistent, testable, and cheap to change — and to make bugs and review friction rare. Follow them even when you'd personally choose otherwise.
- **The layered architecture is the spine.** Every request flows the same way: Route → Controller → Form Request → Action → Service → Repository → Eloquent/DB. Livewire components, queued Jobs, and Console Commands are parallel entry points that drop into the *same* layers.
- **Thin HTTP layer.** Controllers, Livewire components, and commands are glue. They never contain business logic, never touch the database directly, and never reach for `env()`.
- **Repositories own all data access.** Eloquent models are touched in exactly one place. No `Workspace::where(...)` outside a repository — ever.
- **Actions own writes and orchestration.** One action does one meaningful thing (publish a report, invite a member) and is the unit of a use case.
- **Services own reusable capabilities and integrations.** Slack, GitHub, PDF rendering — stateless, injectable, mockable.
- **Types everywhere.** `declare(strict_types=1)` in every file, typed params and return types on every method, no untyped `array` blobs where a DTO belongs.
- **Fail loudly.** Throw on the unexpected. No silent `null`, no empty `catch`, no `@` error suppression.
- **SOLID and DRY are the defaults, not aspirations.** Small classes, single responsibilities, one source of truth for each rule.
- **Tests are part of "done."** A feature without tests is not finished, regardless of whether it works on your machine.
- **The mechanical parts are enforced.** Pint formats; CI fails on style, static analysis, and red tests. Don't argue with the linter — fix the code.

## Why coding standards exist

Standards are a force multiplier for everything other than the first ten minutes of writing a line of code. The line is read, reviewed, debugged, and changed dozens of times over its life; it is written once. We optimise for the long tail.

### Readability

Code is read far more than it is written. A consistent shape — same file layout, same naming, same place to look for "where does the database get touched" — lets a reader load a class into their head in seconds instead of minutes. When every `Action` has a `handle()` method and every repository method returns a DTO or a model, you stop *parsing* and start *understanding*.

### Consistency

Consistency removes a class of decisions entirely. If the rule is "repositories own all data access," nobody has to wonder, in code review, whether a `Workspace::find()` in a controller is acceptable this one time. It isn't. The decision was made once, here, and never has to be made again. This is the single biggest lever standards pull: they convert recurring judgment calls into settled facts.

### Testability

The layering exists largely so that things can be tested in isolation. A thin controller delegating to an action means you test the action directly — no HTTP kernel, no middleware. A service behind an interface means you swap in a fake in tests. Code that's hard to test is almost always code that violated a boundary; the friction is the standard warning you.

### Fewer bugs

Strict types, explicit return types, and "fail loudly" turn whole categories of bug into either a fatal type error at the boundary or a static-analysis failure in CI — found in seconds, not in production. A repository that returns `?Report` forces the caller to handle the miss; a method that silently returns `null` from three different code paths hides it.

### Faster review

When the structure is predictable, review is about *intent and correctness*, not *style and placement*. Reviewers don't spend their budget asking you to move logic out of a controller or to add a return type — Pint and CI already did that. They spend it on the thing that actually matters: is this the right behaviour?

### Safe change

The whole point of boundaries is that you can change one side without touching the other. Swap the GitHub REST client for GraphQL behind the same `GitHubClient` contract and no action changes. Move from one Slack delivery mechanism to another behind `SlackNotifier` and no report-generation code changes. Standards make large refactors *local*.

## The layered architecture is the spine

Every standard in this guide ultimately serves one structure. Learn this picture and most other rules become obvious consequences of it.

```text
                         ┌─────────────────────────────────────────┐
   ENTRY POINTS          │                                         │
                         ▼                                         │
   HTTP Route ──▶ Controller ──▶ Form Request                      │
   (web.php /     (HTTP glue,    (validation +                     │
    api.php)       no logic)      authorization)                   │
                         │                                         │
   Livewire ────────────┤                                         │
   Component             │                                         │
                         ▼                                         │
   Queued Job ──────▶ Action ◀────────────────────────────────────┘
                     (ONE write / orchestration of a use case)
                         │
   Console ─────────────┘
   Command                  │
                            ▼
                         Service ──────▶ external integration
                     (reusable capability:   (Slack, GitHub,
                      stateless, injectable)   PDF, mailer)
                            │
                            ▼
                        Repository
                  (the ONLY place models are touched)
                            │
                            ▼
                      Eloquent / DB
```

Read the diagram top to bottom and one rule jumps out: **data flows down through layers, and each layer only knows about the layer directly below it.** A controller knows about an action; it does not know about a repository. An action knows about services and repositories; it does not know about HTTP. A repository knows about Eloquent; nothing below it knows it exists.

### Route → Controller (HTTP glue)

The controller's entire job is to translate between HTTP and the application. It receives a validated request, calls exactly one action (or, for trivial reads, one repository/query), and returns a response. It contains no `if`-laden business rules and no database calls.

```php
// ✅ Do — thin controller: validate (via Form Request), delegate, respond.
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Actions\Reports\PublishReport;
use App\Http\Requests\PublishReportRequest;
use App\Http\Resources\ReportResource;
use Illuminate\Http\JsonResponse;

final class ReportController
{
    public function store(PublishReportRequest $request, PublishReport $publishReport): JsonResponse
    {
        $report = $publishReport->handle(
            $request->toData(),
        );

        return ReportResource::make($report)
            ->response()
            ->setStatusCode(201);
    }
}
```

```php
// ❌ Avoid — business logic, validation, and data access stuffed into the controller.
final class ReportController
{
    public function store(Request $request): JsonResponse
    {
        if (! $request->filled('workspace_id')) {
            return response()->json(['error' => 'missing workspace'], 422);
        }

        $workspace = Workspace::findOrFail($request->input('workspace_id')); // data access in HTTP layer
        $report = new Report();
        $report->workspace_id = $workspace->id;
        $report->title = $request->input('title');
        $report->save();                                                     // write in HTTP layer

        SlackWebhook::post('#reports', "Published {$report->title}");        // integration in HTTP layer

        return response()->json($report, 201);
    }
}
```

### Form Request (validation / authorization)

Validation rules and authorization belong in a Form Request, not in the controller body. This keeps the controller a one-liner and makes the contract of the endpoint explicit and reusable. Expose a typed accessor so the action receives a DTO, not a loose array.

```php
// ✅ Do — rules and authz live here; hand the action a typed DTO.
declare(strict_types=1);

namespace App\Http\Requests;

use App\Data\PublishReportData;
use Illuminate\Foundation\Http\FormRequest;

final class PublishReportRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('publish', $this->route('workspace'));
    }

    /**
     * @return array<string, array<int, string>>
     */
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:200'],
            'period_days' => ['required', 'integer', 'min:1', 'max:30'],
        ];
    }

    public function toData(): PublishReportData
    {
        return new PublishReportData(
            workspaceId: (int) $this->route('workspace')->id,
            title: $this->string('title')->toString(),
            periodDays: $this->integer('period_days'),
        );
    }
}
```

### Action (one write / one use case)

An action represents a single unit of work that changes the world or orchestrates a use case end to end: *publish a report*, *invite a member*, *archive a workspace*. It is the seam between "the request" and "the domain." It composes services and repositories; it does not talk to HTTP and does not touch Eloquent directly.

```php
// ✅ Do — a stateless action that orchestrates a use case.
declare(strict_types=1);

namespace App\Actions\Reports;

use App\Data\PublishReportData;
use App\Models\Report;
use App\Repositories\ReportRepository;
use App\Services\Reports\ReportGenerator;
use App\Services\Slack\SlackNotifier;
use Illuminate\Support\Facades\DB;

final readonly class PublishReport
{
    public function __construct(
        private ReportGenerator $generator,
        private ReportRepository $reports,
        private SlackNotifier $slack,
    ) {}

    public function handle(PublishReportData $data): Report
    {
        $report = DB::transaction(function () use ($data): Report {
            $generated = $this->generator->forWorkspace($data->workspaceId, $data->periodDays);

            return $this->reports->create($data->workspaceId, $data->title, $generated);
        });

        $this->slack->reportPublished($report);

        return $report;
    }
}
```

One action, one public verb (`execute`). If you find yourself adding a second unrelated public method, that's a second action.

### Service (reusable capability / integration)

A service is a stateless, reusable capability — usually wrapping an integration (Slack, GitHub, a mailer, a PDF renderer) or a cross-cutting domain calculation. Services sit behind interfaces so they can be swapped and faked. They are *not* tied to a single use case; many actions may use the same service.

```php
// ✅ Do — capability behind a contract, injectable and fakeable.
declare(strict_types=1);

namespace App\Services\Slack;

use App\Contracts\SlackNotifier as SlackNotifierContract;
use App\Contracts\SlackTransport;
use App\Models\Report;
use App\Services\Slack\SlackBlockBuilder;

final readonly class SlackNotifier implements SlackNotifierContract
{
    public function __construct(
        private SlackTransport $transport,
        private SlackBlockBuilder $blocks,
    ) {}

    public function reportPublished(Report $report): void
    {
        $this->transport->send(
            $this->blocks->forReport($report),
        );
    }
}
```

### Repository (the only place models are touched)

This is the hardest rule and the most valuable. **Eloquent models are referenced in exactly one layer: repositories.** No `Report::query()`, no `Workspace::find()`, no `->save()` anywhere else in the codebase. Everything that reads or writes the database goes through a repository method with an explicit signature and return type.

Why so strict? Because the day you need to add caching, change a query, soft-delete instead of hard-delete, or audit every write, you want one file to change — not a grep across the whole app. The repository is the firewall between your domain and Eloquent.

```php
// ✅ Do — all model access concentrated here, with typed methods.
declare(strict_types=1);

namespace App\Repositories;

use App\Data\GeneratedReport;
use App\Models\Report;
use Illuminate\Support\Collection;

final readonly class ReportRepository
{
    public function create(int $workspaceId, string $title, GeneratedReport $generated): Report
    {
        return Report::query()->create([
            'workspace_id' => $workspaceId,
            'title' => $title,
            'body' => $generated->markdown,
            'published_at' => now(),
        ]);
    }

    public function findForWorkspace(int $workspaceId, int $reportId): ?Report
    {
        return Report::query()
            ->where('workspace_id', $workspaceId)
            ->find($reportId);
    }

    /**
     * @return Collection<int, Report>
     */
    public function recentForWorkspace(int $workspaceId, int $limit = 10): Collection
    {
        return Report::query()
            ->where('workspace_id', $workspaceId)
            ->latest('published_at')
            ->limit($limit)
            ->get();
    }
}
```

```php
// ❌ Avoid — querying models from an action (or controller, or component).
final readonly class PublishReport
{
    public function handle(PublishReportData $data): Report
    {
        // Eloquent has leaked out of the repository layer.
        $existing = Report::where('workspace_id', $data->workspaceId)
            ->where('title', $data->title)
            ->first();

        // ...
    }
}
```

### Parallel entry points: Livewire, Jobs, Commands

The HTTP route is not special. A Livewire component, a queued job, and a console command are *also* entry points, and they obey the exact same rule: be thin, validate, delegate to an action, return. They never skip layers just because they aren't HTTP.

```php
// ✅ Do — a Livewire component delegating to the same action an HTTP controller would.
declare(strict_types=1);

namespace App\Livewire\Reports;

use App\Actions\Reports\PublishReport;
use App\Data\PublishReportData;
use App\Models\Workspace;
use Livewire\Attributes\Validate;
use Livewire\Component;

final class PublishReportForm extends Component
{
    public Workspace $workspace;

    #[Validate('required|string|max:200')]
    public string $title = '';

    #[Validate('required|integer|min:1|max:30')]
    public int $periodDays = 7;

    public function save(PublishReport $publishReport): void
    {
        $this->authorize('publish', $this->workspace);
        $this->validate();

        $publishReport->handle(new PublishReportData(
            workspaceId: $this->workspace->id,
            title: $this->title,
            periodDays: $this->periodDays,
        ));

        $this->dispatch('report-published');
    }
}
```

```php
// ✅ Do — a console command as a thin entry point over the same action.
declare(strict_types=1);

namespace App\Console\Commands;

use App\Actions\Reports\PublishReport;
use App\Data\PublishReportData;
use App\Repositories\WorkspaceRepository;
use Illuminate\Console\Command;

final class PublishReportCommand extends Command
{
    protected $signature = 'report:publish {workspace} {--days=7}';

    protected $description = 'Generate and publish a report for a workspace.';

    public function handle(PublishReport $publishReport, WorkspaceRepository $workspaces): int
    {
        $workspace = $workspaces->findBySlug($this->argument('workspace'));

        if ($workspace === null) {
            $this->error('Unknown workspace.');

            return self::FAILURE;
        }

        $publishReport->handle(new PublishReportData(
            workspaceId: $workspace->id,
            title: "Digest for {$workspace->name}",
            periodDays: (int) $this->option('days'),
        ));

        return self::SUCCESS;
    }
}
```

Three different doors — HTTP, Livewire, CLI — and behind each one the same `PublishReport::handle()`. That is the architecture working. If a behaviour needs to exist in two entry points, it lives in an action, not copy-pasted into both.

## The non-negotiable core principles

These are not negotiable per project or per reviewer. They are the constitution; the rest of the guide is the statute book.

### SOLID

Each class has a single reason to change, depends on abstractions rather than concretions, and can be extended without being modified. In practice: small classes, one responsibility each, interfaces for services you'll fake or swap, and constructor injection instead of facades reached for at random. See [solid.md](solid.md).

### DRY

Each piece of knowledge — a validation rule, a business calculation, a config value — has one authoritative home. Duplication is a liability the moment the two copies disagree. But DRY means *don't repeat knowledge*, not *don't repeat characters*; resist collapsing two things that merely look alike today. See [dry.md](dry.md).

### Fail loudly

The application should crash, throw, or fail CI the instant something is wrong — never limp along with a silent `null`, a swallowed exception, or a `@`-suppressed warning. A loud failure in development is a bug found; a quiet one is a bug shipped. Throw domain exceptions, let type errors surface, and never write an empty `catch`.

### Types everywhere

`declare(strict_types=1)` at the top of every file. Every parameter, property, and return value is typed. Loose `array` payloads that cross layer boundaries become DTOs / value objects so their shape is enforced by the language, not by hope. This is what makes static analysis and refactoring safe.

### Thin HTTP layer

Controllers, Livewire components, and commands translate between the outside world and the application — nothing more. No business rules, no data access, no `env()`. If a controller method is longer than a few lines, logic has leaked into it and belongs in an action.

### Repositories own all data access

Eloquent is touched in repositories and nowhere else. This is the one rule that, broken once, tends to be broken everywhere — so it is enforced hardest. Every read and write has a named, typed repository method.

### Tests are part of "done"

A feature is not finished when it works; it is finished when it works *and* is covered. Actions and services are unit-tested with fakes; entry points get a feature test. No network calls in tests — bind fakes and use `Http::fake()`. A PR without tests is an incomplete PR.

## How to use this guide

Read this page first, then [architecture.md](architecture.md) — together they give you the mental model that the rest of the guide assumes. After that, treat the guide as a reference: when you reach for a controller, a repository, a DTO, or a Livewire component, open the matching page and follow the paired ✅/❌ examples.

When in doubt, ask: *"Which layer am I in, and what is this layer allowed to do?"* Almost every style question resolves once you answer that.

### The mechanical parts are enforced; don't hand-police them

Formatting, import ordering, and the bulk of style are not things you should be thinking about or arguing over in review. They're enforced by tooling:

- **Pint** formats every file to the Laravel preset. Run it before you commit.
- **CI** runs Pint in check mode, static analysis, and the full test suite. A red pipeline blocks merge.

```bash
./vendor/bin/pint            # format the codebase to the agreed style
./vendor/bin/pint --test     # CI mode: fail (don't fix) if anything is mis-formatted
./vendor/bin/pest            # the test suite — part of "done"
```

Because the machine owns the mechanical rules, this guide and human review can focus entirely on the parts a linter can't see: the right layer, the right abstraction, the right test. If Pint and you disagree, Pint wins — change the code, not the standard. See [code-style.md](code-style.md) for the conventions the tooling enforces and the few it can't.

## See also

- [Architecture](architecture.md)
- [SOLID](solid.md)
- [DRY](dry.md)
- [Code style](code-style.md)
