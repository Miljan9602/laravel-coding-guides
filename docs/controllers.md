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

## See also

- [Form requests](form-requests.md)
- [Actions](actions.md)
- [Repositories](repositories.md)
