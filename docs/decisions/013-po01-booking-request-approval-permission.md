# ADR 013 — Permission riêng `MANAGE_BOOKING_REQUEST`, chỉ ADMIN/MANAGER

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-11, phản hồi Codex review REQUEST CHANGES), giải quyết PO-01. [Permission matrix](../../../hotel/docs/specs/PMS-0034-hotel-website-direct-booking-v2a.md#9-phân-quyền--resolved-in-design-po-01)

## Bối cảnh

Codex review chỉ ra: nếu duyệt/từ chối Booking Request dùng permission `MANAGE_BOOKING` hiện có, sẽ vô tình mở rộng quyền này — xác nhận từ `RolePermissionPolicy.java`: `MANAGE_BOOKING` không thuộc `ADMIN_ONLY_PERMISSIONS` và không thuộc `STAFF_NEVER_PERMISSIONS`, nghĩa là STAFF có thể được cấp `MANAGE_BOOKING` hôm nay (nó bao phủ tạo/sửa/hủy/check-in/check-out reservation nội bộ) và do đó cũng sẽ vô tình được quyền duyệt Booking Request nếu tái dùng.

## Quyết định

Chỉ ADMIN và MANAGER được approve/reject Booking Request. Thiết kế một permission riêng cho việc này, không mở rộng `MANAGE_BOOKING` ra ngoài ý muốn. Không cấp quyền approve/reject cho STAFF.

## Hệ quả

Permission mới `MANAGE_BOOKING_REQUEST` được thêm vào `RolePermissionPolicy.APPLICATION_PERMISSIONS` VÀ `STAFF_NEVER_PERMISSIONS` (thay đổi code, không chỉ migration) — chặn STAFF ở tầng policy bất kể `role_permission` bị gán sai. Migration seed `permission` + `role_permission` cho ADMIN+MANAGER theo đúng mẫu `V64__create_hotel_setting.sql`; ADMIN tự động có quyền qua cơ chế "ADMIN toàn quyền" hiện có (`PermissionCatalog.effectiveCodes`), không cần `role_permission` riêng cho ADMIN. Chi tiết thiết kế: `hotel-system/docs/specs/PMS-0034-data-design.md` M01, `PMS-0034-hotel-website-direct-booking-v2a.md` mục 9, `PMS-0034-codex-review-resolution-2026-11-11.md` mục 1 và 6.

Mở ADR mới nếu Product Owner sau này muốn mở rộng quyền duyệt cho STAFF ở một cấp độ hạn chế hơn (ví dụ chỉ xem, không duyệt).

## Cập nhật Round 3 (2026-11-11)

Migration seed nay là **M01** (foundation, BE-02), đứng **trước** mọi migration phụ thuộc; không còn V79. Bộ permission đầy đủ và các điểm chờ PO (MANAGER có bị ADMIN thu hồi được không — PO-Q3; mô hình `VIEW/MANAGE_WEBSITE_CONFIG` — PO-Q4): [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §10.3. Quyết định PO-01 (chỉ ADMIN/MANAGER; permission riêng; không STAFF) **không đổi**.

## Cập nhật theo PO-Q3 (2026-11-11)

PO duyệt thêm: ADMIN và MANAGER duyệt/từ chối khi có permission; **ADMIN thu hồi được khỏi MANAGER** (không thuộc `MANAGER_ALWAYS_PERMISSIONS`); STAFF không nhận mặc định; backend kiểm **permission hiệu lực**, không theo tên vai trò; không thiết kế lại RBAC. Xem [ADR 018](018-po-decisions-round3-q3-q10.md) và [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §10.3. Lưu ý đã ghi nhận (O1): văn bản PO-Q3 nói STAFF "không nhận mặc định" còn ADR này nói "không cấp cho STAFF"; thiết kế giữ `STAFF_NEVER_PERMISSIONS`.

## Cập nhật Round 4 (2026-11-11) — đồng bộ catalog quyền

Thêm quyền vào `RolePermissionPolicy.APPLICATION_PERMISSIONS` là chưa đủ: `RolePermissionService.GROUPS` phải đồng bộ (nếu lệch, khối `static` ném `IllegalStateException` và Spring context không khởi động), kèm `STAFF_NEVER_REJECTIONS`, thông điệp VI/EN, test cứng số lượng và seed migration. `MANAGE_BOOKING_REQUEST` vào `STAFF_NEVER_PERMISSIONS` nhưng **không** vào `MANAGER_ALWAYS_PERMISSIONS` nên ADMIN thu hồi được khỏi MANAGER. Cơ chế làm mới authority mỗi request đã xác minh bằng đọc source. Quyết định của ADR không đổi. Chi tiết và test: [../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md) §2.
