# Public API v1 — Schema và fixture (GOV-04, hotel-system#126)

**Trạng thái** — Đề xuất, chờ Codex targeted re-review (2026-11-11, Round 4; đã tích hợp quyết định PO-Q1, Q3–Q10 và sửa F1–F4, M1–M4). Mô tả hành vi: `hotel-system/docs/specs/PMS-0034-public-api-contract.md`. Lý do, bằng chứng và kết quả kiểm tra Round 4: `hotel-system/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md`. Thư mục này là phần **máy đọc** của cùng hợp đồng; khi hai nơi khác nhau, sửa cả hai trong cùng lượt thay đổi.

- `schemas/*.schema.json` — JSON Schema 2020-12, `additionalProperties: false` (allowlist), dùng chung `common.schema.json#/$defs/*`. `$id` dùng miền `.invalid` (không phải URL thật, không tra cứu mạng).
- `fixtures/*.json` — ví dụ thành công và **một fixture cho mỗi mã lỗi** (24 mã, kèm biến thể `error.reservation-not-cancellable-pending-approval`). `fixtures/manifest.json` ánh xạ từng fixture → schema → HTTP status (và `responseHeaders` cho biến thể replay), để mock server của FE-02 và test hợp đồng của BE-21 dùng chung một nguồn.
- `negative/negative-controls.json` — 31 payload **phải bị từ chối** (mỗi mục: `id`, `schema`, `reason`, `payload`). Là dữ liệu cho test hợp đồng của FE-02/BE-21; không phải fixture thành công.
- `corpus/room-type-code.json` — bộ mã Room Type kiểm tương thích V1 (26 mã) với phán quyết của validator V1 thật, ràng buộc DB V1 và schema công khai (F4).
- Số tiền là **chuỗi** thập phân (`"3300000"`), không phải số. Ngày là calendar date `YYYY-MM-DD`. Không có trường giá trung bình ở đâu cả: giá luôn là danh sách `{date, rate}` theo đêm.

## Operation → schema → fixture

| Operation | Request schema | Response schema | HTTP | Fixture chính |
| --- | --- | --- | --- | --- |
| `GET /site-config` | — | `site-config` | 200 | `site-config.ok`, `site-config.booking-disabled` |
| `GET /room-types` | — | `room-type-list` | 200 | `room-types.ok` |
| `GET /room-types/{slug}` | — | `room-type-detail` | 200 | `room-type-detail.ok` |
| `GET /room-types/{slug}/images/{imageId}` | — | byte ảnh (không phải JSON; qua proxy Next.js, mục 3.2 của hợp đồng) | 200 | — |
| `POST /availability/search` | `availability-search-request` | `availability-search-response` | 200 | `availability-search.request`, `availability-search.ok` |
| `POST /quotes` | `quote-request` | `quote-response` | 201 | `quote.request`, `quote.request.v1-code-compat`, `quote.request.code-boundaries`, `quote.ok` |
| `POST /bookings` | `booking-submit-request` | `booking-submit-response` | **201 cho cả `CONFIRMED` và `PENDING_APPROVAL`** (M3) | `booking-submit.request`, `booking-submit.request-mode-request`, `booking-submit.confirmed`, `booking-submit.pending-approval` |
| `GET /bookings/reconcile` | — | `reconcile-response` (phân biệt theo `state`, M1) | 200 | `reconcile.succeeded`, `reconcile.succeeded.pending-approval`, `reconcile.rejected`, `reconcile.rejected.price-changed`, `reconcile.processing`, `reconcile.failed-retryable`, `reconcile.not-found` |
| `POST /manage-booking/lookup` | `manage-lookup-request` | `manage-lookup-response` | 202 | `manage-lookup.request`, `manage-lookup.accepted` |
| `POST /manage-booking/exchange` | `manage-exchange-request` | `manage-exchange-response` | 200 | `manage-exchange.request`, `manage-exchange.ok` |
| `GET /manage-booking/detail` | — | `manage-detail-response` | 200 | `manage-detail.cancellable`, `manage-detail.legacy-no-snapshot`, `manage-detail.pending-approval` (không tự hủy, PO-Q8), `manage-detail.expired` (PO-Q9) |
| `POST /manage-booking/cancel` | — (header `Idempotency-Key` + `SESSION`) | `manage-cancel-response` | 200 | `manage-cancel.ok` |
| Mọi phản hồi non-2xx | — | `error-envelope` | theo mã | `error.<code>` cho từng mã |

`error-envelope` ràng buộc theo mã: `details` bắt buộc đúng hình dạng cho `SOLD_OUT`, `PRICE_CHANGED`, `MODE_CHANGED`, `RESERVATION_NOT_CANCELLABLE`, `STAY_RULE_VIOLATION`, `BOOKING_UNAVAILABLE` và **phải rỗng** cho các mã còn lại; `retryable` bị ép theo bảng mã (`REQUEST_IN_PROGRESS`, `RATE_LIMITED`, `INTERNAL_ERROR`, `BOOKING_UNAVAILABLE` = `true`).

## Replay (M3)

