# Livewire components

Livewire components are a **thin UI layer**: they hold public state, validate input, and orchestrate — nothing more. Every write delegates to an Action, every read goes through a Repository, and the component itself stays small enough to read in one screen.

## Rules at a glance

- A component owns three things only: **public state**, **validation**, and **orchestration**. No business logic, no inline DB writes, no raw queries.
- **All writes delegate to Actions.** A Livewire method may decide *when* to act and pass user input, but the mutation lives in an Action.
- **All reads go through Repositories** (often exposed as `#[Computed]` properties). Components never call `Model::query()` or `Model::where(...)` directly.
- Get dependencies into a method one of **two safe ways**: method-inject into a *no-argument* action method (`save(SaveSettings $action)`), or resolve via `app(Interface::class)` *inside* the method when the method also takes JS-passed arguments.
- **Never trust the request-scoped "current tenant" holder after `mount()`.** It is empty during Livewire AJAX update requests. Resolve the workspace once in `mount()` and store its **id** on the component.
- **Lock every identity/authority property with `#[Locked]`.** Public properties are re-hydrated from the client payload on every update request; without `#[Locked]`, `$workspaceId`/`$reportId` are attacker-controlled (a textbook IDOR).
- **`findOrFail($this->id)` does not scope by tenant.** It is a primary-key lookup with no workspace/user constraint; safe only when the id is `#[Locked]` *and* resolved from the authenticated user (prefer a `findForUserOrFail`-style repository method).
- **Re-authorize inside every mutating method.** Route/Form-Request authorization runs only at the initial page load, never on `/livewire/update`; each `save()`/`delete()` calls `$this->authorize(...)` itself.
- **Never put a secret or another tenant's records in a public property.** Public props serialize into the browser HTML; expose only the scalars the view needs.
- Use `#[Computed]` for derived reads; they are memoized per request and keep Blade clean.
- Use `wire:poll` plus an explicit pending/loading state for async job results.
- Redirect with `redirectRoute(..., navigate: true)` to get SPA-style transitions.
- Test with `Livewire::test(...)->set(...)->call(...)->assertSee(...)` — never reach for the browser for logic you can assert in PHP.

## The component is a thin UI layer

### Rule: components hold state, validation, and orchestration — and delegate everything else

**Why.** Livewire components are re-instantiated and re-hydrated on every network round-trip. If business logic lives inside them, it becomes untestable without a full Livewire harness, impossible to reuse from a console command or queued job, and tightly coupled to the rendering lifecycle. By keeping the component thin, the same `SaveWorkspaceSettings` action backs the UI, the API controller, and the nightly reconciliation job — one tested code path.

A component is allowed to:

- declare `public` properties that map to form inputs,
- declare validation via `rules()` or `#[Validate]`,
- decide *when* to call an Action and pass it the user's input,
- expose reads as `#[Computed]` properties backed by repositories,
- handle navigation (`redirectRoute`) and flash messaging.

A component must **not**:

- write to the database directly (`->save()`, `DB::insert`, `Model::create`),
- contain query logic (`Report::where(...)->get()`),
- contain domain rules (quota checks, state machines, pricing).

### Anatomy of a correct component

```php
<?php

declare(strict_types=1);

namespace App\Livewire\Workspace;

use App\Actions\Workspace\SaveWorkspaceSettings;
use App\Data\WorkspaceSettingsData;
use App\Repositories\Contracts\WorkspaceRepository;
use Livewire\Attributes\Computed;
use Livewire\Attributes\Validate;
use Livewire\Component;

final class SettingsForm extends Component
{
    use InteractsWithWorkspace;

    #[Validate('required|string|max:120')]
    public string $displayName = '';

    #[Validate('required|timezone')]
    public string $timezone = 'UTC';

    #[Validate('boolean')]
    public bool $slackEnabled = false;

    public function mount(): void
    {
        $this->bindActiveWorkspace();

        $workspace = app(WorkspaceRepository::class)->findOrFail($this->workspaceId);

        $this->displayName = $workspace->display_name;
        $this->timezone = $workspace->timezone;
        $this->slackEnabled = $workspace->slack_enabled;
    }

    public function save(SaveWorkspaceSettings $action): void
    {
        $this->validate();

        $action->handle(
            $this->workspaceId,
            new WorkspaceSettingsData(
                displayName: $this->displayName,
                timezone: $this->timezone,
                slackEnabled: $this->slackEnabled,
            ),
        );

        session()->flash('status', 'Settings saved.');
    }

    public function render(): \Illuminate\Contracts\View\View
    {
        return view('livewire.workspace.settings-form');
    }
}
```

