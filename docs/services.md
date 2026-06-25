# Services (reusable domain logic & integrations)

A **Service** is a cohesive, stateless unit of domain logic or an integration with an
external system (Slack, GitHub, an AI provider). Services are the reusable *capabilities*
your application leans on — shared by many actions, jobs, commands, and controllers —
and they keep the messy details of HTTP, signatures, and third-party quirks from leaking
into their callers.

## Rules at a glance

- A Service is **stateless**: no per-request mutable fields, no accumulated results. Same
  inputs → same behaviour.
- Declare it `final readonly class` and inject every dependency through a promoted
  `private readonly` constructor.
- Depend on **interfaces (`Contracts\*`) only where you need to fake or swap the concrete
  in a test**. Otherwise type-hint the concrete `final` class directly.
- A Service **owns** its transport. HTTP requests, signature verification, retries, and
  payload shaping stay inside; callers receive typed DTOs and never see a `Response`,
  array of headers, or raw JSON.
- Use a **Service** for a capability reused across many call sites; use an **Action** for
  one write/orchestration triggered by a single use case. (See [Actions](actions.md).)
- Never call `env()` inside a Service. Read settings through `config()`; secrets are wired
  in `config/*.php` from `.env`.
- Return typed values (`Data\*` DTOs, value objects, enums) — never associative arrays.
- Bind interface → concrete in `AppServiceProvider`; use **contextual binding** when two
  callers need different concretes of the same interface.
- Compose Services from Services. A "runner" Service may orchestrate a client, an engine,
  and a repository, but it stays stateless and delegates writes to repositories/actions.

## What a Service is — and what it is not

A Service answers the question *"what can this system do?"* It is a verb-shaped capability:
*exchange this OAuth code*, *verify and apply this webhook*, *summarise this report*. It is
reusable by definition — if only one use case ever needs it, you probably wanted an Action.

### Service vs Action

The distinction is about **cardinality and intent**, not size.

| | **Action** | **Service** |
|---|---|---|
| Trigger | One specific use case | Any number of call sites |
| Shape | One write / one orchestration | A reusable capability |
| Naming | Imperative use case (`PublishReport`) | Capability / integration (`SlackOAuthClient`) |
| Entry point | A single public method (`handle`) | One or more cohesive public methods |
| Lifetime | Often transient, per-request | Long-lived, shared, frequently singleton-ish |

An **Action** like `PublishWeeklyReport` is the *story* of a use case: fetch the report,
summarise it, post to Slack, mark it sent. It is triggered in exactly one place and reads
top-to-bottom. A **Service** like `SlackOAuthClient` is a *tool* that story (and ten others)
reaches for. Actions consume Services; Services rarely consume Actions.

> Rule of thumb: if you find yourself copy-pasting orchestration into a second Action, the
> shared part wants to become a Service.

## Statelessness

### Rule

A Service holds only its injected dependencies. It never accumulates request-scoped state
in properties, so the container can safely share one instance and concurrent calls cannot
interfere.

### Why

Mutable state on a shared object is the classic source of "works in tests, corrupts in
production" bugs: a value set during one call bleeds into the next. Stateless Services are
trivially reusable, trivially testable, and safe to register as singletons.

```php
// ✅ Do — all inputs flow through method parameters; only deps are stored.
<?php

declare(strict_types=1);

namespace App\Services\Slack;

use App\Contracts\HttpGateway;
use App\Data\SlackToken;

final readonly class SlackOAuthClient
{
    public function __construct(
        private HttpGateway $http,
        private string $clientId,
        private string $clientSecret,
        private string $redirectUri,
    ) {}

    public function authorizeUrl(string $state, array $scopes): string
    {
        return 'https://slack.com/oauth/v2/authorize?'.http_build_query([
            'client_id'    => $this->clientId,
            'scope'        => implode(',', $scopes),
            'redirect_uri' => $this->redirectUri,
            'state'        => $state,
        ]);
    }

    public function exchangeCode(string $code): SlackToken
    {
        $payload = $this->http->postForm('https://slack.com/api/oauth.v2.access', [
            'client_id'     => $this->clientId,
            'client_secret' => $this->clientSecret,
            'redirect_uri'  => $this->redirectUri,
            'code'          => $code,
        ]);

        return SlackToken::fromOAuthResponse($payload);
    }
}
```

