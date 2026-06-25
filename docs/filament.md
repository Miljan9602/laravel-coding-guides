# Filament (admin panels)

Filament is a fast way to build admin panels on top of Eloquent, but it is a *framework boundary*: it owns the query lifecycle for the resources you declare against it. Treat it pragmatically — let Filament name models where the framework legitimately requires it, and route everything else (especially widget data) through your normal repository layer.

## Rules at a glance

- **Resources, Tables, Schemas, and Infolists name a model directly — and that is allowed.** This is the one sanctioned place in the app where admin code references an Eloquent model by name, because Filament owns the query builder behind it.
- **Widgets are ordinary app code.** Their `getStats()` / data methods must go through repositories — never `Model::count()` or ad-hoc queries inside a widget.
- **Gate every panel** to your team with the `FilamentUser` contract (`canAccessPanel()`), backed by an `is_super_admin` flag or an email allowlist. A panel with no gate is a public admin panel.
- **Read-only "god-view" Resources** disable create/edit/delete via `canCreate()`, `canEdit()`, `canDelete()` (or by registering no mutating pages/actions).
- **Apply the house style to generated classes.** Filament's `make:*` generators produce classes without `declare(strict_types=1)` and without `final` — add both. Pint can auto-insert `strict_types`.
- **Authorize per-resource, not just per-panel.** Panel access is the front door; Resources still respect Eloquent policies and the resource-level `can*()` methods.
- **Keep business logic out of Resources.** Resources are declarative schema; non-trivial behaviour belongs in actions/services that the Resource calls.

## Why Filament is a special case

Most of this guide forbids naming an Eloquent model outside the repository layer — repositories are the seam between your domain and the database (see [repositories.md](repositories.md)). Filament breaks that rule *by design*: a `Resource` is a declarative description of how the framework should build, query, paginate, filter, and mutate a model. The query builder is constructed and owned by Filament, not by you. Trying to hide the model behind a repository here would mean reimplementing Filament's table/eager-loading/scoping machinery — fighting the framework for no benefit.

So the rule is precise: **declarations** (Resource, Table, Schema, Infolist, Form) may name the model. **Imperative app logic** that happens to live near Filament (widgets, page hooks that compute domain values, custom actions doing real work) must not — it goes through repositories like any other code.

---

## Resources name the model — that is the boundary

### Rule

A `Resource` declares `protected static ?string $model` and configures its table/form/infolist against that model. Let it. Add `declare(strict_types=1)` and `final` to the generated class.

### Why

Filament resolves `$model` to build the underlying `Builder` (`Model::query()`), apply global scopes, eager-load relationships for tables, and handle record resolution for edit/view pages. This is the framework's contract; substituting a repository call would yield an array or DTO that Filament's table/pagination cannot drive.

### How

✅ Do — declare against the model, keep the class `final` and strict:

```php
<?php

declare(strict_types=1);

namespace App\Filament\Resources;

use App\Filament\Resources\ReportResource\Pages;
use App\Models\Report;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Table;

final class ReportResource extends Resource
{
    protected static ?string $model = Report::class;

    protected static ?string $navigationIcon = 'heroicon-o-document-chart-bar';

    protected static ?string $navigationGroup = 'Operations';

    public static function form(Schema $schema): Schema
    {
        return $schema->components([
            \Filament\Forms\Components\TextInput::make('title')
                ->required()
                ->maxLength(255),
            \Filament\Forms\Components\Select::make('workspace_id')
                ->relationship('workspace', 'name')
                ->required(),
            \Filament\Forms\Components\DateTimePicker::make('generated_at'),
        ]);
    }

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('title')->searchable()->sortable(),
                TextColumn::make('workspace.name')->label('Workspace')->sortable(),
                TextColumn::make('contributors_count')->counts('contributors')->label('Contributors'),
                TextColumn::make('generated_at')->dateTime()->sortable(),
            ])
            ->defaultSort('generated_at', 'desc');
    }

    public static function getPages(): array
    {
        return [
            'index' => Pages\ListReports::route('/'),
            'view' => Pages\ViewReport::route('/{record}'),
        ];
    }
}
```

❌ Avoid — wrapping the framework's query in a repository and losing Filament's machinery:

```php
// Wrong: Filament can't paginate/sort/filter a hand-built array,
// and you've reimplemented half the framework.
public static function table(Table $table): Table
{
    $reports = app(ReportRepository::class)->all(); // array of DTOs — dead end here
    // ...nothing downstream can consume this correctly
}
```

> Scope what the framework queries — don't replace it. To narrow the records a Resource exposes, override `getEloquentQuery()` and add constraints there; you are still inside Filament's builder.

```php
public static function getEloquentQuery(): \Illuminate\Database\Eloquent\Builder
{
    return parent::getEloquentQuery()->whereNotNull('generated_at');
}
```

---

## A read-only "god-view" Resource

### Rule

