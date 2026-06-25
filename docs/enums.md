# Enums

A fixed set of values — a report's lifecycle state, a member's role, how approvals flow — is a *type*, not a loose
string. Model it as a native PHP `enum` so the compiler, the IDE, and static analysis all know every legal value, and
illegal ones simply cannot be constructed.

## Rules at a glance

- Model every fixed, closed set of values as a native `enum` — never a bag of `const` strings or magic literals.
- **Back** enums that persist or cross a boundary: `enum ReportStatus: string`. Use a pure (unbacked) enum only for
  in-memory states that are never stored or serialised.
- Prefer **string** backing values over int — they survive in logs, JSON, and database rows as something a human can read.
- Cast enum columns on the model: `casts()` returns `'status' => ReportStatus::class`. Eloquent then hands you the enum,
  not the string.
- Compare against the **enum case** (`$report->status === ReportStatus::Delivered`), never against a raw string.
- At an untrusted boundary (request, webhook, queue payload) use `tryFrom()` and decide the fallback explicitly; reserve
  `from()` for values you already trust.
- Put derived data — labels, colours, Slack emoji, permission checks — as **methods on the enum**, next to the cases.
- Branch on enums with **`match`**, never `switch`; a total `match` with no `default` lets static analysis flag a case
  you forgot when someone adds one.
- Type parameters and return values as the enum (`ReportStatus $status`), so callers cannot pass an arbitrary string.
- Iterate `Enum::cases()` instead of hand-maintaining a separate list of valid values.

## Back the enum when it persists or crosses a boundary

A *backed* enum (`enum Role: string`) pins a stable scalar to each case. That scalar is what you store in a column, send
over the wire, and read back with `from()`/`tryFrom()`. A *pure* enum has cases but no scalar, so it can never be
persisted or reconstructed from input — it lives only in memory.

The rule: if a value is ever written to a database, serialised to JSON, put on a queue, or parsed from a request, it must
be **string-backed**. Choose string over int so a stray value in a log line or a DB row reads as `approve_first`, not
`2`.

```php
✅ Do — a string-backed enum; the scalar is explicit and stable
<?php

declare(strict_types=1);

namespace App\Enums;

/**
 * How a workspace handles incoming pull requests in the digest pipeline.
 */
enum ApprovalMode: string
{
    case Auto = 'auto';
    case ApproveFirst = 'approve_first';
    case Manual = 'manual';
}
```

```php
❌ Avoid — magic strings scattered across the codebase
final class ApprovalMode
{
    public const AUTO = 'auto';
    public const APPROVE_FIRST = 'approve_first';
    public const MANUAL = 'manual';
}

// Nothing stops a typo, and the type system sees only `string`:
if ($workspace->approval_mode === 'aprove_first') { /* silently never true */ }
```

A pure enum is the right tool only when the set never leaves memory — for example a transient classification computed and
consumed within one request:

```php
✅ Do — a pure enum for an in-memory-only distinction
<?php

declare(strict_types=1);

namespace App\Enums;

/**
 * Whether a contributor's activity this window trends up or down.
 * Computed for rendering and never stored.
 */
enum ActivityTrend
{
    case Rising;
    case Flat;
    case Falling;
}
```

If you later need to persist or serialise an `ActivityTrend`, convert it to a backed enum first — do not paper over it by
calling `->name`.

## Cast enum columns on the model

Eloquent's enum cast is the bridge between the string in the column and the enum in your code. With a cast in place,
reading the attribute yields a `ReportStatus`, and assigning a `ReportStatus` writes its `->value`. Without the cast you
get a bare string everywhere and lose every guarantee the enum was meant to provide.

Declare casts in the `casts()` method (the Laravel 11+ convention), not the legacy `$casts` property.

```php
✅ Do — cast the column so the model speaks in enums
<?php

declare(strict_types=1);

namespace App\Models;

use App\Enums\ApprovalMode;
use App\Enums\ReportStatus;
use Illuminate\Database\Eloquent\Model;

final class Report extends Model
{
    protected function casts(): array
    {
        return [
            'status' => ReportStatus::class,
            'approval_mode' => ApprovalMode::class,
            'delivered_at' => 'immutable_datetime',
        ];
    }
}
```

```php
// Reads and writes are typed end to end:
$report = Report::find($id);
$report->status;                      // ReportStatus (enum), not 'delivered'
$report->status = ReportStatus::Failed;
$report->save();                      // column stores 'failed'
```

Querying still uses the scalar at the SQL layer, but you supply it from the case rather than typing a literal:

