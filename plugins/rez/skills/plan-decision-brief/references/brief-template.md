# Brief Templates

Copy the variant that matches the delivery mode. Remove every item that does not
apply. Keep headings in Vietnamese exactly as written so users can scan them. Length
counts every rendered line, including blank lines, table rows, and `---`.

## Plan variant (~50 lines max)

Pinned right after the plan title/frontmatter. Same text is printed in chat.

```markdown
## Tóm tắt quyết định

> Loại: backend · Đọc trong ~1 phút · Chi tiết ở các phần tiếng Anh bên dưới

**Mục tiêu:** <1–2 câu về kết quả cuối>

**Quyết định chính**

| Quyết định | Chọn | Lý do | Phương án đã loại |
|---|---|---|---|
| <...> | <...> | <...> | <...> |

**Phạm vi:** <...> · **Ngoài phạm vi:** <...>

**Thay đổi phá vỡ tương thích:** Không có | <cái gì vỡ, ai bị ảnh hưởng, cách migrate>

**Rủi ro & rollback**
- <rủi ro> → <giảm thiểu / cách rollback>

**Kiến trúc & Design pattern**
- **Mở rộng:** <trục tăng trưởng, điểm nghẽn, giới hạn chấp nhận>

| Pattern | Ở đâu | Tại sao |
|---|---|---|
| <...> | <...> | <...> |

- **<Mục riêng theo loại>:** <một dòng — mỗi mục một list item, vd `- **API contract:** …`>

**Cần user xác nhận**
1. <câu hỏi> — Đề xuất: <câu trả lời khuyến nghị>

---
```

The `---` separates the brief from the English plan body.

## Fix variant (~20 lines max, chat only)

Printed after the root cause is proven and the fix is chosen, before the first
code edit.

```markdown
## Tóm tắt quyết định (fix)

> Loại: frontend

- **Mục tiêu:** <hành vi đúng sau khi sửa>
- **Nguyên nhân gốc:** <một dòng, đã chứng minh bằng …>

| Quyết định | Chọn | Lý do | Phương án đã loại |
|---|---|---|---|
| Cách sửa | <...> | <...> | <...> |

- **Thay đổi phá vỡ tương thích:** Không có
- **Rủi ro & rollback:** <một dòng>
- **Mở rộng:** <một dòng hoặc "Không ảnh hưởng — <lý do>">
- **Pattern:** <Pattern @ vị trí — lý do> | Không áp dụng pattern mới — <lý do>
- **<Mục riêng theo loại, chỉ khi bị đụng tới>:** <một dòng>
- **Cần user xác nhận:** <chỉ khi có câu hỏi quan trọng — chặn việc sửa code tới khi user trả lời>
```

Final report of a fix adds only: `Khác với brief: <...>` or
`Không có thay đổi so với brief`.

## Example — backend plan (excerpt)

```markdown
## Tóm tắt quyết định

> Loại: backend · Đọc trong ~1 phút · Chi tiết ở các phần tiếng Anh bên dưới

**Mục tiêu:** Webhook thanh toán được xử lý đúng một lần, kể cả khi cổng gửi lại.

**Quyết định chính**

| Quyết định | Chọn | Lý do | Phương án đã loại |
|---|---|---|---|
| Chống xử lý trùng | Unique key `provider_event_id` | Đơn giản, DB đảm bảo | Redis lock (thêm hạ tầng) |
| Xử lý | Queue job | Trả 200 nhanh, retry được | Xử lý đồng bộ (timeout) |

**Thay đổi phá vỡ tương thích:** Không có

**Rủi ro & rollback**
- Migration thêm unique index trên bảng lớn → tạo index concurrently; rollback = drop index

**Kiến trúc & Design pattern**
- **Mở rộng:** tăng theo số event/giây; nghẽn ở 1 worker → scale số worker, không đổi thiết kế

| Pattern | Ở đâu | Tại sao |
|---|---|---|
| Strategy | `WebhookHandlerResolver` + `handlers/*` | Thêm cổng mới không sửa handler cũ |
| ~~Event sourcing~~ (không dùng) | — | Quá nặng cho nhu cầu hiện tại (KISS) |

- **DB & migration:** thêm unique index, tương thích ngược, không backfill
- **Bất đồng bộ:** job retry 5 lần, backoff mũ; lỗi cuối vào failed_jobs + alert

**Cần user xác nhận**
1. Giữ log webhook bao lâu? — Đề xuất: 90 ngày
```
