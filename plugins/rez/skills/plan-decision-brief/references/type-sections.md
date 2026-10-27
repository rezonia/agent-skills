# Type-Specific Sections

Add these after the core sections and the architecture item. Include only items
the change actually touches; drop the rest silently (no "N/A"). One line each
unless a table is clearer.

## Detecting the type

| Type | Signals |
|---|---|
| `coding` | Library, SDK, CLI, script, tooling, config, infra-as-code, agent skill — no server request path or UI |
| `backend` | Routes/controllers, services, jobs/queues, DB models/migrations, API schemas, server config |
| `frontend` | Components, pages/screens, styles, client state, routing, assets, mobile UI |
| `fullstack` | Both backend and frontend signals change in the same delivery |

A frontend server component or loader that only calls an existing API is still
`frontend`; it becomes `fullstack` only when server-side API, data, or job code
changes. Ask one short question only when signals conflict or are missing (e.g. a
monorepo package whose consumers are unclear).

An applicable item that the plan has not decided goes to "Cần user xác nhận"
with a recommendation, not into this list as a guess.

## coding

- **Public API / interface:** exported functions, CLI flags, config keys added or changed
- **Dependency mới:** name, version, why — or "Không thêm dependency"
- **Chiến lược test:** which tests are written first (TDD), unit vs integration

## backend

- **API contract:** endpoints added/changed, request/response shape, status codes, versioning
- **DB & migration:** schema change, backward compatible or not, backfill, lock/downtime risk on large tables
- **Toàn vẹn dữ liệu:** transactions, idempotency, concurrency/locking, retries
- **Auth & phân quyền:** who can call it, policy/role changes
- **Bất đồng bộ:** jobs, queues, events, schedules — ordering and failure handling
- **Tích hợp ngoài:** third-party calls, timeouts, behavior when they fail
- **Config/secret mới:** key names only, never values
- **Hiệu năng & observability:** expected load, indexes/caching, logs/metrics/alerts added
- **Deploy:** order of migration vs code, feature flag, rollout plan
- **Chiến lược test:** which tests are written first, integration vs unit

## frontend

- **Luồng UX:** screens/flows affected, entry points, navigation changes
- **Component:** new vs reused, design-system alignment
- **State & dữ liệu:** state ownership, fetching, caching, loading/error/empty states
- **Responsive & a11y:** breakpoints, keyboard, screen reader, contrast
- **i18n:** new strings, locales affected
- **SEO/SSR:** rendering mode, metadata (only when relevant)
- **Hiệu năng:** bundle impact, lazy loading, Core Web Vitals risk
- **Phụ thuộc API:** existing endpoint, new endpoint, or temporary mock
- **Chiến lược test:** component/unit, e2e, visual checks

## fullstack

All backend and frontend items that apply, plus:

- **Contract FE↔BE:** where shared types/schemas live and who owns them; how both sides stay in sync
- **Thứ tự deploy:** BE first or FE first; compatibility while only one side is live
- **Luồng end-to-end:** the main user flow across both sides, in one line
