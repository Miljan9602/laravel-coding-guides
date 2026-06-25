# Skinny controllers

A controller is HTTP glue: it receives a validated request, hands the work to exactly one Action or Service, and turns the result into a response. It owns no business logic, touches no database, and makes no outbound calls — anything heavier belongs one layer down.

## Rules at a glance

- A controller method does three things only: type-hint a **Form Request**, call **one** Action/Service, **return a response**. Nothing else.
- **Never** put a DB query, `save()`, `updateOrCreate()`, `Model::create()`, or any persistence in a controller.
- **Never** call `$request->validate()` or `Validator::make()` in a controller — validation lives in a Form Request.
- **Never** call `Http::`, the `Socialite` facade, `Crypt::`, queue dispatching of business jobs, or any third-party SDK from a controller.
- **No** business rules, branching on domain state, loops over records, or multi-step workflows in a controller.
- The **only** allowed write-like calls are auth/session *establishment*: `Auth::login()`, regenerating the session, and `session()` for a post-login redirect target. That is HTTP orchestration, not a business-data write.
- Prefer **invokable single-action controllers** (`__invoke`) for one-off endpoints; use **resource controllers** for standard CRUD.
- Inject dependencies you need on every method via the **constructor**; inject per-method collaborators as **method parameters** (Laravel resolves both from the container).
- Always declare an **explicit return type** (`RedirectResponse`, `JsonResponse`, `View`, `Response`).
- Every API/JSON response goes through a **`JsonResource`** — never serialise a raw model, an array, or `$model->toArray()` straight to the client.
- When a route model belongs to **another tenant**, abort with **`404`, not `403`**, so you never confirm the resource exists to someone who may not see it.
- Authorization is decided by a **Policy class** (auto-discovered), invoked from a Form Request's `authorize()` or `$this->authorize()` — never a hand-rolled `abort_unless($user->can(...), 403)` in the controller body.
- A controller method should read top-to-bottom in a handful of lines. If it doesn't, the logic is in the wrong place.

## Why controllers stay skinny

Controllers are the one layer you cannot unit-test without booting the HTTP kernel, faking the session, and routing a request. Every line of logic you put there is a line you can only cover with a slow feature test. Push that logic into an Action and it becomes a plain object you instantiate with fakes and assert against directly.

Skinny controllers also keep behaviour reusable. A `ConnectSlackChannel` action can be triggered by an OAuth callback today, a CLI command tomorrow, and a queued retry next week. If the exchange-and-store logic lived in the controller, none of those other entry points could reach it without duplicating it.

Finally, a one-responsibility controller is auditable at a glance. A reviewer should be able to confirm "validates, delegates, responds" without reading the Action's internals. The moment a controller branches on domain state, that guarantee is gone.

## The three-line shape

Every controller method should collapse to: a typed request in, a single delegation, a response out.

### ✅ Do — validate, delegate, respond

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Workspace;

use App\Actions\Workspace\SaveWorkspaceSettings;
use App\Http\Controllers\Controller;
use App\Http\Requests\Workspace\UpdateWorkspaceSettingsRequest;
use Illuminate\Http\RedirectResponse;

