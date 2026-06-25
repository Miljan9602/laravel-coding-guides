# DRY & reuse

Don't Repeat Yourself, applied with judgement. The goal is a single source of truth for any non-trivial piece of behaviour — but only once that behaviour is genuinely the same in every place it appears, not merely similar on the surface.

## Rules at a glance

- Extract repeated **behaviour** into one Action or Service; never copy-paste a procedure into a second caller.
- A piece of domain knowledge (an encryption scheme, an OAuth state contract, a "create pending report then dispatch" sequence) lives in exactly one class.
- Share multi-step orchestration through a single **runner** that both the synchronous and queued entrypoints call.
- DRY your tests too: factories for state, helper functions for arrange/assert, shared fakes for collaborators.
- Apply the **rule of three** — duplicate twice, extract on the third occurrence — and let the third case prove the abstraction is real.
- Prefer duplication over the **wrong abstraction**. Coupling unrelated code because it *looks* alike is more expensive than two honest copies.
- DRY values and contracts (constants, enums, config keys), not just code paths.

## Extract repeated behaviour into ONE Action or Service

### Rule

When the same sequence of steps appears in two callers, move it into a single Action (for a use case) or Service (for a reusable capability) and call that. The callers should describe *what* they want, not re-implement *how*.

### Why

Copy-pasted procedures drift. The second copy gets a bug fix the first never receives; a security tweak lands in one OAuth flow but not the other. A single owner of the logic means one place to fix, one place to test, and one place a reviewer must understand. It also makes the call sites read like intent rather than mechanics.

### How: one `WorkspaceOAuthState` service for every OAuth flow

The GitHub-App installation flow and the Slack install flow both need to round-trip a `state` parameter through the provider: encrypt it before redirecting, then decrypt and verify it on the callback. The *payload* differs (one carries an installation target, the other a Slack team), but the *mechanism* — sign, encrypt, bind to the workspace, enforce a TTL — is identical. That mechanism is the thing to share.

❌ Avoid — the encrypt/verify logic duplicated, one copy per provider:

```php
// GitHubAppController
$state = Crypt::encryptString(json_encode([
    'workspace_id' => $workspace->id,
    'nonce'        => Str::random(40),
    'expires_at'   => now()->addMinutes(10)->timestamp,
]));
// ...and the inverse, hand-rolled again, in the callback...

// SlackInstallController — the same code, pasted and lightly edited
$state = Crypt::encryptString(json_encode([
    'workspace_id' => $workspace->id,
    'nonce'        => Str::random(32),          // drifted: 32 vs 40
    'expires_at'   => now()->addMinutes(15)->timestamp, // drifted: 15 vs 10
]));
```

The TTL and nonce length have already diverged, and only one controller will remember to reject an expired state.

✅ Do — a single service owns the contract; both flows pass a typed payload:

```php
<?php

declare(strict_types=1);

namespace App\Services\OAuth;

use App\Data\OAuthState;
use App\Exceptions\InvalidOAuthStateException;
use Illuminate\Contracts\Encryption\Encrypter;
use Illuminate\Support\Str;

final readonly class WorkspaceOAuthState
{
    public function __construct(
        private Encrypter $encrypter,
        private int $ttlMinutes = 10,
    ) {}

    public function issue(int $workspaceId, string $provider): string
    {
        $state = new OAuthState(
            workspaceId: $workspaceId,
            provider: $provider,
            nonce: Str::random(40),
            expiresAt: now()->addMinutes($this->ttlMinutes)->timestamp,
        );

        return $this->encrypter->encrypt($state->toArray());
    }

    public function verify(string $token, string $expectedProvider): OAuthState
    {
        try {
            $state = OAuthState::fromArray($this->encrypter->decrypt($token));
        } catch (\Throwable $e) {
            throw new InvalidOAuthStateException('State token could not be decrypted.', previous: $e);
        }

        if ($state->provider !== $expectedProvider) {
            throw new InvalidOAuthStateException('State provider mismatch.');
        }

        if ($state->expiresAt < now()->timestamp) {
            throw new InvalidOAuthStateException('State token has expired.');
        }

        return $state;
    }
}
```

```php
// GitHubAppController::redirect()
$state = $this->oauthState->issue($workspace->id, provider: 'github_app');

// SlackInstallController::callback()
$state = $this->oauthState->verify($request->string('state'), expectedProvider: 'slack');
$workspace = Workspace::findOrFail($state->workspaceId);
```

One TTL, one nonce policy, one expiry check, tested once. A fix to the expiry logic protects every provider that calls `verify`.

### How: one `RequestReport` action for three screens

The dashboard "Run now" button, the digest scheduler, and the catch-up screen all do the same use case: create a pending `Report` row and dispatch the job that fills it in. Three screens, one use case — so one Action.

❌ Avoid — each entrypoint re-creates the row and re-dispatches the job:

```php
// DashboardController
$report = $workspace->reports()->create(['status' => 'pending', 'kind' => 'standup']);
GenerateReport::dispatch($report);

// CatchUpController — same two lines, easy to let 'status' drift to 'queued'
$report = $workspace->reports()->create(['status' => 'queued', 'kind' => 'catch_up']);
GenerateReport::dispatch($report);
```

