# Configuration & secrets

Configuration is the single, cacheable surface through which the application reads every tunable value; secrets are the
credentials that surface depends on but which never touch version control. This page draws a hard line between the two:
`env()` lives only in `config/*`, every knob lives in a config file, and every secret lives in `.env` or a secret
manager — encrypted at rest, hashed for lookup, and validated loudly on boot.

## Rules at a glance

- NEVER call `env()` outside `config/*`. Read environment variables into config files; read `config()` everywhere else.
- Every tunable — limits, model names, endpoints, lookback windows, feature flags, caps — lives in a config file. Code never hardcodes them.
- Secrets live in `.env` (gitignored) or a secret manager. They are never committed. Ship a `.env.example` with **blank** values and every key documented.
- In production, prefer reading a secret from a mounted file path (a secret-manager seam) over an inline `.env` value, especially for multi-line keys (PEM, base64).
- Encrypt sensitive columns at rest with the `encrypted` cast; keep a `sha256` hash column alongside for equality lookups, because ciphertext is non-deterministic and unqueryable.
- Fail loudly when a required secret or config value is missing — throw on boot or at the seam. Never silently no-op or fall back to an insecure default.
- Group related config into a dedicated file (`config/digest.php`) rather than scattering keys across `config/services.php`.
- Validate types as you read env into config: cast to `int`, `bool`, `array` in the config file so consumers get real types.

## `env()` only in `config/*`

### Why

Laravel caches configuration with `php artisan config:cache`, which is standard on any production host. Once the config
is cached, **`env()` returns `null` for everything** except the handful of variables the framework reads before caching.
The `.env` file may not even be present in a built container image. So a stray `env('SLACK_WEBHOOK_URL')` deep in an
action works perfectly in local development and then silently returns `null` in production the moment someone runs
`config:cache`. This is one of the most common and most painful Laravel production bugs precisely because it is invisible
until deploy.

The fix is structural: `env()` is an input boundary, not an accessor. Environment variables are read **once**, in
`config/*`, where they are typed, defaulted, and named. The rest of the codebase reads the resulting config tree through
`config()`, which is fully populated whether or not the cache is warm. As a bonus, `config:cache` becomes safe to run,
and the set of environment variables the app consumes is enumerable by reading the config directory.

### How

❌ Avoid — `env()` reached directly from application code:

```php
<?php

declare(strict_types=1);

namespace App\Services\Slack;

use Illuminate\Http\Client\Factory as Http;

final readonly class SlackNotifier
{
    public function __construct(
        private Http $http,
    ) {}

    public function send(string $payload): void
    {
        // ❌ returns null once `config:cache` has run — silent production failure
        $webhook = env('SLACK_WEBHOOK_URL');

        $this->http->post($webhook, ['json' => $payload]);
    }
}
```

✅ Do — read the value into a config file, inject it as config:

```php
<?php

declare(strict_types=1);

// config/digest.php

return [
    'slack' => [
        'webhook_url' => env('SLACK_WEBHOOK_URL'),
        'max_blocks'  => (int) env('SLACK_MAX_BLOCKS', 50),
    ],
];
```

```php
<?php

declare(strict_types=1);

namespace App\Services\Slack;

use Illuminate\Http\Client\Factory as Http;

final readonly class SlackNotifier
{
    public function __construct(
        private Http $http,
        private string $webhookUrl,
        private int $maxBlocks,
    ) {}

    public function send(string $payload): void
    {
        $this->http->post($this->webhookUrl, ['json' => $payload]);
    }
}
```

Bind it in a service provider, resolving config **once** at the composition root rather than reaching for `config()`
scattered through the class:

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use App\Services\Slack\SlackNotifier;
use Illuminate\Support\ServiceProvider;

final class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(SlackNotifier::class, function ($app): SlackNotifier {
            return new SlackNotifier(
                http: $app->make(\Illuminate\Http\Client\Factory::class),
                webhookUrl: (string) config('digest.slack.webhook_url'),
                maxBlocks: (int) config('digest.slack.max_blocks'),
            );
        });
    }
}
```

This keeps `config()` calls at the edges (config files and service providers) and leaves the rest of the code dependent
only on plain typed values it was constructed with — easy to test, with no global state reached at runtime.

### The one exception, and how to confine it

The framework's own bootstrap files (`config/app.php` and the other shipped config files) call `env()` — that is
correct, because config files *are* the boundary. There is no legitimate reason for `env()` to appear under `app/`,
`routes/`, `database/`, or a Blade view. Enforce it with a grep in CI:

```bash
# Fails the build if env() escapes the config directory.
! grep -rn --include='*.php' '\benv(' app routes database resources
```

## Every tunable lives in config — code hardcodes nothing

### Why

A magic number buried in a method (`->take(20)`, `'claude-opus-4-8'`, `now()->subDays(7)`) is invisible to operators,
untestable without editing source, and duplicated the moment a second class needs the same value. Centralising tunables
in a config file gives them a name, a home, and a per-environment override. The lookback window, the contributor cap, the
AI model id, the API endpoint, the Slack block limit — all of these are policy, not logic, and policy belongs in config.

A useful test: if a value might differ between environments, might be tuned without a code review, or appears in more than
one place, it is configuration. Promote it.

### How

❌ Avoid — policy hardcoded into logic:

```php
<?php