```php
✅ Do — pass `->value` to the query builder, never a magic string
Report::query()
    ->where('status', ReportStatus::Delivered->value)
    ->whereDate('delivered_at', '>=', now()->subWeek())
    ->get();

❌ Avoid — a literal the type system can't check
Report::query()->where('status', 'deliverd')->get(); // typo ships silently
```

The migration column is a plain string sized for your longest backing value:

```php
$table->string('status', 32)->default(ReportStatus::Pending->value);
```

## Compare against the case, not the string

Inside the application, compare enums to enum cases. Identity (`===`) on two instances of the same case is `true`, and
the comparison fails to compile-check if you fat-finger the case name — whereas a string literal silently passes the
parser and fails at runtime. Drop to `->value` only at the edges, where you genuinely hold a scalar (a query binding, an
outbound API field).

```php
✅ Do — compare the typed case
if ($report->status === ReportStatus::Delivered) {
    return;
}

❌ Avoid — comparing against a raw string
if ($report->status->value === 'delivered') { // stringly-typed; defeats the enum
    return;
}
```

For "is it one of these?" checks, build the set from cases rather than a string array:

```php
✅ Do
$isTerminal = in_array($report->status, [ReportStatus::Delivered, ReportStatus::Failed], strict: true);
```

The `strict: true` flag matters: it compares by identity so an unrelated value backed by the same scalar can never sneak
a false positive.

## Put derived data and behaviour on the enum

An enum is a class. Labels, colours, Slack emoji, ordering, and small predicates that depend only on the case belong
*on the enum*, beside the cases they describe — not in a `match` copy-pasted across controllers, Blade views, and
notifiers. Centralising them means adding a case forces you to update exactly one file.

```php
✅ Do — the enum owns its presentation and rules
<?php

declare(strict_types=1);

namespace App\Enums;

/**
 * Lifecycle of a generated standup or digest report.
 */
enum ReportStatus: string
{
    case Pending = 'pending';
    case Generating = 'generating';
    case Delivered = 'delivered';
    case Failed = 'failed';

    /** Human-facing label for Slack and email. */
    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Queued',
            self::Generating => 'Generating…',
            self::Delivered => 'Delivered',
            self::Failed => 'Failed',
        };
    }

    /** Slack attachment colour for this state. */
    public function colour(): string
    {
        return match ($this) {
            self::Pending, self::Generating => '#9aa0a6',
            self::Delivered => '#2eb67d',
            self::Failed => '#e01e5a',
        };
    }

    /** A finished report will not change state again. */
    public function isTerminal(): bool
    {
        return match ($this) {
            self::Delivered, self::Failed => true,
            self::Pending, self::Generating => false,
        };
    }
}
```

```php
❌ Avoid — the same mapping duplicated wherever a label is needed
$label = match ($report->status->value) {
    'pending' => 'Queued',
    'generating' => 'Generating…',
    // ...repeated in three other files, drifting out of sync
};
```

Now every call site reads cleanly and stays correct as cases evolve:

```php
$builder->context($report->status->label());
$attachment->color($report->status->colour());
```

For a small, fixed-set role, the same pattern carries permission logic:

```php
✅ Do — a Role enum that answers its own authorisation questions
<?php

declare(strict_types=1);

namespace App\Enums;

enum Role: string
{
    case Admin = 'admin';
    case Member = 'member';

    public function label(): string
    {
        return match ($this) {
            self::Admin => 'Administrator',
            self::Member => 'Member',
        };
    }

    public function canManageWorkspace(): bool
    {
        return $this === self::Admin;
    }
}
```

```php
// Reads like the domain sentence it encodes:
abort_unless($user->role->canManageWorkspace(), 403);
```

## Branch with total `match`, not `switch`

`match` is an expression (it returns a value), compares with `===` (no type juggling), and — crucially — **throws
`\UnhandledMatchError` if no arm matches**. Omit the `default` arm and a total `match` over an enum becomes a checklist:
PHPStan/Larastan and Psalm both report the missing case the moment someone adds one. `switch` does none of this — it
falls through silently and a forgotten `case` just does nothing.

```php
✅ Do — total match, no default; adding a case forces an update here
public function slackEmoji(ReportStatus $status): string
{
    return match ($status) {
        ReportStatus::Pending => ':hourglass:',
        ReportStatus::Generating => ':gear:',
        ReportStatus::Delivered => ':white_check_mark:',
        ReportStatus::Failed => ':x:',
    };
}

❌ Avoid — switch that silently does nothing for an unhandled case
public function slackEmoji(ReportStatus $status): string
{
    switch ($status) {
        case ReportStatus::Pending:
            return ':hourglass:';
        case ReportStatus::Delivered:
            return ':white_check_mark:';
        // Generating and Failed fall through to an empty string — no warning
    }

    return '';
}
```

