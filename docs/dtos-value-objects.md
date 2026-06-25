# DTOs & value objects

DTOs (data transfer objects) and value objects are immutable, typed carriers that move structured data across layer
boundaries — collector → aggregator → summariser → delivery — without leaking Eloquent models or untyped arrays. They
encode invariants once, document their shape for the IDE and static analysers, and stay cheap to construct and compare.

## Rules at a glance

- Declare every DTO and value object as `final readonly class`.
- Promote constructor properties as `public readonly`; expose them directly instead of writing pass-through getters.
- Type every parameter and every return value — no untyped `array` where a typed shape is known.
- Add no behaviour beyond derived getters (`label()`, `isEmpty()`) and copy-with-change methods (`withX()`).
- Return a **new instance** from every `withX()`; never mutate `$this` (readonly forbids it anyway).
- Document complex array shapes with `@param`/`@return array{...}` and `list<...>` phpdoc.
- Validate invariants in the constructor and throw — never construct an invalid value.
- Reach for a DTO/value object over an array when you want type-safety, IDE autocompletion, or enforced invariants.
- Reach for an Eloquent model only when the data is persisted; DTOs cross layers and are never saved.
- The only mutable carrier is a short-lived **accumulator**, which is explicitly *not* `readonly` and never escapes its scope.

## Always `final readonly class`

A DTO is a value: two instances with equal fields are interchangeable, and nothing should ever change a field after
construction. `final` prevents subclasses from bolting on state or overriding the shape your callers rely on, and
`readonly` makes immutability a compiler guarantee rather than a convention. Together they let you pass a DTO anywhere
without defensive copying — no caller can mutate it behind your back.

```php
✅ Do
<?php

declare(strict_types=1);

namespace App\Data;

use Carbon\CarbonImmutable;

/**
 * The closed time interval a report covers, e.g. the last 7 days.
 */
final readonly class ReportWindow
{
    public function __construct(
        public CarbonImmutable $start,
        public CarbonImmutable $end,
    ) {
        if ($start->greaterThan($end)) {
            throw new \InvalidArgumentException('Report window start must not be after end.');
        }
    }

    public function label(): string
    {
        return $this->start->isSameDay($this->end)
            ? $this->start->toFormattedDayDateString()
            : "{$this->start->toFormattedDayDateString()} – {$this->end->toFormattedDayDateString()}";
    }

    public function days(): int
    {
        return $this->start->diffInDays($this->end) + 1;
    }
}
```

```php
❌ Avoid
<?php

// Not final: a subclass can add mutable state and break value semantics.
// Not readonly: any collaborator can overwrite ->end and invalidate the window.
// No invariant check: an inverted window (end before start) silently constructs.
class ReportWindow
{
    public \DateTime $start;
    public \DateTime $end;

    public function setEnd(\DateTime $end): void
    {
        $this->end = $end; // mutation — now every holder of this object is affected
    }
}
```

Prefer `CarbonImmutable` over `Carbon` and `DateTimeImmutable` over `DateTime` inside a value object: a mutable date
member quietly defeats the `readonly` guarantee, because callers can mutate the *object the property points at* even
though they can't reassign the property.

## Promote properties; skip pass-through getters

Constructor property promotion collapses the field declaration, the constructor parameter, and the assignment into one
line. Because the properties are `public readonly`, they are safe to read directly — there is nothing a getter would
protect. A `getStart()` that only does `return $this->start;` is noise that hides the rare getter (`label()`) that
actually computes something.

```php
✅ Do
final readonly class ContributorSummary
{
    public function __construct(
        public string $login,
        public int $commits,
        public int $pullRequestsMerged,
        public string $narrative,
    ) {}

    public function totalContributions(): int
    {
        return $this->commits + $this->pullRequestsMerged;
    }
}

$summary = new ContributorSummary('octocat', 12, 3, 'Shipped the Slack delivery path.');
echo $summary->login;                 // direct read, no getter
echo $summary->totalContributions();  // derived getter earns its keep
```

```php
❌ Avoid
final readonly class ContributorSummary
{
    public function __construct(
        private string $login,           // private + getter buys nothing for a value object
        private int $commits,
    ) {}

    public function getLogin(): string
    {
        return $this->login;             // pure ceremony
    }

    public function getCommits(): int
    {
        return $this->commits;
    }
}
```

Use `private readonly` promotion only when a property is genuinely an implementation detail the value derives from but
never exposes — not as a reflex.

## No behaviour beyond derived getters and `withX()`

A DTO carries data; it does not fetch, persist, deliver, or mutate the world. The moment a value object talks to the
database, calls the GitHub API, or sends a Slack message, it stops being a value and becomes a hidden service with
untestable dependencies. The only methods that belong on a DTO are:

1. **Derived getters** — pure functions of the existing fields (`label()`, `days()`, `isEmpty()`, `totalContributions()`).
2. **Copy-with-change methods** — `withX()` that returns a *new* instance with one field replaced.

