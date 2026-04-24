# Laravel Standards (Rezonia)

## Carbon

Always import the Laravel facade-flavored Carbon:

```php
use Illuminate\Support\Carbon;
```

Never `use Carbon\Carbon;` directly. Reason: Laravel's wrapper plays nicely with localization, testing (`Carbon::setTestNow()`), and serialization throughout the framework.

## Helpers

Prefer Laravel helpers over raw PHP for arrays/strings/data:

| Use case | Prefer | Avoid |
|---|---|---|
| Array key existence / get | `Arr::get`, `Arr::has` | `isset`, `array_key_exists` (in business code) |
| Nested array read | `data_get($payload, 'user.address.city')` | manual `$payload['user']['address']['city'] ?? null` |
| Nested array write | `data_set($payload, 'user.address.city', $city)` | nested assignment |
| Array transform | `Arr::map`, `collect()->map` | `array_map` |
| String case | `Str::camel`, `Str::snake`, `Str::kebab` | manual regex |
| Slug | `Str::slug($title)` | hand-rolled |
| UUID/ULID | `Str::uuid()`, `Str::ulid()` | external libs |
| Structured array | `fluent($payload)` (`Illuminate\Support\Fluent`) | hand-coded value object for one-shot use |

`fluent()` is preferred over plain arrays whenever the payload has a known shape and you want to read keys with `->key` accessor and chain mutations.

## PHP 8 Attributes (preferred over property/array config)

When the framework or a package supports both attributes and property/array configuration, use **attributes**:

- Eloquent scopes — `#[Scope]` on the method instead of `scopeXyz` naming convention.
- Validation — `#[Rule(...)]` on Livewire / Filament forms instead of `$rules` array.
- Console commands — `#[AsCommand(...)]` (Symfony) when applicable.
- Routes (when using attribute-routing packages) — `#[Route(...)]` over `routes/*.php` registration only when the project already uses attribute routing; otherwise stick with the route file.

Attributes keep configuration co-located with behavior and survive refactors better than parallel property arrays.

## Routes

`routes/web.php` and `routes/api.php`:

```php
use App\Http\Controllers\Billing\InvoiceController;

Route::prefix('billing')->name('billing.')->group(function () {
    Route::get('/invoices/{invoiceId}', [InvoiceController::class, 'show'])
        ->name('invoices.show');

    Route::post('/invoices/{invoiceId}/refund', [InvoiceController::class, 'refund'])
        ->name('invoices.refund');
});
```

Rules:
- **URL path**: `kebab-case`. Plural for collections (`/invoices`), singular nested for actions when natural.
- **Route name**: `camelCase`, dot-grouped: `billing.invoices.show`.
- **Route parameter**: `camelCase` — `{invoiceId}`, never `{invoice_id}`.
- **Controller action**: tuple form `[Controller::class, 'action']`. Never the legacy `'Controller@action'` string.
- **Single-action controllers**: `__invoke` + tuple `[ProcessRefundController::class]`.
- Implicit model binding allowed when the parameter matches the model's route key, but explicit `->where('id', '[0-9]+')` constraints encouraged for numeric IDs.

## Config

```
config/billing.php
config/external-services.php
```

- Filename: `kebab-case.php`.
- Top-level keys: `snake_case`.
- Values come from `env()` — but **only** inside `config/*.php`:

```php
// config/billing.php
return [
    'currency' => env('BILLING_CURRENCY', 'JPY'),
    'webhook_secret' => env('BILLING_WEBHOOK_SECRET'),
];
```

Everywhere else:

```php
$currency = config('billing.currency');
$secret = config('billing.webhook_secret');
```

Calling `env()` in controllers / services / jobs is forbidden — `php artisan config:cache` strips it and you'll get `null` in production. Lint with PHPStan rule or grep in CI.

## Artisan commands

- Command **name** (the string the user types): `kebab-case` with `:` namespace.
  - `payslip:calculate-daily`
  - `users:purge-soft-deleted`
- Command **class**: `Action + Command` suffix → `CalculateDailyPayslipsCommand`, `PurgeSoftDeletedUsersCommand`.
- Use `#[AsCommand(name: 'payslip:calculate-daily', description: '...')]` attribute when supported, otherwise the `$signature`/`$description` properties.

### Scheduler

Reference the **command class**, not the string name:

```php
// app/Console/Kernel.php
use App\Console\Commands\CalculateDailyPayslipsCommand;
use App\Console\Commands\PurgeSoftDeletedUsersCommand;

protected function schedule(Schedule $schedule): void
{
    $schedule->command(CalculateDailyPayslipsCommand::class)->dailyAt('02:00');
    $schedule->command(PurgeSoftDeletedUsersCommand::class)->weekly();
}
```

