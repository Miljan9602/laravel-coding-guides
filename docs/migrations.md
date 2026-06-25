# Migrations & schema conventions

Every change to the database shape goes through a migration: there is no manual `ALTER TABLE`, no schema edit in a console, no "I'll fix it on prod." A migration is source code — versioned, reviewed, reversible, and replayable from an empty database to the exact schema the models expect. This page covers how migrations are written so that they are safe to run forward, safe to roll back, and don't silently corrupt data along the way.

## Rules at a glance

- Every migration is an **anonymous class** (`return new class extends Migration`) with `declare(strict_types=1);` and is named `create_<table>_table` or `add_<col>_to_<table>_table`.
- Every migration has a **real, tested `down()`** that exactly reverses `up()` — or is **deliberately irreversible** (a `down()` that `throw`s with a comment). **Never** an empty `down()`.
- Store **money and quantities** as integer **cents** or `decimal(n, m)` — **never** `float`/`double`. Binary floating point cannot represent decimal cents and silently corrupts totals.
- `->change()` **silently drops** every modifier you don't restate (`unsigned()`, `default()`, `nullable()`). Always **restate the full column definition** when changing.
- Use the `foreignId()->constrained()->cascadeOnDelete()` / `->nullOnDelete()` sugar. `->nullable()` must come **before** `->constrained()`, and `constrained()` **already indexes** the column — don't add a redundant index.
- Index the columns you **filter and sort on**; composite indexes follow the **leftmost-prefix** rule, so column order matters. Measure with `EXPLAIN` before adding a speculative index.
- A **data backfill** belongs in a **queued, chunked job**, not inline in a schema migration. Schema migrations change shape; they don't churn rows.

## Migrations are reversible — a real `down()` or none at all

### Why

`down()` is not decoration. It runs during local development every time you `migrate:rollback` to amend the migration you're iterating on, it runs in CI's down-then-up smoke test, and it is the lever you reach for when a deploy goes wrong. An **empty `down()`** is the worst of both worlds: `migrate:rollback` reports success while leaving the table, column, or index in place, so the next `migrate` either errors ("column already exists") or — worse — silently no-ops and you debug a phantom schema for an hour.

There are exactly two honest states for a migration: it is **reversible** (`down()` precisely undoes `up()`), or it is **deliberately irreversible** (dropping a column destroys data that `down()` cannot recreate, so `down()` throws and says so). An empty body claims reversibility it doesn't have.

### How