Replay một thao tác idempotent trả **cùng HTTP status và cùng body JSON (tương đương ngữ nghĩa)** với lần đầu, kèm header `Idempotency-Replayed: true`. Metadata replay **chỉ nằm ở header**, không có trường `replayed` trong body (schema từ chối: negative `NC-29`, `NC-30`). **Không** khẳng định byte-for-byte: `response_body` lưu `JSONB` (không giữ thứ tự khóa/khoảng trắng) và `X-Request-Id`, `Date`, `Idempotency-Replayed` thay đổi theo lần gọi. Manifest ghi các biến thể này bằng `scenario: "REPLAY"` + `responseHeaders`, dùng lại cùng tệp fixture.

## Reconcile là union phân biệt theo `state` (M1)

| `state` | `response` | Ràng buộc |
| --- | --- | --- |
| `SUCCEEDED` | bắt buộc | `httpStatus = 201`; `body` là `booking-submit-response` (`CONFIRMED` hoặc `PENDING_APPROVAL`); không có UUID nội bộ, `reservation_number`, email khách (allowlist, negative `NC-21…NC-23`) |
| `REJECTED` | bắt buộc | cặp `(httpStatus, error.code)` thuộc đúng 9 lỗi nghiệp vụ xác định: `QUOTE_INVALID` 422, `QUOTE_EXPIRED` 409, `MODE_CHANGED` 409, `POLICY_CHANGED` 409, `PRICE_CHANGED` 409, `SOLD_OUT` 409, `CAPACITY_EXCEEDED` 422, `ROOM_TYPE_NOT_PUBLISHED` 422, `STAY_RULE_VIOLATION` 422; `body` là `error-envelope` hợp lệ, `retryable = false` |
| `PROCESSING`, `FAILED_RETRYABLE`, `NOT_FOUND` | **cấm** | chỉ có `state` (lỗi hạ tầng/cấu hình như `INTERNAL_ERROR`, `BOOKING_UNAVAILABLE` là `FAILED_RETRYABLE`, không bao giờ `REJECTED`) |

## `roomTypeCode` (F4)

`roomTypeCode` là `room_type.code` của PMS V1, nguyên văn và bất biến. Schema `common.schema.json#/$defs/roomTypeCode`: chuỗi 1–64 ký tự, không chứa khoảng trắng ASCII. Đây là **tập cha có chủ đích** của quy tắc V1 (không trống, tối đa 64, không khoảng trắng, không chữ thường): PMS vẫn áp dụng đầy đủ quy tắc V1 (kể cả "không chữ thường") và tra cứu khớp tuyệt đối. Không bao giờ đặt mã vào path/query của URL (URL dùng `slug`). `corpus/room-type-code.json` ghi phán quyết của validator V1 thật cho 26 mã (gồm `DBL-SEA`, mã 40 ký tự, ranh giới 1/64/65).

## Kiểm hợp lệ đã chạy (Round 4, 2026-11-11)

Chạy bằng script ngoài repo (thư mục tạm của phiên; `jsonschema` 4.25.1 trong virtualenv tạm, không thêm phụ thuộc vào dự án): generator sinh schema + fixture + negative controls rồi kiểm tất cả; script corpus chạy validator V1 thật (`RoomTypeCreateRequest`, Hibernate Validator 8.0.2.Final, các lớp đã biên dịch của dự án, chỉ đọc) và ràng buộc DB V57 trên PostgreSQL 17 dùng riêng. Kết quả thực: **18 schema hợp lệ về mặt meta-schema, 58/58 mục manifest (55 tệp fixture) hợp lệ, 31/31 negative control bị từ chối, 9/9 kiểm tra tương thích `roomTypeCode`** (30.026 mã gồm 30.000 chuỗi ngẫu nhiên: mọi mã V1 chấp nhận, 7.708 mã, đều được schema chấp nhận). Chi tiết ở `hotel-system/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md` §9. Script không nằm trong repo; khi FE-02 dựng test hợp đồng thật, dùng `fixtures/manifest.json`, `negative/negative-controls.json` và `corpus/room-type-code.json` làm đầu vào. **Chưa** kiểm hợp lệ với một PMS thật (chưa có endpoint).

## Giới hạn

- Chưa phải OpenAPI hoàn chỉnh (không có `paths`, header, mã trạng thái theo operation ngoài `manifest.json`). Cần OpenAPI YAML hay không là quyết định mở của GOV-04; công cụ hợp lệ hóa YAML không có sẵn trong môi trường đã dùng.
- Schema `roomTypeCode` không thể diễn đạt "không chữ thường Unicode" một cách di động (`\p{Ll}` không có trong mọi engine JSON Schema); phần đó do PMS thực thi và test BE-13 kiểm chứng bằng corpus.
- Giá trị trong fixture (tên khách sạn, mã tham chiếu `7K3M9Q2XDA`, UUID) là ví dụ, không phải dữ liệu thật.
- `reservationReference` là mã công khai riêng (PO-Q6 đã duyệt): Crockford base32 10 ký tự, không tuần tự, không phải UUID nội bộ hay `reservation_number`, và một mình không cấp quyền quản lý. Không fixture nào chứa UUID/số của reservation.
