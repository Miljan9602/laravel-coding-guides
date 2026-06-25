# Error handling & exceptions

Fail loudly and predictably. An error that surfaces immediately — with a typed exception, a clear message, and context — is cheap to fix; an error that is swallowed, defaulted away, or rethrown twice is a multi-hour debugging session waiting to happen.

## Rules at a glance

- **Never silently swallow errors.** No empty `catch` blocks. If you catch, you act — log, translate, or rethrow.
- **Throw typed, domain-specific exceptions** (e.g. `InstallationTokenException`), not bare `\Exception` or `\RuntimeException`, and not `return null` on a real failure.
- **Guard clauses + early returns.** Validate preconditions at the top and bail; keep the happy path un-indented.
- **No fallbacks unless the spec asks for one.** A default value that hides a failure is a bug. When graceful degradation *is* required, make it explicit, deliberate, and tested.
- **Separate client errors from server errors.** Bad input → 4xx via Form Request / `abort()`. Broken system → uncaught typed exception → 5xx, reported to error tracking.
- **Log with context, exactly once.** Either handle-and-log *or* rethrow — never both for the same failure.
- **Untrusted input fails closed.** A bad webhook signature, an unverifiable payload, a missing token → reject. Never "allow on doubt".
- **Distinguish "absent" from "broken".** A missing optional record may legitimately be `null`; a failed API call is an exception.
- **`report()` a handled-but-notable degradation.** A catch-and-continue fallback that only `warning()`s is invisible to your error tracker; call `report($e)` so a week of silent degradation surfaces. `throw` the unexpected, `report()`+continue the notable, `warning()` only the mundane.
- **Throttle exception reporting.** In `withExceptions`, `throttle(...)` and `dontReportDuplicates()` so a GitHub/AI outage can't flood and burn paid Sentry/Flare quota.
- **Carry context across the request→job boundary with `Context`.** `Context::add('trace_id', …)` auto-appends to every log line and dehydrates into queued jobs with no plumbing; `Context::addHidden(...)` carries a tenant key without leaking it into logs.
- **Decide a log's format, destination, and routing, not just its content.** JSON to `stderr` in containers, a `stack` channel, and severity routing (`critical` → Slack, `debug` → file).

## Never silently swallow errors

An empty `catch`, or a `catch` that returns a default, destroys the single most valuable piece of debugging information you will ever have: the original stack trace at the original site. The failure still happens — it just happens later, somewhere unrelated, with no breadcrumb back to the cause.

If you catch an exception you must do something meaningful with it: translate it into a domain exception, enrich it with context and rethrow, or — only when the spec demands degradation — take a deliberate, tested alternate path.

### ❌ Avoid — the failure vanishes

```php
public function fetchPullRequests(Workspace $workspace): array
{
    try {
        return $this->github->pullRequests($workspace->repo);
    } catch (\Throwable) {
        return []; // a network blip is now indistinguishable from "no PRs"
    }
}
```

The report renders "0 pull requests this week" and everyone believes the team did nothing. The truth — a 500 from GitHub — is gone.

### ✅ Do — let real failures propagate

```php
public function fetchPullRequests(Workspace $workspace): array
{
    // No try/catch: a transport failure is a real failure and must surface.
    return $this->github->pullRequests($workspace->repo);
}
```

If you genuinely need to add context, catch, enrich, and rethrow — but only once (see "Log once").

## Throw typed, domain-specific exceptions

A generic `\RuntimeException("token failed")` forces every caller to either catch *everything* or catch *nothing*. A typed exception is a contract: callers can catch precisely the failure they know how to handle, leave the rest to propagate, and the type name documents the failure mode at the throw site and in stack traces and error-tracker groupings.

Define exceptions per domain concept, extend the most specific sensible base, and use named constructors to centralise the message and carry structured context.

### ✅ Do — a typed exception with named constructors