```php
// ❌ Avoid — request state stored on a shared instance; a second exchange
// silently observes the first caller's token.
final class SlackOAuthClient
{
    private ?SlackToken $lastToken = null;   // shared mutable state

    public function exchangeCode(string $code): SlackToken
    {
        $this->lastToken = /* ... */;        // bleeds across calls
        return $this->lastToken;
    }
}
```

If a method genuinely needs context for the duration of one operation, pass it as a
parameter or wrap it in an immutable value object — never park it on the Service.

## `final readonly class` + promoted constructor DI

### Rule

Stateless Services are value-like: their dependencies are fixed at construction and never
change. Mark them `final readonly` and promote constructor parameters as `private readonly`.

### Why

- `final` forbids inheritance, so behaviour is composed (inject another Service) rather than
  inherited — no fragile base classes, no surprising overrides.
- `readonly` makes statelessness a *compiler-enforced* property: the class literally cannot
  mutate a field after construction.
- Promotion removes the boilerplate of declaring-then-assigning each dependency.

```php
// ✅ Do
final readonly class GitHubWebhookProcessor
{
    public function __construct(
        private WebhookSignatureVerifier $verifier,
        private WorkspaceRepository $workspaces,
        private Dispatcher $events,
    ) {}
}
```

```php
// ❌ Avoid — non-final, hand-rolled assignment, mutable fields.
class GitHubWebhookProcessor
{
    protected WebhookSignatureVerifier $verifier;

    public function __construct(WebhookSignatureVerifier $verifier)
    {
        $this->verifier = $verifier;
    }
}
```

## Depend on interfaces only where a fake is needed

### Rule

Introduce a `Contracts\*` interface **only** when you must substitute the concrete in a test
or swap implementations at runtime. Everywhere else, type-hint the concrete `final` class.

### Why

Blanket "interface for everything" doubles your file count and indirection for no benefit:
to read `SlackOAuthClient` you must also open `SlackOAuthClientInterface`, and the IDE can no
longer jump straight to the implementation. Interfaces earn their keep at exactly two seams:

1. **Test seam** — something that hits the network/DB and must be faked so tests stay
   offline and deterministic (a GitHub client, an AI provider, an HTTP gateway).
2. **Swap seam** — genuinely interchangeable implementations chosen by config or context
   (`AnthropicProvider` vs `OpenAiProvider`).

This is also *how a `final` class stays mockable*: you do not mock the `final` concrete —
you depend on the interface and bind a different implementation (or an anonymous test
double) for it. `final` + interface gives you both a sealed production class and a clean
test seam.

```php
// ✅ Do — interface because the concrete makes real GitHub calls and must be faked.
<?php

declare(strict_types=1);

namespace App\Contracts;

use App\Data\PullRequest;
use Illuminate\Support\Collection;

interface GitHubClient
{
    /** @return Collection<int, PullRequest> */
    public function mergedPullRequests(string $org, string $repo, \DateTimeImmutable $since): Collection;
}
```

```php
<?php

declare(strict_types=1);

namespace App\Services\GitHub;

use App\Contracts\GitHubClient;
use App\Data\PullRequest;
use Illuminate\Http\Client\Factory as Http;
use Illuminate\Support\Collection;

final readonly class RestGitHubClient implements GitHubClient
{
    public function __construct(
        private Http $http,
        private string $token,
        private string $baseUrl,
    ) {}

    public function mergedPullRequests(string $org, string $repo, \DateTimeImmutable $since): Collection
    {
        $response = $this->http
            ->withToken($this->token)
            ->baseUrl($this->baseUrl)
            ->get("/repos/{$org}/{$repo}/pulls", [
                'state'     => 'closed',
                'sort'      => 'updated',
                'direction' => 'desc',
            ])
            ->throw();

        return collect($response->json())
            ->map(PullRequest::fromApi(...))
            ->filter(fn (PullRequest $pr): bool => $pr->mergedAfter($since))
            ->values();
    }
}
```