declare(strict_types=1);

namespace App\Services\Summaries;

final readonly class PersonSummariser
{
    public function summarise(string $login, string $activity): string
    {
        $response = $this->ai->complete(
            model: 'claude-opus-4-8',                 // ❌ magic string
            endpoint: 'https://api.anthropic.com/v1', // ❌ magic url
            maxTokens: 1024,                          // ❌ magic number
            prompt: $activity,
        );

        return mb_substr($response, 0, 600); // ❌ magic limit
    }
}
```

✅ Do — declare the policy in config, read it as typed values:

```php
<?php

declare(strict_types=1);

// config/digest.php

return [
    'ai' => [
        'primary' => [
            'model'      => env('DIGEST_AI_MODEL', 'claude-opus-4-8'),
            'endpoint'   => env('DIGEST_AI_ENDPOINT', 'https://api.anthropic.com/v1'),
            'max_tokens' => (int) env('DIGEST_AI_MAX_TOKENS', 1024),
        ],
    ],

    'summary' => [
        'max_chars' => (int) env('DIGEST_SUMMARY_MAX_CHARS', 600),
    ],

    'lookback' => [
        'standup_days' => (int) env('DIGEST_STANDUP_DAYS', 1),
        'digest_days'  => (int) env('DIGEST_DIGEST_DAYS', 7),
    ],

    'contributors' => [
        'max_per_report' => (int) env('DIGEST_MAX_CONTRIBUTORS', 40),
    ],

    'features' => [
        'ai_summaries' => (bool) env('DIGEST_AI_SUMMARIES', true),
    ],
];
```

Consumers receive the resolved values and stay free of literals:

```php
<?php

declare(strict_types=1);

namespace App\Services\Summaries;

use App\Contracts\AiClient;

final readonly class PersonSummariser
{
    public function __construct(
        private AiClient $ai,
        private int $maxChars,
    ) {}

    public function summarise(string $login, string $activity): string
    {
        $response = $this->ai->complete($activity);

        return mb_substr($response, 0, $this->maxChars);
    }
}
```

### Cast types where you read them

Environment variables are always strings (`env()` decodes `"true"`/`"false"`/`"null"` to bools/null, but nothing else).
Cast to the type consumers expect **in the config file**, so the value arrives typed and a `declare(strict_types=1)`
consumer never has to coerce a `"7"` into an `int`.

```php
// config/digest.php — types resolved at the boundary
'max_per_report' => (int) env('DIGEST_MAX_CONTRIBUTORS', 40),
'orgs'           => array_filter(explode(',', (string) env('DIGEST_ORGS', ''))),
'ai_summaries'   => (bool) env('DIGEST_AI_SUMMARIES', true),
```

## Secrets live in `.env` or a secret manager — never in git

### Why

A credential committed to version control is compromised the moment it is pushed, and rewriting history does not unshare
it — it must be rotated. `.env` is gitignored by every fresh Laravel install for exactly this reason. The repository
instead carries `.env.example`: a manifest of **which** variables exist, with **blank** values, so a new engineer or a CI
job knows the full set of required configuration without ever seeing a real secret.

### How

`.env.example` documents the contract — every key present, every secret blank:

```dotenv
APP_NAME=Digest
APP_ENV=production
APP_KEY=
APP_DEBUG=false

# GitHub — blank → falls back to `gh auth token` locally
GITHUB_TOKEN=

# AI providers (Anthropic primary, OpenAI fallback)
ANTHROPIC_API_KEY=
OPENAI_API_KEY=

# Slack delivery
SLACK_WEBHOOK_URL=

# Report policy (safe, non-secret defaults documented inline)
DIGEST_ORGS=NIMA-Labs,acme/one-repo
DIGEST_DIGEST_DAYS=7
DIGEST_MAX_CONTRIBUTORS=40

