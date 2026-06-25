# SOLID principles in Laravel

SOLID is not academic ceremony here — it is the load-bearing structure of this codebase. Each principle maps directly onto an architectural seam: SRP gives us thin controllers and single-`handle()` actions, OCP gives us provider fallback chains, LSP lets fakes stand in for real clients in tests, ISP keeps our `Contracts\*` interfaces small, and DIP is enforced by constructor injection plus bindings in a service provider.

## Rules at a glance

- **SRP** — one reason to change. Controllers do HTTP only; an Action exposes a single `handle()`; a Repository does data access only. Split fat methods into focused collaborators.
- **OCP** — extend without editing. Add a new AI provider or token source by writing a new class behind an interface, never by modifying the consumer.
- **LSP** — every implementation of a `Contracts\*` interface is fully substitutable. A `FakeGitHubClient` must honour the exact behaviour contract of `RestGitHubClient`.
- **ISP** — many small interfaces beat one fat one. One repository interface per aggregate; keep per-install `GitHubClient` separate from app-level `GitHubAppClient`.
- **DIP** — depend on abstractions (`Contracts\*`), inject via the constructor, bind interface→implementation in `AppServiceProvider`. Never `new` a collaborator inside a class.

## Single Responsibility Principle (SRP)

**Rule:** a class has exactly one reason to change. In practice this means each layer owns one concern — controllers translate HTTP, actions orchestrate one use case, repositories read and write data, and value objects carry data without behaviour that pulls in other concerns.

**Why:** when HTTP parsing, business orchestration, and a GitHub query all live in one method, a change to any of the three forces you to re-read and re-test the other two. Splitting them means a request-validation change never risks the data-access code, and a GitHub API change never touches the controller. Each class becomes independently testable: the action under a fake client, the controller under a fake action.

### ❌ Avoid: one fat method doing HTTP, data access, AI, and delivery

```php
final class ReportController
{
    public function send(Request $request): JsonResponse
    {
        $orgs = explode(',', (string) $request->input('orgs'));

        $commits = Http::withToken(config('digest.github.token'))
            ->get('https://api.github.com/orgs/'.$orgs[0].'/events')
            ->json();

        $summary = Http::withToken(config('digest.ai.anthropic.key'))
            ->post('https://api.anthropic.com/v1/messages', [
                'model' => 'claude-sonnet-4-5',
                'messages' => [['role' => 'user', 'content' => json_encode($commits)]],
            ])
            ->json('content.0.text');

        Http::post(config('digest.slack.webhook_url'), ['text' => $summary]);

        return response()->json(['status' => 'sent']);
    }
}
```

This method changes for four unrelated reasons: request shape, GitHub's API, the AI provider, and the Slack format. It is also untestable without hitting three networks.

### ✅ Do: a thin controller delegating to a single-purpose action

```php
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Actions\RunReport;
use App\Data\ReportRequest;
use Illuminate\Http\JsonResponse;

final class ReportController
{
    public function __construct(
        private readonly RunReport $runReport,
    ) {}

    public function send(SendReportRequest $request): JsonResponse
    {
        $report = $this->runReport->handle(
            ReportRequest::fromArray($request->validated()),
        );

        return response()->json(['status' => 'sent', 'contributors' => $report->contributorCount()]);
    }
}
```

The action owns orchestration and nothing else — one public `handle()`:

```php
declare(strict_types=1);

namespace App\Actions;

use App\Contracts\GitHubClient;
use App\Data\Report;
use App\Data\ReportRequest;
use App\Services\Aggregators\ReportAggregator;
use App\Services\Slack\SlackNotifier;
use App\Services\Summaries\SummaryWriter;

final readonly class RunReport
{
    public function __construct(
        private GitHubClient $github,
        private ReportAggregator $aggregator,
        private SummaryWriter $summaries,
        private SlackNotifier $slack,
    ) {}

    public function handle(ReportRequest $request): Report
    {
        $activity = $this->github->activityFor($request->orgs, $request->since);
        $report = $this->aggregator->aggregate($activity);
        $report = $this->summaries->enrich($report);

        if (! $request->dryRun) {
            $this->slack->send($report);
        }

        return $report;
    }
}
```

And the repository touches data and only data — no formatting, no notifications:

