# Naming conventions

Names are the cheapest documentation we have: a reader who knows the conventions can infer a class's role, a method's
return shape, and a column's relationship from the name alone. This page is the canonical, exhaustive reference for how
we name every artefact in a Laravel codebase — classes, methods, variables, routes, database objects, config, and views.

## Rules at a glance

- **Controllers**: singular noun + `Controller` — `ArticleController`, `GitHubInstallationController`.
- **Models**: singular noun, no suffix — `User`, `Workspace`, `Report`.
- **Form Requests**: `<Verb><Noun>Request` — `StorePostRequest`, `UpdateWorkspaceRequest`.
- **Actions**: imperative verb phrase, no suffix — `PublishPost`, `ConnectSlackChannel`.
- **Services**: noun or role — `SlackOAuthClient`, `ReportAggregator`.
- **Repositories**: `<Model>Repository` + contract `<Model>RepositoryInterface` — note the **deliberate** `Interface`
  suffix here, unlike capability contracts (`GitHubClient`).
- **Jobs**: imperative verb phrase — `SendInvoice`, `RefreshGitHubInstallation`.
- **Enums**: singular noun — `ReportStatus`, `DeliveryChannel`.
- **DTOs / value objects**: noun — `Report`, `Contributor`, `EmailAddress`.
- **Traits**: adjective ending in `-able`/`-ing` where natural — `Notifiable`, `HasUuids`.
- **Methods & variables**: `camelCase`; booleans read as predicates — `isActive()`, `hasSummary`.
- **Collections**: plural, descriptive — `$activeUsers`; single objects singular — `$activeUser`.
- **Routes**: plural, kebab-case paths — `/articles`, `/github-installations`; named routes dot-notation
  (`articles.show`); route params singular (`{article}`).
- **Tables**: plural `snake_case` — `article_comments`; **pivots**: singular model names, alphabetical — `article_user`.
- **Columns**: `snake_case`, no model-name prefix — `meta_title`, not `article_meta_title`; FKs `<model>_id`; PK `id`.
- **Migrations**: timestamped + descriptive verb phrase — `2026_06_25_140000_add_status_to_reports_table`.
- **Config & lang keys**: `snake_case`; **Blade views**: kebab- or snake-case `.blade.php`.

---

## The complete reference table