A Service that does **not** need a test seam should not be hidden behind an interface:

```php
// ✅ Do — pure logic, deterministic, no I/O. No interface; depend on the concrete.
final readonly class ContributionRanker
{
    /**
     * @param  Collection<int, Contributor>  $contributors
     * @return Collection<int, Contributor>
     */
    public function topByImpact(Collection $contributors, int $limit): Collection
    {
        return $contributors
            ->sortByDesc(fn (Contributor $c): int => $c->impactScore())
            ->take($limit)
            ->values();
    }
}
```

```php
// ❌ Avoid — a ceremonial interface with exactly one implementation and no test
// seam. It is pure indirection.
interface ContributionRankerInterface { /* ... */ }
final readonly class ContributionRanker implements ContributionRankerInterface { /* ... */ }
```

## No HTTP / DB leaks into callers

### Rule

A Service fully encapsulates its transport. Callers pass domain inputs and receive domain
outputs (DTOs, value objects, enums, collections of those). They never touch a
`Response`, an HTTP status, raw JSON, an Eloquent builder, or provider-specific shapes.

### Why

When a `Response` escapes a Service, every caller becomes coupled to that vendor's wire
format — and the day the API changes, you edit twenty call sites instead of one. Keeping the
boundary clean means the Service is the *single* place that knows how Slack or GitHub talks.

```php
// ✅ Do — verify, parse, and translate inside; hand back a typed result.
final readonly class GitHubWebhookProcessor
{
    public function __construct(
        private WebhookSignatureVerifier $verifier,
        private WorkspaceRepository $workspaces,
        private Dispatcher $events,
    ) {}

    public function process(string $rawBody, string $signature): WebhookOutcome
    {
        if (! $this->verifier->matches($rawBody, $signature)) {
            return WebhookOutcome::rejected('invalid signature');
        }

        $payload = WebhookPayload::fromJson($rawBody);

        $workspace = $this->workspaces->findByInstallationId($payload->installationId);
        if ($workspace === null) {
            return WebhookOutcome::ignored('unknown installation');
        }

        $this->events->dispatch(new RepositoryActivityReceived($workspace, $payload));

        return WebhookOutcome::accepted($payload->deliveryId);
    }
}
```

```php
// ❌ Avoid — the controller is now coupled to Slack's HTTP response and JSON.
final readonly class SlackOAuthClient
{
    public function exchangeCode(string $code): Response   // leaks transport
    {
        return $this->http->asForm()->post('https://slack.com/api/oauth.v2.access', [/* ... */]);
    }
}

// Caller forced to know the wire format:
$response = $slack->exchangeCode($code);
if ($response->json('ok') !== true) { /* vendor detail in a controller */ }
$token = $response->json('access_token');
```

Errors leak too. Translate vendor failures into domain exceptions or a result type so callers
handle *your* vocabulary, not Slack's `error` strings.

```php
// ✅ Do — fail loudly in your own terms.
$payload = $this->http->postForm($url, $params);

if (($payload['ok'] ?? false) !== true) {
    throw new SlackOAuthException($payload['error'] ?? 'unknown_error');
}
```

## Composing a runner Service

Some Services tie several collaborators together into one named capability — the canonical
case being a per-workspace report runner that wires *context + client + engine + persistence*.
It is still stateless: it orchestrates, then hands the write off to a repository.

