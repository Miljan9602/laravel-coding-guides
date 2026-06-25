# Quick reference & PR checklist

The whole guide compressed onto one page: a layer-by-layer cheat sheet of the
rules you apply while writing code, and a task-list checklist a reviewer runs
before approving a PR. Skim the cheat sheet when coding; run the checklist when
reviewing.

## Rules at a glance

- Every PHP file starts with `declare(strict_types=1);`.
- Every concrete class is `final`; stateless ones are `final readonly`.
- Controllers orchestrate: Form Request in, one Action/Service, a response out — nothing else.
- Every input-bearing endpoint has a Form Request; never call `$request->validate()` inline.
- App code never touches a model directly — all persistence goes through a repository.
- New write logic lives in an Action; reusable read/capability logic in a Service.
- `config()` everywhere; `env()` only inside `config/*.php` files.
- A unit test per class, a feature test per feature, and no test touches the network.
- `composer check` (Pint + PHPStan + Pest) passes before you push.

## Part 1 — Cheat sheet

Twenty-five rules, one line each, grouped by the layer they govern. Each line is
a claim you can check against a diff in seconds. The detailed pages explain the
*why*; this is the *what*.

### Style

1. **Strict types always.** First line of every PHP file is `declare(strict_types=1);` — see [philosophy](philosophy.md).
2. **Finalize.** Concrete classes are `final`; value objects, DTOs, actions, services and repositories are `final readonly`.
3. **Promote and type.** Use constructor property promotion with `private readonly`, and give every parameter and return value an explicit type.
4. **Pint is law.** Format with the Laravel preset; never hand-tune whitespace that Pint will rewrite.

```php
<?php

declare(strict_types=1);

namespace App\Data;

final readonly class Contributor
{
    public function __construct(
        public string $login,
        public int $commitCount,
        public int $pullRequestCount,
    ) {}
}
```

### Controllers

5. **Thin controllers.** A controller method takes a Form Request, calls exactly one Action or Service, and returns a response — no queries, `Http`, `Crypt`, or `validate()` inside.
6. **No business logic.** Branching on domain rules belongs in an Action/Service, not the controller. See [controllers](controllers.md).
7. **Return, don't build.** Hand collections to a Resource or a view; don't assemble payload arrays by hand in the method.

```php
final class WorkspaceReportController
{
    public function store(GenerateReportRequest $request, GenerateReport $action): JsonResponse
    {
        $report = $action->handle(
            WorkspaceId::fromString($request->validated('workspace_id')),
            $request->lookbackDays(),
        );

        return ReportResource::make($report)->response()->setStatusCode(201);
    }
}
```

### Requests

8. **One Form Request per write endpoint.** Validation, authorization, and input typing live there — never in the controller. See [requests](form-requests.md).
9. **Read through accessors.** Expose typed getters (`lookbackDays(): int`) on the Form Request rather than re-reading raw `$request->input()` downstream.
10. **Authorize explicitly.** Implement `authorize()`; return `false` (or a Gate check) instead of leaving it `true` by default.

```php
final class GenerateReportRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('generate', Report::class);
    }

    public function rules(): array
    {
        return [
            'workspace_id' => ['required', 'uuid'],
            'lookback_days' => ['required', 'integer', 'between:1,30'],
        ];
    }

    public function lookbackDays(): int
    {
        return (int) $this->validated('lookback_days');
    }
}
```

### Actions

11. **One write, one Action.** A single public method (`handle()` or `__invoke()`) that performs one unit of work and returns a typed result. See [actions](actions.md).
12. **Actions own transactions.** Wrap multi-step writes in `DB::transaction()` inside the Action, not the controller.
13. **Inject collaborators.** Depend on repository and service *interfaces* through the constructor; never `new` them or hit a facade for domain data.

```php
final readonly class GenerateReport
{
    public function __construct(
        private ReportRepository $reports,
        private SummaryService $summaries,
    ) {}

    public function handle(WorkspaceId $workspace, int $lookbackDays): Report
    {
        return DB::transaction(function () use ($workspace, $lookbackDays): Report {
            $report = $this->reports->openForWorkspace($workspace, $lookbackDays);

            return $this->reports->finalize(
                $report->withSummary($this->summaries->forReport($report)),
            );
        });
    }
}
```