| Artefact | Convention | ✅ Example | ❌ Avoid |
| --- | --- | --- | --- |
| Controller | Singular noun + `Controller` | `ArticleController` | `ArticlesController`, `ArticleCtrl` |
| Single-action controller | Imperative + `Controller` | `PublishArticleController` | `ArticlePublishController` |
| Model | Singular noun | `Workspace` | `Workspaces`, `WorkspaceModel` |
| Form Request | `<Verb><Noun>Request` | `StorePostRequest` | `PostRequest`, `PostStoreRequest` |
| Action | Imperative verb phrase | `PublishPost` | `PostPublisher`, `PublishPostAction` |
| Service | Noun / role | `SlackOAuthClient` | `SlackService`, `Slack` |
| Repository (impl) | `<Model>Repository` | `ReportRepository` | `ReportRepo`, `ReportsRepository` |
| Repository (contract) | `<Model>RepositoryInterface` | `ReportRepositoryInterface` | `IReportRepository`, `ReportRepoContract` |
| Capability contract | Adjective/noun, **no** `Interface` | `GitHubClient` | `GitHubClientInterface` |
| Job | Imperative verb phrase | `SendInvoice` | `InvoiceJob`, `InvoiceSender` |
| Event | Past-tense fact | `ReportGenerated` | `GenerateReport`, `ReportEvent` |
| Listener | Imperative reaction | `NotifyTeamOfReport` | `ReportListener` |
| Mailable | Noun + `Mail` | `ReportMail` | `ReportMailable`, `SendReport` |
| Notification | Noun / event-ish | `InvoicePaid` | `InvoicePaidNotification` (optional) |
| Enum | Singular noun | `ReportStatus` | `ReportStatuses`, `ReportStatusEnum` |
| DTO / value object | Noun | `Contributor` | `ContributorDTO`, `ContributorData` |
| Trait | Adjective (`-able`/`-ing`) | `Notifiable` | `NotifiableTrait`, `HasNotifications` |
| Interface (behaviour) | Adjective | `Arrayable`, `Summarisable` | `SummarisableInterface` |
| Middleware | Noun / role | `EnsureWorkspaceIsActive` | `WorkspaceMiddleware` |
| Policy | `<Model>Policy` | `ReportPolicy` | `ReportAuthorizer` |
| Console command class | Imperative + `Command` | `SendDigestCommand` | `DigestCommand` |
| Console command signature | `group:verb` | `digest:send` | `sendDigest`, `digest-send` |
| Table | Plural `snake_case` | `article_comments` | `articleComments`, `comment` |
| Pivot table | Singular models, alphabetical | `article_user` | `user_article`, `article_users` |
| Column | `snake_case`, no model prefix | `meta_title` | `articleMetaTitle`, `article_meta_title` |
| Foreign key | `<model>_id` | `workspace_id` | `workspaceId`, `fk_workspace` |
| Primary key | `id` | `id` | `article_id`, `pk` |
| Migration | Timestamp + verb phrase | `..._create_reports_table` | `..._reports`, `..._migration` |
| Route path | Plural, kebab-case | `/github-installations` | `/githubInstallations`, `/github_installations` |
| Named route | `resource.action` | `articles.show` | `showArticle`, `article-show` |
| Route param | Singular | `{article}` | `{articles}`, `{article_id}` |
| Config key | `snake_case` | `digest.slack_block_limit` | `digest.slackBlockLimit` |
| Lang key | `snake_case` | `report.delivery_failed` | `report.deliveryFailed` |
| Blade view | kebab/snake `.blade.php` | `mail/report.blade.php` | `mail/Report.blade.php` |

---

## Classes

### Controllers — singular noun + `Controller`

A controller orchestrates one resource. Naming it after the singular resource keeps the route-to-controller mapping
obvious and matches Laravel's resource-route generator, which expects `ArticleController` for `Route::resource('articles', ...)`.

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Actions\PublishArticle;
use App\Http\Requests\UpdateArticleRequest;
use App\Models\Article;
use Illuminate\Http\RedirectResponse;

final class ArticleController extends Controller
{
    public function update(UpdateArticleRequest $request, Article $article): RedirectResponse
    {
        // ...
    }
}
```

When a controller does exactly one thing, name it after the operation and give it an `__invoke` method — it is still
suffixed `Controller`:

```php
final class PublishArticleController extends Controller
{
    public function __invoke(Article $article, PublishArticle $publish): RedirectResponse
    {
        $publish($article);

        return back();
    }
}
```

✅ Do: `ArticleController`, `GitHubInstallationController`, `PublishArticleController`.
❌ Avoid: `ArticlesController` (plural), `ArticleCtrl` (abbreviated), `ArticlePublishController` (noun-first for a
single action).

### Models — singular noun, no suffix

A model represents one row; the singular reads naturally in code (`$article->author`) and Laravel infers the plural
`snake_case` table name automatically. Never suffix `Model` — the namespace `App\Models` already carries that meaning.

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

final class Article extends Model
{
    protected $fillable = ['title', 'meta_title', 'body'];

    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

✅ Do: `User`, `Workspace`, `Report`.
❌ Avoid: `Users`, `WorkspaceModel`, `tbl_report`.

### Form Requests — `<Verb><Noun>Request`

The verb encodes which write the request validates, so `StorePostRequest` and `UpdatePostRequest` can differ in rules
(e.g. `unique` ignoring the current row on update) without ambiguity. Match the verb to the controller method
(`store` → `Store…Request`, `update` → `Update…Request`).

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

final class StorePostRequest extends FormRequest
{
    /** @return array<string, mixed> */
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'meta_title' => ['nullable', 'string', 'max:60'],
        ];
    }
}
```

