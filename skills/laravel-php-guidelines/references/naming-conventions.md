# Naming Conventions (Rezonia)

Single source of truth for every Laravel artifact. When in doubt, match this table.

## Files & classes

| Artifact | Filename / class | Example | Notes |
|---|---|---|---|
| Controller | `SingularController` | `UserController`, `InvoiceController` | Singular noun. Resource controllers still singular. |
| Single-action controller | `VerbNounController` | `ProcessRefundController`, `ExportPayslipController` | Has only `__invoke()`. |
| Model | Singular `PascalCase` | `User`, `PayrollRun` | Eloquent. In `app/Models/`. |
| Migration | `YYYY_MM_DD_HHMMSS_verb_phrase.php` | `2026_04_23_120000_add_status_to_users_table.php` | Snake_case after timestamp. |
| Seeder | `ModelSeeder` | `UserSeeder`, `RoleSeeder` | |
| Factory | `ModelFactory` | `UserFactory` | Auto-mapped by Eloquent. |
| Form Request | `Action + Model + Request` | `StoreUserRequest`, `UpdateInvoiceRequest` | Per action. |
| Resource | `Model + Resource` | `UserResource` | API transformer. |
| Resource Collection | `Model + Collection` | `UserCollection` | Only when needed beyond default. |
| Policy | `Model + Policy` | `UserPolicy`, `InvoicePolicy` | One per model that has authorization. |
| Observer | `Model + Observer` | `UserObserver` | |
| Middleware | Verb-phrase, no `Middleware` suffix | `EnsureUserHasVerifiedEmail`, `LogRequestPayload` | Reads as a sentence. Alias is kebab-case. |
| Job | `Action + Job` | `SendWelcomeEmailJob`, `GenerateMonthlyReportJob` | Imperative verb. |
| Notification | `Purpose + Notification` | `WelcomeEmailNotification`, `InvoicePaidNotification` | |
| Event | Past-tense or progressive | `UserRegistered`, `UserLoggedIn`, `CommentCreated`, `MediaProcessing` | Describes a fact that happened or is happening. |
| Listener | `Action + Listener` | `SendWelcomeEmailListener`, `RecordLoginAuditListener` | Reads as the action it performs in response. |
| Console command class | `Action + Command` | `CalculateDailyPayslipsCommand`, `PurgeSoftDeletedUsersCommand` | Class name. |
| Console command name | `kebab-case` with `:` namespace | `payslip:calculate-daily`, `users:purge-soft-deleted` | The string the user types. |
| Mailable | `Purpose + Mail` (or just purpose) | `WelcomeEmail`, `PaymentCompleteEmail`, `InvoicePaidEmail` | |
| Enum | Self-explanatory, no suffix | `UserType`, `UserStatus`, `PayslipState` | Cases & values both `UPPER_SNAKE_CASE`. |
| Service | Domain noun + `Service` | `BillingService`, `PayslipCalculator` | When `Calculator`/`Manager`/`Resolver` is more accurate, use that. |
| DTO / Value Object | Domain noun, no suffix | `Money`, `DateRange`, `Address` | `final readonly`. |
| Action class | Verb + Noun | `RegisterUser`, `RefundInvoice` | If the project uses single-action classes. Has `__invoke()` or `handle()`. |
| Trait | Adjective or HasX/CanX | `HasUuid`, `Sluggable`, `CanGenerateInvoice` | |
| Interface / Contract | Domain noun, no `I` prefix | `PaymentGateway`, `InvoiceRepository` | |
| Exception | Specific cause + `Exception` | `InvoiceAlreadyRefundedException` | |

## Views

- Directory + filename: `kebab-case`.
- Path mirrors the route or controller: `resources/views/users/edit-profile.blade.php`.
- Partials prefixed `_`.
- Layouts in `resources/views/layouts/`.
- Components: kebab-case in markup, PascalCase in `app/View/Components/`.

## Routes

- URL path segment: `kebab-case` plural (`/invoices/{invoiceId}`).
- Route parameter: `camelCase` (`{invoiceId}`).
- Route name: `camelCase`, dot-grouped by feature: `billing.invoices.show`.
- Action: tuple `[InvoiceController::class, 'show']`.

## Config

- Filename: `kebab-case.php` — `external-services.php`, `billing.php`.
- Top-level keys: `snake_case` — `'webhook_secret' => ...`.
- Access: `config('billing.webhook_secret')`.
- `env()` only inside `config/*.php`.

## Database

- Tables: `snake_case`, plural — `users`, `payroll_runs`.
- Columns: `snake_case` — `created_at`, `email_verified_at`.
- Foreign keys: `singular_table_id` — `user_id`, `invoice_id`.
- Pivot tables: alphabetical singular pair — `role_user` (not `user_role`).
- Indexes: `<table>_<columns>_index` (Laravel default), explicit name when ambiguous.
- Enums in DB: store as string (`UPPER_SNAKE_CASE`), not int — matches the PHP enum value.

## Translations

- Lang file per feature: `lang/en/users.php`, `lang/en/billing.php`.
- Keys: `snake_case`, dot-grouped within file: `users.profile.updated`.
- Placeholders: `:name`, `:count`.

## Tests

- Feature tests: `tests/Feature/<Feature>/<Action>Test.php` — `tests/Feature/Billing/RefundInvoiceTest.php`.
- Unit tests: `tests/Unit/<Class>Test.php`.
- Pest test names describe behavior: `it('refunds an invoice when authorized', ...)`.

## Variables & methods

- Variables / properties / method names: `camelCase`.
- Constants: `UPPER_SNAKE_CASE`.
- Booleans read as predicates: `$isActive`, `$hasAccess`, `canEdit()`.
- Collections plural: `$users`, `$invoices`. Single items singular.