```php
<?php

declare(strict_types=1);

namespace App\Exceptions;

use RuntimeException;
use Throwable;

final class InstallationTokenException extends RuntimeException
{
    public static function exchangeFailed(int $installationId, int $status): self
    {
        return new self(
            "Failed to mint an installation token for installation {$installationId} (HTTP {$status}).",
        );
    }

    public static function expired(int $installationId): self
    {
        return new self("Installation token for installation {$installationId} has expired.");
    }

    public static function fromTransport(int $installationId, Throwable $previous): self
    {
        return new self(
            "Transport error while minting token for installation {$installationId}.",
            previous: $previous,
        );
    }
}
```

Throw it where the failure is known, preserving the underlying cause via `previous` so the original trace is never lost:

```php
final readonly class InstallationTokenFactory
{
    public function __construct(private GitHubClient $github) {}

    public function mint(int $installationId): InstallationToken
    {
        try {
            $response = $this->github->exchangeToken($installationId);
        } catch (ConnectionException $e) {
            throw InstallationTokenException::fromTransport($installationId, $e);
        }

        if ($response->failed()) {
            throw InstallationTokenException::exchangeFailed($installationId, $response->status());
        }

        return InstallationToken::fromArray($response->json());
    }
}
```

### ❌ Avoid — returning null on a real failure

```php
public function mint(int $installationId): ?InstallationToken
{
    $response = $this->github->exchangeToken($installationId);

    if ($response->failed()) {
        return null; // caller now has to guess WHY, and probably won't check
    }

    return InstallationToken::fromArray($response->json());
}
```

`?InstallationToken` is a lie: it says "this might legitimately be absent" when in fact a `null` here means "GitHub rejected us and the report is about to break". Reserve nullable return types for genuine absence (a lookup that found nothing), not for failure.

## Guard clauses and early returns

Deeply nested `if`/`else` hides the happy path and makes it easy to leave a branch unhandled. Validate every precondition at the top of the method, return or throw immediately on each, and let the main logic sit at a single indentation level. Each guard reads as one fact the rest of the method may now assume.

### ❌ Avoid — the arrow anti-pattern

```php
public function deliver(Report $report, Workspace $workspace): void
{
    if ($workspace->isActive()) {
        if ($workspace->slackWebhook !== null) {
            if ($report->hasContent()) {
                $this->slack->post($workspace->slackWebhook, $report);
            } else {
                // silently do nothing?
            }
        }
    }
}
```

### ✅ Do — guard, then proceed

```php
public function deliver(Report $report, Workspace $workspace): void
{
    if (! $workspace->isActive()) {
        throw WorkspaceNotDeliverable::inactive($workspace->id);
    }

    if ($workspace->slackWebhook === null) {
        throw WorkspaceNotDeliverable::missingSlackWebhook($workspace->id);
    }

    if (! $report->hasContent()) {
        return; // an empty report is a legitimate no-op, not an error
    }

    $this->slack->post($workspace->slackWebhook, $report);
}
```

Note the deliberate distinction: an inactive workspace or a missing webhook is a *misconfiguration* and throws; an empty report is an *expected* state and returns quietly. Guard clauses make that policy explicit and auditable.

## No fallbacks unless the spec asks for one

A "fallback" that papers over a failure — defaulting to an empty array, swapping in a placeholder, retrying forever in silence — converts a loud, fixable error into a quiet, permanent lie. The default rule is: **do not add fallbacks.** Let it fail.

The exception is **deliberate, specified graceful degradation**: a case where the product requirement is "this feature must never block the whole operation, and a reduced result is acceptable." When that is true, the degradation is a designed code path — named, ordered, logged at each transition, and covered by tests for every rung — not a `catch (\Throwable) { /* whatever */ }`.

### The summary chain — a deliberate, tested fallback

Requirement: a digest must always ship, even if AI providers are down. A per-person summary may degrade from "Anthropic-written" to "OpenAI-written" to a "deterministic template", but the report is never blocked. This is explicit graceful degradation, so it is allowed — and tested at every rung.