✅ Do: `StorePostRequest`, `UpdateWorkspaceRequest`, `DestroyReportRequest`.
❌ Avoid: `PostRequest` (which verb?), `PostStoreRequest` (noun-first), `StorePostFormRequest` (redundant `Form`).

### Actions — imperative verb phrase, no suffix

An action *is* a verb in the domain, so the class name reads as a command: `PublishPost`, `ConnectSlackChannel`,
`RequestReport`. The bare imperative makes call sites read like sentences (`$publishPost($post)`), and omitting an
`Action` suffix keeps the focus on intent. The single public entry point is `__invoke` or `handle`.

```php
<?php

declare(strict_types=1);

namespace App\Actions;

use App\Models\Post;
use App\Repositories\PostRepositoryInterface;

final readonly class PublishPost
{
    public function __construct(
        private PostRepositoryInterface $posts,
    ) {}

    public function __invoke(Post $post): Post
    {
        return $this->posts->markPublished($post, now());
    }
}
```

✅ Do: `PublishPost`, `ConnectSlackChannel`, `RefreshInstallationToken`.
❌ Avoid: `PostPublisher` (noun), `PublishPostAction` (redundant suffix), `DoPublish` (vague verb).

### Services — noun or role

A service is a long-lived collaborator named for *what it is* — `SlackOAuthClient`, `ReportAggregator`,
`MultiOrgCollector`. Prefer a concrete role (`…Client`, `…Aggregator`, `…Collector`, `…Builder`) over the empty word
`Service`, which conveys nothing. `SlackOAuthClient` tells you it speaks Slack's OAuth API; `SlackService` does not.

```php
<?php

declare(strict_types=1);

namespace App\Services\Slack;

use App\Contracts\SlackHttp;

final readonly class SlackOAuthClient
{
    public function __construct(
        private SlackHttp $http,
    ) {}

    public function exchangeCode(string $code): SlackToken
    {
        // ...
    }
}
```

✅ Do: `SlackOAuthClient`, `ReportAggregator`, `SlackBlockBuilder`.
❌ Avoid: `SlackService`, `SlackManager`, `SlackHelper` (the noun adds nothing).

### Repositories — `<Model>Repository` and the deliberate `RepositoryInterface` suffix

The repository implementation is `<Model>Repository` (`ReportRepository`), and its contract is
`<Model>RepositoryInterface` (`ReportRepositoryInterface`). This is a **deliberate, project-specific exception** to our
general rule that interfaces are named for behaviour without an `Interface` suffix.

**Why the exception?** Capability contracts describe a *role* an object can play, so an adjective/noun reads best and
the `Interface` suffix would be noise:

```php
// Capability contract — adjective/noun, NO "Interface" suffix.
interface GitHubClient
{
    public function pullRequests(string $org, string $repo): PullRequestPage;
}
```

The repository layer is different: there is almost always exactly one contract and one Eloquent implementation per
model, and they share the `<Model>Repository` stem. Without a suffix the contract and implementation would collide on
the name `ReportRepository`. The explicit `Interface` suffix disambiguates the pair at a glance and signals "this is
the seam you type-hint against."

```php
<?php

declare(strict_types=1);

namespace App\Repositories;

use App\Models\Report;
use Illuminate\Support\Collection;

interface ReportRepositoryInterface
{
    public function find(int $id): ?Report;

    /** @return Collection<int, Report> */
    public function recentForWorkspace(int $workspaceId, int $days): Collection;
}
```

```php
<?php

declare(strict_types=1);

namespace App\Repositories;

use App\Models\Report;
use Illuminate\Support\Collection;

final readonly class ReportRepository implements ReportRepositoryInterface
{
    public function find(int $id): ?Report
    {
        return Report::query()->find($id);
    }

    /** @return Collection<int, Report> */
    public function recentForWorkspace(int $workspaceId, int $days): Collection
    {
        return Report::query()
            ->where('workspace_id', $workspaceId)
            ->where('created_at', '>=', now()->subDays($days))
            ->get();
    }
}
```