final class WorkspaceSettingsController extends Controller
{
    public function update(
        UpdateWorkspaceSettingsRequest $request,
        SaveWorkspaceSettings $saveSettings,
    ): RedirectResponse {
        $saveSettings->handle(
            $request->workspace(),
            $request->settings(),
        );

        return to_route('workspace.settings.edit')
            ->with('status', 'workspace-settings-saved');
    }
}
```

The `UpdateWorkspaceSettingsRequest` has already authorized the user and validated input by the time the method body runs. `workspace()` and `settings()` are typed accessors on the Form Request (see [Form requests](form-requests.md)). The controller never sees a raw `$request->input(...)`.

### ❌ Avoid — the fat controller

```php
public function update(Request $request): RedirectResponse
{
    // Validation belongs in a Form Request.
    $data = $request->validate([
        'display_name' => ['required', 'string', 'max:120'],
        'timezone' => ['required', 'timezone'],
        'lookback_days' => ['required', 'integer', 'between:1,30'],
    ]);

    // Authorization smeared into the action.
    abort_unless($request->user()->can('update', $request->workspace), 403);

    // Persistence in the controller.
    $workspace = $request->workspace;
    $workspace->display_name = $data['display_name'];
    $workspace->timezone = $data['timezone'];
    $workspace->settings = array_merge($workspace->settings ?? [], [
        'lookback_days' => $data['lookback_days'],
    ]);
    $workspace->save();

    // Business side-effect inlined.
    if ($workspace->wasChanged('timezone')) {
        Http::post(config('services.scheduler.url'), [
            'workspace' => $workspace->id,
            'timezone' => $workspace->timezone,
        ]);
    }

    return back()->with('status', 'saved');
}
```

This method validates, authorizes, persists, and fires an HTTP call. None of it is reusable, all of it requires a feature test, and the `Http::post` is invisible to anyone scanning the route. Every concern here belongs elsewhere: validation and authorization in the Form Request, persistence and the scheduler call in `SaveWorkspaceSettings`.

## What is forbidden, and where it goes instead

| Forbidden in a controller | Belongs in |
| --- | --- |
| `$request->validate()`, `Validator::make()` | Form Request (`rules()`) |
| `Workspace::where(...)`, any query builder / Eloquent read | Repository, called by an Action |
| `save()`, `update()`, `updateOrCreate()`, `create()`, `delete()` | Repository, called by an Action |
| `Http::get/post(...)`, third-party SDK, `Socialite::driver(...)->user()` | Service injected into an Action |
| `Crypt::encrypt/decrypt`, signing, hashing of business secrets | Service / Action |
| `dispatch(SomeBusinessJob)` | Action (which may dispatch internally) |
| `if`/`foreach` over domain state, multi-step workflow | Action / Service |

The single carve-out is **auth/session establishment**, covered below.

## Worked refactor: a Slack OAuth callback

Slack redirects back to your app with a `code` and the `state` you signed when you started the flow. The naïve controller does everything inline. Here is the version to avoid.

### ❌ Avoid — everything in the callback

```php
public function callback(Request $request): RedirectResponse
{
    // Raw validation in the controller.
    $request->validate([
        'code' => ['required', 'string'],
        'state' => ['required', 'string'],
    ]);

    // Decrypting and resolving state by hand.
    $state = Crypt::decrypt($request->string('state'));
    $workspace = Workspace::findOrFail($state['workspace_id']);

    // Outbound HTTP straight from the controller.
    $response = Http::asForm()->post('https://slack.com/api/oauth.v2.access', [
        'client_id' => config('services.slack.client_id'),
        'client_secret' => config('services.slack.client_secret'),
        'code' => $request->string('code'),
    ]);

    if (! $response->json('ok')) {
        return to_route('slack.connect')->with('error', 'Slack rejected the request.');
    }

    // Persistence in the controller.
    DeliveryChannel::updateOrCreate(
        ['workspace_id' => $workspace->id, 'type' => 'slack'],
        [
            'target' => $response->json('incoming_webhook.url'),
            'label' => $response->json('incoming_webhook.channel'),
            'active' => true,
        ],
    );

    return to_route('workspace.settings.edit')->with('status', 'slack-connected');
}
```

Five forbidden concerns in one method: `validate()`, `Crypt`, a model read, an `Http` call, and an `updateOrCreate`.

### ✅ Do — validate, resolve, delegate, redirect

Split the work across the layers that own it. The Form Request validates and exposes typed accessors. A small stateless service verifies the signed `state` and resolves the workspace. The Action performs the OAuth exchange and the persistence. The controller is glue.

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Slack;

use App\Actions\Slack\ConnectSlackChannel;
use App\Http\Controllers\Controller;
use App\Http\Requests\Slack\SlackCallbackRequest;
use App\Services\Slack\SlackOAuthState;
use Illuminate\Http\RedirectResponse;

final class SlackCallbackController extends Controller
{
    public function __invoke(
        SlackCallbackRequest $request,
        SlackOAuthState $state,
        ConnectSlackChannel $connect,
    ): RedirectResponse {
        $workspace = $state->resolveWorkspace($request->stateToken());

        $connected = $connect->handle($workspace, $request->authorizationCode());

        return $connected
            ? to_route('workspace.settings.edit')->with('status', 'slack-connected')
            : to_route('slack.connect')->with('error', 'slack-connect-failed');
    }
}
```