Reach for a `default` arm only when you *intend* to collapse "everything else" and that intent is stable. If the goal is
exhaustiveness, leaving `default` off is the feature, not an oversight.

## Parse untrusted input with `tryFrom()` and an explicit fallback

`from()` throws a `\ValueError` on an unknown scalar; `tryFrom()` returns `null`. At a trust boundary — a query
parameter, a webhook body, a queue payload, a config string — use `tryFrom()` and decide what `null` means *right there*.
Falling back implicitly (or letting an exception escape) hides bad input; choosing a default — or rejecting it — makes
the decision visible.

```php
✅ Do — tryFrom with a stated default for an optional, untrusted value
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use App\Enums\ApprovalMode;
use Illuminate\Foundation\Http\FormRequest;

final class UpdateWorkspaceRequest extends FormRequest
{
    public function approvalMode(): ApprovalMode
    {
        // Unknown or absent → fall back to the safe default, explicitly.
        return ApprovalMode::tryFrom((string) $this->input('approval_mode'))
            ?? ApprovalMode::Manual;
    }
}
```

```php
✅ Do — tryFrom and reject when the value must be valid
$status = ReportStatus::tryFrom($payload['status'] ?? '');

if ($status === null) {
    throw new \InvalidArgumentException("Unknown report status: {$payload['status']}.");
}
```

```php
❌ Avoid — from() on raw input, letting a \ValueError escape uncaught
$mode = ApprovalMode::from($request->input('approval_mode')); // 500s on any typo
```

Use `from()` only when the value is already known-good — for instance a scalar you wrote yourself moments earlier, or one
the database (with its enum cast) already guarantees.

For request validation, Laravel's `Enum` rule rejects bad input before it reaches your code, which pairs well with
`tryFrom()` defaults downstream:

```php
✅ Do — validate at the door with the Enum rule
use Illuminate\Validation\Rule;

public function rules(): array
{
    return [
        'approval_mode' => ['required', Rule::enum(ApprovalMode::class)],
    ];
}
```

## At a JSON boundary, serialise the scalar and rebuild with `tryFrom()`

A backed enum implements `JsonSerializable`-friendly behaviour: `json_encode` of an enum emits its backing value, so a
DTO carrying an enum serialises to clean JSON automatically. Coming back the other way, never trust the string — rebuild
the case through `tryFrom()`.

```php
✅ Do — encode emits the scalar; decode goes back through tryFrom()
<?php

declare(strict_types=1);

namespace App\Data;

use App\Enums\ReportStatus;

/**
 * A queue/cache-friendly snapshot of a report's delivery state.
 */
final readonly class ReportStatusPayload implements \JsonSerializable
{
    public function __construct(
        public int $reportId,
        public ReportStatus $status,
    ) {
    }

    public static function fromArray(array $data): self
    {
        $status = ReportStatus::tryFrom((string) ($data['status'] ?? ''));

        if ($status === null) {
            throw new \InvalidArgumentException('Payload carried an unknown report status.');
        }

        return new self(
            reportId: (int) $data['reportId'],
            status: $status,
        );
    }

    /** @return array{reportId: int, status: string} */
    public function jsonSerialize(): array
    {
        return [
            'reportId' => $this->reportId,
            'status' => $this->status->value, // 'delivered', not the enum object
        ];
    }
}
```

```php
// Round-trips losslessly through any string transport:
$json = json_encode(new ReportStatusPayload(42, ReportStatus::Delivered));
// {"reportId":42,"status":"delivered"}

$restored = ReportStatusPayload::fromArray(json_decode($json, true));
$restored->status === ReportStatus::Delivered; // true
```

The same discipline applies to API responses: emit `->value`, and document the field as the enum's set of backing values
so consumers see `pending | generating | delivered | failed`, not "some string".

## Iterate cases instead of maintaining a parallel list

`Enum::cases()` returns every case in declaration order. Use it to seed dropdowns, build validation sets, or render a
legend — anything that needs "all the values" — so a new case appears automatically and you never maintain a second list
that drifts.

```php
✅ Do — derive options straight from the enum
/** @return array<string, string> value => label, e.g. ['auto' => 'Automatic'] */
function approvalModeOptions(): array
{
    return collect(ApprovalMode::cases())
        ->mapWithKeys(fn (ApprovalMode $mode): array => [$mode->value => $mode->label()])
        ->all();
}

❌ Avoid — a hand-written list that forgets the next case you add
$options = ['auto' => 'Automatic', 'manual' => 'Manual']; // ApproveFirst missing
```

## See also

- [Code style](code-style.md)
- [Models & Eloquent](models-eloquent.md)
- [DTOs & value objects](dtos-value-objects.md)
- [Repositories](repositories.md)