✅ Do: `ReportRepository` + `ReportRepositoryInterface`; `GitHubClient` (capability, no suffix).
❌ Avoid: `IReportRepository` (Hungarian `I` prefix), `ReportRepo`, `ReportRepositoryContract`, or dropping the suffix
so the contract and class collide.

### Jobs — imperative verb phrase

A queued job *does* something later, so name the action: `SendInvoice`, `RefreshGitHubInstallation`. Like actions,
jobs are verbs; unlike actions, they implement `ShouldQueue` and run asynchronously. Keep the name about the effect,
not the mechanism (no `…Job` suffix needed — the `App\Jobs` namespace says it).

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Models\Invoice;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

final class SendInvoice implements ShouldQueue
{
    use Queueable;

    public function __construct(
        public readonly Invoice $invoice,
    ) {}

    public function handle(): void
    {
        // ...
    }
}
```

✅ Do: `SendInvoice`, `RefreshGitHubInstallation`, `PruneStaleReports`.
❌ Avoid: `InvoiceJob`, `InvoiceSender` (noun), `EmailInvoiceJob` (redundant suffix).

### Enums — singular noun

An enum is a closed *type*, and each case is one value of that type, so the type name is singular and reads naturally
at the case: `ReportStatus::Pending`. No `Enum` suffix and no plural — `ReportStatuses::Pending` reads wrong.

```php
<?php

declare(strict_types=1);

namespace App\Enums;

enum ReportStatus: string
{
    case Pending = 'pending';
    case Generating = 'generating';
    case Delivered = 'delivered';
    case Failed = 'failed';

    public function isTerminal(): bool
    {
        return $this === self::Delivered || $this === self::Failed;
    }
}
```

Cases themselves are `PascalCase`; the backing value is `snake_case` or `kebab-case` to match how it is stored or sent
over the wire.

✅ Do: `ReportStatus`, `DeliveryChannel`, `Cadence`.
❌ Avoid: `ReportStatuses`, `ReportStatusEnum`, `EReportStatus`.

### DTOs and value objects — noun

A DTO names the *thing* it carries — `Report`, `Contributor`, `RepoSummary` — not its mechanism. Avoid `…DTO` and
`…Data` suffixes; the namespace (`App\Data`) and the `final readonly class` shape already communicate the role.

```php
<?php

declare(strict_types=1);

namespace App\Data;

final readonly class Contributor
{
    /** @param list<RepoSummary> $repos */
    public function __construct(
        public string $login,
        public int $commitCount,
        public int $pullRequestCount,
        public array $repos,
    ) {}

    public function hasActivity(): bool
    {
        return $this->commitCount > 0 || $this->pullRequestCount > 0;
    }
}
```

✅ Do: `Report`, `Contributor`, `EmailAddress`.
❌ Avoid: `ReportDTO`, `ContributorData`, `ContributorVO`.

### Traits — adjective

A trait mixes a *capability* into a class, so an adjective reads correctly at the use site: `class User extends
Authenticatable use Notifiable` — "a User is notifiable." Laravel's own traits set the standard (`Notifiable`,
`HasUuids`, `SoftDeletes`). No `Trait` suffix.

```php
<?php

declare(strict_types=1);

namespace App\Concerns;

trait Summarisable
{
    public function shortSummary(int $limit = 280): string
    {
        return str($this->summary)->limit($limit)->value();
    }
}
```

✅ Do: `Notifiable`, `HasUuids`, `Summarisable`.
❌ Avoid: `NotifiableTrait`, `SummaryTrait`, `WithSummary`.

---

## Methods and variables

### `camelCase`, with booleans as predicates

Methods and variables are `camelCase`. A method or variable that yields a boolean must read as a yes/no question or
assertion: prefix with `is`, `has`, `can`, `should`, or `was`. This lets call sites read like English (`if
($report->hasSummary())`) and signals the return type without consulting the signature.

```php
final class Report
{
    public function isDelivered(): bool
    {
        return $this->status === ReportStatus::Delivered;
    }

