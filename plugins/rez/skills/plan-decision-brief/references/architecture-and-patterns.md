# Architecture & Design Patterns

This item is **always present**, for every task type and both modes. It answers
two questions briefly: will this design hold as things grow, and which patterns
shape the code — where and why.

## Scalability (Khả năng mở rộng)

1–3 lines. Cover what applies:

- **Growth axis:** what grows — data volume, request rate, users, tenants,
  number of modules/screens/integrations, team size.
- **Bottleneck:** the first thing that breaks as it grows (N+1 query, single
  queue worker, global state, synchronous external call, large bundle).
- **Accepted limit:** the limit this design accepts on purpose, and the trigger
  for revisiting it ("đủ cho < 1M rows; quá ngưỡng thì partition theo tháng").

If growth is irrelevant (one-off script, copy change), write one line saying so
and why.

## Design patterns (Pattern áp dụng)

Table, 1–4 rows:

| Pattern | Ở đâu | Tại sao |
|---|---|---|
| Strategy | `PaymentGatewayResolver` + `gateways/*` | Thêm cổng thanh toán mới không sửa code cũ |
| ~~Repository~~ (không dùng) | `OrderService` | Eloquent đã đủ; thêm lớp chỉ tăng boilerplate (KISS) |

Rules:

- **Ở đâu** names a concrete file, class, module, or component — not "the backend".
- **Tại sao** names the problem the pattern solves here (duplication, change
  isolation, testability, extension point), in one clause.
- Include an existing project pattern that the change **follows** when it is a
  meaningful constraint ("theo Action pattern hiện có ở `app/Actions`").
- Add a "không dùng" row when an obvious pattern was **deliberately rejected**,
  with the reason. This shows the choice was evaluated.
- When no pattern is introduced or followed meaningfully, write:
  `Không áp dụng pattern mới — <lý do, vd: thay đổi cục bộ 1 hàm, KISS>`.
  Never leave this item empty.
- Do not list language features or framework defaults as patterns (a React hook
  or a Laravel controller is not a pattern decision by itself). Rendering-mode
  choices (RSC/SSR/CSR) belong in the frontend "SEO/SSR" or "State & dữ liệu" line.

## Fix mode (compact)

Two lines max:

```
- **Mở rộng:** <one line, or "Không ảnh hưởng — <lý do>">
- **Pattern:** <Pattern @ location — why>, or "Không áp dụng pattern mới — <lý do>"
```
