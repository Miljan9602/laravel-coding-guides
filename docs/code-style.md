# Code style & language rules

The mechanical, non-negotiable rules that every PHP file in a Laravel project must
follow. These are enforced by Pint and CI, not by reviewers' goodwill — a diff that
violates them does not merge.

## Rules at a glance

- `declare(strict_types=1);` is the first statement in **every** PHP file.
- Every concrete class is `final`. Only abstract base classes and interfaces are non-final.
- Any class with no mutable instance state is `final readonly class` — DTOs, value objects, stateless services, actions, repositories.
- Dependencies are injected via constructor property promotion as `private readonly`.
- Every property, parameter, and return value is typed — including `void`, `?T`, and union types.
- One class per file; PSR-4 namespacing; imports ordered, deduplicated, and used.
- Import every class — no leading-slash FQNs inside method bodies.
- Short array syntax `[]`; trailing commas in all multiline arrays, arguments, and parameter lists.
- Single-quoted strings unless interpolation or escapes require double quotes.
- A parameter that defaults to `null` is typed as nullable (`?T`).
- Names are descriptive: plural for collections (`$activeWorkspaces`), singular for objects (`$activeWorkspace`).
- Comments explain **why**, never restate the code.
- Outside `config/*`, read configuration with `config()`, never `env()`.

These are mechanical. Run `composer lint` before pushing; CI runs it too. See
[Tooling & CI](tooling-and-ci.md) for the exact `pint.json` ruleset and how the
check is wired.

## Strict types in every file

### Rule

The first statement of every PHP file — before the namespace, before anything —
is `declare(strict_types=1);`.

### Why

Without it, PHP silently coerces scalars at function boundaries: passing the
string `"7"` to an `int $days` parameter just works, and passing `"7 days"`
quietly becomes `7`. Strict types turn these into `TypeError`s at the call site,
where the bug actually is. In a reporting tool that pages people based on
GitHub activity counts, a silent string-to-int coercion is the difference between
a correct digest and a wrong one shipped to a customer.

### How

```php
<?php

declare(strict_types=1);

namespace App\Services\Reporting;

final readonly class ReportWindow
{
    public function __construct(
        public int $days,
    ) {}
}
```

```php
// ✅ Do — the declaration is the very first statement
<?php

declare(strict_types=1);
```

```php
// ❌ Avoid — missing entirely, or placed after the namespace (a fatal error)
<?php

namespace App\Services\Reporting;

declare(strict_types=1); // too late: this is a compile error
```

This applies to *every* file: classes, config files, migrations, route files,
test files. There are no exemptions.

## `final` on every concrete class

### Rule

Mark every concrete class `final`. The only non-final classes are abstract base
classes (which exist to be extended) and interfaces (which have no
implementation to inherit).

### Why

`final` by default makes inheritance an explicit, deliberate decision rather than
an accident. Open classes invite subclassing as a shortcut, which couples
unrelated code to internal implementation details and makes refactoring a
breaking change. When you genuinely need polymorphism, an interface or an
abstract base expresses it more honestly than an inheritable concrete class.
Composition and dependency injection cover the rest. Removing `final` later is a
one-line, non-breaking change; adding it later can break consumers, so default to
the safe direction.

### How

```php
// ✅ Do — concrete class is final
final class SlackNotifier
{
    public function __construct(
        private readonly SlackClient $client,
    ) {}

    public function send(Report $report): void
    {
        $this->client->postBlocks($report->toSlackBlocks());
    }
}
```

```php
// ✅ Do — an abstract base is not final; it is meant to be extended
abstract class ActivityCollector
{
    abstract public function collect(Workspace $workspace): Collection;
}

final class CommitCollector extends ActivityCollector
{
    public function collect(Workspace $workspace): Collection
    {
        // ...
    }
}
```

```php
// ❌ Avoid — an extensible concrete class with no reason to be open
class SlackNotifier
{
    public function send(Report $report): void { /* ... */ }
}
```

Interfaces are never `final` (the keyword is not even legal on them); they are the
designed extension point.

## `final readonly class` for stateless and value classes