```php
// ✅ Do — a runner composed entirely of injected Services/repositories.
<?php

declare(strict_types=1);

namespace App\Services\Reports;

use App\Contracts\GitHubClient;
use App\Data\Report;
use App\Data\WorkspaceContext;

final readonly class WorkspaceReportRunner
{
    public function __construct(
        private GitHubClient $github,
        private ReportAggregator $aggregator,
        private SummaryEngine $summaries,
        private ReportRepository $reports,
    ) {}

    public function run(WorkspaceContext $context): Report
    {
        $activity = $this->github->mergedPullRequests(
            $context->org,
            $context->repo,
            $context->since,
        );

        $report = $this->aggregator->aggregate($activity, $context->displayName);
        $report = $report->withSummaries($this->summaries->summarise($report));

        $this->reports->save($report);

        return $report;
    }
}
```

The runner does not *contain* aggregation or summarisation logic — it delegates to the
`ReportAggregator` and `SummaryEngine` Services and persists through the `ReportRepository`
(see [Repositories](repositories.md)). This keeps each piece independently testable and the
runner readable as a sentence.

## A fallback chain (swap seam in action)

When several interchangeable providers back one capability, depend on the interface and let
a composite Service try them in order. This is the **swap seam**: the interface exists
because more than one real implementation satisfies it.

```php
// ✅ Do — providers implement a shared contract; the chain tries each in turn.
<?php

declare(strict_types=1);

namespace App\Services\Ai;

use App\Contracts\AiProvider;
use App\Data\SummaryRequest;
use Psr\Log\LoggerInterface;

final readonly class AiClient
{
    /**
     * @param  list<AiProvider>  $providers  ordered: primary first, fallbacks after
     */
    public function __construct(
        private array $providers,
        private LoggerInterface $logger,
    ) {}

    public function summarise(SummaryRequest $request): string
    {
        foreach ($this->providers as $provider) {
            try {
                return $provider->complete($request);
            } catch (AiProviderException $e) {
                $this->logger->warning('AI provider failed, falling back', [
                    'provider' => $provider::class,
                    'reason'   => $e->getMessage(),
                ]);
            }
        }

        return $request->deterministicFallback();
    }
}
```

`AnthropicProvider` and `OpenAiProvider` each implement `AiProvider`; the chain ends in a
deterministic, network-free fallback so a degraded report still ships.

## Wiring it up in `AppServiceProvider`

### Rule

Bind interfaces to concretes and supply scalar configuration in `register()`. Read every
tunable through `config()`. Use **contextual binding** when two consumers need different
implementations of the same interface.

### Why

Centralising construction keeps secrets and endpoints in `config/*.php` (sourced from
`.env`), keeps Services free of `env()`, and gives one obvious place to swap an
implementation for a fake or an alternate provider.

```php
// ✅ Do
<?php

declare(strict_types=1);

namespace App\Providers;

use App\Contracts\AiProvider;
use App\Contracts\GitHubClient;
use App\Services\Ai\AiClient;
use App\Services\Ai\AnthropicProvider;
use App\Services\Ai\OpenAiProvider;
use App\Services\GitHub\RestGitHubClient;
use App\Services\Slack\SlackOAuthClient;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\ServiceProvider;
use Psr\Log\LoggerInterface;

final class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Interface → concrete, with config-sourced scalars.
        $this->app->singleton(GitHubClient::class, function (Application $app): RestGitHubClient {
            return new RestGitHubClient(
                http:    $app->make(\Illuminate\Http\Client\Factory::class),
                token:   (string) config('digest.github.token'),
                baseUrl: (string) config('digest.github.base_url'),
            );
        });

        // A concrete Service that needs scalars also gets a factory closure.
        $this->app->singleton(SlackOAuthClient::class, function (): SlackOAuthClient {
            return new SlackOAuthClient(
                http:         $this->app->make(\App\Contracts\HttpGateway::class),
                clientId:     (string) config('digest.slack.client_id'),
                clientSecret: (string) config('digest.slack.client_secret'),
                redirectUri:  (string) config('digest.slack.redirect_uri'),
            );
        });

        // Composite Service with an ordered list of swappable providers.
        $this->app->singleton(AiClient::class, function (Application $app): AiClient {
            return new AiClient(
                providers: [
                    $app->make(AnthropicProvider::class),
                    $app->make(OpenAiProvider::class),
                ],
                logger: $app->make(LoggerInterface::class),
            );
        });
    }
}
```