### Services

14. **Services are reusable capabilities.** Stateless, `final readonly`, behind an interface — AI summarization, Slack delivery, GitHub fetching. See [services](services.md).
15. **No model awareness.** A service works on DTOs and value objects; if it needs persistence it calls a repository, never Eloquent.
16. **Fallback explicitly or fail loudly.** Provide a deterministic fallback only when asked; otherwise let the error surface.

```php
final readonly class AnthropicSummaryService implements SummaryService
{
    public function __construct(private AiClient $ai) {}

    public function forReport(Report $report): string
    {
        return $this->ai->complete(
            model: config('digest.ai.summary_model'),
            prompt: SummaryPrompt::forReport($report),
        );
    }
}
```

### Repositories

17. **Only repositories touch Eloquent.** Every `Model::query()`, `->save()`, and relation query lives behind a repository interface. See [repositories](repositories.md).
18. **Return domain types.** Repositories return DTOs/entities/collections, not raw query builders or arrays leaking column names.
19. **Name by intent.** Methods read like the domain (`openForWorkspace`, `finalize`), not like SQL (`whereWorkspaceAndUpdate`).

```php
final readonly class EloquentReportRepository implements ReportRepository
{
    public function openForWorkspace(WorkspaceId $workspace, int $lookbackDays): Report
    {
        $model = ReportModel::query()->create([
            'workspace_id' => $workspace->toString(),
            'lookback_days' => $lookbackDays,
            'status' => ReportStatus::Open->value,
        ]);

        return $model->toData();
    }
}
```

### Models

20. **Models are persistence shells.** Keep Eloquent models to casts, relations, and scopes — no business logic, no HTTP, no Slack. See [models](models-eloquent.md).
21. **Cast everything.** Declare `casts()` for dates, enums, money, and JSON; never parse a raw attribute downstream.
22. **Guard mass assignment.** Set `$fillable` (or `$guarded = []`) deliberately and back it with a Form Request.

```php
final class ReportModel extends Model
{
    protected $table = 'reports';

    protected $fillable = ['workspace_id', 'lookback_days', 'status'];

    protected function casts(): array
    {
        return [
            'status' => ReportStatus::class,
            'generated_at' => 'immutable_datetime',
        ];
    }
}
```

### Livewire

23. **Components are thin too.** A Livewire component validates, calls an Action/Service, and re-renders; it never queries Eloquent or holds business rules. See [livewire](livewire.md).
24. **Type public state.** Public properties carry explicit types; use `#[Computed]` for derived values and `#[Validate]`/rules for input.

```php
final class GenerateReportForm extends Component
{
    #[Validate('required|uuid')]
    public string $workspaceId = '';

    #[Validate('required|integer|between:1,30')]
    public int $lookbackDays = 7;

    public function generate(GenerateReport $action): void
    {
        $this->validate();

        $action->handle(WorkspaceId::fromString($this->workspaceId), $this->lookbackDays);

        $this->dispatch('report-generated');
    }
}
```

### Jobs

25. **Jobs delegate, they don't decide.** A queued job resolves an Action/Service from the container and calls it; the domain logic stays out of `handle()`. Configure `$tries`/`$backoff` and make the work idempotent. See [jobs](jobs-commands-scheduling.md).

```php
final class GenerateReportJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public array $backoff = [30, 120, 300];

    public function __construct(
        private readonly WorkspaceId $workspace,
        private readonly int $lookbackDays,
    ) {}

    public function handle(GenerateReport $action): void
    {
        $action->handle($this->workspace, $this->lookbackDays);
    }
}
```

### Tests

- **One unit test per class.** Cover each new class in isolation with its collaborators faked. See [testing](testing.md).
- **One feature test per feature.** Exercise the endpoint or command end-to-end through the container.
- **No network in tests.** Bind fake interfaces and use `Http::fake()`; a test that reaches GitHub or Anthropic is a broken test.

