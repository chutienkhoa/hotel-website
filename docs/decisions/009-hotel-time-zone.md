# ADR 009 — Quy tắc thời gian đặt phòng công khai dùng `hotel_setting.time_zone`

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-10, M0), giải quyết [D-06](../audits/issue-30-gap-analysis.md#71-chặn-bắt-đầu); phạm vi áp dụng được thu hẹp lại phần PMS V1 theo finding mới — xem Hệ quả. [Booking Flow §1](../specs/client-specs.md#7-booking-flow)

## Bối cảnh

`hotel_setting.time_zone` đã tồn tại (migration V64, PMS-0040) nhưng chỉ lưu và hiển thị: `DashboardClockConfiguration` hardcode `Clock` nghiệp vụ của toàn PMS là `Asia/Ho_Chi_Minh`, không đọc `hotel_setting.time_zone`. Không có quy tắc min/max-stay, lead-time hay ngày quá khứ nào cho đặt phòng công khai (gap analysis G-25, finding F-09, MAJOR).

## Quyết định

Áp dụng `hotel_setting.time_zone` cho các quy tắc thời gian của khách sạn (ví dụ: xác định "hôm nay" khi kiểm tra lead-time, ngày hết hạn 24 giờ của Booking Request — ADR 004, cutoff miễn phí hủy). Không hardcode `Asia/Ho_Chi_Minh` cho toàn bộ khách sạn.

## Hệ quả

**Finding kỹ thuật cần lưu ý khi triển khai:** `DashboardClockConfiguration` hiện là một `Clock` bean DUY NHẤT cho toàn bộ nghiệp vụ PMS (cutoff check-in/check-out, no-show, báo cáo theo ngày…), không riêng cho Issue #30. Đổi bean này sang đọc `hotel_setting.time_zone` sẽ thay đổi hành vi của mọi luồng V1/PMS Admin tại các khách sạn có `time_zone` khác `Asia/Ho_Chi_Minh` — vượt ra ngoài ràng buộc "không đổi hành vi PMS V1 trong phạm vi Issue #30" áp dụng cho các quyết định khác (ADR 006). `PMS-0034-hotel-website-direct-booking-v2a.md` đề xuất phương án: giữ nguyên `DashboardClockConfiguration` cho các luồng PMS Admin/V1 hiện có, và thêm một nguồn giờ riêng (đọc `hotel_setting.time_zone`, fallback `Asia/Ho_Chi_Minh`) chỉ dùng cho các quy tắc thời gian MỚI của Public Booking API (lead-time, hạn Request, cutoff hủy). Nếu Product Owner muốn áp dụng `time_zone` cho toàn bộ PMS (không riêng Issue #30), cần một quyết định riêng ngoài phạm vi #30 vì ảnh hưởng hồi quy diện rộng. Việc chốt D-06 mở khóa gap G-19/G-25 và AC-19.

Mở ADR mới nếu Product Owner quyết định wire `hotel_setting.time_zone` vào `Clock` nghiệp vụ chung của toàn PMS.