```php
✅ Do — pure, total, side-effect-free
final readonly class Report
{
    /**
     * @param list<ContributorSummary> $contributors
     */
    public function __construct(
        public ReportWindow $window,
        public string $overallFocus,
        public array $contributors,
    ) {}

    public function isEmpty(): bool
    {
        return $this->contributors === [];
    }

    public function topContributor(): ?ContributorSummary
    {
        return collect($this->contributors)
            ->sortByDesc(fn (ContributorSummary $c): int => $c->totalContributions())
            ->first();
    }
}
```

```php
❌ Avoid — the DTO reaches out and does work
final readonly class Report
{
    public function __construct(
        public ReportWindow $window,
        public array $contributors,
    ) {}

    public function deliver(): void
    {
        // A value object must not know about Slack, HTTP, or queues.
        Http::post(config('digest.slack.webhook_url'), $this->toSlackBlocks());
    }
}
```

Delivery belongs in a `SlackNotifier`; assembly belongs in a `ReportAggregator`. The `Report` only *holds* the result.

## Build a derived copy immutably with `withX()`

You cannot reassign a `readonly` property, so "change one field" means "construct a new instance, copying the rest."
Expose this intent through a named `withX()` method rather than forcing callers to retype the whole constructor. The
method is pure: the original is untouched, and the caller decides whether to keep it.

```php
✅ Do
final readonly class ReportWindow
{
    public function __construct(
        public CarbonImmutable $start,
        public CarbonImmutable $end,
    ) {}

    public function withEnd(CarbonImmutable $end): self
    {
        return new self($this->start, $end);
    }

    public function extendedBy(int $days): self
    {
        return new self($this->start, $this->end->addDays($days));
    }
}

$week  = new ReportWindow($monday, $monday->addDays(6));
$month = $week->extendedBy(23); // $week is unchanged; $month is a new value
```

```php
❌ Avoid
// Cannot work: readonly properties can't be reassigned, and even if they could,
// mutating in place corrupts every other holder of the same instance.
public function withEnd(CarbonImmutable $end): self
{
    $this->end = $end; // Error: Cannot modify readonly property
    return $this;
}
```

For objects with many fields, a single private clone helper keeps the `withX()` methods short while preserving
immutability:

```php
✅ Do — one canonical copy point
final readonly class ContributorSummary
{
    public function __construct(
        public string $login,
        public int $commits,
        public int $pullRequestsMerged,
        public string $narrative,
    ) {}

    public function withNarrative(string $narrative): self
    {
        return $this->copy(narrative: $narrative);
    }

    public function withCommits(int $commits): self
    {
        return $this->copy(commits: $commits);
    }

    private function copy(
        ?string $login = null,
        ?int $commits = null,
        ?int $pullRequestsMerged = null,
        ?string $narrative = null,
    ): self {
        return new self(
            $login ?? $this->login,
            $commits ?? $this->commits,
            $pullRequestsMerged ?? $this->pullRequestsMerged,
            $narrative ?? $this->narrative,
        );
    }
}
```

## Document complex array shapes with phpdoc

PHP's `array` type is opaque — it tells the IDE and PHPStan/Larastan nothing about keys or element types. When a DTO
holds or returns a structured array, annotate it with `array{...}` (fixed keys), `list<...>` (zero-indexed sequence),
or `array<key, value>` (a map). This restores autocompletion, lets static analysis catch typos, and documents the
contract at the boundary where raw GitHub JSON enters your typed world.

```php
✅ Do
final readonly class CommitRecord
{
    public function __construct(
        public string $sha,
        public string $authorLogin,
        public string $message,
    ) {}

    /**
     * Build from a raw GitHub REST commit payload.
     *
     * @param array{
     *     sha: string,
     *     commit: array{message: string},
     *     author: array{login: string}|null
     * } $payload
     */
    public static function fromGitHub(array $payload): self
    {
        return new self(
            sha: $payload['sha'],
            authorLogin: $payload['author']['login'] ?? 'unknown',
            message: $payload['commit']['message'],
        );
    }
}

final readonly class Report
{
    /**
     * @param list<ContributorSummary> $contributors  ordered by contribution count, desc
     */
    public function __construct(
        public ReportWindow $window,
        public array $contributors,
    ) {}

    /**
     * @return array<string, int>  login => total contributions
     */
    public function contributionsByLogin(): array
    {
        return collect($this->contributors)
            ->mapWithKeys(fn (ContributorSummary $c): array => [
                $c->login => $c->totalContributions(),
            ])
            ->all();
    }
}
```

```php
❌ Avoid
final readonly class Report
{
    public function __construct(
        public ReportWindow $window,
        public array $contributors, // array of what? Keyed how? IDE and PHPStan are blind.
    ) {}
}
```

Once you have more than two fixed keys, that is a signal the array wants to *become* a DTO — promote it rather than
documenting an ever-growing `array{...}`.

## When to use a DTO/value object vs an array vs a model

These three carry data, but they answer different questions. Choose deliberately.

### DTO / value object — typed data crossing a layer boundary

Reach for a DTO when a value moves between layers (collector → aggregator → notifier), when you want IDE
autocompletion and static-analysis safety, or when the value has invariants that must always hold. The cost is one
small class; the payoff is that every downstream consumer gets a typed, guaranteed-valid object.