### Rule

If a class has no mutable instance state — its properties are set once in the
constructor and never change — declare it `final readonly class`. This covers
DTOs, value objects, stateless services, actions, and repositories.

### Why

A `readonly` class guarantees immutability at the type level: every declared
property is implicitly `readonly`, so the class cannot be mutated after
construction. Immutable objects are safe to share, cache, and reason about — a
`Report` you pass to the Slack notifier cannot be changed out from under the email
notifier. For services and repositories the same guarantee documents intent:
these objects hold only injected collaborators, never per-request mutable state,
so they are trivially reusable and thread-safe to share via the container.

### Readonly-class semantics and the default-value gotcha

Declaring the class `readonly` makes all instance properties `readonly`
automatically — you do not repeat the keyword on each property. The important
constraint: **a `readonly` property cannot have a default value.** This is illegal:

```php
// ❌ Avoid — fatal: readonly property may not have a default value
final readonly class ReportOptions
{
    protected string $format = 'slack';
}
```

The fix is to assign every value through the constructor, which is exactly why
constructor property promotion pairs so naturally with `readonly` classes:

```php
// ✅ Do — supply the "default" via a promoted parameter default
final readonly class ReportOptions
{
    public function __construct(
        public string $format = 'slack',
        public int $contributorCap = 25,
    ) {}
}
```

A constructor *parameter* may have a default; a promoted property gets its value
from that parameter, so this is well-formed.

### How

```php
// ✅ Do — DTO as an immutable value class
final readonly class Contributor
{
    /** @param list<string> $repositories */
    public function __construct(
        public string $login,
        public int $commits,
        public int $pullRequests,
        public array $repositories,
    ) {}

    public function totalActivity(): int
    {
        return $this->commits + $this->pullRequests;
    }
}
```

```php
// ✅ Do — stateless repository: only injected collaborators, no mutable state
final readonly class WorkspaceRepository
{
    public function __construct(
        private GitHubClient $github,
    ) {}

    public function findByOrg(string $org): ?Workspace
    {
        $payload = $this->github->organization($org);

        return $payload === null
            ? null
            : Workspace::fromPayload($payload);
    }
}
```

```php
// ✅ Do — stateless action
final readonly class RunReport
{
    public function __construct(
        private ReportAggregator $aggregator,
        private SlackNotifier $slack,
    ) {}

    public function handle(ReportWindow $window): Report
    {
        $report = $this->aggregator->aggregate($window);
        $this->slack->send($report);

        return $report;
    }
}
```

A class that *does* carry mutable per-instance state — for example a builder that
accumulates Slack blocks across method calls — is `final` but not `readonly`:

```php
// ✅ Do — final, but not readonly, because $blocks mutates
final class SlackBlockBuilder
{
    /** @var list<array<string, mixed>> */
    private array $blocks = [];

    public function header(string $text): self
    {
        $this->blocks[] = ['type' => 'header', 'text' => $text];

        return $this;
    }

    /** @return list<array<string, mixed>> */
    public function build(): array
    {
        return $this->blocks;
    }
}
```

## Constructor property promotion for dependencies

### Rule

Inject dependencies through the constructor and promote them to `private readonly`
properties in the parameter list. Do not declare a separate property and assign it
in the body.

### Why

Promotion removes the threefold repetition (property declaration, parameter,
assignment) that hides the actual logic. `private readonly` states the two facts
that matter about an injected collaborator: nothing outside the class touches it,
and nothing reassigns it after construction. The result is the smallest possible
surface for a dependency.

### How

```php
// ✅ Do
final readonly class DigestService
{
    public function __construct(
        private GitHubClient $github,
        private AiClient $ai,
        private SlackNotifier $slack,
    ) {}
}
```

```php
// ❌ Avoid — manual, verbose, and loses the readonly guarantee
final class DigestService
{
    private GitHubClient $github;
    private AiClient $ai;
    private SlackNotifier $slack;

    public function __construct(GitHubClient $github, AiClient $ai, SlackNotifier $slack)
    {
        $this->github = $github;
        $this->ai = $ai;
        $this->slack = $slack;
    }
}
```