# Path to a mounted private key (preferred over an inline value in prod)
GITHUB_APP_PRIVATE_KEY_PATH=
```

Confirm `.gitignore` excludes the real file and that no secret was ever committed:

```bash
grep -q '^\.env$' .gitignore || echo 'WARNING: .env is not gitignored'
git log --all --full-history -- .env   # must return nothing
```

Non-secret tunables (`DIGEST_DIGEST_DAYS`) can carry meaningful example values; secrets (`ANTHROPIC_API_KEY`) must be
blank. Keeping the example in lockstep with `config/*` means the documented keys and the consumed keys never drift.

## Prefer a mounted file path over an inline secret in production

### Why

Secret managers (Kubernetes Secrets, Vault, AWS/GCP Secret Manager) mount credentials as files, not as environment
variables. File-mounted secrets are rotated without restarting the process, are not exposed via `printenv` or a crash
dump's environment block, and — critically — survive being multi-line. A PEM private key or a base64 blob crammed into a
single `.env` line is fragile: newlines get mangled, quoting breaks, and the value bloats every process's environment.

The robust pattern is a *seam*: config exposes both an inline value and a file path. Prefer the file when present, fall
back to the inline value otherwise. Production mounts the file; local development can paste a value.

### How

```php
<?php

declare(strict_types=1);

// config/services.php

return [
    'github_app' => [
        'app_id' => env('GITHUB_APP_ID'),

        // Preferred: a path a secret manager mounts the key at.
        'private_key_path' => env('GITHUB_APP_PRIVATE_KEY_PATH'),

        // Fallback: an inline value, acceptable for local dev only.
        'private_key' => env('GITHUB_APP_PRIVATE_KEY'),
    ],
];
```

A small resolver reads the path first and fails loudly if neither source yields a key:

```php
<?php

declare(strict_types=1);

namespace App\Services\GitHub;

use App\Exceptions\MissingSecretException;

/**
 * Resolves the GitHub App private key, preferring a secret-manager-mounted
 * file over an inline config value, and never returning an empty key.
 */
final readonly class GitHubAppKeyResolver
{
    public function __construct(
        private ?string $path,
        private ?string $inline,
    ) {}

    public function resolve(): string
    {
        if ($this->path !== null && $this->path !== '') {
            if (! is_readable($this->path)) {
                throw new MissingSecretException(
                    "GitHub App private key path is set but not readable: {$this->path}",
                );
            }

            return $this->readKey($this->path);
        }

        if ($this->inline !== null && $this->inline !== '') {
            return $this->inline;
        }

        throw new MissingSecretException(
            'No GitHub App private key configured: set GITHUB_APP_PRIVATE_KEY_PATH or GITHUB_APP_PRIVATE_KEY.',
        );
    }

    private function readKey(string $path): string
    {
        $contents = file_get_contents($path);

        if ($contents === false || trim($contents) === '') {
            throw new MissingSecretException("GitHub App private key file is empty: {$path}");
        }

        return $contents;
    }
}
```

Wire it from config in a provider, so the path-vs-inline choice is made once:

```php
$this->app->singleton(GitHubAppKeyResolver::class, fn (): GitHubAppKeyResolver => new GitHubAppKeyResolver(
    path: config('services.github_app.private_key_path'),
    inline: config('services.github_app.private_key'),
));
```

## Encrypt sensitive data at rest, hash for lookup

### Why

Some secrets must be **stored**, not just read — a per-workspace Slack token, an installed GitHub App access token, a
user's personal API key. Storing these as plaintext columns means a single database leak exposes every credential. The
`encrypted` cast solves storage: Laravel transparently encrypts on write and decrypts on read using `APP_KEY`, so the
column holds ciphertext at rest while your code sees the plaintext.

But Laravel's encryption is **non-deterministic** — the same plaintext encrypts to a different ciphertext every time
(it includes a random IV). That is a security feature, and it means you **cannot** `where('token', $value)` against an
encrypted column: the stored ciphertext will never equal a freshly-encrypted needle. The standard pattern is a companion
`sha256` hash column: a deterministic, one-way digest you *can* index and query, used solely for equality lookup, while
the encrypted column holds the recoverable secret.

### How

Migration — an encrypted secret plus a deterministic, indexed hash for lookup:

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
        Schema::create('workspace_credentials', function (Blueprint $table): void {
            $table->id();
            $table->foreignId('workspace_id')->constrained()->cascadeOnDelete();

            // Ciphertext at rest — wide column, not indexable.
            $table->text('slack_token');

            // Deterministic digest of the same value — indexed, for lookups.
            $table->string('slack_token_hash', 64)->unique();

            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('workspace_credentials');
    }
};
```

Model — cast the secret to `encrypted`, and set the hash whenever the secret is set:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

