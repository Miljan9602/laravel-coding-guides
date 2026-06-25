# Testing (Pest: unit + feature)

Tests are not a follow-up task — they are part of "done." A change without tests is unfinished. This page defines the two test tiers we write (unit and feature), exactly what each covers, how we fake collaborators without touching the network or the database where we shouldn't, and the gotchas that have actually bitten us.

## Rules at a glance

- **Use Pest.** Every test file is `it(...)` / `test(...)` blocks with the `expect()` API. No PHPUnit `class extends TestCase` test cases.
- **Two tiers.** *Unit* tests exercise ONE class in isolation (no DB, no network, fakes for every collaborator). *Feature* tests use `RefreshDatabase` and drive HTTP / Livewire / full flows end-to-end.
- **Write a unit test for every** Service, Action, Repository (its fake's behaviour), value object, DTO, Job, and Command.
- **Write a feature test for every** controller route, every Livewire screen, every Form Request authorisation rule, and every user-facing flow.
- **No test may hit the network.** `Http::fake()` (or a fake client) is mandatory. A test that opens a socket is a bug.
- **Mock the interface, never a final concrete.** If you need to fake a `final` class, that's the signal to extract a contract and mock the contract.
- **Arrange–Act–Assert**, in that order, with blank lines between the phases.
- **Stay DRY** with factories and small local helper functions — never copy-paste 15 lines of setup into ten tests.
- **Bind fakes** via `$this->app->instance(Interface::class, $fake)` or `Mockery::mock(Interface::class)`; never `new` the system-under-test's dependencies by hand when the container can wire them.
- **Force the deterministic fallback** by leaving AI keys blank in `config` — assert the template path renders without any HTTP stub.

## Use Pest, and structure with Arrange–Act–Assert

### Why

Pest's function-style tests read as specifications: `it('marks a report published')` is a sentence, and the file is a list of behaviours rather than a class of methods. The `expect()` API chains assertions fluently, and shared lifecycle (`beforeEach`, `uses`) is declared once per file. Consistency matters more than preference here — one style across the whole suite means any engineer can open any test and know the shape.

Arrange–Act–Assert is the structure inside each test. Set up the world, perform exactly one action, assert on the outcome. One action per test keeps failures diagnosable: when `it('queues generation')` fails, you know precisely which behaviour broke.

### How

✅ Do — one behaviour, three clear phases:

```php
<?php

declare(strict_types=1);

use App\Data\CommitActivity;
use App\Services\Aggregators\ReportAggregator;

it('merges contributors by login across repositories', function (): void {
    // Arrange
    $activity = [
        new CommitActivity(login: 'octocat', repo: 'NIMA-Labs/digest', additions: 40),
        new CommitActivity(login: 'octocat', repo: 'NIMA-Labs/site', additions: 10),
    ];

    // Act
    $report = (new ReportAggregator())->aggregate($activity);

    // Assert
    expect($report->contributors)->toHaveCount(1)
        ->and($report->contributors->first()->additions)->toBe(50);
});
```

❌ Avoid — PHPUnit-style classes and multiple unrelated actions in one test:

```php
<?php

final class ReportAggregatorTest extends TestCase
{
    public function test_it_does_everything(): void
    {
        // aggregates...
        // then also tests delivery...
        // then also tests the Slack block builder...
        // when this fails, which behaviour broke?
    }
}
```

The Pest file does not start with `declare(strict_types=1)` removed — keep it: test files are PHP files and follow the same code standards as production code.

## Tier 1 — Unit tests: one class, in isolation

### Why

A unit test pins the behaviour of a single class. It must not depend on a database schema, a queue worker, a real HTTP endpoint, or another collaborator's implementation — otherwise a failure no longer tells you *which* class is broken, and the test becomes slow and flaky. Isolation is achieved by injecting fakes for every collaborator: a fake repository, a fake GitHub client, a fake AI provider. The class under test runs against in-memory doubles, so the test is fast, deterministic, and precise.

Every Service, Action, Repository, value object, DTO, Job, and Command gets unit coverage. These are where the domain logic lives; they are also where logic is cheapest to test, because they take inputs and produce outputs or recorded effects with no framework boot required beyond the container.

### How — a Service with a faked GitHub client and `Http::fake()`

The service below talks to GitHub through the `GitHubClient` contract. In the test we bind a fake client; we *also* call `Http::fake()` as a backstop so that if any code path slips past the fake and tries a real request, it fails loudly with a clear "no fake set" error rather than reaching the network.

```php
<?php

declare(strict_types=1);

use App\Contracts\GitHubClient;
use App\Data\RepoActivity;
use App\Services\Collectors\PullRequestCollector;
use Illuminate\Support\Facades\Http;

it('collects merged pull requests for a repository', function (): void {
    // Arrange
    Http::preventStrayRequests(); // any un-faked request throws

    $github = Mockery::mock(GitHubClient::class);
    $github->shouldReceive('mergedPullRequests')
        ->once()
        ->with('NIMA-Labs/digest', 7)
        ->andReturn([
            new RepoActivity(login: 'octocat', title: 'Fix Slack retry', mergedAt: now()),
        ]);

    $collector = new PullRequestCollector($github);

    // Act
    $activity = $collector->collect('NIMA-Labs/digest', days: 7);

    // Assert
    expect($activity)->toHaveCount(1)
        ->and($activity[0]->login)->toBe('octocat');
});
```

`Http::preventStrayRequests()` is the strongest network guard: it converts any request without a matching fake into an exception. Prefer it in unit tests over a bare `Http::fake()` when you expect *zero* outbound calls.

### How — a value object / DTO unit test

Value objects and DTOs are the easiest things to test and the easiest to forget. Test construction, derived accessors, and equality/immutability.

```php
<?php

declare(strict_types=1);

use App\Data\Contributor;

it('sums additions and deletions into a net change', function (): void {
    $contributor = new Contributor(
        login: 'octocat',
        additions: 120,
        deletions: 30,
    );

    expect($contributor->netChange())->toBe(90);
});

it('is immutable — withAdditions returns a new instance', function (): void {
    $original = new Contributor(login: 'octocat', additions: 10, deletions: 0);

    $updated = $original->withAdditions(25);

    expect($updated)->not->toBe($original)
        ->and($original->additions)->toBe(10)
        ->and($updated->additions)->toBe(25);
});
```

### How — a Repository unit test (against a real DB, deliberately)

A repository is the one class whose *job* is to talk to Eloquent, so its "unit" test legitimately uses the database — that is the unit being tested. Use `RefreshDatabase` here even though it's a lower-tier test, because the query *is* the behaviour. Everything else stays isolated: no HTTP, no other services.

```php
<?php

declare(strict_types=1);

use App\Models\Report;
use App\Models\Workspace;
use App\Repositories\ReportRepository;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('returns the latest standup for a workspace', function (): void {
    $workspace = Workspace::factory()->create();

    Report::factory()->for($workspace)->create([
        'type' => 'standup',
        'generated_at' => now()->subDays(2),
    ]);
    $newest = Report::factory()->for($workspace)->create([
        'type' => 'standup',
        'generated_at' => now(),
    ]);

    $found = (new ReportRepository())->latestStandupForWorkspace($workspace);

    expect($found->is($newest))->toBeTrue();
});
```

This is the boundary the rest of the suite is allowed to fake. *Consumers* of the repository (actions, services) bind a fake of `ReportRepositoryInterface` and never touch the DB; the repository itself is the single class that proves the query is correct.

## Tier 2 — Feature tests: end-to-end with `RefreshDatabase`

### Why

Unit tests prove each class works alone. Feature tests prove the wired-together system works: the route resolves, the middleware runs, the Form Request validates, the controller calls the action, the action writes through the repository, and the response is what the user sees. They catch integration mistakes — a missing binding, a route typo, an authorisation gap — that no isolated unit test can.

Every controller route, every Livewire screen, and every user-facing flow gets a feature test. Use `RefreshDatabase` so each test starts from a clean, migrated schema; arrange state with factories; and still **never hit the network** — `Http::fake()` the GitHub/Slack/AI calls even in feature tests.

### How — an HTTP route feature test

```php
<?php

declare(strict_types=1);

use App\Models\User;
use App\Models\Workspace;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Http;

uses(RefreshDatabase::class);

it('requests a report and redirects to its show page', function (): void {
    Http::fake(); // GitHub/Slack are stubbed; the queued job is faked below

    $user = User::factory()->create();
    $workspace = Workspace::factory()->for($user, 'owner')->create();

    $response = $this
        ->actingAs($user)
        ->post(route('reports.store', $workspace), [
            'repository' => 'NIMA-Labs/digest',
            'days' => 7,
        ]);

    $response->assertRedirect();

    $this->assertDatabaseHas('reports', [
        'workspace_id' => $workspace->id,
        'repository' => 'NIMA-Labs/digest',
        'status' => 'pending',
    ]);
});
```

### How — a flow that spans multiple steps

A user-facing flow test walks through the real sequence a user triggers, asserting state and side effects at each meaningful step.

```php
<?php

declare(strict_types=1);

use App\Jobs\GenerateReport;
use App\Models\User;
use App\Models\Workspace;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Queue;

uses(RefreshDatabase::class);

it('runs the full request-report flow from form to queued generation', function (): void {
    Http::fake();
    Queue::fake();

    $user = User::factory()->create();
    $workspace = Workspace::factory()->for($user, 'owner')->create();

    $this->actingAs($user)
        ->get(route('reports.create', $workspace))
        ->assertOk()
        ->assertSee('Generate report');

    $this->actingAs($user)
        ->post(route('reports.store', $workspace), [
            'repository' => 'NIMA-Labs/digest',
            'days' => 7,
        ])
        ->assertRedirect();

    Queue::assertPushed(
        GenerateReport::class,
        fn (GenerateReport $job): bool => $job->report->repository === 'NIMA-Labs/digest',
    );
});
```

## Mock the interface, never a final concrete

### Why

We declare classes `final` so they cannot be subclassed in production. Mockery (and PHPUnit doubles) create a *subclass* to override methods — which means you **cannot** mock a `final` class. That is not an obstacle to work around; it is the type system pointing at a design smell. If you need to fake a collaborator, that collaborator should be behind an interface, and the consumer should depend on the interface. The fake then implements the contract, and the production code is none the wiser.

This is exactly why repositories and external clients (GitHub, Slack, AI) have interfaces: so they are fakeable. A `final readonly class RestGitHubClient` is unmockable on purpose — consumers depend on `GitHubClient` (the contract), and tests mock that.

### How

✅ Do — extract and mock the contract:

```php
<?php

declare(strict_types=1);

namespace App\Contracts;

use App\Data\RepoActivity;

interface GitHubClient
{
    /**
     * @return array<int, RepoActivity>
     */
    public function mergedPullRequests(string $repo, int $days): array;
}
```

```php
// In the test — mock the interface, not RestGitHubClient:
$github = Mockery::mock(GitHubClient::class);
$github->shouldReceive('mergedPullRequests')->andReturn([]);
$this->app->instance(GitHubClient::class, $github);
```

❌ Avoid — trying to mock the final implementation:

```php
// Throws: "Unable to create mock: class App\Services\GitHub\RestGitHubClient is final"
$github = Mockery::mock(RestGitHubClient::class);
```

If you find yourself reaching for `Mockery::mock(SomeFinalClass::class)`, stop and extract an interface. The friction is the design feedback.

## Binding fakes into the container

### Why

The system-under-test should be resolved by the container so its real constructor wiring is exercised — except for the one collaborator you're faking, which you swap in beforehand. Two mechanisms cover every case: `$this->app->instance()` for a hand-written fake or a Mockery double, and Laravel's own facade fakes (`Queue::fake()`, `Bus::fake()`, `Event::fake()`, `Mail::fake()`, `Notification::fake()`, `Storage::fake()`) for framework subsystems.

### How — hand-written fake vs. Mockery double

A hand-written fake is best when several tests share it and you want to assert on accumulated state:

```php
<?php

declare(strict_types=1);

use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Models\Report;
use App\Models\Workspace;
use Illuminate\Support\Collection;

final class FakeReportRepository implements ReportRepositoryInterface
{
    /** @var Collection<int, Report> */
    public Collection $published;

    public function __construct()
    {
        $this->published = collect();
    }

    public function markPublished(Report $report): Report
    {
        $this->published->push($report);

        return $report;
    }

    public function latestStandupForWorkspace(Workspace $workspace): Report
    {
        return $this->published->last() ?? Report::factory()->make();
    }

    // ...remaining interface methods...
}
```

```php
it('publishes through the repository', function (): void {
    $fake = new FakeReportRepository();
    $this->app->instance(ReportRepositoryInterface::class, $fake);

    app(PublishReport::class)->handle(Report::factory()->make());

    expect($fake->published)->toHaveCount(1);
});
```

A Mockery double is best for a one-off interaction assertion (called once, with these arguments):

```php
it('marks the report published exactly once', function (): void {
    $report = Report::factory()->make();

    $repo = Mockery::mock(ReportRepositoryInterface::class);
    $repo->shouldReceive('markPublished')->once()->with($report)->andReturn($report);
    $this->app->instance(ReportRepositoryInterface::class, $repo);

    app(PublishReport::class)->handle($report);
});
```

Both bind through `$this->app->instance(...)` so the container injects the double wherever `ReportRepositoryInterface` is type-hinted — including deep inside other resolved services.

## Testing Livewire components

### Why

A Livewire screen is user-facing, so it gets a feature test. `Livewire::test()` boots the component through the real Livewire runtime: you set properties, call actions, and assert on rendered output and emitted events exactly as the browser would drive it. This catches validation wiring, authorisation, and render bugs that an HTTP test of the wrapping page would miss.

### How

```php
<?php

declare(strict_types=1);

use App\Livewire\RequestReportForm;
use App\Models\User;
use App\Models\Workspace;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Http;
use Livewire\Livewire;

uses(RefreshDatabase::class);

it('validates the repository field and creates a report on submit', function (): void {
    Http::fake();

    $user = User::factory()->create();
    $workspace = Workspace::factory()->for($user, 'owner')->create();

    Livewire::actingAs($user)
        ->test(RequestReportForm::class, ['workspace' => $workspace])
        ->set('repository', '')
        ->call('submit')
        ->assertHasErrors(['repository' => 'required'])
        ->set('repository', 'NIMA-Labs/digest')
        ->set('days', 7)
        ->call('submit')
        ->assertHasNoErrors()
        ->assertSee('Report queued')
        ->assertDispatched('report-requested');

    $this->assertDatabaseHas('reports', [
        'workspace_id' => $workspace->id,
        'repository' => 'NIMA-Labs/digest',
    ]);
});
```

Key methods: `set()` assigns a public property, `call()` invokes a component method (a Livewire action), `assertSee()` checks rendered HTML, `assertHasErrors()` / `assertHasNoErrors()` check the validation bag, `assertDispatched()` checks a browser event was emitted, and `assertRedirect()` checks a `redirect()` from within the component. Use `Livewire::actingAs($user)` to drive it as an authenticated user.

## Testing jobs

### Why

A job has two distinct things to test: that it gets **dispatched** (the responsibility of whoever enqueues it), and that its `handle()` **does the right work** (the job's own responsibility). Conflate them and you get brittle tests. Separate them and each is simple.

### How — assert dispatch with `Queue::fake()`

When testing the *caller*, fake the queue and assert the job was pushed. `Queue::fake()` prevents the job from actually running, so this stays a fast unit/feature test of the dispatcher.

```php
<?php

declare(strict_types=1);

use App\Actions\Report\RequestReport;
use App\Jobs\GenerateReport;
use App\Models\Workspace;
use Illuminate\Support\Facades\Queue;

it('queues report generation after creating the pending report', function (): void {
    Queue::fake();

    $report = app(RequestReport::class)->handle(
        Workspace::factory()->make(),
        repository: 'NIMA-Labs/digest',
        days: 7,
    );

    Queue::assertPushed(
        GenerateReport::class,
        fn (GenerateReport $job): bool => $job->report->is($report),
    );
});
```

### How — run `handle()` with container-injected dependencies

When testing the *job itself*, you want its real `handle()` to run with its dependencies resolved from the container. Do **not** `new GenerateReport(...)` and call `handle()` with manually-built collaborators — let the container inject them. Two idioms:

```php
// Resolve dependencies by type-hint on handle() and invoke through the container:
app()->call([$job, 'handle']);
```

```php
// Or dispatch synchronously — runs the job inline, in-process, with DI:
GenerateReport::dispatchSync($report);
```

A full job unit test, faking only the external boundary:

```php
<?php

declare(strict_types=1);

use App\Contracts\Repositories\ReportRepositoryInterface;
use App\Jobs\GenerateReport;
use App\Models\Report;
use Illuminate\Support\Facades\Http;

it('generates a summary and marks the report complete', function (): void {
    Http::preventStrayRequests();

    $report = Report::factory()->make(['status' => 'pending']);

    $repo = Mockery::mock(ReportRepositoryInterface::class);
    $repo->shouldReceive('markComplete')->once()->with($report)->andReturn($report);
    $this->app->instance(ReportRepositoryInterface::class, $repo);

    GenerateReport::dispatchSync($report);
});
```

`dispatchSync` and `app()->call()` both honour constructor and method DI, so the job runs exactly as it would on a worker — minus the queue transport, which is not the job's concern.

## Testing Form Request authorisation

### Why

A Form Request's `authorize()` is a security boundary. If it returns `false`, the framework aborts with a `403`. That rule must be tested directly: send the request as a user who should be denied, and assert the response is forbidden. Skipping this leaves authorisation untested and easy to silently break.

### How

```php
<?php

declare(strict_types=1);

use App\Models\User;
use App\Models\Workspace;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('forbids requesting a report on a workspace you do not own', function (): void {
    $owner = User::factory()->create();
    $intruder = User::factory()->create();
    $workspace = Workspace::factory()->for($owner, 'owner')->create();

    $this->actingAs($intruder)
        ->post(route('reports.store', $workspace), [
            'repository' => 'NIMA-Labs/digest',
            'days' => 7,
        ])
        ->assertForbidden(); // 403 — authorize() returned false
});

it('allows the workspace owner', function (): void {
    $owner = User::factory()->create();
    $workspace = Workspace::factory()->for($owner, 'owner')->create();

    $this->actingAs($owner)
        ->post(route('reports.store', $workspace), [
            'repository' => 'NIMA-Labs/digest',
            'days' => 7,
        ])
        ->assertRedirect(); // passed authorize() and validation
});
```

Pair an "allowed" and a "forbidden" test for every Form Request, so both branches of `authorize()` are pinned.

## Testing the deterministic fallback

### Why

The AI summariser has a contract: when no API key is configured (Anthropic *and* OpenAI blank), it must fall back to a deterministic, template-rendered summary rather than throwing or calling out. That fallback is what keeps `--dry-run` and key-less environments working. Test it by configuring empty keys and asserting the template path runs — with no HTTP stub at all, proving no request is attempted.

### How

```php
<?php

declare(strict_types=1);

use App\Data\Report;
use App\Services\Summaries\PersonSummariser;
use Illuminate\Support\Facades\Http;

it('renders the deterministic template summary when no AI keys are set', function (): void {
    Http::preventStrayRequests(); // proves we never call out

    config()->set('digest.ai.anthropic.key', '');
    config()->set('digest.ai.openai.key', '');

    $summary = app(PersonSummariser::class)->summarise(
        Report::factory()->make(['contributor' => 'octocat', 'additions' => 42]),
    );

    expect($summary)->toContain('octocat')
        ->and($summary)->toContain('42');
});
```

Note we set configuration with `config()->set(...)`, never by mutating `env`. Production code reads `config('digest.ai.anthropic.key')`, so overriding the config value is what actually changes the code path under test. Because `Http::preventStrayRequests()` is active and no fake is registered, any accidental outbound call fails the test — which is the assertion that the fallback truly stays offline.

## Gotchas that have actually bitten us

### `Http::fake()` called twice in the same test does NOT override the first stub

`Http::fake()` **merges** stubs; it does not replace them. Calling it a second time in the same test to "change" the response leaves the *first* stub winning for matching URLs, so your test asserts against a response you thought you'd overridden. This produces baffling failures where the second stub appears to do nothing.

❌ Avoid — second `Http::fake()` silently ignored:

```php
it('handles rate-limit then success', function (): void {
    Http::fake(['api.github.com/*' => Http::response([], 429)]);

    // ...code that should retry...

    Http::fake(['api.github.com/*' => Http::response(['ok' => true], 200)]); // does NOT override
    // The first 429 stub is still in effect — this assertion fails confusingly.
});
```

✅ Do — split into two `it()` blocks (one scenario each):

```php
it('surfaces a rate-limit error', function (): void {
    Http::fake(['api.github.com/*' => Http::response([], 429)]);
    // assert the 429 path
});

it('parses a successful response', function (): void {
    Http::fake(['api.github.com/*' => Http::response(['ok' => true], 200)]);
    // assert the success path
});
```

✅ Or — model a *sequence* of responses to the same URL with `Http::fakeSequence()`:

```php
it('retries after a rate-limit, then succeeds', function (): void {
    Http::fakeSequence('api.github.com/*')
        ->push([], 429)
        ->push(['ok' => true], 200);

    $result = app(GitHubClient::class)->mergedPullRequests('NIMA-Labs/digest', 7);

    expect($result)->not->toBeEmpty();
});
```

`fakeSequence` returns each pushed response once, in order — the correct tool when one test legitimately needs the *same* endpoint to answer differently across calls.

### `firstOrCreate()` does NOT hydrate DB-default columns onto the new instance

When `firstOrCreate()` (or `firstOrNew()`/`updateOrCreate()`) **creates** a row, the returned in-memory model contains only the attributes you passed plus the auto-incrementing key — it is **not** re-read from the database. Columns whose values come from database `DEFAULT` clauses (a `status` defaulting to `'pending'`, a `created_at`, a generated column) are therefore `null` on that returned instance until you `refresh()` it. Code (and tests) that read those columns straight off the returned model get `null` and behave wrongly.

❌ Avoid — relying on a DB default to be present on the returned model:

```php
$report = $reports->firstOrCreateForWorkspace($workspace, 'NIMA-Labs/digest');

// `status` has a DB default of 'pending', but it was NOT passed in,
// so on a freshly-created row this is null — not 'pending'.
if ($report->status === 'pending') {
    GenerateReport::dispatch($report); // never runs on first create
}
```

✅ Do — set the column explicitly so it lives on the in-memory instance:

```php
$report = $reports->firstOrCreateForWorkspace(
    $workspace,
    repository: 'NIMA-Labs/digest',
    attributes: ['status' => 'pending'], // explicit — present on the returned model
);

if ($report->status === 'pending') {
    GenerateReport::dispatch($report);
}
```

✅ Or — `refresh()` to re-read DB-assigned values when you genuinely need them:

```php
$report = $reports->firstOrCreateForWorkspace($workspace, 'NIMA-Labs/digest');
$report->refresh(); // now reflects DB defaults like status, created_at
```

In tests this bites hardest with `assertDatabaseHas` passing while an in-memory assertion fails — because the *database* has the default but the *returned object* does not. Assert against a `refresh()`ed model, or assert against the database, not the stale in-memory instance.

## See also

- [Repositories](repositories.md) — the data-access seam that consumers fake in unit tests.
- [Jobs, Commands & Scheduling](jobs-commands-scheduling.md) — what `Queue::fake()`, `dispatchSync()`, and `app()->call([$job, 'handle'])` are testing.
