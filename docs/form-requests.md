# Validation & Form Requests

All HTTP input validation and request-level authorization lives in **Form Requests** — dedicated classes that validate, normalise, and authorize a single incoming request before the controller method runs. Controllers receive an already-validated, already-authorized request and never touch raw input.

## Rules at a glance

- **One Form Request per input-bearing controller method.** Never call `$request->validate()` inline in a controller.
- **Name them `<Verb><Noun>Request`** — `CreateReportRequest`, `SwitchWorkspaceRequest`, `GitHubWebhookRequest`.
- **`rules()` returns validation rules.** Use array-of-rules syntax, not pipe strings.
- **`authorize()` performs access checks** — membership, ownership, or signature verification. It **fails closed**: return `false` to abort with `403`.
- **Expose typed accessors** (`installationId(): int`, `state(): string`) that read `validated()`, so the controller pulls clean, typed values.
- **`prepareForValidation()` normalises input** before rules run (trim, lowercase, decode JSON).
- **`messages()` / `attributes()`** customise error text; keep them only where the default is unclear.
- **No Form Request for methods with no user input** — pure redirects and OAuth callbacks that own their own framework-supplied params.
- **Livewire validates in the component** via `rules()` or `#[Validate]`, not in a Form Request.
- Form Requests are **`final`**, declare `strict_types`, and live under `App\Http\Requests\<Domain>\`.

---

## Why Form Requests, not inline validation

Inline `$request->validate()` couples three concerns — validation, authorization, and input shaping — into the controller, where they cannot be reused, are awkward to test in isolation, and bloat the action. A Form Request is a single, named, testable unit: Laravel resolves it from the container, runs `authorize()` then `rules()`, and only then hands a clean object to the controller. Failure paths (`422` for validation, `403` for authorization) are handled by the framework, so the controller body only ever runs on the happy path.

### ✅ Do — push validation into a Form Request

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Actions\CreateReport;
use App\Http\Requests\Report\CreateReportRequest;
use Illuminate\Http\RedirectResponse;

final class ReportController extends Controller
{
    public function store(CreateReportRequest $request, CreateReport $createReport): RedirectResponse
    {
        $report = $createReport->handle(
            workspace: $request->workspace(),
            title: $request->title(),
            lookbackDays: $request->lookbackDays(),
        );

        return to_route('reports.show', $report);
    }
}
```

### ❌ Avoid — validation, authorization, and shaping inline

```php
public function store(Request $request): RedirectResponse
{
    abort_unless($request->user()->can('create', Report::class), 403);

    $data = $request->validate([
        'title' => ['required', 'string', 'max:120'],
        'lookback_days' => ['required', 'integer', 'between:1,90'],
    ]);

    // The controller now owns casting, defaults, and reuse problems.
    $report = Report::create([
        'title' => trim($data['title']),
        'lookback_days' => (int) $data['lookback_days'],
    ]);

    return to_route('reports.show', $report);
}
```

---

## One Form Request per input-bearing method

Each controller method that accepts user input gets its own Form Request, even when two methods look similar. `store` and `update` differ in their rules (uniqueness ignores the current model on update) and often in authorization, so sharing one class forces conditionals that obscure both paths.

Methods that take **no user input** need no Form Request — type-hint the base `Request` (or nothing) instead. Examples: a pure redirect, or a Socialite OAuth callback whose parameters (`code`, `state`) are framework- and provider-supplied rather than user-entered.

### ✅ Do — separate requests, and skip the request where there is no input

```php
public function store(CreateReportRequest $request, CreateReport $action): RedirectResponse { /* ... */ }

public function update(UpdateReportRequest $request, Report $report, UpdateReport $action): RedirectResponse { /* ... */ }

// Socialite callback owns its own params — no Form Request.
public function callback(GitHubAuthService $auth): RedirectResponse
{
    $user = $auth->resolveFromCallback();

    Auth::login($user, remember: true);

    return to_route('dashboard');
}
```

---

## Naming and location