```php
<?php

declare(strict_types=1);

namespace App\Services\Summaries;

use App\Contracts\SummaryProvider;
use App\Data\Contributor;
use Psr\Log\LoggerInterface;
use Throwable;

final readonly class SummaryChain
{
    /**
     * @param  list<SummaryProvider>  $providers  Ordered best-to-worst.
     *                                            The final provider MUST NOT throw.
     */
    public function __construct(
        private array $providers,
        private LoggerInterface $logger,
    ) {}

    public function summarise(Contributor $contributor): string
    {
        foreach ($this->providers as $provider) {
            try {
                return $provider->summarise($contributor);
            } catch (Throwable $e) {
                // Deliberate degradation: log the rung that failed, then try the next.
                $this->logger->warning('Summary provider failed; falling back.', [
                    'provider' => $provider::class,
                    'contributor' => $contributor->login,
                    'exception' => $e->getMessage(),
                ]);
            }
        }

        // Unreachable: the deterministic provider is total. Fail loudly if misconfigured.
        throw SummaryChainExhausted::noTerminalProvider($contributor->login);
    }
}
```

The terminal provider is *total* — it cannot fail — which is what makes the chain safe:

```php
final readonly class TemplateSummaryProvider implements SummaryProvider
{
    public function summarise(Contributor $contributor): string
    {
        return sprintf(
            '%s opened %d PRs and pushed %d commits across %d repositories.',
            $contributor->displayName,
            $contributor->pullRequestCount,
            $contributor->commitCount,
            $contributor->repositoryCount,
        );
    }
}
```

What makes this legitimate and *not* silent swallowing:

1. It is **specified** — the requirement explicitly says the report must never block.
2. It is **ordered and named** — providers are an explicit best-to-worst list.
3. It is **observable** — every degradation logs which rung failed and why.
4. It is **bounded** — the last rung is total, so there is no infinite "try anything".
5. It is **tested** — see below.

```php
it('uses Anthropic when it succeeds', function () {
    $chain = new SummaryChain([
        fakeProvider(returns: 'anthropic copy'),
        fakeProvider(throws: true),
        new TemplateSummaryProvider(),
    ], logger());

    expect($chain->summarise($contributor()))->toBe('anthropic copy');
});

it('falls back to OpenAI when Anthropic throws', function () {
    $chain = new SummaryChain([
        fakeProvider(throws: true),
        fakeProvider(returns: 'openai copy'),
        new TemplateSummaryProvider(),
    ], logger());

    expect($chain->summarise($contributor()))->toBe('openai copy');
});

it('falls back to the deterministic template when both AI providers throw', function () {
    $chain = new SummaryChain([
        fakeProvider(throws: true),
        fakeProvider(throws: true),
        new TemplateSummaryProvider(),
    ], logger());

    expect($chain->summarise($contributor(login: 'octocat')))
        ->toContain('opened');
});
```

Contrast with an *un*-specified fallback, which is never acceptable:

```php
// ❌ Avoid — no requirement said this should default; the failure is now invisible.
public function rateLimit(Workspace $workspace): int
{
    try {
        return $this->github->rateLimitFor($workspace);
    } catch (\Throwable) {
        return 5000; // a fabricated number that will mislead every downstream decision
    }
}
```

## Report designed degradation, do not just log it

A specified fallback (the previous section) is *correct* — but it is also *invisible*. Re-read the `SummaryChain`: when Anthropic throws, it `warning()`s and drops to the next rung. The report still ships, the test still passes, the dashboard stays green. Now imagine Anthropic has been 500ing for a week. Every digest has quietly run on the deterministic template the whole time, the AI you pay for has been dead for seven days, and *nobody knows*, because a `warning()` in a log file is something nobody reads until they already suspect a problem.

That is the trap of a well-built fallback: it converts an outage into a non-event. The fix is to make the degradation tell your error tracker, not just your log file. `report($e)` hands the exception to the same pipeline as an uncaught error — Sentry/Flare, the `report` callbacks, the registered context — *without* rethrowing, so the fallback still proceeds. The error tracker groups identical degradations and shows you the frequency: "this provider has failed 4,000 times in seven days" is a page; one buried log line is not.

