# ADR 010 — Issue #30 là Epic; đặc tả PMS-0034 đặt tại `hotel-system/docs/specs/`

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-10, M0), giải quyết [D-07](../audits/issue-30-gap-analysis.md#71-chặn-bắt-đầu). [Phân chia trách nhiệm](../specs/client-specs.md#13-phân-chia-trách-nhiệm)

## Bối cảnh

Issue #30 (PMS-0034) đang ở Backlog, chưa có tiêu chí nghiệm thu gắn vào Issue, và PMS chưa có tài liệu đặc tả riêng cho PMS-0034. `architecture.md` mục 14–15 của PMS cấm tạo domain/package mới mà chưa được duyệt; các domain hiện có chỉ gồm `customer`, `booking`, `room` (gap analysis G-28, finding F-04, BLOCKER).

## Quyết định

Issue #30 là một Epic, chia task theo mốc M0–M5. Đặc tả PMS-0034 đặt tại `hotel-system/docs/specs/`. Mọi thiết kế kỹ thuật tuân thủ `docs/coding-rules/architecture.md`. Mọi domain hoặc package mới phải được phê duyệt trong thiết kế trước khi implement.

## Hệ quả

Quyết định này đặt đặc tả PMS-0034 ở `docs/specs/PMS-0034-hotel-website-direct-booking-v2a.md` thay vì vị trí `technical-spec-v2.md` mà gap analysis D-07 từng đề xuất làm mặc định — [implementation plan GOV-02](../development/issue-30-implementation-plan.md#gov-02-đặc-tả-pms-0034-trong-pms) cần cập nhật theo vị trí này (xem cập nhật kèm theo trong cùng đợt soạn tài liệu này). `docs/specs/PMS-0034-hotel-website-direct-booking-v2a.md` xác nhận không cần domain mới: mọi entity/class mới (trạng thái Booking Request, snapshot giá theo đêm, Room Type Gallery, outbox email) đặt trong các domain `booking`, `room`, `common` hiện có theo quy tắc `architecture.md` §5.1–5.2, không có package `pricing`/`notification`/`reservation` riêng. Issue #30 vẫn giữ nguyên status (APPROVED), priority (P1) và release (V2) đã ghi ở `master-backlog.md` dòng PMS-0034 và mục 6F; GOV-01 (tách Issue con, chuyển Backlog → Ready) vẫn là quyết định riêng của Product Owner, không tự động hoàn tất bởi ADR này.

Mở ADR mới nếu một task V2A sau này thực sự cần một domain/package mới ngoài `customer`/`booking`/`room`/`common`.
