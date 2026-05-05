# Filament Actions & Notifications

## Notification Rule (Non-Negotiable)

Inside `->action()` callbacks, **always use the injected `$action` object** — never `Filament\Notifications\Notification` facade directly.

```php
// ✅ CORRECT — use injected $action
->action(function (Model $record, Action $action, array $data): void {
    try {
        $record->update($data);
        $action->successNotificationTitle(__('messages.updated'))->success();
    } catch (\Throwable $e) {
        $action->errorNotificationTitle(__('messages.update_failed'))->failure();
    }
})

// ❌ WRONG — do not use Notification facade inside action()
->action(function (Model $record, array $data): void {
    $record->update($data);
    Notification::make()->title('Updated')->success()->send(); // forbidden here
})
```

**Why:** Using `$action->success()` / `->failure()` integrates properly with Filament's action lifecycle (disables the button during execution, closes modals on success, etc.). The facade bypasses this.

## Action Notification API

```php
// Success
$action->successNotificationTitle('Record saved')->success();

// Failure / error
$action->errorNotificationTitle('Save failed')->failure();
```

Both methods are chainable. Call them as the last statement in the `action()` closure.

## Action Injection

Filament resolves action closures from the service container. Inject what you need:

```php
->action(function (
    Model $record,           // the record the action is run against
    Action $action,          // the action instance (for notifications)
    array $data,             // form data from the action's schema()
    Component $livewire,     // the parent Livewire component
): void {
    // ...
})
```

## Standard Action Patterns

**Header action with form:**
```php
Action::make('export')
    ->label(__('actions.export'))
    ->icon('fas-file-export')
    ->schema([
        Select::make('format')
            ->options(['csv' => 'CSV', 'xlsx' => 'Excel'])
            ->required(),
    ])
    ->action(function (Action $action, array $data): void {
        ExportJob::dispatch($data['format']);
        $action->successNotificationTitle(__('actions.export_queued'))->success();
    })
```

**Table row action:**
```php
Tables\Actions\Action::make('approve')
    ->label(__('actions.approve'))
    ->icon('fas-check')
    ->color('success')
    ->requiresConfirmation()
    ->action(function (Model $record, Tables\Actions\Action $action): void {
        $record->approve();
        $action->successNotificationTitle(__('actions.approved'))->success();
    })
```

**Bulk action:**
```php
Tables\Actions\BulkAction::make('archive')
    ->label(__('actions.archive'))
    ->icon('fas-archive')
    ->requiresConfirmation()
    ->action(function (Collection $records, Tables\Actions\BulkAction $action): void {
        $records->each->archive();
        $action->successNotificationTitle(__('actions.archived'))->success();
    })
    ->deselectRecordsAfterCompletion()
```

## Action Class Extraction

For reusable actions, extract to a class (see `code-quality.md`). The `$action` pattern works identically inside extracted action classes.

## When Notification Facade IS Allowed

Outside `->action()` closures — e.g. in Livewire lifecycle hooks, `mount()`, page actions called from page methods — the `Notification` facade is acceptable:

```php
// In a page method (not inside ->action())
public function doSomething(): void
{
    Notification::make()
        ->title(__('messages.done'))
        ->success()
        ->send();
}
```