```php
declare(strict_types=1);

namespace App\Repositories;

use App\Contracts\WorkspaceRepository as WorkspaceRepositoryContract;
use App\Models\Workspace;
use Illuminate\Support\Collection;

final readonly class EloquentWorkspaceRepository implements WorkspaceRepositoryContract
{
    public function activeForOrg(string $org): Collection
    {
        return Workspace::query()
            ->where('org', $org)
            ->where('is_active', true)
            ->orderBy('name')
            ->get();
    }
}
```

Each class now has a single axis of change, and each is testable in isolation.

## Open/Closed Principle (OCP)

**Rule:** a unit of code should be open to extension but closed to modification. You add behaviour by introducing new classes that satisfy an existing interface, not by editing a `switch` inside the consumer.

**Why:** modifying working, tested code to bolt on a new case re-opens the risk surface of everything around it. When extension means "write one new class and register it," the consumer stays frozen and trusted. Two patterns carry this in our codebase: a provider/strategy chain, and swapping a collaborator behind an interface.

### Pattern 1 — a provider chain (Anthropic → OpenAI → deterministic fallback)

The `AiClient` iterates an ordered list of providers and returns the first success. Adding a provider — say a local model — means appending one `AiProvider` implementation; `AiClient` itself never changes.

```php
declare(strict_types=1);

namespace App\Contracts;

interface AiProvider
{
    public function name(): string;

    public function summarize(string $prompt): ?string;
}
```

```php
declare(strict_types=1);

namespace App\Services\Ai;

use App\Contracts\AiProvider;
use Psr\Log\LoggerInterface;

final readonly class AiClient
{
    /**
     * @param  list<AiProvider>  $providers  ordered most- to least-preferred
     */
    public function __construct(
        private array $providers,
        private LoggerInterface $logger,
    ) {}

    public function summarize(string $prompt): string
    {
        foreach ($this->providers as $provider) {
            $result = $provider->summarize($prompt);

            if ($result !== null) {
                return $result;
            }

            $this->logger->warning('AI provider yielded no result', [
                'provider' => $provider->name(),
            ]);
        }

        throw new \RuntimeException('No AI provider produced a summary.');
    }
}
```

The deterministic fallback is just another provider that always succeeds, kept last in the chain:

```php
declare(strict_types=1);

namespace App\Services\Ai;

use App\Contracts\AiProvider;

final readonly class DeterministicFallbackProvider implements AiProvider
{
    public function name(): string
    {
        return 'deterministic-fallback';
    }

    public function summarize(string $prompt): string
    {
        return 'Summary unavailable; see the raw activity report for details.';
    }
}
```

Order is wired once, in configuration/binding (see DIP below). To prefer OpenAI, reorder the array — the chain code is untouched.

### ❌ Avoid: a `switch` the consumer must edit for every new provider

```php
public function summarize(string $prompt, string $provider): string
{
    return match ($provider) {
        'anthropic' => $this->callAnthropic($prompt),
        'openai' => $this->callOpenAi($prompt),
        // every new provider edits THIS method and its tests
    };
}
```

### Pattern 2 — swap a token source behind an interface

A GitHub token may come from config or from a freshly minted GitHub App installation token. The consumer asks an interface for a token; it neither knows nor cares which implementation answers.

```php
declare(strict_types=1);

namespace App\Contracts;

interface GitHubTokenProvider
{
    public function token(string $org): string;
}
```

```php
declare(strict_types=1);

namespace App\Services\GitHub\Token;

use App\Contracts\GitHubTokenProvider;

final readonly class ConfigGitHubTokenProvider implements GitHubTokenProvider
{
    public function __construct(
        private string $token,
    ) {}

    public function token(string $org): string
    {
        return $this->token;
    }
}
```

```php
declare(strict_types=1);

namespace App\Services\GitHub\Token;

use App\Contracts\GitHubAppClient;
use App\Contracts\GitHubTokenProvider;

final readonly class InstallationTokenProvider implements GitHubTokenProvider
{
    public function __construct(
        private GitHubAppClient $app,
    ) {}

    public function token(string $org): string
    {
        return $this->app->installationTokenForOrg($org);
    }
}
```

`RestGitHubClient` depends on `GitHubTokenProvider` and is closed to this change — swap config auth for App auth by changing the binding, not the client.

## Liskov Substitution Principle (LSP)