```php
it('finalizes a report through the repository', function (): void {
    $reports = Mockery::mock(ReportRepository::class);
    $summaries = Mockery::mock(SummaryService::class);
    $report = ReportFactory::open();

    $summaries->shouldReceive('forReport')->andReturn('weekly focus: shipping');
    $reports->shouldReceive('openForWorkspace')->andReturn($report);
    $reports->shouldReceive('finalize')->andReturnUsing(fn (Report $r): Report => $r);

    $result = (new GenerateReport($reports, $summaries))
        ->handle(WorkspaceId::fromString('...'), 7);

    expect($result->summary)->toBe('weekly focus: shipping');
});
```

### Config

- **`config()` outside config files.** Read tunables (orgs, models, limits) via `config('digest.*')`; call `env()` only inside `config/*.php`. See [configuration](config-and-secrets.md).
- **Secrets stay in `.env`.** API keys and webhook URLs come from environment, are mapped through a config file, and are never committed.

```php
// ✅ Do — in a service
$model = config('digest.ai.summary_model');

// ❌ Avoid — env() anywhere but config/*.php
$model = env('DIGEST_SUMMARY_MODEL');
```

## Part 2 — Pull-request checklist

Copy this block into the PR description or run it mentally during review. Every
box must be checked — an unchecked box is a blocking comment, not a nit. Start
from [philosophy](philosophy.md) when a box is contested.

### Style & structure

- [ ] Every PHP file has `declare(strict_types=1);` as its first statement.
- [ ] Every concrete class is `final` (and `final readonly` if it is stateless: DTO, value object, action, service, repository).
- [ ] Constructor property promotion is used with `private readonly`; all params and returns are typed; no `any`/untyped arrays where a DTO fits.
- [ ] `composer check` (Pint + static analysis + Pest) passes locally — no Pint diff, no PHPStan errors.

### Controllers & requests

- [ ] Controllers contain no queries, `Http`, `Crypt`, or `validate()` — only a Form Request, one Action/Service, and a response. ([controllers](controllers.md))
- [ ] Every input-bearing endpoint has a dedicated Form Request with real `rules()` and an explicit `authorize()`. ([requests](form-requests.md))
- [ ] Downstream code reads input through typed Form Request accessors, not raw `$request->input()`.

### Domain logic

- [ ] No app code touches a model directly — there is no `Model::...`, `->save()`, or `->relation()->query()` outside a repository. ([repositories](repositories.md))
- [ ] New write logic lives in an Action with a single public method; reusable read/capability logic lives in a Service behind an interface. ([actions](actions.md), [services](services.md))
- [ ] Actions own their transactions; multi-step writes are wrapped in `DB::transaction()`.
- [ ] No duplicated logic — repeated branches/queries/formatting are extracted (DRY), and existing helpers were checked before writing new ones.
- [ ] Models hold only casts, relations, and scopes — no business logic or I/O. ([models](models-eloquent.md))

### Config & secrets

- [ ] `config()` is used everywhere outside `config/*.php`; no stray `env()` calls in app code. ([configuration](config-and-secrets.md))
- [ ] No secrets committed — keys/webhooks live in `.env`, mapped through a config file, and `.env` is not in the diff.

### Tests

- [ ] A unit test exists for each new class and a feature test for each new feature. ([testing](testing.md))
- [ ] No test hits the network; mocks and fakes target interfaces (`Http::fake()`, fake `GitHubClient`/`AiClient`).
- [ ] Tests assert behaviour and edge cases, not just the happy path; failures fail loudly with clear messages.

### Final gate

- [ ] The diff is minimal and scoped to the request — no drive-by refactors or unrelated files.
- [ ] A staff engineer would approve this diff.

## See also

- [Philosophy](philosophy.md)
- [Controllers](controllers.md)
- [Requests](form-requests.md)
- [Actions](actions.md)
- [Services](services.md)
- [Repositories](repositories.md)
- [Models](models-eloquent.md)
- [Livewire](livewire.md)
- [Jobs](jobs-commands-scheduling.md)
- [Testing](testing.md)
- [Configuration](config-and-secrets.md)
- [Tooling & CI](tooling-and-ci.md)