Pick the right verb for the severity of the event:

- **`throw`** — the failure is unexpected and this code does not know how to continue. Let it propagate to the boundary.
- **`report($e)` then continue** — the failure is *handled* (a specified fallback covers it) but *notable*: it means a paid dependency is down or a quality guarantee is degraded. You want it counted and grouped.
- **`warning()` only** — the event is genuinely mundane and expected at some baseline rate, and nobody needs to be told the count (e.g. a single optional enrichment that is fine to skip).

### ❌ Avoid — a fallback that only logs; the outage is invisible

```php
foreach ($this->providers as $provider) {
    try {
        return $provider->summarise($contributor);
    } catch (Throwable $e) {
        // A week of "Anthropic is down" reduces to log lines nobody greps.
        $this->logger->warning('Summary provider failed; falling back.', [
            'provider' => $provider::class,
            'exception' => $e->getMessage(),
        ]);
    }
}
```

### ✅ Do — `report()` the notable rung, keep the run alive

```php
<?php

declare(strict_types=1);

namespace App\Services\Summaries;

use App\Contracts\SummaryProvider;
use App\Data\Contributor;
use Psr\Log\LoggerInterface;
use Throwable;

final readonly class SummaryChain
{
    /** @param  list<SummaryProvider>  $providers  Ordered best-to-worst; the last MUST NOT throw. */
    public function __construct(
        private array $providers,
        private LoggerInterface $logger,
    ) {}

    public function summarise(Contributor $contributor): string
    {
        $lastIndex = array_key_last($this->providers);

        foreach ($this->providers as $index => $provider) {
            try {
                return $provider->summarise($contributor);
            } catch (Throwable $e) {
                // The terminal rung throwing is a real bug, not a degradation.
                if ($index === $lastIndex) {
                    throw $e;
                }

                // Notable: a paid AI provider just degraded the report's quality.
                // report() surfaces frequency in the error tracker; the run still continues.
                report($e);

                $this->logger->warning('Summary provider failed; falling back.', [
                    'provider' => $provider::class,
                    'contributor' => $contributor->login,
                ]);
            }
        }

        throw SummaryChainExhausted::noTerminalProvider($contributor->login);
    }
}
```

Two things keep this honest. First, the terminal provider throwing is *not* a degradation — there is no further rung — so it rethrows; `report()` is for "we recovered, but you should know". Second, `report()` does **not** rethrow, so it does **not** collide with the "log exactly once" rule: there is exactly one report of this failure and the original `throw`-or-not decision is unchanged. Pair it with the throttling below so a sustained outage counts without flooding.

Edge cases:

- **Don't `report()` client-shaped failures.** A 4xx from a provider because *we* sent a bad request is a bug to throw and fix, not a degradation to count. Reserve `report()`+continue for outages and quality drops you have deliberately chosen to ride through.
- **`report()` is testable.** `Exceptions::fake()` (Laravel 11+) lets a unit test assert `$exceptions->assertReported(AiProviderException::class)` for the middle rung and `assertNotReported(...)` for the success path — add it to the chain's per-rung tests.
- **A bare `report()` with no argument** reports the *current* exception inside a `catch`; prefer the explicit `report($e)` so the call site reads unambiguously.

## Throttle exception reporting

`report()`-on-degradation and "let server errors propagate" are both correct — and both will, during a real outage, fire the *same* exception thousands of times in minutes. The per-minute `reports:dispatch` scheduler fans out one `GeneratePreviewReport` job per workspace; if GitHub or Anthropic is down, every job throws the same way at once. Unthrottled, that is thousands of identical events flooding Sentry/Flare, burning a paid monthly quota in an afternoon and burying the *one* novel error that actually needs you under a wall of duplicates.

Laravel 11+ exposes two controls in `bootstrap/app.php` `withExceptions()`. `dontReportDuplicates()` collapses repeats of the *same exception instance* within a request/job. `throttle()` is the real lever: return a `Lottery` to sample a fixed fraction of a noisy type, or a `Limit` to cap a key at N per minute and drop the rest. You still see that the outage is happening — you just see it at a rate you can afford.