The supporting Form Request keeps validation and accessors out of the controller:

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests\Slack;

use Illuminate\Foundation\Http\FormRequest;

final class SlackCallbackRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user() !== null;
    }

    /**
     * @return array<string, array<int, string>>
     */
    public function rules(): array
    {
        return [
            'code' => ['required', 'string'],
            'state' => ['required', 'string'],
        ];
    }

    public function authorizationCode(): string
    {
        return $this->string('code')->toString();
    }

    public function stateToken(): string
    {
        return $this->string('state')->toString();
    }
}
```

And the Action — the one place where the SDK call and the write live — looks exactly like the real `ConnectSlackChannel`:

```php
<?php

declare(strict_types=1);

namespace App\Actions\Slack;

use App\Contracts\Repositories\DeliveryChannelRepositoryInterface;
use App\Models\DeliveryChannel;
use App\Models\Workspace;
use App\Services\Slack\SlackOAuthClient;

/**
 * Completes the Slack connect flow: exchanges the OAuth code for a webhook and
 * stores it as a Slack delivery channel for the workspace.
 */
final readonly class ConnectSlackChannel
{
    public function __construct(
        private SlackOAuthClient $slack,
        private DeliveryChannelRepositoryInterface $channels,
    ) {}

    public function handle(Workspace $workspace, string $code): bool
    {
        $webhook = $this->slack->exchangeCode($code);

        if ($webhook === null) {
            return false;
        }

        $this->channels->updateOrCreateForWorkspace(
            $workspace->getKey(),
            ['type' => 'slack', 'target_hash' => DeliveryChannel::hashTarget($webhook['url'])],
            ['target' => $webhook['url'], 'label' => $webhook['channel'], 'active' => true, 'team_id' => null],
        );

        return true;
    }
}
```

Now the OAuth exchange is fakeable (`SlackOAuthClient`), the persistence is fakeable (`DeliveryChannelRepositoryInterface`), and the same `ConnectSlackChannel::handle()` can be called from a CLI reconnect command without a single HTTP request in sight. The controller's only job — choosing the redirect — is the one thing that genuinely belongs to HTTP. See [Actions](actions.md) and [Form requests](form-requests.md).

## Invokable single-action controllers

When a route maps to exactly one operation, use an invokable controller (`__invoke`). It removes the redundant method name, keeps the file focused, and routes read cleanly.

### ✅ Do

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Reports;

use App\Actions\RunReport;
use App\Http\Controllers\Controller;
use App\Http\Requests\Reports\GenerateReportRequest;
use Illuminate\Http\RedirectResponse;

final class GenerateReportController extends Controller
{
    public function __invoke(
        GenerateReportRequest $request,
        RunReport $runReport,
    ): RedirectResponse {
        $runReport->handle(
            $request->workspace(),
            $request->reportWindow(),
            dryRun: false,
        );

        return to_route('reports.index')->with('status', 'report-queued');
    }
}
```

Route it without naming a method:

```php
use App\Http\Controllers\Reports\GenerateReportController;

Route::post('/reports', GenerateReportController::class)->name('reports.generate');
```

Reach for invokable controllers when the endpoint is a single verb (generate, connect, switch, retry). Use a multi-method controller only when the methods are genuinely cohesive — which in practice means a resource.

## Resource controllers

