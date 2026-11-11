# ADR 012 — Modern Tropical – Bright & Fresh là template V2A; mockup là điều kiện bắt đầu UI

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-10, M0), giải quyết [D-09](../audits/issue-30-gap-analysis.md#71-chặn-bắt-đầu). [3.2 Modern Tropical](../specs/client-specs.md#32-modern-tropical)

## Bối cảnh

`docs/designs/{mockups,references,screenshots}` chỉ chứa `.gitkeep`; chưa có mockup nào được duyệt cho website công khai hoặc cho các màn hình PMS Admin mới mà V2A cần (Room Type Gallery, Rate Calendar, cấu hình Instant/Request, hàng đợi duyệt Booking Request) (gap analysis G-27, finding F-05, BLOCKER).

## Quyết định

Modern Tropical – Bright & Fresh là template V2A. Cần thiết kế mockup cho cả website công khai và các màn hình PMS Admin liên quan. Không bắt đầu implement UI tương ứng khi mockup chưa được duyệt.

## Hệ quả

[`docs/designs/mockup-requirements-issue-30.md`](../designs/mockup-requirements-issue-30.md) (tài liệu mới, cùng đợt soạn này) liệt kê đầy đủ các màn hình website và PMS Admin cần mockup, theo đúng quy tắc đặt tên và trạng thái duyệt của [designs/README.md](../designs/README.md). Cổng này áp dụng cho UI; phần backend/adapter (schema, API, migration) có thể tiếp tục bằng fixture trong lúc chờ mockup, theo nguyên tắc "Contract before code" của implementation plan. GOV-05 vẫn là bước Product Owner/nhà thiết kế cung cấp mockup thật; ADR này không tự tạo ra mockup. Việc chốt D-09 mở khóa gap G-27 và xác nhận phạm vi mockup cần cho cả hai repo.

Mở ADR mới nếu V2A thêm một template thứ hai ngoài Modern Tropical – Bright & Fresh.