Why: refactor-safe. Renaming the string command is detected by the IDE.

## Events & Listeners

Default to **EventServiceProvider auto-discovery**:

```php
// app/Providers/EventServiceProvider.php
public function shouldDiscoverEvents(): bool
{
    return true;
}

protected function discoverEventsWithin(): array
{
    return [
        $this->app->path('Listeners'),
    ];
}
```

Listener types its `handle` method with the event class — Laravel wires it up:

```php
final class SendWelcomeEmailListener
{
    public function handle(UserRegistered $event): void
    {
        // ...
    }
}
```

Do **not**:
- Maintain `protected $listen = [...]` arrays for simple pairs.
- Call `Event::listen(...)` inside `boot()`.

Exception: complex/dynamic event-listener pairs (closures, runtime resolution, multi-listener fan-out with shared config) — those go in `boot()` with a comment explaining why.

## Models / Eloquent

- Models in `app/Models/`.
- `protected $fillable` declared explicitly. Never `$guarded = []`.
- `$casts` for every typed attribute (dates, enums, JSON, booleans).
- Prefer `#[Scope]` attribute (Laravel 11+) over `scopeXyz` method naming when available.
- Relationships return-typed: `public function posts(): HasMany`.

### Query building

- **Always start queries with `Model::query()`** — gives the IDE/static analyzer a real `Builder<Model>` to chain on, so all subsequent methods get type hints and autocomplete.

  ```php
  // Good — IDE knows the builder type
  $users = User::query()
      ->where('status', UserStatus::ACTIVE)
      ->orderBy('created_at')
      ->get();

  // Avoid — static call returns a less-typed chain in many IDEs
  $users = User::where('status', UserStatus::ACTIVE)->get();
  ```

- **Always use Eloquent. The DB query builder (`DB::table(...)`) is the last resort.** Reach for `DB::` only when Eloquent genuinely cannot express the query (heavy bulk inserts, raw recursive CTEs, performance-critical reporting). Document the reason in a comment when you do.

### Mutations must go through model instances

- **Fetch the model record first, then mutate it** (`update`, `delete`, `save`, ...) so Eloquent model events (`saving`, `saved`, `updating`, `updated`, `deleting`, `deleted`) and observers fire correctly.

  ```php
  // Good — events / observers fire
  $user = User::query()->findOrFail($userId);
  $user->update(['status' => UserStatus::SUSPENDED]);

  $post = Post::query()->findOrFail($postId);
  $post->delete();

  // Avoid — bulk builder mutations skip model events
  User::query()->where('id', $userId)->update(['status' => UserStatus::SUSPENDED]);
  Post::query()->where('id', $postId)->delete();
  ```

- **Exception**: when you explicitly want to bypass model events — e.g. data migrations, large backfills, seeders, or maintenance scripts where event side-effects are undesirable — bulk builder mutations are acceptable. Add a short comment stating the intent, e.g. `// bulk update — intentionally bypassing model events for migration`.

## Form Requests

- One per action: `StoreUserRequest`, `UpdateUserRequest`.
- Validation lives there, not in the controller.
- `authorize()` returns the actual policy check, not `true`.

## Resources

- API responses go through `JsonResource` / `ResourceCollection`.
- Field names: `camelCase` in API output even though DB is `snake_case`.

## Jobs

- One job = one action. Put it in `app/Jobs/`.
- Implement `ShouldQueue`. Set `tries`, `backoff`, `timeout` explicitly when non-default.
- Use `#[WithoutRelations]` on the model property when serialization size matters.

## Notifications & Mailables

- Notifications in `app/Notifications/`, suffix `Notification`.
- Mailables in `app/Mail/`, suffix `Mail` (e.g. `WelcomeEmail`).
- Subject + view both translatable through `__()`.

## Middleware

- Class name describes what it enforces: `EnsureUserHasVerifiedEmail`, not `VerifyEmailMiddleware`.
- Aliased in the kernel with `kebab-case`: `'verified-email' => EnsureUserHasVerifiedEmail::class`.

## Testing

- Pest preferred. Group by feature: `tests/Feature/Billing/RefundInvoiceTest.php`.
- Use `RefreshDatabase` only when needed; prefer `LazilyRefreshDatabase`.
- Factories typed: `User::factory()->verified()->create()` with explicit states, not magic ad-hoc closures.