    public function hasContributors(): bool
    {
        return $this->contributors !== [];
    }
}
```

```php
$canPublish = $user->can('publish', $article);
$shouldRetry = $attempt < $maxAttempts;
$isStale = $installation->refreshedAt->isBefore(now()->subHour());
```

✅ Do: `isActive()`, `hasSummary`, `canPublish`, `shouldRetry`.
❌ Avoid: `active()`/`$active` for a boolean (reads as a verb or a noun), `getIsActive()`, `flag`, `status` for a
boolean.

Reserve `get`/`set` prefixes for genuine accessors that wrap non-trivial logic; for plain reads prefer the bare noun
(`$report->title`, `$user->fullName()`), which is idiomatic Laravel.

### Collections plural, single objects singular

Plurality of the *name* must match plurality of the *value*. A variable holding many items is plural (`$activeUsers`,
`$pendingReports`); a variable holding one item is singular (`$activeUser`). This prevents the classic bug of iterating
a single object or calling a scalar method on a collection.

```php
/** @var Collection<int, User> $activeUsers */
$activeUsers = $this->users->active();

foreach ($activeUsers as $activeUser) {
    $this->notify($activeUser);
}

$primaryWorkspace = $activeUser->workspaces->first();
```

✅ Do: `$activeUsers` (many), `$activeUser` (one), `$repoSummaries`, `$repoSummary`.
❌ Avoid: `$activeUser` for a collection, `$users` for a single model, `$data`/`$list`/`$items` (undescriptive).

---

## Routes

### Plural, kebab-case paths

Resource URLs are plural nouns (the collection), and multi-word segments use `kebab-case` because hyphens are the
URL-friendly, SEO-preferred separator and never need encoding. `/github-installations`, not `/githubInstallations` or
`/github_installations`.

```php
use App\Http\Controllers\ArticleController;
use App\Http\Controllers\GitHubInstallationController;

Route::resource('articles', ArticleController::class);
Route::resource('github-installations', GitHubInstallationController::class);
```

### Named routes use dot-notation; params are singular

Named routes group by resource then action with a dot (`articles.index`, `articles.show`, `github-installations.store`),
which mirrors the resource hierarchy and is what `Route::resource` generates. Route parameters are **singular** because
each binds to one model (`{article}`), and Laravel's implicit model binding resolves `{article}` to an `Article`.

```php
Route::get('/articles/{article}', [ArticleController::class, 'show'])
    ->name('articles.show');