Note what is *absent*: no `Workspace::find()`, no `->update()`, no business rule about what "saving settings" means. The component validates, then hands a typed DTO to an action.

## Getting dependencies into a method

Livewire resolves method dependencies from the container, but the rules are subtle once a method also receives arguments from the front end (`wire:click="approve(42)"`). There are exactly two patterns you should use — and one trap to understand so you know *why*.

### The pitfall: argument ordering

Livewire fills a method's parameters by walking them left to right. Parameters whose types it can resolve from the container are injected; the remaining parameters are filled, **positionally**, from the JS-passed arguments. When a method mixes injected dependencies and JS arguments, the mapping becomes fragile — reorder the signature, add a dependency, and the JS argument silently lands in the wrong slot. Avoid mixing the two in one signature.

### Pattern A — method-inject into a *no-argument* action method

Use this when the method takes **no** arguments from the front end. It is the cleanest form: the dependency is explicit in the signature and trivially fakeable in tests.

```php
public function save(SaveWorkspaceSettings $action): void
{
    $this->validate();

    $action->handle($this->workspaceId, $this->toData());
}
```

```blade
<button type="button" wire:click="save">Save</button>
```

✅ **Do** — no JS args, so injection is unambiguous:

```php
public function refresh(RebuildReportCache $action): void
{
    $action->handle($this->workspaceId);
}
```

❌ **Avoid** — mixing an injected dependency with a JS-passed argument:

```php
// Fragile: Livewire must guess which parameter is the JS arg ($reportId)
// and which is container-resolved ($action). Reordering breaks it silently.
public function archive(ArchiveReport $action, int $reportId): void
{
    $action->handle($this->workspaceId, $reportId);
}
```

```blade
<button wire:click="archive({{ $report->id }})">Archive</button>
```

### Pattern B — resolve via `app(Interface::class)` inside the method

Use this whenever the method **also** takes JS-passed arguments. Keep the method signature free of injected dependencies, accept only the JS arguments (typed), and resolve the action from the container inside the body. This sidesteps the ordering pitfall entirely.

✅ **Do**:

```php
public function archive(int $reportId): void
{
    app(ArchiveReport::class)->handle($this->workspaceId, $reportId);
}
```

```blade
<button type="button" wire:click="archive({{ $report->id }})">Archive</button>
```

Resolve against the **interface** when one exists — it keeps the component decoupled and lets tests swap a fake:

```php
public function send(string $channel): void
{
    app(SlackNotifier::class)->postReport($this->workspaceId, $channel);
}
```

A quick rule of thumb:

| Method receives JS arguments? | Use |
| --- | --- |
| No | **Pattern A** — method-inject the action |
| Yes | **Pattern B** — `app(Interface::class)` inside the body |

## Tenancy: resolve the workspace once in `mount()`

### Rule: never rely on the request-scoped "current tenant" holder after `mount()`

**Why — this is the single most common Livewire tenancy bug.** Your application almost certainly has a request-scoped holder for the active workspace (a singleton populated by a middleware, e.g. `TenantManager::current()`, `Filament::getTenant()`, or a `CurrentWorkspace` context object). That holder is populated on the **initial full page request**, when the middleware runs. But Livewire's *update* requests — the AJAX calls fired by every `wire:click`, `wire:model`, and `wire:poll` — hit a dedicated Livewire endpoint (`/livewire/update`) that does **not** pass through your tenancy middleware. During those requests the holder is **empty**. Code that reads `TenantManager::current()->id` in an action method works in `mount()` and then throws "null on null" the moment a user clicks a button.