```php
// ❌ Avoid — env() inside the Service, no binding, untyped scalars.
final readonly class RestGitHubClient implements GitHubClient
{
    public function mergedPullRequests(/* ... */): Collection
    {
        $token = env('GITHUB_TOKEN');   // config cache will return null in prod
        // ...
    }
}
```

### Contextual binding for two consumers of one interface

When a `RealtimeSummaryAction` needs the fast model and a `NightlyDigestJob` needs the
thorough one, bind by consumer instead of branching inside the Service.

```php
// ✅ Do
$this->app->when(\App\Actions\RealtimeSummaryAction::class)
    ->needs(AiProvider::class)
    ->give(AnthropicProvider::class);

$this->app->when(\App\Jobs\NightlyDigestJob::class)
    ->needs(AiProvider::class)
    ->give(OpenAiProvider::class);
```

### Tagging a set of implementations

When the *order* of a provider set is a deployment concern, tag the providers and resolve
the tagged group into the chain.

```php
// ✅ Do — tag, then inject the tagged group (resolution order follows tag order).
$this->app->tag([AnthropicProvider::class, OpenAiProvider::class], 'ai.providers');

$this->app->singleton(AiClient::class, function (Application $app): AiClient {
    return new AiClient(
        providers: iterator_to_array($app->tagged('ai.providers')),
        logger:    $app->make(LoggerInterface::class),
    );
});
```

## Testing Services

### Rule

Test the concrete Service through its interface seam: bind a fake `Contracts\*` or use
`Http::fake()` so no test touches the network. Assert on the typed return value, not on wire
shapes.

### Why

The interface seams you added for swapping doubles as your test seam. Faking at the boundary
keeps tests fast and deterministic while still exercising the Service's real logic.

```php
// ✅ Do — fake the gateway, drive the real SlackOAuthClient, assert the DTO.
it('exchanges an oauth code for a workspace token', function (): void {
    $gateway = new FakeHttpGateway([
        'https://slack.com/api/oauth.v2.access' => [
            'ok'           => true,
            'access_token' => 'xoxb-test-token',
            'team'         => ['id' => 'T123', 'name' => 'NIMA Labs'],
        ],
    ]);

    $client = new SlackOAuthClient(
        http:         $gateway,
        clientId:     'cid',
        clientSecret: 'secret',
        redirectUri:  'https://app.test/slack/callback',
    );

    $token = $client->exchangeCode('code-from-slack');

    expect($token)->toBeInstanceOf(SlackToken::class)
        ->and($token->accessToken)->toBe('xoxb-test-token')
        ->and($token->teamId)->toBe('T123');
});
```

```php
// ✅ Do — for clients built on the Http facade, fake the facade instead.
it('returns only pull requests merged after the cutoff', function (): void {
    Http::fake([
        'api.github.com/*' => Http::response([
            ['number' => 7, 'merged_at' => '2026-06-20T10:00:00Z'],
            ['number' => 8, 'merged_at' => null],
        ]),
    ]);

    $client = app(GitHubClient::class);

    $prs = $client->mergedPullRequests('NIMA-Labs', 'digest', new DateTimeImmutable('2026-06-19'));

    expect($prs)->toHaveCount(1)
        ->and($prs->first()->number)->toBe(7);
});
```

## See also

- [Actions](actions.md) — single use-case writes that consume Services
- [Repositories](repositories.md) — the persistence seam runner Services delegate to
- [SOLID](solid.md) — the dependency-inversion and interface-segregation principles behind the interface seams
