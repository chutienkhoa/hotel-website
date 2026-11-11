# ADR 016 — Booking V1 (legacy) không tự hủy qua website

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-11, phản hồi Codex review REQUEST CHANGES — MAJOR-07), giải quyết PO-04. [Cancellation Policy](../../../hotel/docs/specs/PMS-0034-codex-review-resolution-2026-11-11.md#47-major-07--cancellation-policy--resolved-in-design)

## Bối cảnh

Manage Booking (Issue #30) cho khách tự hủy trong cửa sổ miễn phí hủy, dựa trên một snapshot chính sách lưu tại thời điểm đặt (AC-18, ADR 004/005). Booking tạo trước Issue #30 (V1) không có snapshot này — cột `cancellation_free_until` là MỚI, không backfill.

## Quyết định

Booking V1 không có cancellation policy snapshot: không cho khách tự hủy qua website; hiển thị hướng dẫn liên hệ khách sạn; không tự áp chính sách 24h hiện tại cho booking legacy. Không làm thay đổi chức năng hủy booking hiện có trong PMS Admin.

## Hệ quả

`hotel-system/docs/specs/PMS-0034-data-design.md` M07a/M07b: `reservation.cancellation_free_until IS NULL` cho mọi booking tạo trước Issue #30 (không backfill một giá trị suy diễn). Manage Booking (`PMS-0034-public-api-contract.md` mục 6) kiểm tra `cancellation_free_until IS NOT NULL` trước khi cho tự hủy; NULL → trả lỗi `CANCELLATION_WINDOW_PASSED` kèm message hướng dẫn liên hệ khách sạn, KHÔNG áp 24h mặc định. `Reservation.cancel()` dùng bởi nhân viên trong PMS Admin không đổi — nhân viên vẫn hủy booking legacy qua PMS Admin như hôm nay, không qua website.

Mở ADR mới nếu Product Owner sau này muốn cho phép backfill một chính sách mặc định cho booking legacy (ví dụ áp 24h hồi tố) — đây sẽ là một quyết định nghiệp vụ mới, không tự suy diễn.

## Cập nhật Round 3 (2026-11-11)

Snapshot gồm ba cột trên `reservation` — `cancellation_policy_revision_id`, `cancellation_free_until`, `cancellation_policy_accepted_at` — cùng bảng `cancellation_policy_revision` bất biến (giờ + **văn bản** chính sách theo phiên bản), ghi **lúc submit**; cả ba NULL cho booking V1 và booking nhân viên tạo (ràng buộc all-or-none). Không có job hay migration nào tự hủy hoặc gán chính sách mặc định cho booking thiếu snapshot. Manage Booking trả `CANCELLATION_NOT_AVAILABLE` + hướng dẫn liên hệ. Quyết định PO-04 **không đổi**. Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §7.

## Cập nhật theo PO-Q7 (2026-11-11)

Đã duyệt: chính sách khách chấp nhận lúc submit được giữ nguyên; nhân viên đổi ngày không âm thầm thay; nếu sau này cần áp chính sách khác thì phải có quy trình chấp nhận tường minh của khách (định nghĩa trong round 3 §7.2, ngoài phạm vi V2A). Hành vi hủy PMS Admin V1 không đổi. Xem [ADR 018](018-po-decisions-round3-q3-q10.md).
