# Core Sections

Every brief has these items, in this order. Headings and content are in
Vietnamese; code identifiers, file paths, and technical terms stay as-is.

| # | Heading | Content | Rule |
|---|---|---|---|
| 1 | **Mục tiêu** | 1–2 sentences: the end state the user will see or operate | Outcome, not activity ("API trả về X", not "sửa service") |
| 2 | **Quyết định chính** | Table: Quyết định \| Chọn \| Lý do \| Phương án đã loại | 1–6 rows; only decisions that change behavior, contract, cost, or risk. Use `—` when no alternative was considered; small decisions without alternatives go in type-specific lines instead |
| 3 | **Phạm vi / Ngoài phạm vi** | Two short lists | Ngoài phạm vi = nearby work this delivery explicitly will not absorb |
| 4 | **Thay đổi phá vỡ tương thích** | What breaks, who is affected, migration path | Write "Không có" when none — users scan for this. A new error status, limit, or required field on an existing contract counts as breaking for clients |
| 5 | **Rủi ro & rollback** | Top risks and how to undo | Each risk paired with a mitigation or rollback step. A mitigation the plan did not decide is marked `(đề xuất)`; if it changes scope or a contract, move it to "Cần user xác nhận" |
| 6 | **Kiến trúc & Design pattern** | See `architecture-and-patterns.md` | Always present |
| 7 | Type-specific items | See `type-sections.md` | Only items that apply |
| 8 | **Cần user xác nhận** | Open questions with a recommended answer | Always last; omit when nothing is open |

## Writing rules

- Lead with the decision, then the reason. One line per item where possible.
- Use the plan's own decisions. If the plan does not decide something material,
  list it under "Cần user xác nhận" with a recommendation.
- Rejected options in "Quyết định chính" must be real alternatives that were
  considered — never filler.
- Link to the English section of the plan for detail (`xem Phase 2`) instead of
  copying it.
- Say each fact once. A rejected option that is also out of scope stays in the
  decision table only; do not repeat it under "Ngoài phạm vi".
- Use names that exist in the plan or code. When a class or file name is not
  known yet, name the event, route, or module instead of inventing one.
- Type-specific items are list items with a bold label (`- **API contract:** …`)
  in the order of `type-sections.md`, with no shared heading — in both variants.
  Never put bold lines on consecutive lines without `- `; Markdown merges them
  into one paragraph.

## Fix-mode reduction

In fix mode (~20 lines), keep: Mục tiêu (expected repaired behavior), Nguyên nhân
gốc (one line, proven root cause), Quyết định chính (fix chosen vs alternatives),
Thay đổi phá vỡ tương thích, Rủi ro & rollback, Kiến trúc & Design pattern
(compact form), and only the type-specific items the fix actually touches.
Drop "Phạm vi / Ngoài phạm vi" unless the fix deliberately leaves related bugs open.
Add "Cần user xác nhận" only for a material open question; it blocks the first
code edit until answered.