For standard CRUD on a model, a resource controller gives you the seven conventional methods with predictable names. Keep every one of them skinny.

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Reports;

use App\Actions\Reports\ArchiveReport;
use App\Http\Controllers\Controller;
use App\Http\Requests\Reports\StoreReportRequest;
use App\Models\Report;
use App\Queries\WorkspaceReportsQuery;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;

final class ReportController extends Controller
{
    public function index(WorkspaceReportsQuery $reports): View
    {
        return view('reports.index', [
            'reports' => $reports->forCurrentWorkspace()->paginate(),
        ]);
    }

    public function show(Report $report): View
    {
        $this->authorize('view', $report);

        return view('reports.show', ['report' => $report]);
    }

    public function store(StoreReportRequest $request, RunReport $runReport): RedirectResponse
    {
        $runReport->handle($request->workspace(), $request->reportWindow(), dryRun: false);

        return to_route('reports.index')->with('status', 'report-queued');
    }

    public function destroy(Report $report, ArchiveReport $archive): RedirectResponse
    {
        $this->authorize('delete', $report);

        $archive->handle($report);

        return to_route('reports.index')->with('status', 'report-archived');
    }
}
```

Notes that keep these methods honest:

- **`index` delegates its read to a query object** (`WorkspaceReportsQuery`), not an inline `Report::where(...)`. The controller never builds an Eloquent query itself. See [Repositories](repositories.md).
- **Route-model binding** (`Report $report`) resolves the model for you — that is framework routing, not a controller-issued query, so it is allowed.
- **`$this->authorize(...)`** is acceptable when there is no Form Request to host it; with a Form Request, authorization moves to `authorize()`. Either way the controller never hand-rolls an `abort_unless` permission check.

Register it conventionally:

```php
use App\Http\Controllers\Reports\ReportController;

Route::resource('reports', ReportController::class)
    ->only(['index', 'show', 'store', 'destroy']);
```

## Dependency injection: constructor vs. method

Both work; choose by lifetime of need.

- **Constructor injection** for a dependency every method uses (a presenter, a policy gateway, a per-request context object). Promote it `private readonly`.
- **Method injection** for a collaborator only one method needs (the Action that method delegates to). This keeps the constructor empty and the dependency visible at the call site.

### ✅ Do — method injection for per-action collaborators

```php
final class WorkspaceController extends Controller
{
    public function switch(
        SwitchWorkspaceRequest $request,
        SwitchWorkspace $switchWorkspace,
    ): RedirectResponse {
        $switchWorkspace->handle($request->user(), $request->targetWorkspaceId());

        return to_route('dashboard');
    }
}
```

### ✅ Do — constructor injection for a cross-method dependency

```php
final class ReportController extends Controller
{
    public function __construct(
        private readonly ReportPresenter $presenter,
    ) {}

    public function show(Report $report): View
    {
        return view('reports.show', [
            'report' => $this->presenter->forView($report),
        ]);
    }
}
```

Note the controller itself is `final` but not `readonly`: Laravel's base `Controller` is mutable, and constructor-injected dependencies on a controller are conventional rather than value state. The Actions it calls are `final readonly` (see [Actions](actions.md)).

## The one exception: auth and session establishment

Logging a user in *is* HTTP/session orchestration — it manipulates the session and the auth guard, not your domain tables. So a controller may call `Auth::login()`, regenerate the session, and read/write `session()` for the post-login redirect. The provisioning of the *user record* still belongs in an Action.

### ✅ Do — Action provisions, controller establishes the session

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Auth;

use App\Actions\Auth\AuthenticateWithGitHub;
use App\Http\Controllers\Controller;
use App\Services\Auth\GitHubSocialite;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;

final class GitHubCallbackController extends Controller
{
    public function __invoke(
        GitHubSocialite $socialite,
        AuthenticateWithGitHub $authenticate,
    ): RedirectResponse {
        // Socialite itself is wrapped in a service so the facade never appears here.
        $githubUser = $socialite->user();

        // The Action owns all persistence: find-or-create the user, workspace, membership.
        $user = $authenticate->handle($githubUser);

        // Allowed: session/auth establishment is HTTP orchestration, not a data write.
        Auth::login($user, remember: true);
        request()->session()->regenerate();

        return redirect()->intended(route('dashboard'));
    }
}
```