### ❌ Avoid — every job reports every failure, unthrottled

```php
->withExceptions(function (Exceptions $exceptions): void {
    // A GitHub outage during reports:dispatch sends thousands of identical
    // events in one minute and exhausts the Flare quota by lunchtime.
    $exceptions->context(fn (): array => ['team' => config('digest.team')]);
})
```

### ✅ Do — sample the noisy, cap the rest, drop exact duplicates

```php
<?php

use App\Exceptions\AiProviderException;
use App\Exceptions\GitHubRateLimitException;
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Support\Lottery;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    // Collapse repeats of one instance (e.g. retried within a single job).
    $exceptions->dontReportDuplicates();

    $exceptions->throttle(function (Throwable $e): Lottery|Limit|null {
        // Outage-shaped, high-volume: a rate-limit storm only needs sampling.
        if ($e instanceof GitHubRateLimitException) {
            return Lottery::odds(1, 100);
        }

        // A provider outage: cap per type so the tracker shows it without flooding.
        if ($e instanceof AiProviderException) {
            return Limit::perMinute(30)->by($e::class);
        }

        // null => never throttled: novel, low-volume errors always report in full.
        return null;
    });
})
```

`Lottery::odds(1, 100)` reports roughly one in a hundred — right for a type that fires constantly and whose *count* you read from request volume anyway. `Limit::perMinute(30)->by($e::class)` reports up to 30 of that class per minute then drops the rest until the window resets; the `->by()` key keeps two different outages in separate buckets so one cannot starve the other.

Edge cases:

- **Return `null` (or nothing) for the default.** Anything you don't recognise is presumed rare and *novel* — exactly what you must never sample away. Only throttle types you have identified as high-volume.
- **Throttling is about the tracker, not the run.** A throttled exception is still thrown, still fails the job, still hits retry/backoff. You are rate-limiting the *report*, not changing program behaviour.
- **`dontReportDuplicates()` is per request/job lifecycle**, not global — it dedupes the same instance bubbling through nested handlers, not the same *kind* recurring across thousands of jobs. That second case is what `throttle()` is for.

## Observability: carry context across the request→job boundary

Elsewhere we warn hard against ambient request state — reaching for `request()`, the authenticated user, or the resolved tenant from deep inside a service. That rule is right, but it leaves a real gap: when a Livewire action dispatches `GeneratePreviewReport`, the job runs *later*, on a *different* worker, with *no* request. How does a log line written inside that job know which workspace, which user, which original click it belongs to? Threading a `trace_id` through every constructor and method signature by hand is exactly the plumbing we are trying to avoid.

Laravel's `Context` facade is the sanctioned mechanism. `Context::add('trace_id', …)` does two things for free: it appends the key to **every** subsequent log line in this lifecycle, and it **dehydrates into the payload of any job dispatched afterwards**, then rehydrates inside that job — so the worker's logs carry the same `trace_id` as the web request that queued it, with zero plumbing. This is *not* the ambient-state anti-pattern: it is explicit, write-once metadata for correlation, never read back as business input.

### ✅ Do — stamp a trace id at the edge; it follows the work everywhere

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;
use Symfony\Component\HttpFoundation\Response;

final class AssignTraceId
{
    public function handle(Request $request, Closure $next): Response
    {
        // Honour an upstream id (load balancer / front proxy) or mint one.
        $traceId = $request->header('X-Request-Id') ?? (string) Str::uuid();

        // Auto-appended to every log line AND dehydrated into queued jobs.
        Context::add('trace_id', $traceId);

        $response = $next($request);

        // Echo it back so a user-reported error maps straight to the logs.
        $response->headers->set('Request-Id', $traceId);

        return $response;
    }
}
```

Now a log line written deep inside the queued job — on another machine, minutes later — carries the same `trace_id` without a single parameter being passed:

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Contracts\Repositories\WorkspaceRepositoryInterface;
use App\Services\Reports\WorkspaceReportRunner;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\Log;

final class GeneratePreviewReport implements ShouldQueue
{
    public function __construct(private int $workspaceId) {}

    public function handle(
        WorkspaceRepositoryInterface $workspaces,
        WorkspaceReportRunner $runner,
    ): void {
        $workspace = $workspaces->findOrFail($this->workspaceId);

        // trace_id rehydrated automatically; this line correlates with the web request.
        Log::info('Generating preview report.', ['workspace' => $workspace->id]);

        $runner->run($workspace);
    }
}
```

