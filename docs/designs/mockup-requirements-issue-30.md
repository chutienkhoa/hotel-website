# Yêu cầu Mockup — Issue #30 (PMS-0034)

**Trạng thái** — Danh mục mockup cần thiết, lập theo [ADR 012](../decisions/012-mockup-gate.md) (D-09, Đã chốt: Modern Tropical – Bright & Fresh là template V2A; không implement UI trước khi mockup duyệt). Đây là **danh sách yêu cầu**, không phải mockup đã có — GOV-05 của [implementation plan](../development/issue-30-implementation-plan.md#gov-05-mockup-đã-duyệt) vẫn là bước Product Owner/nhà thiết kế cung cấp mockup thật, đặt tại `docs/designs/mockups/` theo quy tắc của [designs/README.md](README.md).

Tại thời điểm soạn, `docs/designs/{mockups,references,screenshots}` chỉ có `.gitkeep` (gap G-27 / finding F-05). Danh mục dưới đây là điều kiện để đóng gap đó.

## 1. Website công khai (`hotel-website`, Modern Tropical – Bright & Fresh)

| Màn hình | Trạng thái cần có | Viewport |
| --- | --- | --- |
| Trang chủ | Mặc định; branding theo PMS | mobile 390, tablet 768, desktop 1440 |
| Room Catalog | Đầy đủ dữ liệu; thiếu ảnh tùy chọn; có Room Type Sold Out | mobile, tablet, desktop |
| Room Type Detail | Đầy đủ gallery; gallery chỉ 1 ảnh (fallback cover) | mobile, tablet, desktop |
| Search / Select Rooms | Loading, kết quả có phòng, Sold Out, lỗi tải | mobile, tablet, desktop |
| Guest Info | Trạng thái lỗi validate | mobile, tablet, desktop |
| Review & Confirm | Giá thay đổi cần re-consent; chế độ đổi cần re-confirm | desktop tối thiểu |
| Booking Result — Instant | Đã xác nhận | mobile, tablet, desktop |
| Booking Result — Request | Đã nhận, chờ duyệt | mobile, tablet, desktop |
| Booking Result — Unknown outcome | Timeout/không rõ kết quả | desktop tối thiểu |
| Manage Booking — tra cứu | Form + thông báo chung (không lộ tồn tại booking) | mobile, tablet, desktop |
| Manage Booking — chi tiết & hủy | Còn trong hạn hủy miễn phí; đã quá hạn | mobile, tablet, desktop |
| Gallery / Contact / Location | Mặc định | mobile, tablet, desktop |
| Trạng thái chung | empty, loading, error cho mọi màn hình trên | mobile, tablet, desktop |

Theo `designs/README.md`: đặt tên `<màn hình>-<viewport>-v0X-<draft|approved>.png`; mỗi ảnh ghi trạng thái duyệt; dùng dữ liệu giả.

## 2. PMS Admin (`hotel-system`, Thymeleaf) — màn hình mới cho PMS-0034

Theo `master-backlog.md` mục 6F, các trách nhiệm PMS Admin dưới đây hiện chưa có giao diện tương ứng và cần mockup trước khi implement UI (phần backend/API có thể dùng fixture, không bị chặn bởi mockup theo nguyên tắc "Contract before code"):

| Màn hình | Phục vụ quyết định |
| --- | --- |
| Quản lý Room Type Gallery (tải/xoá/sắp xếp ảnh, chọn cover) | D-07 deliverable #4 (`PMS-0034-data-design.md` M03) |
| Chọn Room Type bán trên website + sắp xếp thứ tự (`show_on_website`, `display_order`) | D-02/D-07 (`PMS-0034-data-design.md` M02) |
| Room Rate Calendar (sửa từng ngày, sửa theo khoảng + lọc thứ, preview, lịch sử) | D-02 (`PMS-0034-data-design.md` M04–M05; preview trả `expectedRevision`, xung đột ⇒ `409` yêu cầu preview lại — Round 3 §9) |
| Cấu hình Instant Booking / Booking Request theo hotel | D-01, D-04 |
| Cấu hình chính sách hủy (giờ miễn phí hủy) | D-01/D-04 (`PMS-0034-data-design.md` M07a; chính sách theo phiên bản, lưu cả văn bản) |
| Website Settings: cảnh báo cấu hình nguồn `HOTEL_WEBSITE` (thiếu/không active/không hợp lệ) kèm bước sửa; chỉ hiện và chỉ sửa được với permission `MANAGE_WEBSITE_CONFIG` | PO-Q4, PO-Q5 (round 3 §10.3–10.4) |
| Hàng đợi Booking Request: nút duyệt/từ chối chỉ hiện khi người dùng có permission hiệu lực `MANAGE_BOOKING_REQUEST` (ẩn nút không phải kiểm soát bảo mật); trạng thái `PENDING_APPROVAL`/`REJECTED`/`EXPIRED` riêng; thông báo khi Request tự bị từ chối lúc duyệt (hết phòng, Room Type không còn đủ điều kiện) hoặc đã hết hạn | PO-Q3, PO-Q9, PO-Q10 |
| Hàng đợi duyệt Booking Request (xem, duyệt, từ chối, xem hạn xử lý) | D-01 |
| Website Settings mở rộng (homepage, domain, gallery mục 1) nếu chưa có | D-07 (đã có một phần theo PMS-0040, chỉ bổ sung phần mới) |

Các màn hình này không thuộc Modern Tropical (đó là template website); dùng theme/khuôn UI Thymeleaf hiện có của PMS Admin, không cần mockup theo chuẩn thiết kế riêng trừ khi Product Owner yêu cầu.

## 3. Ghi chú

- Mockup không tự mở khóa phần backend/schema — các phần đó theo `PMS-0034-hotel-website-direct-booking-v2a.md` và có thể tiến hành song song bằng fixture.
- Khi mockup được duyệt, cập nhật trạng thái tại đây và trong GOV-05 của implementation plan, không tạo tài liệu trùng.