The fix is to resolve the workspace **once** during `mount()`, when the holder is still populated, and persist its **scalar id** as component state. Livewire serializes that id into the page and rehydrates it on every subsequent request, so it is always available. Store the **id**, not the Eloquent model — models are heavy to serialize and Livewire will try to re-fetch them on hydration.

### A reusable trait

```php
<?php

declare(strict_types=1);

namespace App\Livewire\Concerns;

use App\Support\Tenancy\TenantManager;
use Livewire\Attributes\Locked;

trait InteractsWithWorkspace
{
    /**
     * The active workspace id, captured during mount() while the
     * request-scoped tenant holder is still populated. #[Locked] so
     * Livewire rejects any client attempt to mutate it on a subsequent
     * /livewire/update request — without this attribute the id is
     * re-hydrated from the browser payload and is attacker-controlled.
     */
    #[Locked]
    public int $workspaceId = 0;

    public function bindActiveWorkspace(): void
    {
        $workspace = app(TenantManager::class)->current();

        if ($workspace === null) {
            // We are *not* inside a Livewire update request here — mount()
            // runs on the full page load, so a null tenant is a real error.
            abort(403, 'No active workspace for this request.');
        }

        $this->workspaceId = $workspace->id;
    }
}
```

✅ **Do** — capture the id in `mount()`, use the id everywhere after:

```php
public function mount(): void
{
    $this->bindActiveWorkspace();          // holder is populated here
}

public function save(SaveWorkspaceSettings $action): void
{
    $action->handle($this->workspaceId, $this->toData());   // id, always present
}
```

❌ **Avoid** — reading the holder inside an action method:

```php
public function save(SaveWorkspaceSettings $action): void
{
    // BOOM during the wire:click update request: current() is null because
    // the tenancy middleware never ran for /livewire/update.
    $action->handle(app(TenantManager::class)->current()->id, $this->toData());
}
```

If you must re-fetch the full model inside a method, look it up by the stored id through the repository — but understand that **this lookup does not re-assert the tenant scope on its own**. `findOrFail($this->workspaceId)` is a primary-key-only query (`where id = ? limit 1`) with no workspace or user constraint. If `$workspaceId` is an *unlocked public property*, a crafted `/livewire/update` payload can set it to any other tenant's id and this call will happily return that tenant's record — a textbook IDOR. The lookup is only safe when `$workspaceId` is locked against client tampering (next section) **and** the action re-authorizes against the authenticated user. Resolve from the authenticated user, never from a client-supplied id:

```php
// Safe only because $workspaceId is #[Locked] and resolved from auth in mount().
$workspace = app(WorkspaceRepository::class)
    ->findForUserOrFail(auth()->id(), $this->workspaceId);
```

## Lock server-authoritative state with `#[Livewire\Attributes\Locked]`

### Rule: every identity or authority property is `#[Locked]`

**Why — a public Livewire property is part of the request payload, not server state.** Livewire serializes every `public` property into the HTML it sends to the browser, and on every `/livewire/update` AJAX call it **re-hydrates those properties from the client payload** before running your method. That means a `public int $workspaceId` you carefully set from the authenticated tenant in `mount()` is, on the very next `wire:click`, whatever value the browser sends. An attacker opens dev-tools (or replays the request with `curl`) and rewrites `workspaceId` to **another tenant's id**. Your `mount()` never re-runs on an update request, so nothing re-validates it — your `save()` happily writes to, and your `#[Computed]` happily reads from, a workspace the user has no claim to.

`#[Livewire\Attributes\Locked]` closes this hole: Livewire records the property's value server-side and **throws a `CannotUpdateLockedPropertyException` if the incoming payload tries to change it**. Lock *every* property that names an identity or carries authority — `$workspaceId`, `$reportId`, `$contributorId`, `$deliveryChannelId`, `$githubInstallationId`, any role/permission flag. Form-input properties (`$displayName`, `$channel`) stay unlocked precisely *because* they are meant to come from the client; identity is not.