For an overview panel where operators should *look but not touch*, disable creation, editing, and deletion at the Resource level and register only `index`/`view` pages.

### Why

Defence in depth: hiding buttons is not authorization. The `can*()` static methods are consulted by Filament before rendering actions *and* before executing them, so even a hand-crafted request to a mutating route is refused. Registering no `create`/`edit` pages removes the routes entirely.

### How

✅ Do — a Resource that is structurally read-only:

```php
<?php

declare(strict_types=1);

namespace App\Filament\Resources;

use App\Filament\Resources\ReportResource\Pages;
use App\Models\Report;
use Filament\Resources\Resource;
use Illuminate\Database\Eloquent\Model;

final class ReportResource extends Resource
{
    protected static ?string $model = Report::class;

    public static function canCreate(): bool
    {
        return false;
    }

    public static function canEdit(Model $record): bool
    {
        return false;
    }

    public static function canDelete(Model $record): bool
    {
        return false;
    }

    public static function canDeleteAny(): bool
    {
        return false;
    }

    public static function getPages(): array
    {
        // No `create` or `edit` routes are registered at all.
        return [
            'index' => Pages\ListReports::route('/'),
            'view' => Pages\ViewReport::route('/{record}'),
        ];
    }
}
```

❌ Avoid — relying on UI to be read-only while leaving the routes live:

```php
// The edit page still exists and is reachable by URL.
public static function getPages(): array
{
    return [
        'index' => Pages\ListReports::route('/'),
        'edit' => Pages\EditReport::route('/{record}/edit'), // mutating route still open
    ];
}
// ...and no can*() overrides, so anyone who reaches /reports/5/edit can save.
```

For a view-only detail page, present an **Infolist** rather than a disabled form — it is the read-only counterpart and also declares against the model:

```php
use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Schema;

public static function infolist(Schema $schema): Schema
{
    return $schema->components([
        TextEntry::make('title'),
        TextEntry::make('workspace.name')->label('Workspace'),
        TextEntry::make('generated_at')->dateTime(),
    ]);
}
```

---

## Gate the panel to your team

### Rule

Implement `Filament\Models\Contracts\FilamentUser` on your `User` model and return access from `canAccessPanel()`. Back the decision with an `is_super_admin` boolean column or an email allowlist read from config.

### Why

Without this contract, **any authenticated user can reach the panel** in production — Filament only requires authentication, not authorization, by default. `canAccessPanel()` is the single, framework-honoured choke point for "is this person allowed in this admin panel at all". Per-panel gating composes with per-resource policies: the panel is the front door, policies guard individual rooms.

### How

✅ Do — gate by a flag, with a config-driven allowlist fallback:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Filament\Models\Contracts\FilamentUser;
use Filament\Panel;
use Illuminate\Foundation\Auth\User as Authenticatable;

final class User extends Authenticatable implements FilamentUser
{
    protected $casts = [
        'is_super_admin' => 'boolean',
    ];

    public function canAccessPanel(Panel $panel): bool
    {
        if ($this->is_super_admin) {
            return true;
        }

        // config, never env(), outside config files
        $allowlist = config('digest.admin_emails', []);

        return in_array($this->email, $allowlist, true)
            && $this->hasVerifiedEmail();
    }
}
```

Define the allowlist in config so secrets/tunables stay out of code:

```php
// config/digest.php
return [
    'admin_emails' => array_filter(explode(',', (string) env('DIGEST_ADMIN_EMAILS', ''))),
];
```

❌ Avoid — leaving the panel ungated, or reading `env()` at request time:

```php
// No FilamentUser contract at all → every logged-in user is an admin.
final class User extends Authenticatable
{
    // ...nothing here
}

// Or: env() outside a config file — slow, and null once config is cached.
public function canAccessPanel(Panel $panel): bool
{
    return $this->email === env('ADMIN_EMAIL'); // returns false under config:cache
}
```

### Gate the panel provider too

Apply an auth/authorization guard at the provider level so the whole panel sits behind your middleware, and name the panel explicitly:

```php
<?php

declare(strict_types=1);

namespace App\Providers\Filament;

use Filament\Http\Middleware\Authenticate;
use Filament\Panel;
use Filament\PanelProvider;

final class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->id('admin')
            ->path('admin')
            ->login()
            ->discoverResources(in: app_path('Filament/Resources'), for: 'App\\Filament\\Resources')
            ->discoverWidgets(in: app_path('Filament/Widgets'), for: 'App\\Filament\\Widgets')
            ->authMiddleware([Authenticate::class]);
    }
}
```

`Authenticate` ensures a user exists; `canAccessPanel()` decides whether *that* user may enter. You need both.

---

## Widgets are ordinary app code — go through repositories

### Rule

A widget's `getStats()` / data methods are your code, not the framework's query boundary. Pull every number from a repository. Do **not** call `Report::count()`, `Workspace::where(...)->count()`, or any Eloquent query inside a widget.

### Why

Widgets compute *domain* values ("reports generated this week", "active workspaces"). That logic must live behind the same repository seam as the rest of the app so it is reusable, testable in isolation (no Filament, no HTTP), and consistent with how the Slack/email reports compute the same figures. Inlining `Model::count()` in a widget duplicates query knowledge and silently drifts from the canonical definition.

### How

✅ Do — inject repositories and ask them for the counts:

```php
<?php

