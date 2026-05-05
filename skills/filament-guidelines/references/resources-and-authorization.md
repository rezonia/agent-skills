# Filament Resources & Authorization

## Simple vs Full Resources

**Default: always use simple resource** unless the UI is inherently complex (e.g. relation managers, custom sub-pages, multi-step wizards) or the user specifies otherwise.

```bash
# Simple resource — single Manage page with create/edit modals
php artisan make:filament-resource Customer --simple

# Full resource — separate Create, Edit, List, View pages
php artisan make:filament-resource Customer
```

Simple resources generate a `ManageCustomers` page that combines listing and modal-based create/edit. They have no `getRelations()` method since relation managers require separate Edit/View pages.

**When to use full resource:**
- Resource has relation managers that must appear on Edit/View pages
- Create/edit UI is too complex for a modal (multi-step, lots of tabs/sections)
- User explicitly requests separate pages

## Model Policy — Required for Every Resource

Every Filament resource must have a corresponding Model Policy unless the user explicitly opts out.

**Generate:**
```bash
php artisan make:policy CustomerPolicy --model=Customer
```

**Register** (Laravel auto-discovers policies in `app/Policies/` matching the model name convention — verify `AuthServiceProvider` if using non-standard paths):
```php
// app/Providers/AuthServiceProvider.php
protected $policies = [
    Customer::class => CustomerPolicy::class,
];
```

**Policy methods Filament respects automatically:**

| Method | Effect |
|---|---|
| `viewAny()` | Hides resource from nav; blocks list page |
| `create()` | Hides/disables create button |
| `update()` | Hides/disables edit button per record |
| `delete()` | Hides/disables delete button per record |
| `deleteAny()` | Controls bulk delete action |
| `restore()` | Controls restore action (soft deletes) |
| `forceDelete()` | Controls permanent delete |

**Minimal policy template:**
```php
<?php

declare(strict_types=1);

namespace App\Policies;

use App\Models\Customer;
use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;

class CustomerPolicy
{
    use HandlesAuthorization;

    public function viewAny(User $user): bool
    {
        return $user->can('view_any_customer');
    }

    public function create(User $user): bool
    {
        return $user->can('create_customer');
    }

    public function update(User $user, Customer $record): bool
    {
        return $user->can('update_customer');
    }

    public function delete(User $user, Customer $record): bool
    {
        return $user->can('delete_customer');
    }

    public function deleteAny(User $user): bool
    {
        return $user->can('delete_any_customer');
    }
}
```

**Bulk actions — per-record authorization:**
```php
// Use authorizeIndividualRecords() when bulk action needs per-record checks
BulkAction::make('approve')
    ->authorizeIndividualRecords()
    ->action(fn (Collection $records) => $records->each->approve());
```

**Skipping authorization (explicit user opt-out only):**
```php
// Add a comment explaining why
protected static bool $shouldSkipAuthorization = true; // public demo panel, no auth needed
```

## Resource Naming

| Artifact | Pattern | Example |
|---|---|---|
| Resource | `ModelResource` | `CustomerResource` |
| Simple page | `ManageModels` | `ManageCustomers` |
| List page | `ListModels` | `ListCustomers` |
| Create page | `CreateModel` | `CreateCustomer` |
| Edit page | `EditModel` | `EditCustomer` |
| View page | `ViewModel` | `ViewCustomer` |
| Policy | `ModelPolicy` | `CustomerPolicy` |
| Schema class | `ModelForm` | `CustomerForm` |
| Table class | `ModelsTable` | `CustomersTable` |
