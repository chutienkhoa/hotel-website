# ADR 005 — Snapshot giá theo đêm, không hồi tố theo Rate Calendar

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-10, M0), giải quyết [D-02](../audits/issue-30-gap-analysis.md#71-chặn-bắt-đầu). [Pricing & Room Rate Calendar](../specs/client-specs.md#6-pricing--room-rate-calendar)

## Bối cảnh

`reservation_room.nightly_rate` hiện là một giá trị cố định cho toàn bộ kỳ ở của một phòng; `total_amount = nightly_rate × số đêm`. Không có bảng nào lưu giá theo từng đêm, nên không thể biểu diễn Daily Rate Override hoặc giữ nguyên giá đã chốt khi Rate Calendar đổi sau đó (gap analysis G-11…G-14, finding F-01, BLOCKER).

## Quyết định

Lưu snapshot giá từng đêm cho từng phòng trong reservation. Giá của một reservation đã xác nhận không thay đổi khi Rate Calendar sau đó bị sửa. Migration bổ sung phải tương thích với dữ liệu V1 hiện có (không yêu cầu backfill suy diễn ngoài giá theo đêm đã biết).

## Hệ quả

`PMS-0034-hotel-website-direct-booking-v2a.md` và `PMS-0034-data-design.md` (repo `hotel-system`) đề xuất kỹ thuật cho quyết định này: một bảng snapshot giá theo đêm cho mỗi `reservation_room` (một dòng mỗi đêm), giữ `reservation_room.nightly_rate`/`total_amount` hiện có cho tương thích hiển thị, cùng bảng Base Rate/Daily Override ở `room_type`. Thiết kế bảng cụ thể còn chờ Codex và Product Owner xác nhận trước khi tạo migration thật. Việc chốt D-02 mở khóa BE-09, BE-10, BE-11 và gap G-11–G-14.

Mở ADR mới nếu V2A cần một mô hình rate plan phức tạp hơn một giá hiệu lực mỗi Room Type/ngày.

## Cập nhật Round 3 (2026-11-11)

Phần "Hệ quả" ở trên (một bảng snapshot theo đêm cho mỗi `reservation_room`, giữ `nightly_rate`/`total_amount` "cho tương thích hiển thị") được thực hiện bằng hai chế độ giá `FLAT`/`DAILY` và bảng `room_night_rate` theo [ADR 017](017-nightly-pricing-flat-and-daily-modes.md) (đề xuất, chờ PO-Q1). Quyết định của ADR này (snapshot giá từng đêm, không hồi tố, tương thích dữ liệu V1) **không đổi**; cách "giữ `nightly_rate`" dưới dạng giá trung bình phái sinh đã bị loại.