`AuthenticateWithGitHub` is the real Action — it wraps the whole provisioning in `DB::transaction(...)`, matches on `github_id`, and spins up a first workspace and membership for new users. That logic is emphatically *not* allowed in the controller. What *is* allowed is the `Auth::login()` / `session()->regenerate()` pair, because establishing the session is the controller's legitimate HTTP job.

### ❌ Avoid — calling Socialite and provisioning inline

```php
public function callback(): RedirectResponse
{
    // Socialite facade in the controller — forbidden.
    $githubUser = Socialite::driver('github')->user();

    // Provisioning logic in the controller — forbidden.
    $user = User::updateOrCreate(
        ['github_id' => $githubUser->getId()],
        ['name' => $githubUser->getName(), 'email' => $githubUser->getEmail()],
    );

    if ($user->wasRecentlyCreated) {
        $workspace = Workspace::create(['owner_id' => $user->id, 'name' => "{$user->name}'s workspace"]);
        Membership::create(['user_id' => $user->id, 'workspace_id' => $workspace->id, 'role' => 'admin']);
        $user->update(['current_workspace_id' => $workspace->id]);
    }

    Auth::login($user);

    return redirect()->route('dashboard');
}
```

The `Socialite` facade, the `updateOrCreate`, the branching on `wasRecentlyCreated`, and the three follow-up writes are all forbidden. Only the `Auth::login(...)` line survives the move to a skinny controller.

## Return types are not optional

Declare the concrete response type on every method. It documents intent, lets static analysis catch a method that forgets to return, and stops a method from silently returning `null` (a blank 200).

### ✅ Do

```php
public function store(StoreReportRequest $request, RunReport $runReport): RedirectResponse
{
    $runReport->handle($request->workspace(), $request->reportWindow(), dryRun: false);

    return to_route('reports.index');
}

public function show(Report $report): JsonResponse
{
    return response()->json(ReportResource::make($report));
}
```

### ❌ Avoid — untyped, ambiguous return

```php
public function store(StoreReportRequest $request, RunReport $runReport)
{
    $runReport->handle($request->workspace(), $request->reportWindow(), dryRun: false);
    // Forgetting the return here yields a blank 200 with no type to catch it.
}
```

Use the precise type: `RedirectResponse`, `JsonResponse`, `View`, `StreamedResponse`, or `Response`. Reach for the union `View|RedirectResponse` only when a method genuinely returns either, and prefer splitting the endpoint if that branch encodes a business decision.

## Shape responses with API Resources

A model is your storage shape; an API response is your public contract. The two diverge the moment you add an internal column (`webhook_secret`, `installation_token`), rename a field, or want to expose a computed value. An **API Resource** — an `Illuminate\Http\Resources\Json\JsonResource` subclass — is the transformation layer that sits between them: it takes a model (or collection) and returns exactly the JSON you mean to ship. Serialising a raw model leaks every `$fillable`/`$visible` column you forget to hide, couples the wire format to the table, and turns a migration into a breaking API change. The Resource decouples them: the column rename is one line in `toArray()`, the contract is unchanged.

> A `JsonResource` is the **HTTP** API Resource — not a Filament Resource. Filament's `Resource` (`app/Filament/Resources/*`) is an admin-panel CRUD declaration; the `JsonResource` here (`app/Http/Resources/*`) is the JSON serialiser for your public endpoints. Different layers, unrelated base classes; see [Filament](filament.md).

The rule: **every JSON response renders through a `JsonResource`**. Never return `$model`, `$model->toArray()`, or a hand-built array from a JSON endpoint.

### ✅ Do — a Resource owns the wire shape