Name the class `<Verb><Noun>Request` and namespace it by domain under `App\Http\Requests\`. This keeps related requests grouped (`Requests\GitHub\`, `Requests\Workspace\`, `Requests\Report\`) and makes the controller signature self-documenting.

```
app/Http/Requests/
├── GitHub/
│   ├── GitHubWebhookRequest.php
│   └── InstallationCallbackRequest.php
├── Report/
│   ├── CreateReportRequest.php
│   └── UpdateReportRequest.php
└── Workspace/
    └── SwitchWorkspaceRequest.php
```

---

## `rules()` — the validation contract

`rules()` returns an array keyed by field name, with each value an array of rules. Prefer the **array form** over pipe-delimited strings: arrays let you compose `Rule::*` objects, are diff-friendly, and never break when a value contains a `|`.

### ✅ Do — array rules, with `Rule` objects where needed

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests\Report;

use App\Models\Report;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

final class CreateReportRequest extends FormRequest
{
    /**
     * @return array<string, array<int, mixed>>
     */
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:120'],
            'type' => ['required', Rule::in(['standup', 'digest'])],
            'lookback_days' => ['required', 'integer', 'between:1,90'],
            'orgs' => ['required', 'array', 'min:1'],
            'orgs.*' => ['string', 'regex:/^[A-Za-z0-9._-]+(\/[A-Za-z0-9._-]+)?$/'],
            'slack_channel' => ['nullable', 'string', 'starts_with:#'],
        ];
    }
}
```

### ❌ Avoid — pipe strings that hide structure and break on edge cases

```php
public function rules(): array
{
    return [
        'title' => 'required|string|max:120',
        // A regex containing a pipe would silently corrupt this rule.
        'orgs.*' => 'string|regex:/^[A-Za-z0-9._-]+(\/[A-Za-z0-9._-]+)?$/',
    ];
}
```

### Context-dependent rules with `Rule::unique()->ignore()`

On update, uniqueness must ignore the record being edited. Reach the route-bound model through `$this->route()`.

```php
public function rules(): array
{
    $report = $this->route('report');

    return [
        'title' => [
            'required',
            'string',
            'max:120',
            Rule::unique('reports', 'title')
                ->where('workspace_id', $this->user()->current_workspace_id)
                ->ignore($report),
        ],
    ];
}
```

---

## `authorize()` — fail-closed access control

`authorize()` decides whether the request may proceed at all. Returning `false` aborts with `403` before validation rules run. **Always default to denial**: any missing precondition (no secret configured, no membership, an absent route model) returns `false`. Never `return true` "to be filled in later".

`authorize()` is resolved from the container, so you can type-hint dependencies (repositories, policies) directly into the method signature.

### Authorize by membership / ownership

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests\Workspace;

use App\Contracts\Repositories\MembershipRepositoryInterface;
use App\Models\Workspace;
use Illuminate\Foundation\Http\FormRequest;

/**
 * Authorises switching to a workspace: the user must be a member of it.
 */
final class SwitchWorkspaceRequest extends FormRequest
{
    public function authorize(MembershipRepositoryInterface $memberships): bool
    {
        $workspace = $this->route('workspace');

        return $workspace instanceof Workspace
            && $memberships->userBelongsToWorkspace(
                (int) $this->user()->getKey(),
                (int) $workspace->getKey(),
            );
    }

    /**
     * @return array<string, array<int, mixed>>
     */
    public function rules(): array
    {
        return [];
    }
}
```

### Authorize a webhook by HMAC-SHA256 signature

Webhooks have no authenticated user — they prove authenticity with a signature. Compute the expected HMAC over the **raw request body** using the shared secret, then compare in constant time with `hash_equals()`. Fail closed when the secret is unconfigured or the header is missing.

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests\GitHub;

use Illuminate\Foundation\Http\FormRequest;

/**
 * Authorises a GitHub App webhook by verifying its HMAC-SHA256 signature against
 * the App webhook secret. Fails closed: a missing secret or bad signature → 403.
 */
final class GitHubWebhookRequest extends FormRequest
{
    public function authorize(): bool
    {
        $secret = (string) config('github_app.webhook_secret');

        if ($secret === '') {
            return false;
        }

        $expected = 'sha256='.hash_hmac('sha256', $this->getContent(), $secret);

        return hash_equals($expected, (string) $this->header('X-Hub-Signature-256'));
    }

    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        return [];
    }

    public function event(): string
    {
        return (string) $this->header('X-GitHub-Event');
    }

    /**
     * @return array<string, mixed>
     */
    public function payload(): array
    {
        return $this->json()->all();
    }
}
```

