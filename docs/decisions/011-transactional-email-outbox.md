# ADR 011 — Outbox email tối thiểu trong PMS, không phụ thuộc Issue #33

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-10, M0), giải quyết [D-08](../audits/issue-30-gap-analysis.md#71-chặn-bắt-đầu). [Email, Security & Operations](../specs/client-specs.md#10-email-security--operations)

## Bối cảnh

PMS chưa có `spring-boot-starter-mail`, `JavaMailSender`, scheduler hay bảng outbox nào. Issue #33 (PMS-0037B, Email Notifications) vẫn ở trạng thái PROPOSED, phạm vi có thể trùng (gap analysis G-21, finding F-10, MAJOR).

## Quyết định

Thiết kế transactional email và outbox tối thiểu ngay trong phạm vi PMS cho Issue #30, không phụ thuộc vào việc Issue #33 có được duyệt hay triển khai trước hay không. Email provider và địa chỉ gửi được cấu hình qua biến môi trường, không hardcode.

## Hệ quả

`PMS-0034-hotel-website-direct-booking-v2a.md` và `PMS-0034-data-design.md` (repo `hotel-system`) đề xuất một bảng outbox tối thiểu (sự kiện, payload, trạng thái gửi, số lần thử) và một tiến trình gửi định kỳ, chỉ phục vụ các sự kiện của Issue #30 (xác nhận Instant, Request đã nhận/được duyệt/bị từ chối, xác nhận hủy, liên kết Manage Booking). Gửi email chạy SAU khi transaction đặt phòng thành công; lỗi gửi email không được làm hoàn tác reservation đã tạo. Hạ tầng này không mở rộng để bao phủ toàn bộ phạm vi Issue #33; khi #33 được duyệt, Issue đó quyết định có tái dùng hay mở rộng outbox này hay không. Thiết kế bảng/cấu hình cụ thể còn chờ Codex và Product Owner xác nhận. Việc chốt D-08 mở khóa BE-15, BE-18, BE-19 và gap G-21.

Mở ADR mới nếu Issue #33 được duyệt và cần hợp nhất hai outbox thành một hạ tầng notification chung.

## Cập nhật Round 3 (2026-11-11) — làm rõ thiết kế

Dispatcher claim bằng một câu `UPDATE … WHERE id IN (SELECT … FOR UPDATE SKIP LOCKED) RETURNING` (owner + lease), ghi kết quả có hàng rào theo owner, retry có backoff và `FAILED` sau số lần thử tối đa, phục hồi bằng lease hết hạn. Ngữ nghĩa phát là **at-least-once** (không exactly-once). Quyết định của ADR này không đổi. Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §8.

## Cập nhật Round 4 (2026-11-11) — SQL, ràng buộc và giới hạn thử

SQL của Round 3 có ba lỗi (đã tái hiện bằng PostgreSQL 17.11): ràng buộc claim quá lỏng, câu `SENT`/retry vi phạm chính ràng buộc, phục hồi lease không giới hạn lượt thử. Thiết kế hiện hành: `SENDING` ⇔ có đủ chủ và hạn lease, mọi trạng thái khác ⇒ cả hai trường NULL; ghi kết quả có hàng rào `(id, claimed_by, attempts)`; giới hạn lượt thử áp dụng cho cả đường thường và đường phục hồi (event cạn lượt ⇒ `FAILED`); claim nguyên tử bằng `FOR UPDATE SKIP LOCKED`. Ngữ nghĩa vẫn là **at-least-once** (có thể gửi trùng khi crash giữa lúc SMTP chấp nhận và lúc ghi `SENT`); **không** có exactly-once. Quyết định của ADR không đổi. Chi tiết: [../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md) §3.
