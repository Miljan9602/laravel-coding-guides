# Tooling & CI

The standards in this guide are only real if a machine enforces them. This page gives you the complete, copy-pasteable toolchain — Pint for formatting, Composer scripts as the single entry point, and a GitHub Actions workflow as the gate — so that no unformatted, untested, or non-strict code ever reaches `main`.

## Rules at a glance

- **Pint is the source of truth for style.** Use the `laravel` preset plus an explicit rule set; never hand-format.
- **`declare_strict_types` is enforced by Pint** and auto-inserts `declare(strict_types=1);` at the top of every file.
- **`final` is added by hand**, never by the auto `final_class` fixer — that fixer is too blunt and breaks legitimate extension points.
- **Static analysis is enforced by Larastan/PHPStan** at the highest level the codebase sustains; type errors fail the build like a broken test.
- **All entry points are Composer scripts**: `lint`, `lint:test`, `analyse`, `test`, and `check`. Engineers and CI invoke the same commands.
- **`composer check` runs before every commit** — it is `lint:test` + `analyse` + `test` and must be green locally before you push.
- **CI runs on every push and PR**: it checks formatting (`lint:test`), runs static analysis (`analyse`), audits dependencies for known CVEs (`audit`), then runs the full Pest suite.
- **Tests run on SQLite `:memory:` for speed** but a Postgres service is available in CI for migration-fidelity checks.
- **PHP 8.4 everywhere** — local, CI, and production must agree on the runtime.
- **Each enforcement tool owns one axis**: Pint = style, PHPStan = types, Pest `arch()` = architecture (the rules in this guide), Pest = behaviour. A rule with no enforcing tool is a suggestion, not a standard.
- **Security is a CI gate, not an afterthought.** `composer audit` fails the build on a dependency with a known CVE, and `roave/security-advisories` makes Composer *refuse to install* a vulnerable version in the first place.
- **Third-party Actions are pinned to a commit SHA**, never a moving `@v4` tag — a tag can be re-pointed at malicious code; a SHA cannot.
- **The workflow runs least-privilege**: a top-level `permissions: { contents: read }`, escalated per-job only where a job genuinely needs to write.
- **`APP_DEBUG=false` in production**, always — Ignition's debug page leaks environment variables, secrets, and stack traces to anyone who triggers an error.

## Pint: the formatter