Public promoted properties are appropriate for value classes whose data is the
public contract (see `Contributor` above); for injected services, use `private`.

## Type everything

### Rule

Every property, every parameter, and every return value carries a declared type.
That includes `void` for methods that return nothing, `?T` for nullable values,
and explicit union types where a value really is one of several types.

### Why

Types are the cheapest documentation and the first line of defence. With
`strict_types` on, they are also enforced at runtime, so a typed boundary is a
contract the engine checks for free. Untyped code defers every type question to a
human reader and to production.

### How

```php
// ✅ Do — typed property, params, nullable return, and void
final readonly class ReportAggregator
{
    public function __construct(
        private int $contributorCap,
    ) {}

    /** @param list<Contributor> $contributors */
    public function topContributor(array $contributors): ?Contributor
    {
        return collect($contributors)
            ->sortByDesc(fn (Contributor $c): int => $c->totalActivity())
            ->first();
    }

    public function assertWithinCap(int $count): void
    {
        if ($count > $this->contributorCap) {
            throw new TooManyContributorsException($count, $this->contributorCap);
        }
    }
}
```

```php
// ✅ Do — union type when the value genuinely varies
public function durationLabel(int|string $window): string
{
    return is_int($window) ? "{$window} days" : $window;
}
```

```php
// ❌ Avoid — untyped parameter and missing return type
public function topContributor($contributors)
{
    return collect($contributors)->first();
}
```

Generic shapes that PHP cannot express (the element type of an array, for example)
go in a docblock `@param`/`@return` annotation, as shown above — the native type
stays `array`, and the docblock refines it.

## One class per file, PSR-4, and imports

### Rule

Exactly one class (or interface, trait, enum) per file. The file's location must
match its namespace under PSR-4. Import every external class with `use`, order the
imports, and remove unused ones. Inside method bodies, reference classes by their
imported short name — never a leading-slash FQN.

### Why

One class per file makes a symbol findable from its name alone and keeps diffs
scoped. PSR-4 lets the autoloader resolve `App\Services\GitHub\RestGitHubClient`
to a single predictable path. Imports gathered at the top give a reader the file's
dependency list at a glance; a `\App\...` reference buried mid-method hides that
dependency and defeats the import ordering Pint maintains.

### How

```php
// ✅ Do — file: app/Services/GitHub/RestGitHubClient.php
<?php

declare(strict_types=1);

namespace App\Services\GitHub;

use App\Contracts\GitHubClient;
use App\Data\Workspace;
use Illuminate\Support\Facades\Http;

final readonly class RestGitHubClient implements GitHubClient
{
    public function organization(string $org): ?array
    {
        return Http::withToken($this->token)
            ->get("https://api.github.com/orgs/{$org}")
            ->json();
    }
}
```

```php
// ❌ Avoid — leading-slash FQN in the body instead of an import
public function organization(string $org): ?array
{
    return \Illuminate\Support\Facades\Http::get(/* ... */)->json();
}
```

```php
// ❌ Avoid — two classes in one file
final class Workspace { /* ... */ }
final class Contributor { /* ... */ } // belongs in its own file
```

Pint's `ordered_imports` and `no_unused_imports` rules keep the `use` block
alphabetised and free of dead entries automatically.

## Arrays, quotes, commas, and nullable defaults

### Rule

Use short array syntax `[]`. Add a trailing comma after the last element of any
multiline array, the last argument of any multiline call, and the last parameter
of any multiline signature. Use single quotes unless the string needs
interpolation or escape sequences. A parameter whose default is `null` is typed
nullable.

### Why

Short syntax is the modern standard and the only one Pint allows. Trailing commas
make adding or removing a line a one-line diff instead of a two-line one and keep
`git blame` honest. Single quotes signal "no interpolation here" and avoid the
reader scanning for a `$`. A `null` default with a non-nullable type is a latent
bug — the type lies about what the parameter can hold.

### How