✅ Do — `down()` mirrors `up()` exactly:

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('contributor_summaries', function (Blueprint $table): void {
            $table->id();
            $table->foreignId('report_id')->constrained()->cascadeOnDelete();
            $table->string('login');
            $table->text('body');
            $table->timestamps();

            $table->unique(['report_id', 'login']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('contributor_summaries');
    }
};
```

✅ Do — when reversal genuinely can't recreate the data, **declare it irreversible**:

```php
public function up(): void
{
    Schema::table('reports', function (Blueprint $table): void {
        // Replaced by `metrics` (jsonb); the raw column is being dropped.
        $table->dropColumn('legacy_metrics_blob');
    });
}

public function down(): void
{
    // Irreversible: the dropped blob is not reconstructable from the new
    // `metrics` column. Restore from a backup if you must roll back past this.
    throw new \RuntimeException(
        'Cannot reverse: legacy_metrics_blob was destructively migrated.',
    );
}
```

❌ Avoid — the empty `down()` that lies about reversibility:

```php
public function up(): void
{
    Schema::table('reports', function (Blueprint $table): void {
        $table->string('slug')->nullable();
    });
}

// ❌ rollback "succeeds" but leaves `slug` in place; the next migrate breaks.
public function down(): void
{
    //
}
```

The reversal must be **exact**, not approximate. If `up()` added a column *and* an index, `down()` drops both. If `up()` renamed `team` to `workspace`, `down()` renames it back — not "drops workspace." A `down()` that leaves residue is as misleading as an empty one. Prove it: run `php artisan migrate`, then `php artisan migrate:rollback`, then `php artisan migrate` again — a correct pair round-trips cleanly. See [Testing](testing.md) for wiring that round-trip into CI.

## Money and quantities: integer cents or DECIMAL, never float

### Why

`float` and `double` are IEEE-754 binary floating point. They cannot exactly represent most decimal fractions — `0.10` and `0.20` are stored as the nearest binary approximation, so `0.10 + 0.20` is `0.30000000000000004`, not `0.30`. Accumulate that error across an invoice, a usage meter, or a billing rollup and the totals drift, comparisons (`=== 0.30`) fail, and reconciliation reports go red. The corruption is silent: nothing throws, the numbers just stop adding up.

The two correct representations are **integer minor units** (store cents — `1999` for `$19.99` — in a `bigInteger`/`unsignedBigInteger`) or **fixed-point `decimal(precision, scale)`**, which the database stores and arithmetics exactly. Integer cents is the default for currency in this codebase; the model casts it back to a money value object on read. `checklists.md` already assumes a money cast exists — these columns are what it casts.

### How

✅ Do — integer minor units, cast to a money value object on the model:

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('workspace_invoices', function (Blueprint $table): void {
            $table->id();
            $table->foreignId('workspace_id')->constrained()->cascadeOnDelete();

            // Money as integer minor units (cents). Exact, no rounding.
            $table->unsignedBigInteger('amount_cents');
            $table->string('currency', 3);

            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('workspace_invoices');
    }
};
```

✅ Also fine — fixed-point `decimal` when you need fractional units the integer-cents model can't carry (tax rates, per-unit rates):

```php
// 12 digits total, 4 after the point: rates like 0.0825 stay exact.
$table->decimal('tax_rate', 12, 4);
```

❌ Avoid — `float`/`double` for any money or exact quantity:

```php
Schema::create('workspace_invoices', function (Blueprint $table): void {
    // ❌ 0.10 + 0.20 !== 0.30. Totals drift, equality checks fail silently.
    $table->float('amount');
    $table->double('balance');
});
```

Match the model to the column: an `amount_cents` integer column is cast through a money value object (see [DTOs & value objects](dtos-value-objects.md)) so application code never juggles raw cents, and a `decimal` column casts to `'decimal:4'` so it materialises as a string, not a lossy float. `float` is acceptable only for genuinely approximate scientific values — never money, never counts.

## `->change()` silently drops the modifiers you don't restate

### Why

`->change()` does not patch a column — it **redefines** it from the definition you give. Any modifier you omit is **dropped**, silently. Change a column to add a default and forget to restate `->unsigned()`, and the column becomes signed. Forget `->nullable()`, and an existing nullable column becomes `NOT NULL` — which then **fails on the existing NULL rows** mid-migration, or, on databases that coerce, rewrites them to `0`/`''`. This is data loss with no error at definition time; you find out in production.

The rule is mechanical: when you `->change()` a column, write its **complete** target definition — type, length, *and every modifier it should keep* — exactly as if you were declaring it fresh.

### How

✅ Do — restate the full definition, keeping every modifier that must survive:

```php
public function up(): void
{
    Schema::table('reports', function (Blueprint $table): void {
        // Full restated definition: still unsigned, still nullable, new default.
        $table->unsignedInteger('contributor_count')
            ->nullable()
            ->default(0)
            ->change();
    });
}

public function down(): void
{
    Schema::table('reports', function (Blueprint $table): void {
        // Reverse is just as explicit — restate the prior full definition.
        $table->unsignedInteger('contributor_count')
            ->nullable()
            ->default(null)
            ->change();
    });
}
```

❌ Avoid — a partial `->change()` that drops modifiers by omission:

```php
Schema::table('reports', function (Blueprint $table): void {
    // ❌ Omits unsigned() and nullable(): the column becomes a signed,
    //    NOT NULL integer — the redefinition silently strips both, and
    //    existing NULL rows break or get coerced to 0.
    $table->integer('contributor_count')->default(0)->change();
});
```

Two edge cases to plan for. First, `->change()` on the `doctrine/dbal`-backed path historically couldn't see certain modifiers at all (notably enum columns) — when in doubt, prefer a `create → copy → drop → rename` migration for a fraught change over an in-place `->change()`. Second, a "widening" change (e.g. `string(100)` → `string(255)`) is safe to restate, but a "narrowing" one (`text` → `string(255)`) **truncates** any longer existing value — back-fill or validate first, and write a `down()` that is honest about the lost length (often irreversible).

## Foreign keys: the `constrained()` sugar, ordering, and indexing

### Why

`foreignId('report_id')->constrained()` is the canonical way to declare a foreign key: it creates an **unsigned big integer** column, infers the referenced table from the column name (`report_id` → `reports.id`), adds the foreign-key constraint, **and creates the index the constraint needs**. The expressive `->cascadeOnDelete()` / `->nullOnDelete()` modifiers state the referential action inline, which is exactly where a reviewer wants to see "deleting a report deletes its summaries." But two details bite people: modifier **ordering** and a **redundant index**.

### How

✅ Do — `constrained()` plus an explicit on-delete action; nullable FK orders `->nullable()` first:

```php
public function up(): void
{
    Schema::create('contributor_summaries', function (Blueprint $table): void {
        $table->id();

        // Required FK: deleting the report deletes its summaries.
        $table->foreignId('report_id')
            ->constrained()
            ->cascadeOnDelete();

        // Optional FK: ->nullable() MUST precede ->constrained().
        // On delete of the channel, null the column rather than the row.
        $table->foreignId('delivery_channel_id')
            ->nullable()
            ->constrained()
            ->nullOnDelete();

        $table->timestamps();
    });
}

public function down(): void
{
    Schema::dropIfExists('contributor_summaries');
}
```

❌ Avoid — wrong modifier order and a redundant index:

```php
Schema::create('contributor_summaries', function (Blueprint $table): void {
    $table->id();

    // ❌ ->nullable() AFTER ->constrained() applies to the wrong builder and
    //    the column ends up NOT NULL — the nullable() is effectively ignored.
    $table->foreignId('delivery_channel_id')
        ->constrained()
        ->nullable()
        ->nullOnDelete();

    // ❌ constrained() already created an index on report_id; this is a
    //    duplicate index — wasted writes and storage, no read benefit.
    $table->foreignId('report_id')->constrained();
    $table->index('report_id');
});
```

Edge cases. When the column name doesn't follow the convention, name the table explicitly: `->constrained(table: 'github_installations')`, or use `->references('id')->on('...')` for a fully custom target. The `->cascadeOnDelete()` action is enforced **by the database**, so it fires for raw deletes too (not just Eloquent) — that is the point; don't reimplement it in a model event. And remember the **drop order**: in `down()` you drop the child table (or `dropForeign` then `dropColumn`) before the parent, or the constraint blocks you — `dropConstrainedForeignId('report_id')` drops the FK, its index, and the column in one call.

## Indexing: index what you query, mind column order, measure first

### Why

An index makes reads that **filter or sort** on a column fast, at the cost of slower writes and more storage. So you index the columns the application actually queries on — and *only* those. The codebase looks up a workspace by `github_token_hash` on the GitHub-App webhook hot path; without an index that is a full table scan on every call. With one it is a single seek. The two failure modes are equal and opposite: **missing** the index your hot query needs, and **adding speculative indexes** "just in case" that only tax every write.

Composite indexes add a subtlety: they obey the **leftmost-prefix rule**. An index on `(workspace_id, status, created_at)` can serve queries filtering on `workspace_id`, on `workspace_id + status`, and on all three — but **not** a query that filters on `status` alone, because `status` isn't a leftmost prefix. Order the columns from most-selective / always-present to least.

### How

✅ Do — index the real lookup column; order a composite by the access pattern:

```php
public function up(): void
{
    Schema::table('workspaces', function (Blueprint $table): void {
        // Deterministic hash of the encrypted GitHub token; the webhook path
        // looks workspaces up by this exact value — index the equality lookup.
        $table->string('github_token_hash')->index();
    });

    Schema::table('reports', function (Blueprint $table): void {
        // Reads are "this workspace's reports of a status, newest first".
        // Leftmost prefix = workspace_id, then status, then the sort column.
        $table->index(['workspace_id', 'status', 'created_at']);
    });
}

public function down(): void
{
    Schema::table('workspaces', function (Blueprint $table): void {
        $table->dropIndex(['github_token_hash']);
    });

    Schema::table('reports', function (Blueprint $table): void {
        $table->dropIndex(['workspace_id', 'status', 'created_at']);
    });
}
```

❌ Avoid — speculative indexes and a column order the queries can't use:

```php
Schema::table('reports', function (Blueprint $table): void {
    // ❌ Indexing every column "to be safe" — each one slows inserts/updates
    //    and none was justified by a measured query.
    $table->index('title');
    $table->index('body');
    $table->index('metrics');

    // ❌ Leading with the low-selectivity, not-always-filtered column means
    //    the common "by workspace" query can't use this index at all.
    $table->index(['status', 'workspace_id']);
});
```

Before adding an index to fix a slow query, **measure it**: run the actual query under `EXPLAIN` (or `EXPLAIN ANALYZE`), confirm it's doing a sequential scan, add the index, and re-run `EXPLAIN` to confirm the planner now uses it. An index the planner ignores is pure write-tax. Use `->unique()` when the column is a business identity (a `(report_id, login)` summary can't duplicate) — that gives you both the constraint and the index. Foreign keys created with `constrained()` are **already indexed**; don't double up (see the section above).

