# Filament Code Quality — Structure & Organization

Source: https://filamentphp.com/docs/5.x/resources/code-quality-tips

## Schema & Table Classes

Extract `form()`, `table()`, and `infolist()` definitions into dedicated classes to prevent bloated resource files.

**Directory layout:**
```
Resources/Customers/
├── CustomerResource.php
├── Pages/
├── Schemas/
│   ├── CustomerForm.php
│   └── Components/
│       └── CustomerNameInput.php
└── Tables/
    ├── CustomersTable.php
    ├── Columns/
    │   └── CustomerNameColumn.php
    └── Filters/
        └── CustomerCountryFilter.php
```

**Schema class:**
```php
namespace App\Filament\Resources\Customers\Schemas;

use Filament\Schemas\Schema;
use Filament\Forms\Components\TextInput;

class CustomerForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            CustomerNameInput::make(),
        ]);
    }
}
```

**Resource usage:**
```php
public static function form(Schema $schema): Schema
{
    return CustomerForm::configure($schema);
}
```

These classes intentionally have no enforced parent/interface — pass custom config variables freely and reuse with variations.

## Component Classes

Break large `configure()` methods by extracting individual components.

**Form input component:**
```php
class CustomerNameInput
{
    public static function make(): TextInput
    {
        return TextInput::make('name')
            ->label(__('customers.fields.name'))
            ->required()
            ->maxLength(255);
    }
}
```

**Table column component:**
```php
class CustomerNameColumn
{
    public static function make(): TextColumn
    {
        return TextColumn::make('name')
            ->label(__('customers.fields.name'))
            ->searchable()
            ->sortable();
    }
}
```

## Action Classes

Extract actions to an `Actions/` directory alongside the resource:

```php
// app/Filament/Resources/Customers/Actions/EmailCustomerAction.php
class EmailCustomerAction
{
    public static function make(): Action
    {
        return Action::make('email')
            ->label(__('customers.actions.email'))
            ->icon('fas-envelope')
            ->schema([
                TextInput::make('subject')->required(),
                Textarea::make('body')->autosize()->required(),
            ])
            ->action(function (Customer $record, Action $action, array $data): void {
                SendCustomerEmailJob::dispatch($record, $data);
                $action->successNotificationTitle(__('customers.actions.email_sent'))->success();
            });
    }
}
```

**Usage in header / table:**
```php
protected function getHeaderActions(): array
{
    return [EmailCustomerAction::make()];
}

// table row actions
->recordActions([EmailCustomerAction::make()])
```

## File Size Rule

Keep every Filament PHP file under 200 lines. If a resource, schema, or table class grows beyond that, split further into component classes.