final class WorkspaceCredential extends Model
{
    protected $fillable = ['workspace_id', 'slack_token'];

    protected function casts(): array
    {
        return [
            'slack_token' => 'encrypted',
        ];
    }

    /**
     * Keep the hash in lockstep with the encrypted token: any write to
     * `slack_token` updates `slack_token_hash` with its deterministic digest.
     */
    protected function slackToken(): Attribute
    {
        return Attribute::make(
            set: fn (string $value): array => [
                'slack_token'      => $value,
                'slack_token_hash' => self::hashToken($value),
            ],
        );
    }

    public static function hashToken(string $token): string
    {
        return hash('sha256', $token);
    }
}
```

Because the `encrypted` cast already runs on the `slack_token` key, the value placed in the returned array is encrypted by
the cast layer before persistence, while `slack_token_hash` is stored verbatim. Look the record up by hash — never by the
ciphertext:

```php
<?php

declare(strict_types=1);

namespace App\Repositories;

use App\Contracts\Repositories\WorkspaceCredentialRepositoryInterface;
use App\Models\WorkspaceCredential;

final readonly class WorkspaceCredentialRepository implements WorkspaceCredentialRepositoryInterface
{
    public function findBySlackToken(string $token): ?WorkspaceCredential
    {
        // ✅ query the deterministic hash, not the encrypted column
        return WorkspaceCredential::query()
            ->where('slack_token_hash', WorkspaceCredential::hashToken($token))
            ->first();
    }
}
```

❌ Avoid — querying the encrypted column directly (always misses):

```php
// ❌ ciphertext is non-deterministic; this never matches a stored row
WorkspaceCredential::query()
    ->where('slack_token', $token)
    ->first();
```

A note on rotation: `encrypted` columns are tied to `APP_KEY`. When rotating the key, configure `APP_PREVIOUS_KEYS` so
old ciphertext still decrypts, then re-save rows to re-encrypt under the new key. The `sha256` hash is unaffected by key
rotation, which is another reason to look up by hash.

## Fail loudly when a required secret or config is missing

### Why

A missing credential is a fatal configuration error, not a runtime condition to paper over. If `ANTHROPIC_API_KEY` is
unset and the code silently skips AI summaries, you ship a degraded report that looks fine until someone notices the
summaries vanished a week ago. Silent fallbacks turn a five-minute "the env var is missing" fix into a multi-day
investigation. Fail at the earliest possible point — on boot or at the seam where the secret is first needed — with a
message that names the exact missing key.

### How

Validate required configuration as the service is constructed, and throw a typed exception:

```php
<?php

declare(strict_types=1);

namespace App\Services\Ai;

use App\Contracts\AiClient;
use App\Exceptions\MissingSecretException;

final readonly class AnthropicClient implements AiClient
{
    public function __construct(
        private string $apiKey,
        private string $model,
    ) {
        if ($this->apiKey === '') {
            throw new MissingSecretException(
                'ANTHROPIC_API_KEY is required for AI summaries but is not set.',
            );
        }
    }

    public function complete(string $prompt): string
    {
        // ... real call, guaranteed a non-empty key by construction
    }
}
```

A dedicated exception type makes the failure mode explicit and catchable:

```php
<?php

declare(strict_types=1);

namespace App\Exceptions;

use RuntimeException;

final class MissingSecretException extends RuntimeException
{
}
```

❌ Avoid — silently degrading when a required secret is absent:

```php
public function complete(string $prompt): string
{
    if ($this->apiKey === '') {
        return ''; // ❌ silent no-op — the report is quietly diminished
    }

    // ...
}
```

A *deliberate* fallback (Anthropic → OpenAI) is different from a silent no-op: it is an explicit, configured chain where
each link is itself a fully-functional client, and exhausting the chain still throws. "No provider configured at all" is
a loud failure; "primary provider is down, try the secondary" is designed resilience. The distinction is whether the
degraded path was chosen on purpose and still guarantees a correct result.

For boot-time validation of the whole config surface, assert in a service provider's `boot()` so the process refuses to
start misconfigured rather than failing on the first request:

```php
public function boot(): void
{
    foreach (['digest.slack.webhook_url', 'services.anthropic.key'] as $required) {
        if (blank(config($required))) {
            throw new MissingSecretException("Required config '{$required}' is not set.");
        }
    }
}
```

## See also

- [Error handling](error-handling.md) — typed exceptions, failing loudly, and the difference between a designed fallback and a silent no-op.
- [Models & Eloquent](models-eloquent.md) — the `encrypted` cast, attribute mutators, and where Eloquent is allowed to live.
