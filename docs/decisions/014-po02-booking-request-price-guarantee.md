# ADR 014 — Bảo đảm giá Booking Request từ lúc gửi, không hồi tố theo Rate Calendar

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-11, phản hồi Codex review REQUEST CHANGES — BLOCKER-02), giải quyết PO-02. [Thiết kế Booking Request](../../../hotel/docs/specs/PMS-0034-codex-review-resolution-2026-11-11.md#3-blocker-02--unassigned-booking-request)

## Bối cảnh

Booking Request (ADR 004) không giữ tồn phòng và có thể chờ duyệt tới 24 giờ. Trong thời gian đó, Rate Calendar (Room Rate Calendar, BE-11) có thể bị nhân viên sửa. Nếu giá được tính lại tại thời điểm duyệt, khách có thể bị tính giá cao hơn giá đã thấy khi gửi Request — vi phạm kỳ vọng hợp lý của khách và có thể vi phạm chính sách nêu trên website.

## Quyết định

Giữ nguyên giá đã báo khi khách gửi Booking Request, kể cả khi Rate Calendar thay đổi trong thời gian chờ duyệt. Khi duyệt: kiểm tra lại availability và capacity; không tự động tăng giá đã báo; nếu không còn đủ phòng, không xác nhận Request; nếu không thể đáp ứng theo điều kiện đã cam kết, không tự động đổi giá hoặc chính sách hủy. Phải lưu snapshot giá đầy đủ từ lúc Request được tạo.

## Hệ quả

`hotel-system/docs/specs/PMS-0034-data-design.md` V71/V72 thiết kế bảng `booking_request_line`/`booking_request_line_nightly_rate`: giá breakdown theo đêm được chốt NGAY lúc khách gửi Request (không phải lúc duyệt), copy sang `reservation_room_nightly_rate` khi duyệt thành công, không đọc lại Rate Calendar. Nếu kiểm tra lại availability/capacity lúc duyệt thất bại, Reservation chuyển sang trạng thái từ chối (ADR 004) — không có đường nào tự động đổi giá cao hơn hoặc rút ngắn chính sách hủy để "cứu" một Request không còn đủ điều kiện. Làm rõ xử lý khi chính sách hủy thay đổi trong lúc Request chờ duyệt: chính sách áp dụng cũng được snapshot cùng lúc với giá (không đọc lại `cancellation_policy` khi duyệt) — không tự suy diễn rằng Product Owner đã phê duyệt việc đổi chính sách giữa chừng.

Mở ADR mới nếu Product Owner muốn cho phép một hình thức "báo giá lại, khách xác nhận lại" khi Request hết hạn do giá đã đổi, thay vì từ chối thẳng.

## Cập nhật Round 3 (2026-11-11)

Bảng nay là `booking_request_line` và `booking_request_line_nightly_rate` (migration **M12**, BE-18), và breakdown được sao chép sang `room_night_rate` (M06) khi duyệt — không còn `reservation_room_nightly_rate` (V73). Chính sách hủy được snapshot **lúc khách gửi Request** (không phải lúc duyệt) trên `reservation`; duyệt không ghi lại. Giá đã báo bằng giá của **quote** khách thấy; lúc duyệt `Σ` phòng tạo ra phải bằng giá đó. Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §3, §7.