Two details that matter for HMAC verification:

- Use `$this->getContent()` — the **raw, unparsed body**. Re-serialising `$request->all()` to JSON would not byte-match what the sender signed, so the comparison would always fail.
- Use `hash_equals()`, never `===`. A plain string comparison short-circuits on the first differing byte and leaks timing information; `hash_equals()` is constant-time.

### ❌ Avoid — open-by-default authorization

```php
public function authorize(): bool
{
    return true; // Every caller is now trusted. Fail-open is a vulnerability.
}
```

---

## Typed accessors — keep the controller clean

`validated()` returns `mixed`, so reading it directly in the controller scatters casts and array keys everywhere. Instead, expose **intention-revealing, typed accessor methods** on the Form Request. The controller asks for `installationId()` and gets an `int`; the cast lives in exactly one place.

### ✅ Do — accessors over `validated()`

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests\GitHub;

use Illuminate\Foundation\Http\FormRequest;

/**
 * Validates the GitHub App post-install callback.
 */
final class InstallationCallbackRequest extends FormRequest
{
    /**
     * @return array<string, array<int, string>>
     */
    public function rules(): array
    {
        return [
            'installation_id' => ['required', 'integer', 'min:1'],
            'state' => ['required', 'string'],
        ];
    }

    public function installationId(): int
    {
        return (int) $this->validated('installation_id');
    }

    public function state(): string
    {
        return (string) $this->validated('state');
    }
}
```

The controller then reads like prose:

```php
public function callback(InstallationCallbackRequest $request, LinkInstallation $action): RedirectResponse
{
    $action->handle(
        installationId: $request->installationId(),
        state: $request->state(),
    );

    return to_route('settings.integrations');
}
```

### ❌ Avoid — re-casting `validated()` in the controller

```php
public function callback(InstallationCallbackRequest $request, LinkInstallation $action): RedirectResponse
{
    $action->handle(
        installationId: (int) $request->validated('installation_id'),
        state: (string) $request->validated('state'),
    );

    return to_route('settings.integrations');
}
```

---

## `prepareForValidation()` — normalise before rules run

`prepareForValidation()` runs **before** `rules()`, letting you reshape input so the rules see canonical values. Use it to trim whitespace, lowercase identifiers, split a comma list into an array, or merge in a derived field. Mutate input with `$this->merge()`.

```php
final class CreateReportRequest extends FormRequest
{
    protected function prepareForValidation(): void
    {
        $orgs = $this->input('orgs');

        $this->merge([
            'title' => trim((string) $this->input('title')),
            // Accept "NIMA-Labs, acme/one-repo" as a string and normalise to an array.
            'orgs' => is_string($orgs)
                ? array_values(array_filter(array_map('trim', explode(',', $orgs))))
                : $orgs,
            'slack_channel' => $this->filled('slack_channel')
                ? Str::start(trim((string) $this->input('slack_channel')), '#')
                : null,
        ]);
    }

    // rules() now validates clean, array-shaped `orgs` and a normalised channel.
}
```

Do **not** put authorization or persistence side effects here — it is for input shaping only.

---

## `messages()` and `attributes()` — readable errors

Override `messages()` only where Laravel's default is unclear to an end user, and `attributes()` to give fields human names. Keep both sparse: every overridden message is something you must maintain.

```php
/**
 * @return array<string, string>
 */
public function messages(): array
{
    return [
        'orgs.*.regex' => 'Each org must look like "org" or "org/repo".',
        'lookback_days.between' => 'The lookback window must be between 1 and 90 days.',
    ];
}

/**
 * @return array<string, string>
 */