**Rule:** any implementation of an interface must be substitutable for any other without breaking callers. Substitutes must honour the same behaviour contract: same return shapes, same exception types, no surprising side effects.

**Why:** this is what makes tests trustworthy. If `FakeGitHubClient` returned a slightly different shape, or threw `\Exception` where the real client throws a typed `GitHubRequestException`, a green test would be a lie. LSP turns the interface into a real contract that fakes and real implementations both obey, so a passing test against the fake predicts behaviour against the real client.

### The contract

```php
declare(strict_types=1);

namespace App\Contracts;

use App\Data\Activity;
use App\Exceptions\GitHubRequestException;
use Illuminate\Support\Carbon;

interface GitHubClient
{
    /**
     * @param  list<string>  $orgs
     *
     * @throws GitHubRequestException on a non-2xx response
     */
    public function activityFor(array $orgs, Carbon $since): Activity;
}
```

### ✅ Do: a fake that honours the exact same contract

```php
declare(strict_types=1);

namespace Tests\Fakes;

use App\Contracts\GitHubClient;
use App\Data\Activity;
use App\Exceptions\GitHubRequestException;
use Illuminate\Support\Carbon;

final readonly class FakeGitHubClient implements GitHubClient
{
    public function __construct(
        private Activity $activity,
        private bool $shouldFail = false,
    ) {}

    public function activityFor(array $orgs, Carbon $since): Activity
    {
        if ($this->shouldFail) {
            // Same exception type the real client throws — substitutable on the error path too.
            throw new GitHubRequestException('Simulated GitHub failure.');
        }

        return $this->activity;
    }
}
```

Because the fake returns a real `Activity` DTO and throws the real exception type, the action behaves identically whether wired to `RestGitHubClient` or `FakeGitHubClient`:

```php
it('aggregates contributors from GitHub activity', function (): void {
    $activity = ActivityFactory::withCommits(login: 'octocat', count: 3);

    $report = app(RunReport::class)
        ->handle(new ReportRequest(orgs: ['NIMA-Labs'], since: now()->subDay(), dryRun: true));

    expect($report->contributorFor('octocat')->commitCount)->toBe(3);
});
```

### ❌ Avoid: a fake that weakens the contract

```php
final class FakeGitHubClient implements GitHubClient
{
    public function activityFor(array $orgs, Carbon $since): Activity
    {
        // LSP violation: real client throws GitHubRequestException, never returns null.
        if ($this->shouldFail) {
            return null; // breaks the typed contract and the caller's assumptions
        }

        // LSP violation: returns an empty stub that the real client never would,
        // silently masking aggregation bugs.
        return new Activity([]);
    }
}
```

A substitute that narrows return types, returns sentinel values the real one never emits, or throws a different exception type is not LSP-safe — and any test relying on it is unreliable.

## Interface Segregation Principle (ISP)

**Rule:** clients should not be forced to depend on methods they do not use. Prefer many small, role-specific interfaces over one fat "do everything" interface.

**Why:** a fat interface forces every implementer to stub methods it has no business implementing, and forces every consumer to know about capabilities irrelevant to it. Small interfaces mean a fake only implements what a test needs, and a consumer's dependencies honestly describe what it uses. Two concrete splits in this codebase: one repository interface per aggregate, and separating per-install GitHub access from app-level access.

### Split GitHub access by audience