✅ **Do** — identity is locked and resolved from the **authenticated user**, never the client value:

```php
<?php

declare(strict_types=1);

namespace App\Livewire\Concerns;

use App\Support\Tenancy\TenantManager;
use Livewire\Attributes\Locked;

trait InteractsWithWorkspace
{
    #[Locked]
    public int $workspaceId = 0;

    public function bindActiveWorkspace(): void
    {
        // Resolved server-side from the request-scoped tenant holder,
        // which the middleware derives from the *authenticated* user —
        // never from a property the browser can set.
        $workspace = app(TenantManager::class)->current();

        abort_if($workspace === null, 403, 'No active workspace for this request.');

        $this->workspaceId = $workspace->id;
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Livewire\Reports;

use App\Livewire\Concerns\InteractsWithWorkspace;
use Livewire\Attributes\Locked;
use Livewire\Component;

final class ReportViewer extends Component
{
    use InteractsWithWorkspace;   // brings in #[Locked] public int $workspaceId

    #[Locked]
    public int $reportId = 0;     // identity → locked

    public string $note = '';     // user input → intentionally unlocked

    public function render(): \Illuminate\Contracts\View\View
    {
        return view('livewire.reports.report-viewer');
    }
}
```

❌ **Avoid** — the unlocked version, which is the IDOR:

```php
final class ReportViewer extends Component
{
    public int $workspaceId = 0;   // re-hydrated from the browser every update
    public int $reportId = 0;      // attacker sets this to any report id

    // On a crafted /livewire/update payload, $workspaceId and $reportId
    // arrive as whatever the client chose. findForWorkspace() then returns
    // — and addNote() then mutates — another tenant's report.
    public function addNote(): void
    {
        app(ReportRepository::class)
            ->findForWorkspace($this->workspaceId, $this->reportId)
            ->update(['note' => $this->note]);
    }
}
```

**Edge cases.**