declare(strict_types=1);

namespace App\Filament\Widgets;

use App\Repositories\Contracts\ReportRepository;
use App\Repositories\Contracts\WorkspaceRepository;
use Filament\Widgets\StatsOverviewWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;

final class OperationsOverview extends StatsOverviewWidget
{
    protected function getStats(): array
    {
        $reports = app(ReportRepository::class);
        $workspaces = app(WorkspaceRepository::class);

        return [
            Stat::make('Reports this week', $reports->countGeneratedSince(now()->subWeek()))
                ->description('Standups + digests')
                ->color('success'),

            Stat::make('Active workspaces', $workspaces->countActive())
                ->color('primary'),

            Stat::make('Workspaces missing GitHub token', $workspaces->countWithoutGitHubToken())
                ->color('danger'),
        ];
    }
}
```

The repository owns the query and names the model — the legitimate place to do so:

```php
<?php

declare(strict_types=1);

namespace App\Repositories;

use App\Models\Report;
use App\Repositories\Contracts\ReportRepository as ReportRepositoryContract;
use DateTimeInterface;

final readonly class EloquentReportRepository implements ReportRepositoryContract
{
    public function countGeneratedSince(DateTimeInterface $since): int
    {
        return Report::query()
            ->where('generated_at', '>=', $since)
            ->count();
    }
}
```

❌ Avoid — querying Eloquent straight from the widget:

```php
final class OperationsOverview extends StatsOverviewWidget
{
    protected function getStats(): array
    {
        return [
            // Model named outside the repository layer; definition of
            // "this week" now lives in two places and will drift.
            Stat::make('Reports this week', Report::where('generated_at', '>=', now()->subWeek())->count()),
            Stat::make('Active workspaces', Workspace::where('is_active', true)->count()),
        ];
    }
}
```

This also pays off in tests: the widget is verified by binding a fake `ReportRepository` that returns fixed counts — no database, no Filament render, no `Http::fake()`.

```php
it('shows the weekly report count from the repository', function (): void {
    $reports = Mockery::mock(ReportRepository::class);
    $reports->shouldReceive('countGeneratedSince')->andReturn(7);
    app()->instance(ReportRepository::class, $reports);

    $stats = (new OperationsOverview())->getStatsForTesting();

    expect($stats)->toContain('7');
});
```

> The same rule covers custom **page hooks** and **table actions** that compute or mutate domain state: declarative column/field definitions stay in the Resource, but any real work delegates to an action/service that uses repositories. Keep the Resource thin.

---

## Apply the house style to generated classes

### Rule

Every `php artisan make:filament-resource` / `make:filament-widget` output gets `declare(strict_types=1)` at the top and `final` on the class, exactly like hand-written code. See [code-style.md](code-style.md).

### Why

Generated code is still your code and is held to the same standard: strict types catch silent coercions, and `final` documents that these classes are not designed for inheritance (Filament classes are extended *by the framework's base classes you extend*, not by your own subclasses). Letting generators bypass the standard creates a two-tier codebase.

### How

Run Pint after generating; it can auto-add `declare(strict_types=1)` when the rule is enabled:

```json
// pint.json
{
    "preset": "laravel",
    "rules": {
        "declare_strict_types": true,
        "final_class": false
    }
}
```

```bash
php artisan make:filament-resource Report --view
./vendor/bin/pint app/Filament   # adds declare(strict_types=1)
# then add `final` by hand where the class is yours to seal
```

✅ Do:

```php
<?php

declare(strict_types=1);

namespace App\Filament\Resources\ReportResource\Pages;

use App\Filament\Resources\ReportResource;
use Filament\Resources\Pages\ViewRecord;

final class ViewReport extends ViewRecord
{
    protected static string $resource = ReportResource::class;
}
```

❌ Avoid — shipping the raw generator output:

```php
<?php

namespace App\Filament\Resources\ReportResource\Pages;

use App\Filament\Resources\ReportResource;
use Filament\Resources\Pages\ViewRecord;

class ViewReport extends ViewRecord // no strict_types, not final
{
    protected static string $resource = ReportResource::class;
}
```

> Note on `final`: `final_class` is left off in Pint because Filament page classes are sometimes extended within larger plugins; seal *your* classes deliberately rather than letting a blanket rule fight third-party packages. Resources, widgets, and pages you own should be `final`.

## See also

- [Repositories](repositories.md) — the model-naming boundary that widgets and page hooks must route through.
- [Code style](code-style.md) — `declare(strict_types=1)`, `final`, Pint configuration applied to generated classes.