Per-installation operations (reading commits/PRs with an installation token) and app-level operations (minting installation tokens with the App's JWT) are different responsibilities used by different callers. Keep them apart.

```php
declare(strict_types=1);

namespace App\Contracts;

use App\Data\Activity;
use Illuminate\Support\Carbon;

interface GitHubClient
{
    /** @param  list<string>  $orgs */
    public function activityFor(array $orgs, Carbon $since): Activity;
}
```

```php
declare(strict_types=1);

namespace App\Contracts;

interface GitHubAppClient
{
    public function installationTokenForOrg(string $org): string;
}
```

A collector needs only `GitHubClient`; the `InstallationTokenProvider` needs only `GitHubAppClient`. Neither is burdened by the other's methods.

### One repository interface per aggregate

```php
declare(strict_types=1);

namespace App\Contracts;

use App\Models\Workspace;
use Illuminate\Support\Collection;

interface WorkspaceRepository
{
    public function activeForOrg(string $org): Collection;

    public function findBySlug(string $slug): ?Workspace;
}
```

```php
declare(strict_types=1);

namespace App\Contracts;

use App\Models\Report;

interface ReportRepository
{
    public function store(Report $report): void;

    public function latestForOrg(string $org): ?Report;
}
```

### ❌ Avoid: a god interface every implementer must satisfy

```php
interface GitHubRepository
{
    public function activityFor(array $orgs, Carbon $since): Activity;
    public function installationTokenForOrg(string $org): string;
    public function storeReport(Report $report): void;
    public function latestReport(string $org): ?Report;
    public function activeWorkspaces(string $org): Collection;
}
```

A fake for a collector test would have to stub four unrelated methods. The split interfaces above let each fake implement only what it must.

## Dependency Inversion Principle (DIP)

**Rule:** high-level code depends on abstractions, not concretions. Inject collaborators through the constructor as `Contracts\*` interfaces, and bind each interface to its implementation in a service provider. Never `new` a collaborator inside a class.

**Why:** `new RestGitHubClient(...)` inside an action hard-wires it to HTTP, the network, and a specific token strategy — you cannot test it without the network, and you cannot swap implementations. Depending on `Contracts\GitHubClient` and letting the container resolve it means tests inject a fake, OCP swaps (config vs installation token) happen at the binding, and LSP/ISP guarantees hold because the consumer only ever sees the interface.

### ❌ Avoid: constructing collaborators inline

```php
final class RunReport
{
    public function handle(ReportRequest $request): Report
    {
        // Hard-wired: untestable without the network, no way to swap auth.
        $github = new RestGitHubClient(
            new ConfigGitHubTokenProvider(env('GITHUB_TOKEN')),
        );

        return $this->aggregate($github->activityFor($request->orgs, $request->since));
    }
}
```

Note also the `env()` call outside a config file — forbidden, because cached config makes `env()` return `null` in production.

### ✅ Do: inject the abstraction; resolve config in config files

```php
declare(strict_types=1);

namespace App\Actions;

use App\Contracts\GitHubClient;
use App\Data\Report;
use App\Data\ReportRequest;

final readonly class RunReport
{
    public function __construct(
        private GitHubClient $github,
    ) {}

    public function handle(ReportRequest $request): Report
    {
        return $this->aggregate(
            $this->github->activityFor($request->orgs, $request->since),
        );
    }
}
```

### Bind interface → implementation in a service provider

```php
declare(strict_types=1);

namespace App\Providers;

use App\Contracts\AiProvider;
use App\Contracts\GitHubClient;
use App\Contracts\GitHubTokenProvider;
use App\Services\Ai\AiClient;
use App\Services\Ai\AnthropicProvider;
use App\Services\Ai\DeterministicFallbackProvider;
use App\Services\Ai\OpenAiProvider;
use App\Services\GitHub\RestGitHubClient;
use App\Services\GitHub\Token\ConfigGitHubTokenProvider;
use App\Services\GitHub\Token\InstallationTokenProvider;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\ServiceProvider;

final class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Choose the token source by config — OCP swap, one line.
        $this->app->bind(GitHubTokenProvider::class, function (Application $app): GitHubTokenProvider {
            return config('digest.github.app.enabled')
                ? $app->make(InstallationTokenProvider::class)
                : new ConfigGitHubTokenProvider(config('digest.github.token'));
        });

        $this->app->bind(GitHubClient::class, RestGitHubClient::class);

        // Provider chain order lives in the binding, not in AiClient.
        $this->app->singleton(AiClient::class, function (Application $app): AiClient {
            return new AiClient(
                providers: [
                    $app->make(AnthropicProvider::class),
                    $app->make(OpenAiProvider::class),
                    $app->make(DeterministicFallbackProvider::class),
                ],
                logger: $app->make(\Psr\Log\LoggerInterface::class),
            );
        });
    }
}
```

With this binding, `app(RunReport::class)` resolves a fully wired action in production, while a test rebinds the interface to a fake before resolving:

```php
beforeEach(function (): void {
    $this->app->instance(
        GitHubClient::class,
        new FakeGitHubClient(ActivityFactory::empty()),
    );
});
```

The consumer (`RunReport`) is identical in both cases — that is dependency inversion delivering on its promise.

## See also

- [Architecture](architecture.md)
- [Repositories](repositories.md)
- [Services](services.md)