public function attributes(): array
{
    return [
        'lookback_days' => 'lookback window',
        'slack_channel' => 'Slack channel',
    ];
}
```

---

## A complete Form Request

Putting it together: normalisation, ownership authorization, rules, custom messages, and typed accessors in one class.

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests\Report;

use App\Contracts\Repositories\MembershipRepositoryInterface;
use App\Models\Workspace;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Support\Str;

/**
 * Validates and authorises creating a Report inside the current workspace.
 * The user must be a member of the workspace they are reporting on.
 */
final class CreateReportRequest extends FormRequest
{
    public function authorize(MembershipRepositoryInterface $memberships): bool
    {
        $workspace = $this->route('workspace');

        return $workspace instanceof Workspace
            && $memberships->userBelongsToWorkspace(
                (int) $this->user()->getKey(),
                (int) $workspace->getKey(),
            );
    }

    protected function prepareForValidation(): void
    {
        $orgs = $this->input('orgs');

        $this->merge([
            'title' => trim((string) $this->input('title')),
            'orgs' => is_string($orgs)
                ? array_values(array_filter(array_map('trim', explode(',', $orgs))))
                : $orgs,
            'slack_channel' => $this->filled('slack_channel')
                ? Str::start(trim((string) $this->input('slack_channel')), '#')
                : null,
        ]);
    }

    /**
     * @return array<string, array<int, mixed>>
     */
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:120'],
            'type' => ['required', Rule::in(['standup', 'digest'])],
            'lookback_days' => ['required', 'integer', 'between:1,90'],
            'orgs' => ['required', 'array', 'min:1'],
            'orgs.*' => ['string', 'regex:/^[A-Za-z0-9._-]+(\/[A-Za-z0-9._-]+)?$/'],
            'slack_channel' => ['nullable', 'string', 'starts_with:#'],
        ];
    }

    /**
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'orgs.*.regex' => 'Each org must look like "org" or "org/repo".',
            'lookback_days.between' => 'The lookback window must be between 1 and 90 days.',
        ];
    }

    public function workspace(): Workspace
    {
        $workspace = $this->route('workspace');

        assert($workspace instanceof Workspace);

        return $workspace;
    }

    public function title(): string
    {
        return (string) $this->validated('title');
    }

    public function type(): string
    {
        return (string) $this->validated('type');
    }

    public function lookbackDays(): int
    {
        return (int) $this->validated('lookback_days');
    }

    /**
     * @return list<string>
     */
    public function orgs(): array
    {
        /** @var list<string> $orgs */
        $orgs = $this->validated('orgs');

        return $orgs;
    }

    public function slackChannel(): ?string
    {
        $channel = $this->validated('slack_channel');

        return $channel === null ? null : (string) $channel;
    }
}
```

---

## Livewire's component-side validation

Form Requests govern controller-handled HTTP requests. **Livewire components do not use Form Requests** — their input is bound to public properties, not an HTTP request body, so validation lives on the component. The two equivalent styles:

### `#[Validate]` attributes — declarative, per property

```php
<?php

declare(strict_types=1);

namespace App\Livewire\Reports;

use App\Actions\CreateReport;
use Livewire\Attributes\Validate;
use Livewire\Component;

final class CreateReportForm extends Component
{
    #[Validate('required|string|max:120')]
    public string $title = '';

    #[Validate('required|integer|between:1,90')]
    public int $lookbackDays = 7;

    public function save(CreateReport $createReport): void
    {
        $this->validate();

        $createReport->handle(
            workspace: auth()->user()->currentWorkspace,
            title: $this->title,
            lookbackDays: $this->lookbackDays,
        );

        $this->redirectRoute('reports.index');
    }
}
```

### `rules()` method — when rules are dynamic or share messages

Use a `rules()` method (mirroring the Form Request shape) when rules depend on component state or you want `messages()` / `validationAttributes()` alongside them.

```php
final class CreateReportForm extends Component
{
    public string $title = '';

    public int $lookbackDays = 7;

    /**
     * @return array<string, array<int, mixed>>
     */
    protected function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:120'],
            'lookbackDays' => ['required', 'integer', 'between:1,90'],
        ];
    }

    /**
     * @return array<string, string>
     */
    protected function messages(): array
    {
        return [
            'lookbackDays.between' => 'The lookback window must be between 1 and 90 days.',
        ];
    }

    public function save(CreateReport $createReport): void
    {
        $this->validate();
        // ...
    }
}
```

Prefer `#[Validate]` for simple static rules and the `rules()` method when logic or shared messages justify it. Either way, call `$this->validate()` in the action method before doing work — never trust property values straight from the wire.

## See also

- [Controllers](controllers.md)
- [Livewire](livewire.md)