### Carry a secret across the boundary without leaking it

The job needs the tenant's GitHub installation token to build the `WorkspaceReportContext`, and that token must travel with the dispatched job — but a token in a log line is a security incident. `Context::addHidden(...)` stores a value that **dehydrates into the job exactly like visible context** but is **never written to logs**. It carries the secret across the request→job seam without it ever surfacing in your log destination.

```php
// In the action that dispatches the job, after minting the installation token:
Context::add('workspace_id', $workspace->id);          // safe to log
Context::addHidden('installation_token', $token->value); // travels, never logged

GeneratePreviewReport::dispatch($workspace->id);
```

```php
// Inside the job: read the hidden value explicitly to bind the engine seam.
$token = Context::getHidden('installation_token');
$context = WorkspaceReportContext::forInstallation($workspace, $token);
```

### ❌ Avoid — reaching for ambient request state inside the job

```php
public function handle(): void
{
    // request() is null on the queue worker; auth() is the system, not the user.
    // This either fatals or silently attributes the work to the wrong tenant.
    $traceId = request()->header('X-Request-Id');
    $token = auth()->user()->currentWorkspace->installation_token;
}
```

Edge cases:

- **Hidden context still *travels*.** `addHidden` is about log redaction, not transport secrecy — the value is serialized into the queue payload. Treat your queue store with the same care as any secret-bearing channel; this is not a substitute for encryption at rest.
- **Don't reach for `Context` to fetch business inputs.** The job's identity (`$workspaceId`) belongs in its constructor where it is explicit and serialized as a typed argument. `Context` is for cross-cutting *correlation* metadata (trace id, request id) and the occasional carried secret — not for smuggling domain arguments past the type system.
- **Set the response header even on error responses.** Register the middleware globally so a 500's `Request-Id` is the exact key the user quotes when they report the failure.

## Structured logging: format, destination, routing

The rest of this page is about a log's *content* — attach identifiers, log exactly once, never both log and rethrow. That guidance stands unchanged. But content is only half of observability; *where* the line goes and in *what shape* decides whether anyone can act on it. A human-readable line printed to a file is invisible inside a container that ships `stderr` to a log aggregator, and a `CRITICAL` that lands in the same file as a `DEBUG` is a page that nobody is paged for.

Three decisions live in `config/logging.php`, not in the calling code — the call site stays `Log::warning($message, $context)` regardless:

1. **Format.** In a containerized run, emit JSON so the aggregator can index every context key as a field. Laravel ships `Monolog\Formatter\JsonFormatter`; the message and the entire context array become queryable structured fields instead of an unparseable string.
2. **Destination.** Containers expect logs on `stderr`, not a file the orchestrator never reads. Use the `stderr` channel (the `monolog` driver over `php://stderr`) in production, a readable file locally.
3. **Routing by severity.** A `stack` channel fans one write out to several handlers, and Monolog's `level` gates each one — so `Log::critical(...)` reaches Slack while `Log::debug(...)` only ever hits a file.

### ✅ Do — JSON to stderr, a stack, and severity routing

