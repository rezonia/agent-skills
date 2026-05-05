# Filament UI Conventions

## Icons — Prefer Font Awesome Solid (`fas-*`)

Use `fas-*` (Font Awesome Solid) icons by default. Only use Heroicons when the project already uses them or the user specifies.

**Install blade-fontawesome if not present:**
```bash
composer require owenvoke/blade-fontawesome
```

**Usage:**
```php
->icon('fas-user')
->icon('fas-trash')
->icon('fas-pencil')
->icon('fas-eye')
->icon('fas-plus')
->icon('fas-check')
->icon('fas-times')
->icon('fas-envelope')
->icon('fas-file-export')
->icon('fas-arrow-left')
```

**Common mappings (Heroicon → Font Awesome):**

| Purpose | Heroicon (avoid) | Font Awesome (prefer) |
|---|---|---|
| Edit | `heroicon-o-pencil` | `fas-pencil` |
| Delete | `heroicon-o-trash` | `fas-trash` |
| View | `heroicon-o-eye` | `fas-eye` |
| Create | `heroicon-o-plus` | `fas-plus` |
| Save | `heroicon-o-check` | `fas-check` |
| Close | `heroicon-o-x-mark` | `fas-times` |
| User | `heroicon-o-user` | `fas-user` |
| Email | `heroicon-o-envelope` | `fas-envelope` |
| Export | `heroicon-o-arrow-down-tray` | `fas-file-export` |
| Back | `heroicon-o-arrow-left` | `fas-arrow-left` |

## SPA Mode Compatibility

Filament panels can enable SPA mode via `->spa()` in panel config. All components must work with `wire:navigate`.

**Panel config:**
```php
// app/Providers/Filament/AdminPanelProvider.php
->spa()
// optional — prefetch on hover
->spa(hasPrefetching: true)
```

**Navigation links — always use Filament's link component with `wire:navigate`:**
```blade
{{-- ✅ correct --}}
<x-filament::link :href="route('filament.admin.resources.customers.index')" wire:navigate>
    @lang('nav.customers')
</x-filament::link>

{{-- ❌ avoid — breaks SPA navigation --}}
<a href="{{ route('filament.admin.resources.customers.index') }}">
    {{ __('nav.customers') }}
</a>
```

**Redirects inside actions/pages — use Filament's redirect, not JS:**
```php
// ✅ in action or page method
$this->redirect(CustomerResource::getUrl('index'));

// ❌ avoid
$this->js("window.location.href = '/admin/customers'");
```

**Exclude URLs from SPA if needed (e.g. OAuth callbacks):**
```php
->spaUrlExceptions(fn (): array => [
    url('/admin/oauth/callback'),
])
```

## Dark Mode

All UI must work in both dark and light mode without visual breakage.

**Rules:**
- Use Filament's semantic color aliases — they auto-adapt:
  `primary`, `success`, `danger`, `warning`, `info`, `gray`
- Never hardcode light-only Tailwind shades without a `dark:` counterpart:
  ```blade
  {{-- ❌ breaks dark mode --}}
  <span class="text-gray-900 bg-white">...</span>

  {{-- ✅ correct --}}
  <span class="text-gray-900 dark:text-gray-100 bg-white dark:bg-gray-800">...</span>

  {{-- ✅ even better — use semantic Filament component --}}
  <x-filament::badge color="primary">...</x-filament::badge>
  ```
- Never hardcode hex colors in PHP or Blade without a dark variant strategy.
- When using custom Blade, always check the `dark:` class exists for background, text, and border colors.

## Filament Blade Components

Prefer Filament's built-in Blade components over raw HTML to get automatic dark mode, consistent styling, and SPA compatibility.

| Use this | Instead of |
|---|---|
| `<x-filament::link>` | `<a>` |
| `<x-filament::button>` | `<button>` |
| `<x-filament::badge>` | `<span class="badge ...">` |
| `<x-filament::section>` | `<div class="p-4 ...">` |
| `<x-filament::icon>` | `<svg>` or raw icon embed |
| `<x-filament::loading-indicator>` | custom spinner |

**Example:**
```blade
{{-- ✅ filament link with SPA navigation --}}
<x-filament::link
    :href="route('filament.admin.resources.customers.edit', $record)"
    wire:navigate
    icon="fas-pencil"
>
    @lang('actions.edit')
</x-filament::link>

{{-- ✅ filament badge with semantic color --}}
<x-filament::badge color="success">
    @lang('statuses.active')
</x-filament::badge>
```