✅ Do — the action owns "pending + dispatch"; callers choose only the inputs:

```php
<?php

declare(strict_types=1);

namespace App\Actions\Reports;

use App\Enums\ReportKind;
use App\Enums\ReportStatus;
use App\Jobs\GenerateReport;
use App\Models\Report;
use App\Models\Workspace;
use Carbon\CarbonImmutable;

final readonly class RequestReport
{
    public function handle(
        Workspace $workspace,
        ReportKind $kind,
        CarbonImmutable $since,
        CarbonImmutable $until,
    ): Report {
        $report = $workspace->reports()->create([
            'kind'     => $kind,
            'status'   => ReportStatus::Pending,
            'since'    => $since,
            'until'    => $until,
        ]);

        GenerateReport::dispatch($report);

        return $report;
    }
}
```

```php
// Dashboard "Run now"
$report = $this->requestReport->handle($workspace, ReportKind::Standup, $since, $until);

// Digest scheduler
$report = $this->requestReport->handle($workspace, ReportKind::Digest, $weekStart, $weekEnd);

// Catch-up screen
$report = $this->requestReport->handle($workspace, ReportKind::CatchUp, $lastSeen, now()->toImmutable());
```

The status enum, the relationship, and the dispatch are decided once. If pending reports later need an idempotency key or a rate-limit check, you add it in `RequestReport` and all three screens inherit it.

## Share a runner for multi-step orchestration

### Rule

When a long sequence of steps must run from more than one trigger — a preview request and a scheduled cron, say — put the sequence in a single **runner** object. The triggers become thin shells whose only job is to assemble inputs and call `run()`.

### Why

Orchestration is where copy-paste hurts most: it is long, it has ordering constraints, and the two triggers are tempting to evolve independently. A shared runner guarantees the preview a user sees and the digest that ships overnight were produced by *exactly* the same pipeline — collect, aggregate, summarise, build blocks. Divergence here means "the preview looked fine but the real one was wrong," the worst kind of bug to debug.

### How: `WorkspaceReportRunner` behind both jobs

❌ Avoid — the preview job and the scheduled job each wire up the pipeline:

```php
// PreviewReportJob
$activity = $this->collector->collect($workspace, $since, $until);
$report   = $this->aggregator->aggregate($activity);
$report   = $this->summariser->summarise($report);   // present here...

// ScheduledDigestJob — pasted, then the summarise step quietly omitted
$activity = $this->collector->collect($workspace, $since, $until);
$report   = $this->aggregator->aggregate($activity);
// ...missing here — the shipped digest has no AI summaries
```

✅ Do — one runner; both jobs delegate to it:

```php
<?php

declare(strict_types=1);

namespace App\Services\Reports;

use App\Data\Report;
use App\Enums\ReportKind;
use App\Models\Workspace;
use App\Services\Aggregators\ReportAggregator;
use App\Services\Collectors\MultiOrgCollector;
use App\Services\Summaries\ReportSummariser;
use Carbon\CarbonImmutable;

final readonly class WorkspaceReportRunner
{
    public function __construct(
        private MultiOrgCollector $collector,
        private ReportAggregator $aggregator,
        private ReportSummariser $summariser,
    ) {}

    public function run(
        Workspace $workspace,
        ReportKind $kind,
        CarbonImmutable $since,
        CarbonImmutable $until,
    ): Report {
        $activity = $this->collector->collect($workspace, $since, $until);
        $report   = $this->aggregator->aggregate($activity, $kind);

        return $this->summariser->summarise($report);
    }
}
```

```php
final class PreviewReportJob implements ShouldQueue
{
    public function handle(WorkspaceReportRunner $runner): void
    {
        $report = $runner->run($this->workspace, ReportKind::Standup, $this->since, $this->until);
        Cache::put($this->previewKey(), $report, now()->addMinutes(15));
    }
}

final class ScheduledDigestJob implements ShouldQueue
{
    public function handle(WorkspaceReportRunner $runner, SlackNotifier $slack): void
    {
        $report = $runner->run($this->workspace, ReportKind::Digest, $this->weekStart, $this->weekEnd);
        $slack->send($this->workspace, $report);
    }
}
```

The jobs differ only in what they do with the finished `Report` (cache it vs. ship it). The pipeline itself — and any future step inserted into it — exists once.

## DRY in tests

### Rule

Tests repeat themselves more than any other code. Factor the repetition with the three standard tools: **factories** for model state, **helper functions** for arrange/assert blocks, and **shared fakes** for collaborators. Keep each test's *unique* intent on screen; push the boilerplate out of sight.

### Why

A test suite full of near-identical setup is brittle: change one column and you edit forty tests. Worse, the duplication hides the one line that actually differs between two cases, so readers can't tell what each test is really checking. Shared setup makes the variable part obvious and the schema change a one-file edit.

### How: factories carry the shared state

