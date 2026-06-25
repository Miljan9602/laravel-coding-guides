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

- [Configuration & secrets](config-and-secrets.md) — where webhook secrets and API keys come from, and why it's `config()` not `env()`.
- [Testing](testing.md) — how to test thrown exceptions, the fallback chain, and fail-closed webhook handling.