```php
✅ Do
final readonly class StandupResult
{
    /**
     * @param list<ContributorSummary> $contributors
     */
    public function __construct(
        public ReportWindow $window,
        public string $overallFocus,
        public array $contributors,
    ) {}
}

function deliver(StandupResult $result): void
{
    // Autocomplete on $result->window->label(), guaranteed-present overallFocus.
}
```

### Array — small, local, throwaway, or already-shapeless data

An array is right for a short-lived bag of data that never leaves a single method, for genuinely dynamic shapes (raw
decoded JSON before mapping), or for Laravel APIs that expect arrays (config, validated request data, query bindings).
Don't pass a raw array across three layers and re-document its shape in each — map it into a DTO at the boundary.

```php
✅ Do — array stays local, immediately mapped to DTOs
$payloads = $this->github->commits($repo); // raw array<int, array<string,mixed>>

return array_map(CommitRecord::fromGitHub(...), $payloads);
```

```php
❌ Avoid — opaque array threaded through the whole pipeline
function summarise(array $report): string
{
    // What keys exist? No IDE help, no PHPStan, typos fail at runtime in prod.
    return $report['contributors'][0]['narrative'];
}
```

### Eloquent model — persisted state with identity and a lifecycle

A model represents a row that lives in the database, has an identity (primary key), and participates in
relationships, events, and queries. DTOs are the opposite: they are **not** persisted, have no identity, and exist
only to ferry data between layers. A `Report` is computed fresh every run from GitHub data and never saved, so it is
a DTO, not a model. A `Workspace` row that you store, query, and relate is a model.

| Concern | DTO / value object | Eloquent model |
| --- | --- | --- |
| Persisted to a database | No | Yes |
| Has identity (primary key) | No | Yes |
| Mutability | Immutable (`readonly`) | Mutable (dirty tracking, `save()`) |
| Crosses layer boundaries | Yes, freely | Avoid leaking past the repository |
| Equality | By value | By identity / primary key |
| Construction | `new`, always valid | Hydrated from a row or `make()`/`create()` |

```php
✅ Do — a model owns persistence; a method returns a DTO across the boundary
final class Workspace extends Model
{
    /**
     * Snapshot the workspace's report config as an immutable value
     * that the reporting pipeline can consume without touching the DB.
     */
    public function reportWindow(): ReportWindow
    {
        $end = CarbonImmutable::now($this->timezone);

        return new ReportWindow($end->subDays($this->lookback_days), $end);
    }
}
```

Do not invert this by giving a DTO `save()`, relationships, or query scopes — that is a model wearing a DTO's name.

## Mutable accumulators are the rare, explicit exception

Building a `Report` often means tallying counts across a stream of commits and pull requests. A `readonly` object can't
accumulate, and producing a fresh copy per event is wasteful. The sanctioned pattern is a small, **mutable**
accumulator that lives entirely inside one aggregation method and emits an immutable DTO at the end. Make its mutability
loud: it is *not* `final readonly`, and it never escapes the scope that owns it.

```php
✅ Do — mutable on purpose, sealed into an immutable result
<?php

declare(strict_types=1);

namespace App\Services\Aggregators;

use App\Data\ContributorSummary;

/**
 * Mutable accumulator — deliberately NOT readonly. Lives only inside
 * ReportAggregator::aggregate() and is never returned or stored.
 */
final class ContributorTally
{
    private int $commits = 0;
    private int $pullRequestsMerged = 0;

    public function __construct(
        public readonly string $login,
    ) {}

    public function addCommit(): void
    {
        $this->commits++;
    }

    public function addMergedPullRequest(): void
    {
        $this->pullRequestsMerged++;
    }

    public function freeze(string $narrative): ContributorSummary
    {
        return new ContributorSummary(
            $this->login,
            $this->commits,
            $this->pullRequestsMerged,
            $narrative,
        );
    }
}
```

```php
final readonly class ReportAggregator
{
    /**
     * @param list<CommitRecord> $commits
     * @return list<ContributorSummary>
     */
    public function aggregate(array $commits): array
    {
        /** @var array<string, ContributorTally> $tallies */
        $tallies = [];

        foreach ($commits as $commit) {
            $tallies[$commit->authorLogin] ??= new ContributorTally($commit->authorLogin);
            $tallies[$commit->authorLogin]->addCommit();
        }

        return array_map(
            fn (ContributorTally $t): ContributorSummary => $t->freeze(''),
            array_values($tallies),
        );
    }
}
```

```php
❌ Avoid — leaking a mutable accumulator as if it were the result
public function aggregate(array $commits): array
{
    // Returning ContributorTally objects lets callers keep mutating the
    // "result" long after aggregation — the data is never truly settled.
    return array_values($tallies);
}
```

The rule of thumb: mutability is permitted only as a private, in-method optimisation that produces an immutable DTO.
If a mutable object reaches a return type or a property of another class, refactor it to freeze into a `readonly` value
first.

## See also

- [Code style](code-style.md)
- [Models & Eloquent](models-eloquent.md)
- [Repositories](repositories.md)
- [Actions](actions.md)