```php
<?php

// config/logging.php

use Monolog\Formatter\JsonFormatter;
use Monolog\Handler\StreamHandler;

return [
    'default' => env('LOG_CHANNEL', 'stack'),

    'channels' => [
        // One write fans out to all listed channels.
        'stack' => [
            'driver' => 'stack',
            'channels' => ['stderr', 'slack'],
            'ignore_exceptions' => false,
        ],

        // Structured JSON on stderr: the container runtime ships it to the aggregator.
        'stderr' => [
            'driver' => 'monolog',
            'level' => env('LOG_LEVEL', 'debug'),
            'handler' => StreamHandler::class,
            'formatter' => JsonFormatter::class,
            'with' => ['stream' => 'php://stderr'],
        ],

        // Severity routing: only critical+ ever pages the on-call Slack channel.
        'slack' => [
            'driver' => 'slack',
            'url' => env('LOG_SLACK_WEBHOOK_URL'),
            'level' => 'critical',
        ],
    ],
];
```

With that stack in place, the severity of the *call* decides the destination, and the calling code never changes:

```php
// Mundane degradation: indexed JSON on stderr, never pages anyone.
Log::warning('Summary provider failed; falling back.', [
    'workspace' => $workspace->id,
    'provider' => $provider::class,
]);

// A delivery invariant broke: same call shape, but this one reaches Slack.
Log::critical('Report delivery dispatched with no channel configured.', [
    'workspace' => $workspace->id,
    'report' => $report->id,
]);
```

### ❌ Avoid — pre-formatted strings to a file the container can't see

```php
// Defeats structured logging: the context is melted into the message,
// so the aggregator can't filter by workspace, and a single-file 'daily'
// channel inside a container is written to a path nobody ever reads.
Log::channel('daily')->warning(
    "Summary provider {$provider} failed for workspace {$workspace->id}",
);
```

Edge cases:

- **`env()` belongs in `config/logging.php` and nowhere else.** Config files are the one sanctioned place to read `env()`; every call site uses `config()`/the channel name. (See [Configuration & secrets](config-and-secrets.md).)
- **Match the format to the runtime.** JSON-to-stderr is correct in production containers and miserable to read while developing; key it off the channel so local stays human-readable and prod stays machine-parseable.
- **`'ignore_exceptions' => false`** on the stack means a broken Slack webhook surfaces instead of silently dropping your logs — fail loudly, even in the logging layer.
- **Severity is routing, not decoration.** Reserve `critical`/`emergency` for things that should page a human; if everything is `error`, the Slack handler becomes noise and gets muted, and you are back to no alerting.

## Client errors vs server errors

These are two different categories with two different correct responses, and conflating them produces both noise and blind spots.

- **Client error** — the caller sent something invalid or unauthorised. The system is healthy. Respond **4xx**. Do **not** report it to error tracking; a 422 is not a bug. Use Form Request validation and `abort()`.
- **Server error** — the system itself is broken (a dependency failed, an invariant was violated). Respond **5xx**, let the typed exception propagate, and **report it** to your error tracker.

### ✅ Do — client errors via Form Request and `abort()`

```php
final class ScheduleDigestRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('manage', $this->route('workspace'));
    }

    /** @return array<string, mixed> */
    public function rules(): array
    {
        return [
            'days' => ['required', 'integer', 'min:1', 'max:30'],
            'channel' => ['required', 'string', 'starts_with:#'],
        ];
    }
}
```

Validation failure → automatic 422; failed `authorize()` → automatic 403. Neither pollutes error tracking. Use `abort()` for authorization-shaped checks outside a Form Request:

```php
public function show(Workspace $workspace, Report $report): View
{
    abort_unless($report->workspace_id === $workspace->id, 404);

    return view('reports.show', ['report' => $report]);
}
```

### ✅ Do — let server errors become reported 5xx

```php
public function handle(): int
{
    // If the token can't be minted, this typed exception propagates,
    // Laravel renders a 500, and the handler reports it to error tracking.
    $token = $this->tokenFactory->mint($this->installationId);

    $this->collector->run($token);

    return self::SUCCESS;
}
```

Register the line between the two categories once, centrally, so the rest of the codebase doesn't repeat the policy. In Laravel 11+ this lives in `bootstrap/app.php`:

```php
->withExceptions(function (Exceptions $exceptions): void {
    // Client-shaped failures are expected; don't page anyone.
    $exceptions->dontReport([
        ValidationException::class,
        AuthorizationException::class,
        WebhookSignatureException::class,
    ]);

    // Enrich every reported (server) exception with shared context.
    $exceptions->context(fn (): array => [
        'team' => config('digest.team'),
    ]);
})
```