- `#[Locked]` only prevents the *client* from changing a property; your own server code may assign it freely (as `mount()` does). It is not `readonly`.
- A locked property can still be *read* in the browser payload — it is **not** secret, only tamper-proof. Never put a secret in any public property, locked or not (see the next section's last rule).
- `#[Locked]` is not a substitute for authorization. It guarantees the id is the one *you* set; it does **not** prove the current user may act on that id. You still re-authorize in every action — that is the next section.

## Re-authorize inside every action; public props are client-readable

### Rule: every mutating Livewire method calls `$this->authorize(...)` itself

**Why — route and Form Request authorization never runs on a Livewire update.** The middleware stack, route `->can()` gates, and any Form Request `authorize()` execute exactly **once**, on the initial full-page `GET` that boots the component. Every subsequent `wire:click` / `wire:model` / `wire:poll` is a `POST` to the shared `/livewire/update` endpoint — it does **not** re-run your route's authorization, and there is no Form Request in the loop at all. So a method that mutated data on the strength of "the route already checked this" is, on the update request, **completely unauthorized**. Combined with client-rehydrated properties, that is how one user edits another's data.

Treat each public method as its own entry point: **re-authorize at the top of every mutating method**, using a policy keyed off the authenticated user and the locked id. Validate user input with `$this->validate()`, then delegate the write to an Action — the Action stays the single tested code path, the component owns the gate.

✅ **Do** — re-authorize in both `save()` and `delete()`:

```php
<?php

declare(strict_types=1);

namespace App\Livewire\Reports;

use App\Actions\Reports\DeleteReport;
use App\Actions\Reports\UpdateReportNote;
use App\Data\ReportNoteData;
use App\Livewire\Concerns\InteractsWithWorkspace;
use App\Repositories\Contracts\ReportRepository;
use Livewire\Attributes\Locked;
use Livewire\Attributes\Validate;
use Livewire\Component;

final class ReportEditor extends Component
{
    use InteractsWithWorkspace;

    #[Locked]
    public int $reportId = 0;

    #[Validate('required|string|max:2000')]
    public string $note = '';

    public function mount(int $reportId): void
    {
        $this->bindActiveWorkspace();
        $this->reportId = $reportId;

        $report = app(ReportRepository::class)
            ->findForWorkspaceOrFail($this->workspaceId, $this->reportId);

        $this->authorize('update', $report);   // gate at load, too
        $this->note = $report->note ?? '';
    }

    public function save(UpdateReportNote $action): void
    {
        $report = app(ReportRepository::class)
            ->findForWorkspaceOrFail($this->workspaceId, $this->reportId);

        $this->authorize('update', $report);   // re-checked on this update request
        $this->validate();

        $action->handle(
            $this->workspaceId,
            $this->reportId,
            new ReportNoteData(note: $this->note),
        );

        session()->flash('status', 'Note saved.');
    }

    public function delete(DeleteReport $action): void
    {
        $report = app(ReportRepository::class)
            ->findForWorkspaceOrFail($this->workspaceId, $this->reportId);

        $this->authorize('delete', $report);   // its own gate; delete ≠ update

        $action->handle($this->workspaceId, $this->reportId);

        $this->redirectRoute('reports.index', navigate: true);
    }

    public function render(): \Illuminate\Contracts\View\View
    {
        return view('livewire.reports.report-editor');
    }
}
```

❌ **Avoid** — trusting the page-load check and skipping re-authorization on the update:

```php
public function delete(DeleteReport $action): void
{
    // No authorize() here. The route gate that ran on the initial GET does
    // NOT run for this /livewire/update POST, so any authenticated user who
    // can reach the component can delete this report. Pairing this with an
    // unlocked $reportId lets them delete *any* report.
    $action->handle($this->workspaceId, $this->reportId);
}
```

**Never store secrets — or another tenant's records — in a public property.** Public properties serialize into the HTML delivered to the browser, so anything you assign to one is readable by the user (and tamperable unless `#[Locked]`). A `GithubInstallation` access token, a `DeliveryChannel` webhook secret, the raw `WorkspaceReportContext`, or a `Collection` of some *other* workspace's `ContributorSummary` records all leak the instant they touch a public property. Keep secrets in `config()`/the server-side context and out of the component; expose only the scalars the view actually needs.

✅ **Do** — keep the secret server-side, surface only a derived boolean:

```php
#[Computed]
public function slackConnected(): bool
{
    // Repository reads the token server-side and never returns it.
    return app(DeliveryChannelRepository::class)
        ->hasActiveSlack($this->workspaceId);
}
```

❌ **Avoid** — a secret on a public property:

```php
public string $slackWebhookSecret = '';   // serialized straight into the page HTML
```

## Computed properties for derived reads

### Rule: expose reads as `#[Computed]` properties backed by repositories

**Why.** `#[Computed]` properties are lazily evaluated and **memoized for the duration of a single request**, so referencing `$this->recentReports` five times in a Blade view runs the query once. They keep `render()` free of fetching, give the view clean accessors (`$this->recentReports` reads as a property, no parentheses), and — critically — they let you route the read through a repository instead of querying in the component. Properties are not serialized into the page payload, so a heavy collection does not bloat every round-trip.

✅ **Do** — computed read through a repository:

```php
use Illuminate\Support\Collection;
use Livewire\Attributes\Computed;

#[Computed]
public function recentReports(): Collection
{
    return app(ReportRepository::class)
        ->recentForWorkspace($this->workspaceId, limit: 10);
}

#[Computed]
public function hasReports(): bool
{
    return $this->recentReports->isNotEmpty();
}
```

```blade
@if ($this->hasReports)
    <ul>
        @foreach ($this->recentReports as $report)
            <li wire:key="report-{{ $report->id }}">{{ $report->title }}</li>
        @endforeach
    </ul>
@else
    <p>No reports yet.</p>
@endif
```

❌ **Avoid** — querying inside `render()` (re-runs every round-trip, couples the component to Eloquent):

```php
public function render(): \Illuminate\Contracts\View\View
{
    return view('livewire.reports.index', [
        'reports' => Report::query()
            ->where('workspace_id', $this->workspaceId)
            ->latest()
            ->limit(10)
            ->get(),
    ]);
}
```

Computed properties are **memoized per request**. If an action mutates data mid-request and you need a fresh read afterward, bust the cache explicitly:

```php
public function archive(int $reportId): void
{
    app(ArchiveReport::class)->handle($this->workspaceId, $reportId);

    unset($this->recentReports);   // forget the memoized value
}
```

## Async results with `wire:poll` and a pending state

### Rule: poll for async job results and always render an explicit pending/loading state

**Why.** Report generation runs on a queue — the component dispatches the job, then nothing is ready yet. Without an explicit pending state the UI either shows stale data or an empty void. `wire:poll` re-renders the component on an interval so it can pick up the finished result; pairing it with a status flag gives the user honest feedback and lets you **stop polling** once the work is done (polling forever wastes requests).

✅ **Do** — dispatch, then poll until done:

```php
<?php

declare(strict_types=1);

namespace App\Livewire\Reports;

use App\Actions\Reports\StartReportGeneration;
use App\Enums\ReportStatus;
use App\Repositories\Contracts\ReportRepository;
use Livewire\Attributes\Computed;
use Livewire\Component;

final class Generator extends Component
{
    use \App\Livewire\Concerns\InteractsWithWorkspace;

    public ?int $reportId = null;

    public function mount(): void
    {
        $this->bindActiveWorkspace();
    }

    public function start(StartReportGeneration $action): void
    {
        $this->reportId = $action->handle($this->workspaceId)->id;
    }

    #[Computed]
    public function status(): ReportStatus
    {
        if ($this->reportId === null) {
            return ReportStatus::Idle;
        }

        return app(ReportRepository::class)
            ->findOrFail($this->reportId)
            ->status;
    }

    public function render(): \Illuminate\Contracts\View\View
    {
        return view('livewire.reports.generator');
    }
}
```

```blade
<div @if ($this->status === \App\Enums\ReportStatus::Running) wire:poll.2s @endif>
    @switch($this->status)
        @case(\App\Enums\ReportStatus::Idle)
            <button type="button" wire:click="start">Generate report</button>
            @break

        @case(\App\Enums\ReportStatus::Running)
            <p role="status" aria-busy="true">Generating your report…</p>
            @break

        @case(\App\Enums\ReportStatus::Completed)
            <a href="{{ route('reports.show', $this->reportId) }}">View report</a>
            @break

        @case(\App\Enums\ReportStatus::Failed)
            <p class="error">Generation failed.</p>
            <button type="button" wire:click="start">Retry</button>
            @break
    @endswitch
</div>
```

The `wire:poll` directive is rendered **conditionally** — it only appears while `status` is `Running`, so polling stops automatically the moment the job completes or fails. Pair the button with `wire:loading` for instant feedback before the first response returns:

```blade
<button type="button" wire:click="start" wire:loading.attr="disabled">
    <span wire:loading.remove wire:target="start">Generate report</span>
    <span wire:loading wire:target="start">Starting…</span>
</button>
```

## Navigation: `redirectRoute` with `navigate`

### Rule: redirect with `navigate: true` for SPA-style transitions

**Why.** `navigate: true` opts into Livewire's `wire:navigate` engine: instead of a full browser reload, Livewire fetches the destination over the network and swaps the page in place, preserving scroll and avoiding a flash of unstyled content. The result feels like a single-page app without writing any SPA code.

✅ **Do**:

```php
public function save(SaveWorkspaceSettings $action): void
{
    $this->validate();

    $action->handle($this->workspaceId, $this->toData());

    $this->redirectRoute('workspace.settings', navigate: true);
}
```

Pass route parameters as the second argument:

```php
public function create(CreateReport $action): void
{
    $report = $action->handle($this->workspaceId);

    $this->redirectRoute('reports.show', ['report' => $report->id], navigate: true);
}
```

❌ **Avoid** — string concatenation and a hard reload:

```php
$this->redirect('/reports/' . $report->id);   // no named route, no navigate
```

## Refactor: from inline DB writes to actions + repositories

The most common Livewire smell is a component that has quietly grown into a controller-with-state: it queries, mutates, and enforces rules inline. Here is a realistic "before" and the "after" that follows these standards.

### ❌ Before — logic trapped in the component

```php
<?php

namespace App\Livewire;

use App\Models\Workspace;
use App\Models\SlackChannel;
use Illuminate\Support\Facades\Http;
use Livewire\Component;

class WorkspaceSlack extends Component
{
    public $channel = '';
    public $channels = [];

    public function mount()
    {
        // Reads the request-scoped tenant — fine here, but the pattern leaks below.
        $workspace = tenant()->current();
        $this->channels = SlackChannel::where('workspace_id', $workspace->id)
            ->orderBy('name')
            ->get()
            ->toArray();
    }

    public function connect()
    {
        // Inline validation, inline rule, inline write, inline HTTP call —
        // none of it testable without a browser, none of it reusable.
        if (trim($this->channel) === '') {
            $this->addError('channel', 'Required');
            return;
        }

        $workspace = tenant()->current();   // NULL on this update request → crash

        if (SlackChannel::where('workspace_id', $workspace->id)->count() >= 5) {
            $this->addError('channel', 'Channel limit reached');
            return;
        }

        $response = Http::withToken(env('SLACK_TOKEN'))
            ->post('https://slack.com/api/conversations.join', [
                'channel' => $this->channel,
            ]);

        SlackChannel::create([
            'workspace_id' => $workspace->id,
            'name' => $this->channel,
            'joined' => $response->json('ok'),
        ]);

        return redirect('/workspace/slack');
    }

    public function render()
    {
        return view('livewire.workspace-slack');
    }
}
```

What is wrong: the tenant holder is read in `connect()` (it is null there), the channel cap is a domain rule sitting in the UI, the write and the Slack API call are inline, `env()` is read outside config, and the redirect is a hard reload to a hardcoded path. None of this can be unit-tested.

### ✅ After — thin component, logic in an action and repository

The component shrinks to state + validation + orchestration:

```php
<?php

declare(strict_types=1);

namespace App\Livewire\Workspace;

use App\Actions\Slack\ConnectSlackChannel;
use App\Livewire\Concerns\InteractsWithWorkspace;
use App\Repositories\Contracts\SlackChannelRepository;
use Illuminate\Support\Collection;
use Livewire\Attributes\Computed;
use Livewire\Attributes\Validate;
use Livewire\Component;

final class SlackChannels extends Component
{
    use InteractsWithWorkspace;

    #[Validate('required|string|starts_with:#|max:80')]
    public string $channel = '';

    public function mount(): void
    {
        $this->bindActiveWorkspace();
    }

    #[Computed]
    public function channels(): Collection
    {
        return app(SlackChannelRepository::class)
            ->forWorkspace($this->workspaceId);
    }

    public function connect(ConnectSlackChannel $action): void
    {
        $this->validate();

        $action->handle($this->workspaceId, $this->channel);

        $this->channel = '';
        unset($this->channels);   // refresh the memoized list

        $this->redirectRoute('workspace.slack', navigate: true);
    }

    public function render(): \Illuminate\Contracts\View\View
    {
        return view('livewire.workspace.slack-channels');
    }
}
```

The domain rule (the five-channel cap), the Slack API call, and the persistence move into a single tested action:

```php
<?php

declare(strict_types=1);

namespace App\Actions\Slack;

use App\Exceptions\ChannelLimitReached;
use App\Repositories\Contracts\SlackChannelRepository;
use App\Services\Slack\SlackNotifier;

final readonly class ConnectSlackChannel
{
    public function __construct(
        private SlackChannelRepository $channels,
        private SlackNotifier $slack,
    ) {}

    public function handle(int $workspaceId, string $channel): void
    {
        if ($this->channels->countForWorkspace($workspaceId) >= config('digest.slack.max_channels')) {
            throw new ChannelLimitReached($workspaceId);
        }

        $joined = $this->slack->joinChannel($channel);

        $this->channels->create($workspaceId, $channel, joined: $joined);
    }
}
```

Now the cap is enforced in one place (and reads its limit from `config()`, not `env()`), the action is unit-testable with a fake repository and fake notifier, and the component is small enough to verify at a glance.

## Testing components

### Rule: assert component behaviour with `Livewire::test()`, not a browser

**Why.** Livewire's test helper drives the full hydrate → call → render cycle in-process: you `set()` properties, `call()` methods, and assert on resulting state, emitted events, and rendered HTML — fast, deterministic, no headless browser. Because writes go through actions and reads through repositories, you bind fakes for those and assert that the component *delegated correctly*, keeping the test focused on the UI layer.

✅ **Do** — set, call, assert:

```php
<?php

declare(strict_types=1);

use App\Actions\Slack\ConnectSlackChannel;
use App\Livewire\Workspace\SlackChannels;
use App\Repositories\Contracts\SlackChannelRepository;
use Livewire\Livewire;

it('delegates channel connection to the action', function (): void {
    $action = Mockery::mock(ConnectSlackChannel::class);
    $action->shouldReceive('handle')
        ->once()
        ->with($this->workspaceId, '#standup');
    app()->instance(ConnectSlackChannel::class, $action);

    Livewire::test(SlackChannels::class)
        ->set('channel', '#standup')
        ->call('connect')
        ->assertHasNoErrors()
        ->assertRedirect(route('workspace.slack'));
});

it('validates the channel format', function (): void {
    Livewire::test(SlackChannels::class)
        ->set('channel', 'no-hash')
        ->call('connect')
        ->assertHasErrors(['channel' => 'starts_with']);
});

it('renders channels from the repository', function (): void {
    $repository = Mockery::mock(SlackChannelRepository::class);
    $repository->shouldReceive('forWorkspace')
        ->andReturn(collect([(object) ['id' => 1, 'name' => '#general']]));
    app()->instance(SlackChannelRepository::class, $repository);

    Livewire::test(SlackChannels::class)
        ->assertSee('#general');
});
```

Useful assertions in the same fluent chain: `assertSet('channel', '')` (state after an action), `assertDispatched('report-ready')` (events), `assertSeeHtml(...)`, `assertNoRedirect()`, and `assertStatus(200)`. Because `mount()` calls `bindActiveWorkspace()`, set up the active workspace in your test's `beforeEach` so the tenant holder is populated for the initial mount — exactly as it would be on a real full page load.

❌ **Avoid** — reaching for a browser test to verify pure logic:

```php
// Slow, flaky, and unnecessary: this is a state transition, not a JS interaction.
$browser->visit('/workspace/slack')
        ->type('channel', '#standup')
        ->press('Connect')
        ->assertSee('Connected');
```

Reserve browser-level tests (Dusk / Livewire's browser testing) for genuinely JS-dependent behaviour — file uploads, third-party widgets, drag-and-drop — not for logic the in-process test helper already covers.

## See also

- [Actions](actions.md) — where every write lives; components call them, never duplicate them.
- [Repositories](repositories.md) — the only place components read data from; add a tenant-scoped `findForWorkspaceOrFail`/`findForUserOrFail` rather than a bare `findOrFail`.
- [Controllers](controllers.md) — why route/Form-Request authorization runs only at page load and must be re-asserted on every Livewire update.
- [Error handling](error-handling.md) — how `AuthorizationException` and `CannotUpdateLockedPropertyException` surface and the 403 you should expect.
- [Testing](testing.md) — Pest conventions, fakes, and the `Livewire::test()` patterns above.