```php
class WorkspaceFactory extends Factory
{
    protected $model = Workspace::class;

    public function definition(): array
    {
        return [
            'name'     => $this->faker->company(),
            'timezone' => 'Europe/Belgrade',
            'orgs'     => ['NIMA-Labs'],
        ];
    }

    public function withSlack(): static
    {
        return $this->state(fn (): array => ['slack_webhook_url' => 'https://hooks.slack.test/x']);
    }
}
```

```php
it('ships the digest to Slack when configured', function (): void {
    $workspace = Workspace::factory()->withSlack()->create();
    // ...the test states only what makes it different: Slack is configured.
});
```

### How: helper functions for repeated arrange/assert

Pest's `Pest.php` is the home for project-wide helpers. Name them for the question they answer.

```php
// tests/Pest.php
function fakeActivity(int $commits = 3, int $pullRequests = 1): WorkspaceActivity
{
    return new WorkspaceActivity(
        commits: Commit::factory()->count($commits)->make()->all(),
        pullRequests: PullRequest::factory()->count($pullRequests)->make()->all(),
    );
}

function assertReportHasContributor(Report $report, string $login): void
{
    expect($report->contributors->pluck('login'))->toContain($login);
}
```

```php
it('merges contributors across repos', function (): void {
    $report = app(ReportAggregator::class)->aggregate(fakeActivity(), ReportKind::Standup);

    assertReportHasContributor($report, 'octocat');
});
```

### How: one shared fake per collaborator

Build the fake GitHub client and fake AI provider once, bind them in a `beforeEach`, and let every test reuse them. Don't re-stub the same network shape in twenty files.

```php
// tests/Pest.php — bound centrally, configured per test
function fakeGitHub(): FakeGitHubClient
{
    $client = new FakeGitHubClient();
    app()->instance(GitHubClient::class, $client);

    return $client;
}
```

```php
it('collects commits for the workspace orgs', function (): void {
    $github = fakeGitHub();
    $github->withCommits('NIMA-Labs/digest', count: 5);

    $activity = app(MultiOrgCollector::class)->collect($workspace, $since, $until);

    expect($activity->commits)->toHaveCount(5);
});
```

The fake is one class; the per-test calls (`withCommits(...)`) express only this test's scenario. See [Testing](testing.md) for the full fake-vs-mock guidance.

## The counter-rule: avoid the wrong abstraction

### Rule

Do **not** unify two pieces of code merely because they currently look alike. Wait for the **rule of three**, confirm the cases share a *reason to change*, and prefer leaving honest duplication in place over inventing a parameterised super-method that serves no one well.

### Why

The wrong abstraction is more costly than duplication. Duplication is visible and local; you can see both copies and change one. A premature abstraction is invisible coupling: two callers now share a method, so a change demanded by one forces a `bool $isDigest` flag, then a second flag, then a branch, until the "shared" method is a thicket of conditionals that no one can safely touch. Removing duplication is mechanical; unwinding a bad abstraction means untangling every caller that grew to depend on it.

A reliable signal: if making the shared code fit a new caller means *adding a parameter that only one caller ever passes*, the abstraction is wrong. Back it out — re-inline — and let the cases live apart.

❌ Avoid — two genuinely different formats forced through one method by a flag:

```php
final readonly class MessageFormatter
{
    // Slack and email only look similar. The flags multiply with every difference.
    public function format(Report $report, bool $forSlack, bool $includeAvatars, bool $truncate): string
    {
        if ($forSlack) {
            $text = $this->slackHeader($report);
            $text .= $truncate ? $this->truncated($report) : $this->full($report);
            // ...email never reaches here, yet must pass these flags...
        } else {
            $text = $this->emailHeader($report);
        }

        return $text;
    }
}
```

✅ Do — let the two outputs be two classes; share only what is truly common:

```php
final readonly class SlackBlockBuilder
{
    public function build(Report $report): array
    {
        // Slack-specific: blocks, 50-block cap, mrkdwn truncation.
    }
}

final readonly class ReportMailRenderer
{
    public function render(Report $report): string
    {
        // Email-specific: a Blade view, full prose, inline avatars.
    }
}
```

If both later need the same "humanise a commit count" helper, extract *that one genuine commonality* into a small shared function — not the whole formatting flow.

### Knowing when NOT to DRY

- **Coincidental duplication.** Two validation rule sets that happen to read the same today but answer to different stakeholders (a Slack webhook URL vs. a GitHub org slug) will diverge. Keep them separate.
- **Different rates of change.** If two snippets change for unrelated reasons, sharing them couples those reasons. The single source of truth principle is about *one reason to change*, not *one set of characters*.
- **Test clarity over test DRY.** A test that inlines its assertion is sometimes clearer than one that hides intent behind a helper. DRY tests where setup is noise; keep the *meaning* explicit.
- **Premature generalisation.** A method with three boolean parameters and a `match` over a `$mode` string is duplication wearing a disguise. Two readable copies beat one unreadable union.

The discipline is symmetric: extract the moment a third real case proves the behaviour is shared (the OAuth state, the report request, the runner) — and resist extracting when the similarity is skin-deep.

## See also

- [Actions](actions.md)
- [Services](services.md)
- [Repositories](repositories.md)
- [Testing](testing.md)