## Log with context, exactly once

Two failure modes here, both common, both wrong:

1. **Logging without context.** `Log::error('failed')` tells you nothing. Always attach the identifiers needed to reproduce: the workspace, the repo, the installation id, the status code.
2. **Logging *and* rethrowing the same exception.** If you log and then rethrow, the next handler logs it again — you get the same failure twice in your tracker, with double the alert noise and no added information. Pick one: **handle here and log here**, or **let it propagate and log at the boundary**. The framework's exception handler already logs uncaught exceptions, so prefer propagation.

### ❌ Avoid — log-and-rethrow (double reporting)

```php
try {
    $this->slack->post($webhook, $report);
} catch (SlackDeliveryException $e) {
    Log::error('Slack delivery failed', ['error' => $e->getMessage()]);
    throw $e; // the global handler will log this AGAIN
}
```

### ✅ Do — propagate and let the boundary log

```php
// Throws SlackDeliveryException on failure; the framework's handler
// reports it once, with the context we registered in bootstrap/app.php.
$this->slack->post($webhook, $report);
```

### ✅ Do — when you genuinely handle here, log here and stop

```php
try {
    $this->slack->post($webhook, $report);
} catch (SlackDeliveryException $e) {
    // We are HANDLING this — the run continues without Slack — so log once and do not rethrow.
    $this->logger->warning('Slack delivery skipped; report still emailed.', [
        'workspace' => $workspace->id,
        'channel' => $workspace->slackChannel,
        'exception' => $e->getMessage(),
    ]);
}
```

(Only do the above if "continue without Slack" is a *specified* behaviour — otherwise let it propagate.)

## Untrusted input fails closed

For anything you did not generate — webhooks, callbacks, signed URLs, third-party payloads — the default decision on any doubt is **reject**. A signature you cannot verify, a payload you cannot parse, a timestamp outside tolerance: treat all of them as hostile and `abort()`. "Fail open" (process it anyway because verification was inconvenient) is how forged requests get in.

### ✅ Do — verify the signature before touching the body

```php
final readonly class VerifyGitHubSignature
{
    public function __construct(private string $secret) {}

    public function handle(Request $request, Closure $next): Response
    {
        $signature = $request->header('X-Hub-Signature-256');

        // Guard: no signature at all -> reject, do not "assume trusted".
        if ($signature === null) {
            abort(401, 'Missing webhook signature.');
        }

        $expected = 'sha256=' . hash_hmac('sha256', $request->getContent(), $this->secret);

        // Constant-time compare; any mismatch fails closed.
        if (! hash_equals($expected, $signature)) {
            abort(401, 'Invalid webhook signature.');
        }

        return $next($request);
    }
}
```

Two details that matter: use `hash_equals` (constant-time) rather than `===` to avoid timing attacks, and read the **raw** request body for the HMAC, not the parsed/normalised array. Only after the signature passes do you decode and act on the payload.

### ❌ Avoid — fail open

```php
if ($signature !== null && ! hash_equals($expected, $signature)) {
    abort(401);
}
// BUG: a request with NO signature skips the check entirely and is processed.
```

The missing-header case must be its own guard. Absence of proof is not proof.

## See also

- [Configuration & secrets](config-and-secrets.md) — where webhook secrets and API keys come from, why it's `config()` not `env()`, and the one place (`config/logging.php`) `env()` is allowed.
- [Tooling & CI](tooling-and-ci.md) — wiring the error tracker (Sentry/Flare), log destinations per environment, and what the pipeline does with reported failures.
- [Jobs, commands & scheduling](jobs-commands-scheduling.md) — the `reports:dispatch` fan-out and `GeneratePreviewReport` job whose failures the throttling and `Context` correlation above are built for.
- [Testing](testing.md) — how to test thrown exceptions, the fallback chain, `Exceptions::fake()` reporting assertions, and fail-closed webhook handling.
