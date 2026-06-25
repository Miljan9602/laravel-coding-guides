# Livewire components

Livewire components are a **thin UI layer**: they hold public state, validate input, and orchestrate — nothing more. Every write delegates to an Action, every read goes through a Repository, and the component itself stays small enough to read in one screen.

## Rules at a glance

- A component owns three things only: **public state**, **validation**, and **orchestration**. No business logic, no inline DB writes, no raw queries.
- **All writes delegate to Actions.** A Livewire method may decide *when* to act and pass user input, but the mutation lives in an Action.
- **All reads go through Repositories** (often exposed as `#[Computed]` properties). Components never call `Model::query()` or `Model::where(...)` directly.
- Get dependencies into a method one of **two safe ways**: method-inject into a *no-argument* action method (`save(SaveSettings $action)`), or resolve via `app(Interface::class)` *inside* the method when the method also takes JS-passed arguments.
- **Never trust the request-scoped "current tenant" holder after `mount()`.** It is empty during Livewire AJAX update requests. Resolve the workspace once in `mount()` and store its **id** on the component.
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
    use ResolvesActiveWorkspace;

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
use Illuminate\Validation\ValidationException;

trait ResolvesActiveWorkspace
{
    /**
     * The active workspace id, captured during mount() while the
     * request-scoped tenant holder is still populated. Safe to read
     * on every subsequent Livewire update request.
     */
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

If you must re-fetch the full model inside a method, look it up by the stored id through the repository — which also re-asserts the scope:

```php
$workspace = app(WorkspaceRepository::class)->findOrFail($this->workspaceId);
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
    use \App\Livewire\Concerns\ResolvesActiveWorkspace;

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
use App\Livewire\Concerns\ResolvesActiveWorkspace;
use App\Repositories\Contracts\SlackChannelRepository;
use Illuminate\Support\Collection;
use Livewire\Attributes\Computed;
use Livewire\Attributes\Validate;
use Livewire\Component;

final class SlackChannels extends Component
{
    use ResolvesActiveWorkspace;

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
- [Repositories](repositories.md) — the only place components read data from.
- [Testing](testing.md) — Pest conventions, fakes, and the `Livewire::test()` patterns above.
