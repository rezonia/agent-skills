# PHP Standards (Rezonia)

Base style: **PER Coding Style 2.0** (successor to PSR-12). Enforced strictly.

## File header

Every new `.php` file begins with:

```php
<?php

declare(strict_types=1);

namespace App\...;
```

Blank line after `<?php`, blank line after `declare`, blank line after `namespace`, then `use` imports grouped and alphabetical.

## Imports

- Import every class you use — no inline FQN like `\App\Services\Foo`.
- Group order: classes, then `function`, then `const`. Blank line between groups.
- No unused imports.

```php
use App\Models\User;
use Illuminate\Support\Carbon;
use Illuminate\Support\Collection;

use function array_filter;
```

## Types

Priority: **PHP native type** > **docblock**.

- All class properties typed.
- All function/method arguments typed.
- All function/method return types declared (including `void`, `never`, `self`, `static`).
- Nullable via `?Type`, union via `Type|OtherType`, intersection via `Type&Other`.
- `mixed` is a **last resort**. Prefer explicit unions.
- For generics the runtime can't express, use docblocks:

```php
/**
 * @return Collection<int, User>
 */
public function activeUsers(): Collection
{
    // ...
}
```

- For array shapes, prefer `array{key: type, ...}` or dedicated DTOs / value objects.

## Control flow — early return, no else, happy path last

```php
// Bad
public function price(Product $product): int
{
    if ($product->isOnSale()) {
        return $product->salePrice();
    } else {
        return $product->regularPrice();
    }
}

// Good — happy path last, guard clauses on top
public function price(Product $product): int
{
    if ($product->isOnSale()) {
        return $product->salePrice();
    }

    return $product->regularPrice();
}
```

Rules:
- No `else` after a `return`, `throw`, `continue`, or `break`.
- Error / edge cases fail fast at the top; the success path flows to the bottom.
- Maximum one level of meaningful nesting inside a method. Extract helpers when you need more.

## Always use braces

```php
// Bad
if ($user->isAdmin()) return true;

foreach ($users as $u) doSomething($u);

// Good
if ($user->isAdmin()) {
    return true;
}

foreach ($users as $u) {
    doSomething($u);
}
```

Applies to `if`, `else`, `elseif`, `for`, `foreach`, `while`, `do`.

## Ternary operators

Short ternaries inline:

```php
$label = $user->isAdmin() ? 'admin' : 'user';
```

Longer ternaries: split by component, one per line, `?` and `:` at the start of the continuation:

```php
$url = $user->isAdmin()
    ? route('admin.dashboard')
    : route('user.dashboard');
```

Avoid nested ternaries — use `match` instead:

```php
$label = match (true) {
    $user->isAdmin() => 'admin',
    $user->isEditor() => 'editor',
    default => 'user',
};
```

`match` is preferred over `switch` for value mapping.

## Enums

- Backed enums use string or int values. Default to string.
- Case NAMES and string VALUES are both `UPPER_SNAKE_CASE`.
- Class name is self-describing, **no `Enum` suffix**.

```php
<?php

declare(strict_types=1);

namespace App\Enums;

enum UserType: string
{
    case ADMIN = 'ADMIN';
    case STAFF = 'STAFF';
    case EXTERNAL_CONTRACTOR = 'EXTERNAL_CONTRACTOR';

    public function label(): string
    {
        return match ($this) {
            self::ADMIN => __('user.type.admin'),
            self::STAFF => __('user.type.staff'),
            self::EXTERNAL_CONTRACTOR => __('user.type.external_contractor'),
        };
    }
}
```

## Comments

- **If the code is self-explanatory, write no comment.** Rename the variable/function instead.
- Inline short comment: `//`
- Block explanation: `/* ... */`
- Only use `/** ... */` docblocks when adding information the types cannot (generics, `@throws`, `@deprecated`, semantics).
- Never duplicate the signature in a docblock.

```php
$amount = 1000; // cents, not dollars

/*
 * Two-phase commit:
 * 1. Reserve inventory.
 * 2. Charge card; on failure, release reservation.
 */
$this->commit($order);
```

Do NOT write:

```php
/**
 * Get the user.
 *
 * @return User
 */
public function getUser(): User
```

The signature already says it.

## Strings

Prefer, in order:

1. `__('greeting :name', ['name' => $name])` — translatable + safe.
2. `"hello {$name}"` — interpolation, when not user-facing.
3. `'hello ' . $name` — concatenation, only when the others don't fit.

## Classes

- One class per file. Filename matches class name (PSR-4).
- `final` by default for service classes, DTOs, value objects, and enums.
- `readonly` for DTO / value-object properties.
- Constructor property promotion for value objects and services:

```php
final readonly class Money
{
    public function __construct(
        public int $amount,
        public string $currency,
    ) {}
}
```

- Visibility on every method and property: `public` / `protected` / `private` — do not rely on defaults.

## Null & Booleans

- Use `?Type` for nullables, never `Type|null` mixed with `Type`.
- Avoid returning `null` when an empty collection / default object works — signal intent with types.
- Boolean methods read as predicates: `isActive()`, `hasAccess()`, `canEdit()`.

## Error handling

- Throw domain exceptions (`App\Exceptions\...`), not `\Exception`.
- `try`/`catch` the narrowest type you handle; re-throw everything else.
- Never swallow exceptions silently; at minimum `report($e);` before continuing.

## Tests

- Pest preferred; PHPUnit acceptable when the project is already on it.
- One behavior per test.
- Names describe behavior: `it_charges_the_card_on_renewal()` / `test_renewal_charges_the_card()`.
- No logic in tests — extract helpers or fixtures.