```php
// ✅ Do — short arrays, trailing commas, single quotes
$blocks = [
    'type' => 'section',
    'fields' => [
        'commits',
        'pull_requests',
    ],
];

$report = $aggregator->aggregate(
    window: $window,
    org: 'NIMA-Labs',
);
```

```php
// ✅ Do — nullable type for a null default
public function summarise(Contributor $contributor, ?string $tone = null): string
{
    return $this->ai->summary($contributor, $tone ?? 'neutral');
}
```

```php
// ✅ Do — double quotes only when interpolating
$url = "https://api.github.com/repos/{$org}/{$repo}/commits";
```

```php
// ❌ Avoid — long array syntax, no trailing comma, needless double quotes,
//            and a non-nullable type with a null default
$blocks = array(
    "type" => "section"
);

public function summarise(Contributor $contributor, string $tone = null): string
{
    // implicit-nullable: deprecated in PHP 8.4
}
```

> Note: in PHP 8.4 an implicitly-nullable parameter (`string $tone = null`) is
> deprecated. Always write the type as `?string`.

## Naming and comments

### Rule

Name variables, properties, and methods for what they hold or do. A collection is
plural; a single object is singular. Comments justify a decision; they never
paraphrase the line below them.

### Why

A name like `$activeWorkspaces` tells the reader it is iterable and what it
contains, so the loop body needs no explanation. A name like `$data` or `$tmp`
forces the reader to reconstruct meaning from usage. Comments that restate code
rot the moment the code changes and become lies; comments that capture *why* —
a non-obvious constraint, a workaround, a business rule — stay valuable.

### How

```php
// ✅ Do — plural collection, singular element, self-explaining names
$activeWorkspaces = $repository->activeForOrg($org);

foreach ($activeWorkspaces as $activeWorkspace) {
    $report = $this->aggregator->aggregate($activeWorkspace->window());
    $this->slack->send($report);
}
```

```php
// ✅ Do — comment explains a non-obvious WHY
// GitHub caps search results at 1000 items, so we page by date range
// rather than offset to capture busy repositories fully.
$commits = $this->github->commitsByDateRange($org, $window);
```

```php
// ❌ Avoid — vague names and comments that restate the code
$d = $repo->get($o); // get the data

foreach ($d as $x) {
    // loop over the data
    $this->slack->send($this->agg->go($x));
}
```

## `config()` over `env()` outside config files

### Rule

Call `env()` only inside files under `config/`. Everywhere else — services,
actions, commands, controllers — read settings through `config()`.

### Why

When the framework caches configuration (`php artisan config:cache`, the default
in production), `env()` calls *outside* config files return `null`, because the
real environment is no longer read at runtime. A service that reads
`env('SLACK_WEBHOOK_URL')` works in local development and silently breaks in
production. Routing every read through `config/digest.php` also gives one
authoritative place to see and document every knob.

### How

```php
// config/digest.php — env() lives here, with sane defaults
return [
    'slack_webhook_url' => env('SLACK_WEBHOOK_URL'),
    'contributor_cap' => (int) env('DIGEST_CONTRIBUTOR_CAP', 25),
];
```

```php
// ✅ Do — read the cached config value in application code
final readonly class SlackNotifier
{
    public function send(Report $report): void
    {
        Http::post(
            config('digest.slack_webhook_url'),
            ['blocks' => $report->toSlackBlocks()],
        );
    }
}
```

```php
// ❌ Avoid — env() outside config: returns null under config:cache
public function send(Report $report): void
{
    Http::post(env('SLACK_WEBHOOK_URL'), [/* ... */]); // null in production
}
```

## See also

- [Tooling & CI](tooling-and-ci.md) — the `pint.json` ruleset and the `composer lint` check that enforce this page.
- [Project structure](architecture.md) — where each class lives under PSR-4.
- [Data objects & DTOs](dtos-value-objects.md) — `final readonly class` value objects in depth.
- [Services & actions](services.md) — stateless, constructor-injected collaborators.
- [Repositories](repositories.md) — the repository pattern and its immutability.
- [Configuration](config-and-secrets.md) — `config()` access patterns and the config-cache contract.