```php
<?php

declare(strict_types=1);

namespace App\Http\Resources;

use App\Models\Report;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

/**
 * @mixin Report
 */
final class ReportResource extends JsonResource
{
    /**
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'type' => $this->type->value,
            'period_start' => $this->period_start->toIso8601String(),
            'period_end' => $this->period_end->toIso8601String(),
            'status' => $this->status->value,
            // Relationship is only serialised when the caller eager-loaded it.
            'contributors' => ContributorSummaryResource::collection(
                $this->whenLoaded('contributorSummaries'),
            ),
            // Permission-gated field: present only for workspace admins.
            'delivery_log' => $this->when(
                $request->user()?->can('viewDeliveryLog', $this->resource) ?? false,
                fn (): array => $this->delivery_log,
            ),
        ];
    }
}
```

The controller stays a one-liner, with a `JsonResponse` return type:

```php
public function show(Report $report): JsonResource
{
    $this->authorize('view', $report);

    return ReportResource::make($report->load('contributorSummaries'));
}
```

Return the `JsonResource` directly — Laravel serialises it to a `JsonResponse` for you (declaring `JsonResource` as the return type is honest about what the method produces; wrapping in `response()->json(ReportResource::make(...))` is equivalent and also fine). Either way, **the model never reaches `response()->json()` unwrapped.**

### ❌ Avoid — serialising the model straight to JSON

```php
public function show(Report $report): JsonResponse
{
    // Ships every column: status, internal flags, and any future secret column
    // someone adds to the table — silently, the day they add it.
    return response()->json($report);
}

public function index(WorkspaceReportsQuery $reports): JsonResponse
{
    // ->toArray() has the identical leak, plus an N+1 if the view touches relations.
    return response()->json($reports->forCurrentWorkspace()->get()->toArray());
}
```

`response()->json($report)` makes the table the contract. Add a column, leak a column. Hide it with `$hidden` and you have now coupled the model's serialisation to one endpoint's needs — the next endpoint that *does* want that column can't have it. The Resource is where each endpoint states its own shape.

### `whenLoaded`, `when`, `mergeWhen` — conditional keys

Three conditionals keep a Resource both safe and cheap:

- **`whenLoaded('relation')`** — include a relationship's data **only if it was eager-loaded**. This is your N+1 guard inside the Resource: if the relation isn't loaded, the key is omitted entirely rather than triggering a lazy query per row. Pair it with an explicit `->load()` / `->with()` at the call site so the eager-load is a deliberate decision, not an accident.
- **`when($condition, $value)`** — include a key only when a condition holds, typically a permission gate. Pass a **closure** for `$value` when computing it is non-trivial, so it runs only when the condition is true.
- **`mergeWhen($condition, [...])`** — merge several keys at once behind one condition, e.g. an admin-only block of operational fields.

```php
return [
    'id' => $this->id,
    'period_end' => $this->period_end->toIso8601String(),
    // N+1-safe: only serialised when ->load('contributorSummaries') ran.
    'contributors' => ContributorSummaryResource::collection(
        $this->whenLoaded('contributorSummaries'),
    ),
    // Admin-only operational block, gated once.
    ...$this->mergeWhen($request->user()?->can('administer', $this->resource) ?? false, [
        'delivery_log' => $this->delivery_log,
        'last_dispatched_at' => $this->last_dispatched_at?->toIso8601String(),
    ]),
];
```

Use `ReportResource::collection($reports)` for lists (it wraps each item and handles pagination metadata). Eager-load the relations the Resource references **before** you hand the models over — `whenLoaded` protects you from N+1 only if you never lazy-load in the first place.

## Tenant-scoped routes: 404, not 403

Every route-bound model in a multi-tenant app must be proven to belong to the **current** workspace before the controller acts on it. The interesting question is what to do when it doesn't. The instinct is `403 Forbidden`. The correct answer is almost always `404 Not Found`.

