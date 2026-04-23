# Blade Standards (Rezonia)

## File location & naming

- All Blade views live under `resources/views/`.
- Directory + file names: `kebab-case`.
- Mirror controller/feature structure: `resources/views/users/edit-profile.blade.php` for `UserController@editProfile`.
- Partials prefixed with `_`: `_form.blade.php`, `_flash-message.blade.php`.
- Layouts in `resources/views/layouts/`: `app.blade.php`, `admin.blade.php`.
- Components (Blade components) in `resources/views/components/`, class-based in `app/View/Components/`.

## Translation

Inside Blade:

```blade
<h1>@lang('users.edit_profile')</h1>
<p>@lang('users.greeting', ['name' => $user->name])</p>
```

Use `@lang()` **inside Blade**. Reserve `{{ __('...') }}` for cases where you need escaping control or string concatenation in an expression. Never hard-code user-facing strings.

Translation key convention:
- Dot-grouped: `users.edit_profile`, `billing.refund_success`.
- Snake_case keys.
- File per feature: `lang/en/users.php`, `lang/en/billing.php`.

Placeholders use `:name` syntax:

```php
// lang/en/users.php
return [
    'greeting' => 'Hello, :name!',
];
```

## Echo & escaping

- Default: `{{ $var }}` — auto-escapes. Use this 99% of the time.
- Raw HTML: `{!! $var !!}` — only when `$var` is trusted (policy-rendered Markdown, sanitized via an allowlist). Comment why it's safe.
- Never emit user input through `{!! !!}`.

## Directives

- Prefer `@if`, `@unless`, `@foreach`, `@forelse`, `@switch`/`@case`, `@isset`, `@empty`.
- `@forelse` over `@foreach` + manual empty check.
- Always a matching `@endif`/`@endforeach`/etc. — no loose tags.
- Indent directives like PHP control flow. One level per nesting.

```blade
@forelse ($users as $user)
    <li>{{ $user->name }}</li>
@empty
    <li>@lang('users.none_found')</li>
@endforelse
```

## Components & slots

Prefer Blade components over `@include` for anything with logic:

```blade
<x-button.primary :href="route('users.create')">
    @lang('users.create')
</x-button.primary>
```

- Component names: `kebab-case` in markup, `PascalCase` class in `app/View/Components/`.
- Namespaced: `<x-forms.input>` → `resources/views/components/forms/input.blade.php`.
- Props declared in the class constructor with types. No untyped props.

## Layouts

```blade
{{-- resources/views/users/edit-profile.blade.php --}}
<x-layouts.app :title="__('users.edit_profile')">
    <form method="POST" action="{{ route('users.update', $user) }}">
        @csrf
        @method('PUT')

        {{-- fields --}}
    </form>
</x-layouts.app>
```

- Forms: always `@csrf`. Non-POST verbs: `@method('PUT')` / `@method('DELETE')`.
- Never inline `_token` hidden fields manually.

## Comments

- Blade-only comments: `{{-- ... --}}` (not rendered in HTML).
- HTML comments `<!-- ... -->` only when the comment should reach the browser.

## Logic

- **No heavy logic in Blade.** No queries, no aggregations, no Carbon formatting chains. Move those to the controller, view composer, or a dedicated ViewModel / data object.
- Simple checks (`@if ($user->isAdmin())`) are fine. If it's more than 2 method calls on one line, extract.

## Assets

- Vite: `@vite(['resources/js/app.js', 'resources/css/app.css'])` in layout head.
- Never hard-code `/build/...` paths.

## Accessibility baseline

- Every form field has a `<label for="...">`.
- Buttons say what they do: `@lang('users.save')`, not `@lang('generic.submit')`.
- Alt text on images, via translation keys when dynamic.