// Generating the URL:
route('articles.show', $article);
```

✅ Do: path `/github-installations`, name `github-installations.show`, param `{githubInstallation}`.
❌ Avoid: name `showArticle` or `article-show`, param `{articles}` or `{article_id}`.

---

## Database

### Tables — plural `snake_case`

A table holds many rows, so its name is the plural of the model in `snake_case`: `User` → `users`, `Article` →
`articles`, `ArticleComment` → `article_comments`. Following this lets Eloquent infer the table from the model with no
`$table` override.

### Pivots — singular model names, alphabetical

A many-to-many pivot is named from the two **singular** model names joined by an underscore, in **alphabetical
order**: `Article` + `User` → `article_user` (not `user_article`, not `article_users`). Alphabetical order makes the
name deterministic regardless of which side you start from, and it is the order Laravel's `belongsToMany` assumes by
default.

```php
Schema::create('article_user', function (Blueprint $table): void {
    $table->foreignId('article_id')->constrained()->cascadeOnDelete();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->primary(['article_id', 'user_id']);
});
```

### Columns — `snake_case`, no model-name prefix

Columns are `snake_case` and must **not** repeat the table/model name. On the `articles` table the column is
`meta_title`, not `article_meta_title` — the table already provides that context, and the prefix only adds noise to
every query and form field.

```php
Schema::create('articles', function (Blueprint $table): void {
    $table->id();                              // PK is always "id"
    $table->foreignId('workspace_id')          // FK is "<model>_id"
        ->constrained()
        ->cascadeOnDelete();
    $table->string('title');
    $table->string('meta_title')->nullable();  // not "article_meta_title"
    $table->timestamps();
});
```

- **Primary key**: always `id`. Never `article_id` on the `articles` table, never `pk`.
- **Foreign key**: `<referenced_model>_id` in singular `snake_case` — `workspace_id`, `user_id`. This is the convention
  `foreignId()->constrained()` and `belongsTo()` both assume, so following it removes boilerplate.
- **Booleans**: predicate-style, `is_`/`has_` prefixed — `is_published`, `has_attachments`.
- **Timestamps**: `_at` suffix for moments (`published_at`, `deleted_at`), `_on` for dates if you must distinguish.

✅ Do: table `article_comments`, pivot `article_user`, column `meta_title`, FK `workspace_id`, PK `id`.
❌ Avoid: table `articleComments`, pivot `user_article`, column `article_meta_title`, FK `workspaceId`, PK `article_id`.

### Migrations — timestamped + descriptive verb phrase

Laravel prefixes every migration filename with a UTC timestamp (`2026_06_25_140000_`) so they apply in creation order.
The remainder describes the change as a verb phrase ending in the table it touches: `create_reports_table`,
`add_status_to_reports_table`, `rename_meta_title_on_articles_table`. The class name follows from the description.

```
database/migrations/
  2026_06_25_140000_create_reports_table.php
  2026_06_25_141200_add_status_to_reports_table.php
  2026_06_26_090500_create_article_user_table.php
```

✅ Do: `..._add_status_to_reports_table`.
❌ Avoid: `..._reports`, `..._update_table`, `..._migration_2`, or hand-editing the timestamp out of order.

---

## Config, lang, and views

### Config and lang keys — `snake_case`

Keys in `config/*.php` and `lang/**/*.php` are `snake_case`, accessed by dotted path: `config('digest.slack_block_limit')`,
`__('report.delivery_failed')`. Outside config files themselves, always read configuration through `config()` — never
`env()` — so cached config (`php artisan config:cache`) works in production.

```php
// config/digest.php
return [
    'slack_block_limit' => (int) env('DIGEST_SLACK_BLOCK_LIMIT', 50),
    'lookback_days' => (int) env('DIGEST_LOOKBACK_DAYS', 7),
];
```

```php
// Application code — config(), never env().
$limit = config('digest.slack_block_limit');
$message = __('report.delivery_failed', ['channel' => $channel]);
```

✅ Do: `digest.slack_block_limit`, `report.delivery_failed`.
❌ Avoid: `digest.slackBlockLimit`, `report.deliveryFailed`, reading `env('DIGEST_SLACK_BLOCK_LIMIT')` in app code.

### Blade views — kebab- or snake-case `.blade.php`

View files and their directories are lowercase `kebab-case` (or `snake_case` to mirror a table), never `PascalCase`,
because view names map case-sensitively to dotted identifiers and mixed casing breaks on case-sensitive filesystems.
A view at `resources/views/mail/report.blade.php` is referenced as `mail.report`.

```
resources/views/
  mail/
    report.blade.php          // view('mail.report')
  github-installations/
    show.blade.php            // view('github-installations.show')
```

✅ Do: `mail/report.blade.php`, `github-installations/show.blade.php`.
❌ Avoid: `Mail/Report.blade.php`, `githubInstallations/Show.blade.php`.

---

## See also

- [Architecture](architecture.md) — where each named layer lives and how data flows between them.
- [Actions](actions.md) — the imperative-verb action pattern in depth.
- [Repositories](repositories.md) — the `<Model>RepositoryInterface` contract/implementation pair.
- [DTOs & value objects](dtos-value-objects.md) — naming and shaping immutable carriers.
- [Code style](code-style.md) — `strict_types`, `final`, promotion, and the formatting rules these names sit inside.
- [SOLID](solid.md) — why capability contracts and repository interfaces are named differently.