A `403` is an admission: *"this resource exists, you just can't have it."* That single bit leaks existence. An attacker who can tell `403` (exists, not yours) from `404` (doesn't exist) can **enumerate** your resources — walk `/reports/1`, `/reports/2`, … and map out which IDs are live across other tenants, how many reports a competitor runs, when one appeared. For anything scoped to a tenant the safest posture is to make "not yours" and "doesn't exist" **indistinguishable**: return `404` for both. The caller learns nothing they didn't already know.

### ✅ Do — cross-tenant access is a 404

```php
public function show(Report $report): JsonResource
{
    // Belongs to another workspace → behave exactly as if it never existed.
    abort_unless($report->workspace_id === $request->workspace()->getKey(), 404);

    return ReportResource::make($report->load('contributorSummaries'));
}
```

At the policy level, return the same `404` instead of a bare denial. `Illuminate\Auth\Access\Response::denyAsNotFound()` produces a denial that the framework renders as `404` rather than the default `403`:

```php
<?php

declare(strict_types=1);

namespace App\Policies;

use App\Models\Report;
use App\Models\User;
use Illuminate\Auth\Access\Response;

final class ReportPolicy
{
    public function view(User $user, Report $report): Response
    {
        return $report->workspace_id === $user->current_workspace_id
            ? Response::allow()
            // Same 404 a missing report would return — existence is never confirmed.
            : Response::denyAsNotFound();
    }
}
```

Now both `$this->authorize('view', $report)` and a Form Request's `authorize()` produce a `404` for a cross-tenant report, with no special-casing in the controller.

### ❌ Avoid — leaking existence with a 403

```php
public function show(Report $report): JsonResource
{
    // 403 confirms the report exists and belongs to someone else — enumerable.
    abort_unless($report->workspace_id === $request->workspace()->getKey(), 403);

    return ReportResource::make($report);
}
```

**Edge cases.** A genuine *permission* failure within the user's own tenant — a member trying an admin-only action on a report they can legitimately see — is a real `403`; they already know the resource exists, so hiding it serves nothing. Reserve `404`-for-authorization for the **cross-tenant boundary**, where existence itself is the secret. The same `abort_unless(..., 404)` guard appears in [error handling](error-handling.md); this section is the *why* behind it.

## Authorization: policy classes & `Gate::before`

Scattered `if ($user->role === 'admin')` checks rot: the rule for "who may delete a report" ends up copied into a controller, a Livewire component, and a Blade `@can`, and they drift. A **Policy** centralises every authorization decision for one model into one class with one method per ability. Every layer then asks the same question the same way — `$user->can('delete', $report)`, `$this->authorize('delete', $report)`, `@can('delete', $report)`, a Form Request's `authorize()` — and gets the same answer.

### A real Policy

Map one ability method per action. `viewAny` and `create` take no model (there is none yet); `view`, `update`, and `delete` receive the resolved model.

```php
<?php

declare(strict_types=1);

namespace App\Policies;

use App\Models\Report;
use App\Models\User;
use Illuminate\Auth\Access\Response;

final class ReportPolicy
{
    public function viewAny(User $user): bool
    {
        return $user->current_workspace_id !== null;
    }

    public function view(User $user, Report $report): Response
    {
        return $report->workspace_id === $user->current_workspace_id
            ? Response::allow()
            : Response::denyAsNotFound();
    }

    public function create(User $user): bool
    {
        return $user->isAdminOf($user->current_workspace_id);
    }

    public function update(User $user, Report $report): bool
    {
        return $report->workspace_id === $user->current_workspace_id
            && $user->isAdminOf($report->workspace_id);
    }

    public function delete(User $user, Report $report): Response
    {
        return $report->workspace_id === $user->current_workspace_id
            ? Response::allow()
            : Response::denyAsNotFound();
    }
}
```

**Auto-discovery.** Laravel maps `App\Models\Report` to `App\Policies\ReportPolicy` by naming convention — no registration needed as long as the names line up. Override only if a policy lives off-convention, via `Gate::policy(Report::class, ReportPolicy::class)` in a service provider.

**`authorizeResource` wires a resource controller in one call.** It registers `before`-style middleware mapping each resource method to its ability (`index → viewAny`, `show → view`, `store → create`, `update → update`, `destroy → delete`), so you delete the per-method `$this->authorize(...)` calls:

```php
final class ReportController extends Controller
{
    public function __construct()
    {
        // Every resource method now runs its matching policy ability automatically.
        $this->authorizeResource(Report::class, 'report');
    }

    public function show(Report $report): JsonResource
    {
        // No $this->authorize('view', $report) needed — authorizeResource did it.
        return ReportResource::make($report->load('contributorSummaries'));
    }
}
```

The route parameter name (`'report'`) must match the model binding (`Report $report`) for the implicit-model methods to resolve the right instance.

### `Gate::before` — grant super-admin once

A `Gate::before` callback runs **before every policy check** and short-circuits the lot: return `true` and the ability is granted regardless of the policy. This is the one clean place to encode a global super-admin so the rule lives in exactly one spot instead of being threaded through every policy method.

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use App\Models\User;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;

final class AuthServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Super-admins bypass every policy. Returning null falls through to the
        // policy; only an explicit bool decides the ability here.
        Gate::before(function (User $user, string $ability): ?bool {
            return $user->is_super_admin ? true : null;
        });
    }
}
```

Today `is_super_admin` is hand-checked only in Filament's `canAccessPanel()`. Centralising it in `Gate::before` means a super-admin transparently passes every `can`/`authorize`/`@can` across the app, with the grant defined once. Return **`null`** (not `false`) for the non-super-admin case — `false` would *deny* the ability outright and skip the policy, whereas `null` lets the policy decide.

> **Footgun.** `Gate::before` is **skipped for any ability whose policy class lacks a matching method.** If `ReportPolicy` has no `export` method and you call `$user->can('export', $report)`, Laravel never reaches `before` — the check just fails. A super-admin you "granted everything" will be denied an ability the policy doesn't define. Either define the method on the policy or use a `Gate::define('export', ...)` ability so `before` has something to intercept.

### Keep the controller thin

Authorization is a *decision*, and like every decision it leaves the controller body. Host it in the Form Request's `authorize()` when one exists, or `$this->authorize()` (directly or via `authorizeResource`) when there isn't. The controller never hand-rolls `abort_unless($user->can(...), 403)`.

### ✅ Do — authorize in the Form Request, controller stays glue

```php
final class DeleteReportRequest extends FormRequest
{
    public function authorize(): bool
    {
        // Delegates to ReportPolicy::delete via the gate; fails closed → 403/404.
        return $this->user()->can('delete', $this->route('report'));
    }
}
```

```php
public function destroy(DeleteReportRequest $request, Report $report, ArchiveReport $archive): RedirectResponse
{
    // Authorization already happened in the Form Request. Just delegate.
    $archive->handle($report);

    return to_route('reports.index')->with('status', 'report-archived');
}
```

### ❌ Avoid — inlining the permission check in the controller

```php
public function destroy(Report $report, ArchiveReport $archive): RedirectResponse
{
    // Authorization logic smeared into the controller body.
    abort_unless($request->user()->can('delete', $report), 403);
    abort_unless($report->workspace_id === $request->user()->current_workspace_id, 403);

    $archive->handle($report);

    return to_route('reports.index');
}
```

Both checks belong in `ReportPolicy::delete` (which already encodes tenant scoping *and* the `404`-over-`403` choice), reached through the Form Request. The controller is left with one job: delegate and redirect. See [Form requests](form-requests.md) for `authorize()` and [Error handling](error-handling.md) for how failed authorization renders.

## See also

- [Form requests](form-requests.md)
- [Actions](actions.md)
- [Repositories](repositories.md)
- [Error handling](error-handling.md)
- [Models & Eloquent](models-eloquent.md)
- [Livewire](livewire.md)
