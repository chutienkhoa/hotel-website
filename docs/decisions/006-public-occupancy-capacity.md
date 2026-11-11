# ADR 006 — Quy tắc sức chứa adults+children chỉ áp dụng cho Public Booking API

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-10, M0), giải quyết [D-03](../audits/issue-30-gap-analysis.md#71-chặn-bắt-đầu). [Guest Occupancy & Multi-Room Booking](../specs/client-specs.md#5-guest-occupancy--multi-room-booking)

## Bối cảnh

Quy tắc sức chứa hiện hành của PMS (`AdultCapacityRules`) chỉ tính người lớn; trẻ em không chiếm sức chứa. Đặc tả client lại yêu cầu `adults + children` cho website công khai — hai quy tắc mâu thuẫn trực tiếp (gap analysis G-09, finding F-06, MAJOR).

## Quyết định

Public Booking API tính sức chứa bằng `adults + children`. Hành vi PMS V1 (quy tắc chỉ tính người lớn cho nghiệp vụ nhân viên) không đổi trong phạm vi Issue #30.

## Hệ quả

PMS giữ nguyên `AdultCapacityRules` cho mọi luồng nhân viên hiện có (tạo/sửa reservation trong PMS Admin, Stay Extension, Room Change…). `PMS-0034-hotel-website-direct-booking-v2a.md` đề xuất một quy tắc sức chứa mới, áp dụng riêng cho đường Public Booking API (`adults ≥ 1`, `adults + children ≤ Σ capacity các phòng đã chọn`), không sửa `AdultCapacityRules`. Tài liệu đặc tả PMS phải ghi rõ cả hai định nghĩa sức chứa (nhân viên và công khai) để không ai nhầm lẫn. Việc chốt D-03 mở khóa gap G-09 và AC-05.

Mở ADR mới nếu Product Owner sau này muốn thống nhất một quy tắc sức chứa duy nhất cho cả PMS Admin và website.