[Laravel Pint](https://laravel.com/docs/pint) is a zero-config wrapper around PHP-CS-Fixer with a Laravel-flavoured default. We do not run it zero-config: we pin an explicit `pint.json` so that every machine — laptop, CI, the next engineer's clone — produces byte-identical output. Drift between developers' formatters is a class of merge noise we refuse to tolerate.

### Why an explicit rule set

The `laravel` preset is a moving target across Pint releases, and it omits several rules we consider non-negotiable (strict types, import hygiene, single quotes). By listing rules explicitly on top of the preset, we make our intent legible and version-stable: a reviewer reading `pint.json` knows exactly what is enforced without cross-referencing Pint's changelog.

### The `declare_strict_types` rule

The `declare_strict_types` fixer **automatically inserts `declare(strict_types=1);`** as the first statement of every PHP file. This is why our hand-written examples can open straight into the namespace — Pint backfills the declaration on the first run. You never type `declare(strict_types=1);` defensively hoping it's there; Pint guarantees it.

### Why `final` is NOT auto-fixed

PHP-CS-Fixer ships a `final_class` fixer that marks every non-abstract class `final`. We deliberately **do not enable it**. It is too blunt: it would finalize classes that are legitimate extension points (form requests, base test cases, abstract-but-concrete service bases, Livewire components you intend to subclass). Finalization is a design decision, not a formatting one. We add `final` (and `final readonly` for value classes) **by hand**, and code review checks it. The formatter handles whitespace and imports; humans handle the type system.

### `pint.json`

Place this at the project root.

```json
{
    "preset": "laravel",
    "rules": {
        "declare_strict_types": true,
        "fully_qualified_strict_types": true,
        "ordered_imports": {
            "sort_algorithm": "alpha"
        },
        "no_unused_imports": true,
        "nullable_type_declaration_for_default_null_value": true,
        "single_quote": true,
        "trailing_comma_in_multiline": {
            "elements": ["arrays", "arguments", "parameters"]
        }
    }
}
```

What each rule buys you:

- **`declare_strict_types`** — inserts `declare(strict_types=1);`. No silent type juggling; passing `'7'` where an `int` is expected throws instead of coercing.
- **`fully_qualified_strict_types`** — normalizes how imported types are written in signatures and docblocks, so a `WorkspaceReport` referenced in a return type matches its `use` import rather than appearing as a stray fully-qualified `\App\Reports\WorkspaceReport`.
- **`ordered_imports` (alpha)** — `use` statements sorted alphabetically. Eliminates "where do I add this import" debates and keeps diffs minimal.
- **`no_unused_imports`** — deletes dangling `use` lines, the most common leftover after a refactor.
- **`nullable_type_declaration_for_default_null_value`** — a parameter defaulting to `null` gets a `?` on its type. `string $cursor = null` becomes `?string $cursor = null`, which is what 8.4 actually requires for an explicit nullable.
- **`single_quote`** — single quotes unless interpolation or escapes demand double. Consistent and marginally cheaper to parse.
- **`trailing_comma_in_multiline`** — trailing commas in multi-line arrays, call arguments, and parameter lists. Adding a line to an array or a constructor signature touches one line, not two — cleaner blame and review.

### What clean code looks like under this config

A value class — note the absent `declare(strict_types=1);` in your source becomes present after Pint, the hand-written `final readonly`, the alphabetized imports, and the trailing comma in the promoted constructor.

```php
<?php // declare(strict_types=1); is inserted by Pint on first run

namespace App\Reports\Data;

use App\GitHub\Data\Repository;
use Carbon\CarbonImmutable;

final readonly class WorkspaceReport
{
    public function __construct(
        public Repository $repository,
        public CarbonImmutable $generatedAt,
        public int $mergedPullRequests,
        public int $openPullRequests,
        public ?string $slackThreadTs = null,
    ) {}
}
```

```php
<?php

namespace App\Reports\Actions;

use App\Reports\Data\WorkspaceReport;
use App\Reports\Repositories\ReportRepository;
use App\Slack\SlackNotifier;

final readonly class PublishReport
{
    public function __construct(
        private ReportRepository $reports,
        private SlackNotifier $slack,
    ) {}

    public function handle(WorkspaceReport $report): void
    {
        $this->reports->store($report);
        $this->slack->post($report);
    }
}
```

### Running Pint directly

```bash
./vendor/bin/pint          # format the codebase in place
./vendor/bin/pint --test   # report violations, change nothing, exit non-zero on drift
./vendor/bin/pint -v       # show each file and the rules applied
./vendor/bin/pint app/Reports   # scope to a path
```

`--test` is the CI mode: it never writes files, it just fails the build if anything is mis-formatted.

## Static analysis: Larastan / PHPStan

Pint enforces *style*; it cannot tell you that a method returns `null` where the caller expects a `Report`, that a `@param array` is missing keys, or that you called a method that no longer exists. That is the job of [PHPStan](https://phpstan.org), driven through [Larastan](https://github.com/larastan/larastan) — a PHPStan extension that teaches the analyser Laravel's magic (Eloquent dynamic properties, facades, container resolution, relationship return types). Static analysis is a compile-time type-checker for a language that has no compiler. It is the second leg of the `composer check` tripod, and it catches an entire class of bugs that neither the formatter nor the test suite reliably will.

### Why static analysis, on top of strict types and tests

- **`declare(strict_types=1)` only checks call boundaries at runtime.** PHPStan proves the types line up *without executing the code*, across branches your tests may not cover.
- **It finds the bug your test forgot.** A null-safety gap on an error path, a typo'd array key, an `@return` that lies — PHPStan flags these statically, before the code ships.
- **It keeps the type system honest.** Generics annotations like `Collection<int, Report>` (the ones the repository layer leans on) are only meaningful if something verifies them. PHPStan is that something.
- **It documents intent mechanically.** When the analyser is green at a high level, the type annotations are *true*, not aspirational — so they can be trusted while reading.

### Install

```bash
composer require --dev larastan/larastan
```

Larastan pulls in PHPStan itself as a transitive dependency; you do not require `phpstan/phpstan` separately.

### `phpstan.neon`

Place this at the project root. Larastan's own rule set is included via the package path.

```neon
includes:
    - vendor/larastan/larastan/extension.neon

parameters:
    level: 6

    paths:
        - app
        - config
        - database
        - routes

    # Tighten the screws beyond the level's defaults:
    checkMissingIterableValueType: true
    checkGenericClassInNonGenericObjectType: true
    treatPhpDocTypesAsCertain: false

    ignoreErrors:
        # Example: a framework boundary PHPStan can't model. Pin each
        # ignore to a message + path so it can't silently widen.
        # - message: '#Unsafe usage of new static#'
        #   path: app/Support/SomeBase.php
```

What the key settings buy you:

- **`level: 6`** — PHPStan has ten levels (0–9, plus `max`). Level 6 demands type hints on everything and catches most real defects without drowning a fresh codebase in array-shape noise. **Start where the codebase goes green, then ratchet up one level per PR** until you plateau — a falling error count is a healthier signal than a wall of ignores. New green-field projects should aim for `level: max`.
- **`checkMissingIterableValueType`** — forces `array<int, Report>` over a bare `array`, so collection contents are typed. This is what makes the repository layer's `Collection<int, Report>` annotations enforceable.
- **`treatPhpDocTypesAsCertain: false`** — tells PHPStan not to fully trust a docblock over the real type, surfacing places where an annotation has drifted from reality.
- **`ignoreErrors`** — the escape hatch, used sparingly. Pin every entry to a specific `message` *and* `path` so an ignore can never silently swallow a new, unrelated error. An unpinned baseline of ignores is how a "level 6" project quietly rots to level 2.

### A failure it catches that tests miss

```php
final readonly class ReportRepository implements ReportRepositoryInterface
{
    public function latestStandupForWorkspace(Workspace $workspace): Report
    {
        // ❌ ->first() returns ?Report, but the signature promises Report.
        // PHPStan: "Method ... should return App\Models\Report but returns App\Models\Report|null."
        return Report::query()
            ->where('workspace_id', $workspace->id)
            ->where('type', 'standup')
            ->latest('generated_at')
            ->first();
    }
}
```

A happy-path test that seeds a standup row passes green — the row exists, so `first()` returns a `Report`. PHPStan fails the build anyway, because on an *empty* workspace this method returns `null` and breaks its own contract. Swapping `->first()` for `->firstOrFail()` makes both the analyser and the empty-workspace edge case correct.

### Running it

```bash
composer analyse                       # analyse the configured paths
./vendor/bin/phpstan analyse app/Actions   # scope to a path while iterating
./vendor/bin/phpstan analyse --generate-baseline   # snapshot existing errors (use with care)
```

A baseline freezes today's violations so you can enforce "no *new* errors" on a legacy codebase. Treat it as debt to burn down, never as a place to hide fresh failures — every line added to the baseline is a type hole you have chosen to keep.

## Composer scripts: the single entry point

Engineers should not memorize tool flags, and CI should run the **exact same commands** a developer runs locally. We achieve this by routing everything through Composer scripts. The contract is: if `composer check` is green, your branch is mergeable.

### `composer.json` scripts

Add the `scripts` block to `composer.json` (the rest of the file is elided).

```json
{
    "scripts": {
        "lint": "./vendor/bin/pint",
        "lint:test": "./vendor/bin/pint --test",
        "analyse": "./vendor/bin/phpstan analyse --memory-limit=512M",
        "audit": "composer audit --no-dev=false",
        "test": "php artisan test",
        "check": [
            "@lint:test",
            "@analyse",
            "@test"
        ]
    },
    "scripts-descriptions": {
        "lint": "Format the codebase with Pint (writes files).",
        "lint:test": "Check formatting without writing; fails on drift.",
        "analyse": "Run Larastan/PHPStan static analysis.",
        "audit": "Fail if any installed dependency has a known CVE.",
        "test": "Run the Pest test suite.",
        "check": "Run formatting check + static analysis + tests; the pre-commit gate."
    }
}
```

Notes:

- **`test` uses `php artisan test`.** This is Laravel's test runner, which delegates to Pest (or PHPUnit) and applies the framework's environment bootstrapping. If you prefer to invoke Pest directly, `"./vendor/bin/pest"` is equivalent for this project — pick one and keep it consistent between local and CI.
- **`analyse` runs PHPStan via the Larastan extension** (covered in the next section). `--memory-limit=512M` keeps large analyses from aborting; raise it on bigger codebases.
- **`audit` runs `composer audit`** against the full dependency tree (see *Security as a CI gate* below). It is **deliberately not part of `check`** — the audit result depends on the upstream advisory database, not on your diff, so a freshly-disclosed CVE could turn a developer's local `composer check` red on code they never touched. Dependency health is a *pipeline* concern, gated on its own CI step and a scheduled run; `check` stays a pure function of your branch so "it's green locally" always means "my changes are correct".
- **`check` references other scripts with `@`.** `@lint:test`, `@analyse`, and `@test` invoke the named scripts in order. The array stops at the first non-zero exit, so a formatting failure short-circuits before analysis, and an analysis failure short-circuits before tests — you fix problems cheapest-first.

### Invoking the scripts

```bash
composer lint        # auto-format everything
composer lint:test   # CI-style format check, no writes
composer analyse     # run static analysis
composer audit       # fail on any dependency with a known CVE
composer test        # run the suite
composer check       # lint:test + analyse + test — run this before you commit
```

You can pass extra arguments through to the underlying tool after `--`:

```bash
composer test -- --filter=PublishReport      # run one test
composer test -- --parallel                  # parallelize the suite
```

> **Run `composer check` before every commit.** It is the same gate CI enforces. Catching a formatting slip or a red test locally costs seconds; catching it in CI after a push costs a round-trip and a re-review. If you want it automatic, wire it into a Git `pre-commit` hook or a tool like [GrumPHP](https://github.com/phpro/grumphp) / [Husky](https://typicode.github.io/husky/) — but the script is the contract either way.

## GitHub Actions: the gate

CI exists to make the standards non-optional. A pull request cannot merge unless formatting is clean and the suite is green. The workflow mirrors the local commands exactly — there is no CI-only magic, which means "it passes on my machine" and "it passes in CI" converge.

### Design choices

- **PHP 8.4** with `pdo_pgsql` and `pdo_sqlite` extensions. The matrix is a single version on purpose: we ship one runtime, so we test one runtime.
- **Tests run on SQLite `:memory:`** for speed — an in-memory database is dramatically faster than a network round-trip per test and needs no service to boot. The vast majority of the suite is database-agnostic.
- **A Postgres service is still provisioned** so that migration-fidelity tests (anything exercising Postgres-specific column types, `jsonb`, partial indexes, or `LISTEN/NOTIFY`) have a real engine to run against. Point those specific tests at the `pgsql` connection; leave the rest on SQLite.
- **`composer lint:test` runs before tests.** Formatting is cheap to check and fails fast — no point running a five-minute suite on code that isn't even formatted.

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    name: Lint & Test (PHP 8.4)
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: digest
          POSTGRES_PASSWORD: digest
          POSTGRES_DB: digest_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U digest"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup PHP 8.4
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: pdo_pgsql, pdo_sqlite, mbstring, bcmath
          coverage: none
          tools: composer:v2

      - name: Cache Composer dependencies
        uses: actions/cache@v4
        with:
          path: vendor
          key: composer-${{ hashFiles('composer.lock') }}
          restore-keys: composer-

      - name: Install dependencies
        run: composer install --no-interaction --no-progress --prefer-dist

      - name: Prepare environment
        run: |
          cp .env.example .env
          php artisan key:generate

      - name: Check formatting
        run: composer lint:test

      - name: Static analysis
        run: composer analyse

      - name: Run tests
        run: composer test
        env:
          DB_CONNECTION: sqlite
          DB_DATABASE: ':memory:'
          # Postgres connection available for migration-fidelity specs:
          DB_PGSQL_HOST: 127.0.0.1
          DB_PGSQL_PORT: 5432
          DB_PGSQL_DATABASE: digest_test
          DB_PGSQL_USERNAME: digest
          DB_PGSQL_PASSWORD: digest
```

### Why each step

- **`actions/checkout@v4`** — pulls the branch under test.
- **`shivammathur/setup-php@v2`** — the canonical PHP setup action. We request `pdo_pgsql` (for the Postgres service) and `pdo_sqlite` (for the in-memory suite), plus the usual `mbstring`/`bcmath` Laravel relies on. `coverage: none` keeps the run fast; enable Xdebug/PCOV only on a dedicated coverage job.
- **`actions/cache@v4`** — caches `vendor/` keyed by `composer.lock`, so unchanged dependencies don't reinstall on every run.
- **`composer install`** — `--prefer-dist --no-progress --no-interaction` is the CI-friendly install.
- **Prepare environment** — `cp .env.example .env` then `php artisan key:generate`. Laravel refuses to boot without an `APP_KEY`; generating a throwaway one per run keeps secrets out of the repo.
- **Check formatting** — `composer lint:test`. The build dies here if anything is mis-formatted, before analysis or a single test runs.
- **Static analysis** — `composer analyse`. PHPStan/Larastan fails the build on type defects the formatter and a green happy-path test would both miss.
- **Run tests** — `composer test` against SQLite `:memory:`, with the Postgres connection details exported so migration-fidelity specs can opt into `pgsql`.

### Connecting the Postgres service to config

The workflow exports `DB_PGSQL_*` variables; your `config/database.php` reads them so a second named connection points at the CI service. This keeps the default `sqlite` connection fast while making a real Postgres reachable by name.

```php
// config/database.php — connections array (excerpt)
'pgsql' => [
    'driver' => 'pgsql',
    'host' => env('DB_PGSQL_HOST', '127.0.0.1'),
    'port' => env('DB_PGSQL_PORT', '5432'),
    'database' => env('DB_PGSQL_DATABASE', 'digest'),
    'username' => env('DB_PGSQL_USERNAME', 'digest'),
    'password' => env('DB_PGSQL_PASSWORD', ''),
    'charset' => 'utf8',
    'prefix' => '',
    'search_path' => 'public',
    'sslmode' => 'prefer',
],
```

`env()` is correct **here** — this is a config file, the one place reading the environment directly is sanctioned. Application code calls `config('database.connections.pgsql')`, never `env()`.

A migration-fidelity test then targets the connection explicitly:

```php
<?php

use App\Reports\Models\Report;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('stores jsonb metadata round-trip on postgres', function (): void {
    config(['database.default' => 'pgsql']);

    $report = Report::factory()->create([
        'metadata' => ['merged' => 12, 'repos' => ['digest', 'api']],
    ]);

    expect($report->fresh()->metadata)->toBe([
        'merged' => 12,
        'repos' => ['digest', 'api'],
    ]);
})->group('pgsql');
```

Tag these `->group('pgsql')` so they're easy to run or skip in isolation.

## Security as a CI gate

Pint, PHPStan, and the suite all check *your* code. None of them check the code you *depend on* — and in a Laravel app that is the larger attack surface. A SaaS that holds OAuth tokens, GitHub App installation credentials, and Slack webhooks cannot afford to ship a transitive dependency with a published, exploitable CVE. The cautionary tale is recent and exact: **Livewire shipped a critical remote-code-execution vulnerability, [CVE-2025-54068](https://github.com/livewire/livewire/security/advisories/GHSA-3xq2-wv9p-7h7x)** (property-update payloads could be coerced into arbitrary command execution). Our entire authenticated surface — login, the OAuth connect flow, every `InteractsWithWorkspace` Livewire screen — runs on Livewire. An app with that footprint that does not mechanically gate against a known-vulnerable Livewire release is one `composer update` away from disaster. So dependency security is a build gate, enforced two ways: *detection* (`composer audit`) and *prevention* (`roave/security-advisories`).

### Detection: `composer audit` in the pipeline

`composer audit` cross-references every installed package against the [PHP Security Advisories Database](https://github.com/FriendsOfPHP/security-advisories) and exits non-zero if any installed version is affected. It is a *reactive* check — it tells you the lockfile you already resolved contains something vulnerable.

```bash
composer audit            # audit installed packages; non-zero exit on any advisory
composer audit --locked   # audit composer.lock without a full install (fast CI pre-check)
```

We wire it in as its own CI step (shown in the workflow below) and as the `composer audit` script. Run it on a **schedule** too, not only on push: a CVE disclosed against a dependency you haven't touched in months should turn the pipeline red the morning it lands, not the next time someone happens to open a PR. A nightly `schedule:` trigger on the workflow does exactly that.

### Prevention: `roave/security-advisories` as a rejection list

Detection tells you *after* you've resolved a bad version. Prevention stops it resolving at all. [`roave/security-advisories`](https://github.com/Roave/SecurityAdvisories) is a `require-dev` package that ships **no code** — it is a giant set of `conflict` constraints, one per known-vulnerable release of every package in the advisory database. Because Composer's solver treats those conflicts as hard, `composer require` or `composer update` will **refuse to install** a version with a known CVE: the dependency resolver fails loudly instead of silently pulling in the exploitable release.

```bash
composer require --dev roave/security-advisories:dev-latest
```

```json
{
    "require-dev": {
        "roave/security-advisories": "dev-latest"
    }
}
```

With this in place, an attempt to land a vulnerable Livewire is rejected at resolve time:

✅ **Do** — let the solver block the bad version before it ever enters the lockfile:

```bash
$ composer require livewire/livewire:^3.6.0
# Your requirements could not be resolved to an installable set of packages.
#
#   Problem 1
#     - roave/security-advisories dev-latest conflicts with livewire/livewire 3.6.0
#       (CVE-2025-54068).
#     - Root composer.json requires livewire/livewire ^3.6.0 -> satisfiable only by
#       a version conflicting with roave/security-advisories.
```

❌ **Avoid** — relying on humans to read advisories and a CI step as the *only* line of defence:

```bash
# Without roave/security-advisories, this resolves happily…
$ composer require livewire/livewire:^3.6.0   # installs the CVE'd release
# …and the vulnerability only surfaces later, when `composer audit` runs in CI —
# by which point the bad lockfile is already committed and pushed.
```

The two tools are complementary, not redundant: `roave/security-advisories` is a **prevention** wall at `composer require` time on the developer's machine; `composer audit` is the **detection** backstop in CI that also catches advisories published *after* a version was already installed (a conflict list is only as fresh as your last `composer update` of it). Run both.

### One-line production hardening: `APP_DEBUG=false`

The cheapest security fix in any Laravel app is one environment variable. **`APP_DEBUG` must be `false` in production.** With it `true`, Laravel's debug-error page (Ignition) renders a full stack trace, the resolved request, and — critically — a dump of the environment on any unhandled exception, handing an attacker your `APP_KEY`, database password, `ANTHROPIC_API_KEY`, GitHub App private key, and Slack webhook in a single error. Keep `APP_DEBUG=true` for local development only; assert `false` in your production `.env` and, ideally, fail the deploy if it isn't.

## Harden the GitHub Actions workflow

The workflow itself is code that runs with access to your repository and — via `secrets` — your tokens. A CI pipeline that pulls in third-party Actions by a moving tag, runs with broad write permissions, and never cancels superseded runs is a supply-chain liability bolted onto your security gate. Three changes close the gaps: pin Actions to a commit SHA, run least-privilege, and cancel stale runs.

### Pin third-party Actions to a commit SHA

A tag like `@v4` is a **moving pointer**. If a maintainer's account is compromised (or they retag a release), `@v4` can be silently re-pointed at malicious code that now runs with access to your `GITHUB_TOKEN` and secrets — and your pipeline picks it up on the next run with no diff to review. A 40-character commit SHA is immutable: it can only ever resolve to the exact tree you audited. Pin every third-party Action to a SHA, and leave a trailing comment naming the human-readable version so dependency-update bots (and humans) can read it.

✅ **Do** — pin to a full commit SHA, annotated with the version:

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

  - name: Setup PHP 8.4
    uses: shivammathur/setup-php@cf4cade2721041f288618926317de27f8e2a4c97 # v2.34.1
```

❌ **Avoid** — a moving tag that can be re-pointed under you:

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4          # mutable: @v4 can be retagged at any time

  - name: Setup PHP 8.4
    uses: shivammathur/setup-php@v2    # mutable: trusts the maintainer's account forever
```

> First-party `actions/*` are lower-risk than community Actions, but the rule is uniform: pin *everything* third-party to a SHA. Let Dependabot (`package-ecosystem: github-actions`) propose SHA bumps as reviewable PRs so pinning doesn't mean going stale.

### Least-privilege `permissions`

By default the automatic `GITHUB_TOKEN` is handed to the workflow with broad read/write scopes. A lint-and-test job needs to *read* the repo and nothing more. Declare a restrictive **top-level** `permissions:` so every job starts from least privilege, then escalate per-job only where a job genuinely writes (publishing coverage, commenting on a PR, pushing a tag). If the token is exfiltrated by a compromised Action, `contents: read` sharply limits the blast radius.

✅ **Do** — deny by default at the top, grant narrowly per job:

```yaml
# Top of the workflow: every job inherits read-only unless it overrides.
permissions:
  contents: read

jobs:
  check:
    # inherits contents: read — a lint/analyse/audit/test job needs nothing more.
    runs-on: ubuntu-latest

  coverage-comment:
    # this job posts a coverage comment, so it escalates exactly one scope:
    permissions:
      contents: read
      pull-requests: write
    runs-on: ubuntu-latest
```

❌ **Avoid** — silent broad scopes, or escalating everything globally:

```yaml
# No permissions: block at all → the token keeps the broad default scopes.
# Or, worse, granting write globally so one job's need leaks to every job:
permissions:
  contents: write
  pull-requests: write
```

### Cancel superseded runs with `concurrency`

Pushing twice to the same PR in quick succession starts two full runs; the older one is wasted compute and can finish *after* the newer one, reporting a stale status. A `concurrency` group keyed by workflow + ref cancels the in-flight run when a newer commit arrives. Key it by `github.ref` so each branch/PR has its own group and runs never cancel across branches.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### The amended workflow header

Folded together, the top of `.github/workflows/ci.yml` becomes:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 6 * * *'   # nightly: re-audit deps against freshly disclosed CVEs

# Least privilege by default; jobs escalate only what they need.
permissions:
  contents: read

# Cancel an in-flight run when a newer commit lands on the same ref.
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  check:
    name: Lint, Analyse, Audit & Test (PHP 8.4)
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Setup PHP 8.4
        uses: shivammathur/setup-php@cf4cade2721041f288618926317de27f8e2a4c97 # v2.34.1
        with:
          php-version: '8.4'
          extensions: pdo_pgsql, pdo_sqlite, mbstring, bcmath
          coverage: none
          tools: composer:v2

      # …caching, install, key:generate, lint:test, analyse as before…

      - name: Audit dependencies
        run: composer audit

      - name: Run tests
        run: composer test
        env:
          DB_CONNECTION: sqlite
          DB_DATABASE: ':memory:'
```

The SHAs above are illustrative — copy the real ones from each Action's release page when you pin. The audit step sits **after** static analysis and **before** the suite: it's fast, and a vulnerable dependency should fail the build whether or not the tests pass.

## Putting it together

Four tools cover four orthogonal axes, and every standard in this guide belongs to exactly one of them:

- **Pint enforces *style*** — formatting, imports, `declare(strict_types=1)`.
- **PHPStan enforces *types*** — the signatures line up across branches your tests don't cover.
- **Pest `arch()` tests enforce *architecture*** — the layering rules this guide describes are not just prose. Controllers touch no Eloquent, Actions expose one `handle()`, models are `final`, repository interfaces are the only seam to the database: each is an executable assertion. See [Testing](testing.md) for the `arch()` rule set. An architecture rule with no `arch()` test is a code-review opinion, not a standard; this is what makes the guide self-enforcing.
- **Pest enforces *behaviour*** — the unit and feature suites prove the code does what it claims.

Layered on top, **`composer audit` + `roave/security-advisories` enforce *dependency security*** — the one axis that can go red without anyone changing a line of code.

The full enforcement loop is:

1. You write code. Pint's `declare_strict_types` and import rules mean you don't fuss over boilerplate; you do add `final` by hand. `roave/security-advisories` already blocks any vulnerable dependency from resolving when you `composer require`.
2. Before committing you run **`composer check`** — `lint:test`, then `analyse`, then the suite (which includes the `arch()` architecture tests) — and it's green.
3. You push. **CI** runs the identical commands, plus `composer audit` (and a nightly re-audit), plus a real Postgres for migration-fidelity specs.
4. The PR cannot merge unless that job passes.

Because steps 2 and 3 invoke the same Composer scripts, there is no daylight between "passes locally" and "passes in CI" — with one deliberate exception: `composer audit` lives in CI only (see the *Composer scripts* notes), because dependency health is a function of the upstream advisory feed, not of your branch. That equivalence is the whole point of the tooling.

### Optional: make the gate automatic

The Composer scripts are the contract; a Git hook just stops you forgetting them. A minimal `.git/hooks/pre-commit` (mark it executable with `chmod +x`):

```bash
#!/bin/sh
# Block the commit unless the full gate is green.
composer check || {
    echo "composer check failed — commit aborted. Fix the issues above and retry."
    exit 1
}
```

For a hook that is version-controlled and shared across the team (rather than living in each clone's untracked `.git/`), use [GrumPHP](https://github.com/phpro/grumphp) or point `core.hooksPath` at a tracked directory. Either way the hook only ever runs the same `composer` scripts — never a parallel, drifting copy of the rules.

### Editor consistency: `.editorconfig`

Pint fixes formatting on demand, but an `.editorconfig` keeps editors from fighting it as you type — fewer reformats, smaller diffs. Place this at the project root; every modern editor honours it.

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 4

[*.{yml,yaml,json,neon}]
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

## See also

- [Code Style](code-style.md)
- [Testing](testing.md) — the Pest `arch()` rules that enforce this guide's architecture
- [Architecture](architecture.md) — the layering the `arch()` tests assert
- [Config & Secrets](config-and-secrets.md) — `APP_DEBUG=false` and keeping secrets out of error pages
- [Livewire](livewire.md) — the authenticated surface the Livewire CVE gate protects
- [Philosophy](philosophy.md)