## Anonymous classes, naming, and where backfills belong

### Why

Modern Laravel migrations are **anonymous classes** (`return new class extends Migration`) — there is no class name to collide, so two migrations adding a column on the same day don't clash. The **file name** is the contract: `create_workspaces_table`, `add_github_token_hash_to_workspaces_table`. A reader scanning `database/migrations/` learns the entire schema history from the file names alone, and Laravel infers the default table name from `create_<table>_table`.

The other half of the rule is about **what a migration must not do**: churn data. A schema migration changes *shape* — it runs once, synchronously, holding a connection, often inside a deploy step with a timeout. Loop over a million rows to backfill a new column inside `up()` and you get a migration that takes ten minutes, times out the deploy, locks the table, and has no retry. Backfills are **queued, chunked jobs** — they run after the column exists, in the background, in batches, idempotently.

### How

✅ Do — anonymous class, conventional name; the migration only adds the column:

```php
<?php

declare(strict_types=1);

// File: 2026_06_25_120000_add_github_token_hash_to_workspaces_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('workspaces', function (Blueprint $table): void {
            $table->string('github_token_hash')->nullable()->index();
        });
    }

    public function down(): void
    {
        Schema::table('workspaces', function (Blueprint $table): void {
            $table->dropIndex(['github_token_hash']);
            $table->dropColumn('github_token_hash');
        });
    }
};
```

