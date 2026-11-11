# ADR 007 — Gán phòng vật lý tự động khi xác nhận, trễ đến khi duyệt với Request

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-10, M0), giải quyết [D-04](../audits/issue-30-gap-analysis.md#71-chặn-bắt-đầu). [Multi-Room Booking](../specs/client-specs.md#5-guest-occupancy--multi-room-booking)

## Bối cảnh

`reservation_room.room_id` hiện là khóa ngoại bắt buộc (`NOT NULL`): mọi đặt phòng trong PMS luôn gán một phòng vật lý cụ thể ngay khi tạo. Booking Request công khai cần giữ chỗ theo Room Type trước khi có phòng vật lý cụ thể, vì phòng chỉ nên được gán sau khi nhân viên duyệt (gap analysis G-07, finding F-08, MAJOR).

## Quyết định

Instant Booking tự động gán phòng vật lý ngay khi xác nhận. Booking Request chỉ gán phòng vật lý khi được duyệt. Cả hai luồng phải bảo đảm transaction, locking và chống overbooking tương đương cơ chế hiện có của PMS.

## Hệ quả

`PMS-0034-data-design.md` (repo `hotel-system`) đề xuất cho phép `reservation_room.room_id` là NULL trong khi Reservation ở trạng thái Request chờ duyệt (ADR 004), cùng cột `room_type_id` để biết loại phòng đã chọn trước khi có phòng vật lý; việc gán phòng khi duyệt hoặc khi Instant Booking xác nhận chạy trong cùng transaction với kiểm tra khóa/availability hiện có của `RoomAvailabilityService`. Thiết kế cột/migration cụ thể còn chờ Codex và Product Owner xác nhận. Việc chốt D-04 mở khóa gap G-07 và phối hợp với ADR 004/005 cho BE-12, BE-17, BE-18.

Mở ADR mới nếu V2A sau này cho khách chọn phòng vật lý cụ thể (ngoài phạm vi hiện tại).

## Cập nhật (2026-11-11, Round 2–3) — sửa phần "Hệ quả" lỗi thời

Phần "Hệ quả" ở trên đề xuất `reservation_room.room_id` NULL trong khi Request chờ duyệt: **đã bị loại bỏ** (vi phạm bất biến V1: 4 điểm NPE đã xác nhận và các truy vấn âm thầm loại dòng). Quyết định của ADR này **không đổi** (Instant gán phòng khi xác nhận; Request chỉ gán phòng khi duyệt). Thực hiện: Request không tạo `reservation_room` nào trước khi duyệt; dữ liệu nằm ở `booking_request_line*` (migration M12, BE-18); khi duyệt, tạo `reservation_room` mới với `room_id NOT NULL` như luồng Instant, dưới cùng thứ tự khóa (phòng theo id tăng dần → reservation). Xem [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §3.
