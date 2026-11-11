# ADR 018 — Quyết định Product Owner Round 3 (PO-Q3 đến PO-Q10)

**Trạng thái** — **Đã chốt** theo quyết định Product Owner (2026-11-11), sau Codex re-review #2 (REQUEST CHANGES). PO-Q1 (chế độ giá `FLAT`/`DAILY`) được ghi ở [ADR 017](017-nightly-pricing-flat-and-daily-modes.md). Thiết kế kỹ thuật thực hiện các quyết định này vẫn chờ Codex re-review #3; ADR này không tuyên bố thiết kế đã được Codex xác nhận.

## Quyết định

| # | Quyết định | Thiết kế thực hiện |
| --- | --- | --- |
| PO-Q3 | ADMIN và MANAGER duyệt/từ chối Booking Request khi có permission; ADMIN thu hồi được khỏi MANAGER; STAFF không nhận mặc định; backend kiểm permission hiệu lực, không theo tên vai trò; không thiết kế lại RBAC | `MANAGE_BOOKING_REQUEST` (không thuộc `MANAGER_ALWAYS_PERMISSIONS`); xem [ADR 013](013-po01-booking-request-approval-permission.md) |
| PO-Q4 | Một permission riêng quản lý cấu hình Hotel Website; ADMIN mặc định; vai trò khác cần cấp tường minh; tách sửa cấu hình khỏi truy cập website công khai | `MANAGE_WEBSITE_CONFIG`, không `ADMIN_ONLY`, không seed cho vai trò khác; khóa service công khai không có permission nào |
| PO-Q5 | ADMIN cấu hình và kích hoạt `HOTEL_WEBSITE` trước go-live; không tự kích hoạt nguồn inactive; không ghi đè cấu hình; không fallback `DIRECT`; thiếu/inactive/không hợp lệ ⇒ fail-closed với lỗi cấu hình có thể hành động; không tạo/kích hoạt như tác dụng phụ | `PublicBookingReadiness` (`SOURCE_MISSING`/`SOURCE_INACTIVE`/`SOURCE_INVALID`), `503 BOOKING_UNAVAILABLE`, cảnh báo ở Website Settings; không migration seed |
| PO-Q6 | Mã tham chiếu công khai riêng, không tuần tự, không đoán được; không lộ UUID nội bộ; không dùng `reservation_number` làm credential; mã một mình không cấp quyền; giữ xác minh email và phiên ngắn hạn | `reservation.public_reference` (Crockford base32 10 ký tự), vòng đời và va chạm theo round 3 §4.5 |
| PO-Q7 | Giữ chính sách đã chấp nhận lúc submit; nhân viên đổi ngày không âm thầm thay; nếu cần đổi thì phải có quy trình chấp nhận tường minh; giữ PMS Admin V1 | Snapshot bất biến ([ADR 016](016-po04-legacy-booking-cancellation.md)); quy trình chấp nhận được định nghĩa nhưng ngoài phạm vi V2A |
| PO-Q8 | Website V2A không hỗ trợ khách tự rút/tự hủy Request chờ duyệt; khách liên hệ khách sạn; quy trình nhân viên theo lifecycle đã duyệt | `cancel` ⇒ `RESERVATION_NOT_CANCELLABLE` cho `PENDING_APPROVAL`; `detail` hiển thị hướng dẫn liên hệ |
| PO-Q9 | `EXPIRED` là trạng thái cuối riêng, không gộp `REJECTED`; duyệt kiểm hạn dưới khóa; Request đã hết hạn không được duyệt lại | enum `EXPIRED`; "hạn thắng" dưới khóa ([ADR 004](004-booking-request-lifecycle.md)) |
| PO-Q10 | Room Type phải còn active và đủ điều kiện lúc duyệt; sức chứa/occupancy/tồn kho đạt; giữ giá đã đảm bảo lúc submit khi duyệt hợp lệ; không thay Room Type khác; thất bại ⇒ lifecycle từ chối đã duyệt | Vị từ công bố của catalog (diễn giải O3), giá từ snapshot, thất bại ⇒ `REJECTED` (`ROOM_TYPE_UNAVAILABLE`) |

## Hệ quả

Chi tiết thiết kế, test, Issue chủ sở hữu và các diễn giải còn cần xác nhận (O1…O7, không chặn review): `hotel-system/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md` mục 13 và `PMS-0034-codex-review-traceability-2026-11-11.md`. Các diễn giải đã ghi nhận để PO có thể điều chỉnh: STAFF bị chặn cứng (`STAFF_NEVER`) thay vì chỉ "không mặc định" (O1); Rate Calendar/Gallery giữ `MANAGE_ROOM_TYPE` (O2); "đủ điều kiện" (O3); đổi ngày sớm hơn cutoff đã chấp nhận (O4).

Mở ADR mới nếu PO muốn đổi bất kỳ diễn giải nào ở trên hoặc duyệt quy trình đổi chính sách hủy như một feature.

## Cập nhật Round 4 (2026-11-11)

Chi tiết tích hợp PO-Q3/PO-Q4 vào catalog quyền (tám điểm đồng bộ, test chấp nhận P01–P14, làm mới authority đã xác minh) và PO-Q5 (fail-closed S2S: ứng dụng không khởi động; nguồn `HOTEL_WEBSITE` vẫn lỗi lúc chạy `503`): [../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md) §2 và §6.2. Các quyết định PO không đổi.