✅ Do — backfill the new column in a **queued, chunked** job, run *after* the migration:

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Contracts\Repositories\WorkspaceRepositoryInterface;
use App\Models\Workspace;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\Queueable;

final class BackfillGithubTokenHashes implements ShouldQueue
{
    use Dispatchable;
    use Queueable;

    public function handle(WorkspaceRepositoryInterface $workspaces): void
    {
        $workspaces->eachUnhashedInChunks(500, function (Workspace $workspace): void {
            $workspace->github_token = $workspace->github_token; // mutator re-derives the hash
            $workspace->save();
        });
    }
}
```

❌ Avoid — a backfill loop wedged inside the schema migration:

```php
public function up(): void
{
    Schema::table('workspaces', function (Blueprint $table): void {
        $table->string('github_token_hash')->nullable();
    });

    // ❌ Synchronous, unbatched data churn inside a schema migration:
    //    - blocks the deploy step and can time out
    //    - holds a lock on the whole table
    //    - no retry, no chunking, no idempotency
    //    - touches models in a context Eloquent events don't expect
    foreach (Workspace::all() as $workspace) {
        $workspace->github_token_hash = hash_hmac(
            'sha256',
            $workspace->github_token,
            (string) config('app.key'),
        );
        $workspace->save();
    }
}
```

Edge cases. The chunked iteration belongs behind a **repository method** (`eachUnhashedInChunks`), not a raw `Workspace::chunk()` in the job — data access funnels through repositories (see [Repositories](repositories.md)). Make the job **idempotent**: re-running it must converge, so it selects only rows still needing the backfill rather than blindly rewriting all of them. And keep the new column **`nullable()` first**: add it empty, deploy, run the backfill, then a *later* migration tightens it to `NOT NULL` once every row is populated — three steps, never one, so the schema change and the data change never race.

## See also

- [Models & Eloquent](models-eloquent.md)
- [Naming conventions](naming-conventions.md)
- [Repositories](repositories.md)
