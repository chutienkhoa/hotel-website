# Technical Gap Analysis — Issue #30 (PMS-0034): Hotel Website & Direct Booking V2A

**Loại tài liệu:** audit kỹ thuật, chỉ đọc. Không sửa mã ứng dụng, không commit, không push, không tạo PR.
**Ngày audit:** 2026-11-10
**Người thực hiện:** Claude Code (theo yêu cầu của chủ repo). Kết quả cần Codex review độc lập và Product Owner (PO) xác nhận các quyết định ở mục 7.
**Tài liệu đi kèm:** [Implementation plan Issue #30](../development/issue-30-implementation-plan.md)

## 1. Phạm vi, phương pháp và giới hạn

### 1.1 Revision đã audit

| Repo | Đường dẫn | Revision | Trạng thái |
| --- | --- | --- | --- |
| Backend PMS (`hotel-system`) | `/Users/khoact/Documents/code/AI/hotel` | nhánh `develop-v2`, HEAD `60821ff` ("fix ui", 2026-11-10) | working tree sạch |
| Frontend | `/Users/khoact/Documents/code/AI/hotel-website` | nhánh `develop`, 1 commit (`1cd7827`) | `.agents/`, `.claude/`, `AGENTS.md`, `CLAUDE.md`, `docs/` chưa track. **Chưa có mã nguồn**: không có `package.json`, `src/`, `tsconfig`, lockfile |

Revision PMS khớp mốc đối chiếu `60821ff` ghi trong [docs/README.md](../README.md).

### 1.2 Đã đọc

- Frontend: toàn bộ `docs/` (specs, coding-rules, security, ADR 001–003, development, testing, deployment, designs, requirements), `AGENTS.md`, `CLAUDE.md`. Đường dẫn `docs/architecture/client-architecture.md` trong yêu cầu **không tồn tại**; tài liệu kiến trúc nằm ở [docs/coding-rules/client-architecture.md](../coding-rules/client-architecture.md).
- PMS: `AGENTS.md`, `docs/coding-rules/{architecture,api-standards,security-standards}.md`, `docs/project-management/workflow.md`, `docs/requirements/{pms-0040-hotel-settings,technical-spec-v2}.md`, các đoạn liên quan của `technical-spec-v1.md` (grep theo từ khóa, **không đọc toàn bộ 6.701 dòng**), `docs/operations/production-deployment.md`, Issue GitHub #30, #55, #33, #35, #89, #90 (chỉ đọc).
- Mã PMS: các migration Flyway liên quan đến reservation, room, room type, hotel setting, branding, booking source, idempotency (thư mục có 64 file V1–V67, không có V59–V61; các file đã đọc nằm ở phụ lục); entity `Reservation`, `ReservationRoom`, `Room`, `RoomType`; `ReservationService`, `RoomAvailabilityService`, `RoomRepository`, `AdultCapacityRules`, `RoomTypeManagementService`, `RoomImageService`, `HotelSettingService`, `HotelBrandingController`, `GuestService`, `PaymentIdempotencyService`, `SecurityConfig`, `CurrentActor`, `ClientIpResolver`, `pom.xml`, `application.yml`, `Dockerfile`.

### 1.3 Không làm / chưa xác minh

- Không chạy build, test, ứng dụng hay truy vấn database. Mọi kết luận dựa trên đọc mã, migration và tài liệu. Hành vi runtime (đặc biệt đồng thời, múi giờ phiên CSDL, Caddy thực tế) là **chưa xác minh bằng thực nghiệm**.
- Không có quyền xem môi trường staging/production, nên cấu hình Caddy, DNS, TLS, nhà cung cấp email thật chưa được đối chiếu.
- Không tìm thấy endpoint, bảng hoặc trạng thái nào ngoài danh sách đã đọc; những gì ghi "chưa có" nghĩa là **không tìm thấy trong mã ở revision trên** (grep theo từ khóa `@Scheduled`, `JavaMailSender`, `spring.mail`, `RateLimit`, `Cors`, `slug`, `publish`, `cancellationPolicy`, `RateCalendar`, `quote`, `amenit`, `/api/public`).
- Chi tiết định danh/DTO/endpoint của hợp đồng mới **không được suy đoán** ở đây; mọi tên trong kế hoạch là đề xuất cần chốt ở hợp đồng riêng.

### 1.4 Thang đánh giá

| Nhãn | Ý nghĩa |
| --- | --- |
| **Đã có** | Mã/schema hiện tại đáp ứng đủ yêu cầu của đặc tả |
| **Có một phần** | Có nền tảng tái sử dụng được nhưng thiếu hoặc khác đặc tả |
| **Chưa có** | Không có mã/schema, hoặc mô hình hiện tại không tái sử dụng được |
| Ưu tiên P0 | Nền tảng/quyết định, chặn nhiều việc khác |
| Ưu tiên P1 | Cần cho luồng đặt phòng Instant đầu-cuối |
| Ưu tiên P2 | Cần cho đủ phạm vi V2A (Request, Manage, hủy, email) |
| Ưu tiên P3 | Hoàn thiện, vận hành, hardening |
| Mức finding | **BLOCKER** (không thể bắt đầu vùng liên quan) · **MAJOR** (sai hành vi/rủi ro lớn nếu bỏ qua) · **MINOR** |

## 2. Tóm tắt điều hành

- **28 gap** (G-01…G-28): **0 Đã có · 14 Có một phần · 14 Chưa có**. Không nghiệp vụ nào của Issue #30 đã sẵn sàng đầy đủ.
- **21 finding** (F-01…F-21): **5 BLOCKER · 9 MAJOR · 7 MINOR**, gồm 4 mâu thuẫn trực tiếp giữa đặc tả client và PMS hiện tại (sức chứa, gallery, giá theo đêm, múi giờ).
- PMS có **nền tốt cho phần lõi**: khóa bi quan theo phòng, truy vấn xung đột tồn phòng dùng chung, vòng đời `DRAFT → CONFIRMED → CANCELLED`, audit log, mẫu idempotency (cho Payment/Charge), lưu và xử lý ảnh, branding logo/favicon, login throttle. Phần này nên **tái sử dụng**, không viết lại.
- PMS **chưa có bất kỳ bề mặt công khai nào** cho website: không có `/api/public`, không có xác thực service-to-service, không có nội dung website, không có catalog theo loại phòng, không có lịch giá, không có quote, không có Manage Booking, không có chính sách hủy, không có email/outbox/scheduler, không có rate limiting.
- Frontend hiện **chỉ có tài liệu**. Toàn bộ ứng dụng Next.js, hợp đồng API, fixture và mockup đã duyệt đều chưa tồn tại (`docs/designs/mockups/` chỉ có `.gitkeep`).
- Điều kiện mở khóa V2 ở backlog đã thỏa: PMS-0028 (#64) **Done**, PMS-0053 (#55) **CLOSED** (snapshot `project-status.json` cũ vẫn ghi In Progress). Tuy nhiên Issue #30 vẫn **Backlog**, chưa có tiêu chí chấp nhận trong Issue, và PMS chưa có mục đặc tả PMS-0034.

### Các BLOCKER

| ID | Nội dung ngắn | Vì sao chặn |
| --- | --- | --- |
| F-01 | Schema/folio lưu **một `nightly_rate` phẳng** cho mỗi phòng, không biểu diễn được giá override theo từng đêm | Chặn Daily Override, Rate Calendar, snapshot giá (AC-08/09) |
| F-02 | Vòng đời **Request** chưa định nghĩa; `DRAFT` hiện tại không giữ tồn, không hết hạn, không có từ chối | Chặn chế độ Request (AC-11/12) |
| F-03 | Chưa có hợp đồng public API và **mô hình xác thực Next.js → PMS** | Chặn mọi tích hợp thật |
| F-04 | **Governance**: Issue #30 chưa Ready, PMS chưa có đặc tả PMS-0034, kiến trúc PMS cấm tạo domain/package mới khi chưa được duyệt | Chặn bắt đầu code backend theo quy trình PMS |
| F-05 | Không có **mockup đã duyệt** trong repo website | Chặn dựng giao diện Modern Tropical đúng thiết kế |

## 3. Bảng tổng hợp gap

| ID | Nghiệp vụ | Trạng thái | Ưu tiên | Finding liên quan |
| --- | --- | --- | --- | --- |
| G-01 | Website Settings — mô hình nội dung (General, Appearance, Homepage, Contact, Booking & Publishing) | Có một phần | P0 | F-13 |
| G-02 | Draft/Published, preview, publish có audit, làm mới cache | Chưa có | P1 | F-14 |
| G-03 | Branding (tên, logo, favicon) cho website | Có một phần | P1 | F-20 |
| G-04 | Room Catalog (bật bán, thứ tự, slug, nội dung, tiện nghi) | Có một phần | P1 | F-12, F-14 |
| G-05 | Room Type Gallery (cover, thứ tự, alt text) | Chưa có | P1 | F-07 |
| G-06 | Public Booking API: bề mặt, xác thực, hợp đồng, lỗi | Chưa có | P0 | F-03, F-21 |
| G-07 | Multi-Room Booking (Room Type × quantity, gán phòng, Guest, actor) | Có một phần | P1 | F-08 |
| G-08 | Availability theo Room Type, Sold Out, giới hạn đặt | Có một phần | P1 | F-12 |
| G-09 | Capacity Validation (adults + children) | Có một phần | P1 | F-06 |
| G-10 | Chống overbooking trên đường đặt công khai | Có một phần | P1 | F-12 |
| G-11 | Base Rate theo Room Type | Có một phần | P1 | F-01 |
| G-12 | Daily Rate Override | Chưa có | P1 | F-01 |
| G-13 | Room Rate Calendar (sửa 1 ngày, bulk, preview, audit) | Chưa có | P2 | F-01 |
| G-14 | Snapshot giá reservation, tiền tệ, thuế/phí | Có một phần | P1 | F-01, F-18 |
| G-15 | Quote có định danh/thời hạn | Chưa có | P1 | — |
| G-16 | Instant Booking | Có một phần | P1 | F-08 |
| G-17 | Booking Request lifecycle | Chưa có | P2 | F-02 |
| G-18 | Manage Booking + xác thực email | Chưa có | P2 | F-11 |
| G-19 | Chính sách hủy theo khách sạn (cấu hình, snapshot, cutoff) | Chưa có | P2 | F-09, F-13 |
| G-20 | Giao dịch tự hủy của khách | Có một phần | P2 | F-17 |
| G-21 | Email giao dịch + outbox/retry | Chưa có | P1 | F-10 |
| G-22 | Idempotency và đối soát kết quả chưa rõ | Có một phần | P1 | — |
| G-23 | Bảo mật API công khai (rate limit, anti-enumeration, CORS, PII, request-id) | Có một phần | P1 | F-11 |
| G-24 | Domain, TLS, canonical, xác minh custom domain | Chưa có | P3 | F-19 |
| G-25 | Múi giờ khách sạn, giờ check-in, quy tắc đặt (min/max stay, đặt trước) | Có một phần | P1 | F-09, F-12 |
| G-26 | Ứng dụng frontend Next.js | Chưa có | P0 | F-15 |
| G-27 | Mockup Modern Tropical đã duyệt, asset thiết kế | Chưa có | P0 | F-05 |
| G-28 | Governance: Issue Ready, đặc tả PMS-0034, duyệt domain/package mới | Chưa có | P0 | F-04 |

## 4. Gap chi tiết theo nghiệp vụ

Quy ước đường dẫn: `hotel:` là gốc `/Users/khoact/Documents/code/AI/hotel`, mã Java ở `src/main/java/com/pms/hotel/` được viết tắt `…/`. `web:` là gốc `hotel-website`.

### G-01 Website Settings — mô hình nội dung

- **Trạng thái:** Có một phần.
- **Hiện có:** bảng singleton `hotel_setting` (V64): `name`, `address`, `phone`, `email`, `check_in_time` (mặc định 14:00), `check_out_time` (12:00), `time_zone`, `review_status`, `version` (optimistic lock), audit. `HotelSettingService`, màn hình Thymeleaf, quyền `VIEW_HOTEL_SETTING`/`MANAGE_HOTEL_SETTING` (V64).
- **Thiếu:** tagline, giới thiệu, hero, điểm nổi bật, tiện nghi, link bản đồ, kênh liên hệ/mạng xã hội, mã template, màu thương hiệu (token cho phép), chế độ Instant/Request, cửa sổ hủy 24/48/72h, trạng thái draft/published, thông tin domain. Không có REST cho cài đặt. Không tìm thấy từ khóa `tagline`, `hero`, `amenit` trong mã chính (chỉ `DemoDataSeeder`).
- **Cần bổ sung:** mô hình nội dung website mới (khuyến nghị bảng riêng, singleton, có `version`, tránh nhồi vào `hotel_setting` vì bảng đó mang ngữ nghĩa `PENDING_REVIEW`/`legacy_*`), màn hình quản trị, validation text-only (không HTML/JS tùy ý), quyền mới.
- **Ưu tiên/dependency:** P0; phụ thuộc quyết định phạm vi (F-13) và G-28.
- **Rủi ro:** `hotel:docs/requirements/pms-0040-hotel-settings.md` (mục "Ngoài phạm vi") loại trừ rõ chính sách hủy, API REST cho cài đặt, màu chủ đề, multi-tenant. Mở rộng cần sửa các ghi chú loại trừ đó và PO duyệt, nếu không sẽ vi phạm quy tắc "không suy diễn nghiệp vụ".

### G-02 Draft/Published, preview, publish, cache

- **Trạng thái:** Chưa có.
- **Hiện có:** không có khái niệm draft/published cho nội dung website (`grep published|publish` chỉ khớp comment về ghi file ảnh nguyên tử trong `common/image/ImageAssetStore`). `AuditLog` dùng lại được cho audit publish.
- **Cần bổ sung:** hai trạng thái nội dung (hoặc snapshot đã publish), hành động publish có quyền + audit, preview có bảo vệ và `noindex`, tín hiệu làm mới cache (version/ETag hoặc webhook revalidate sang Next.js).
- **Ưu tiên/dependency:** P1; phụ thuộc G-01.
- **Rủi ro:** lộ nội dung chưa công bố qua preview; cache cũ sau publish gây sai thứ tự phòng/ảnh (AC-03). Cần thiết kế tập trung, không để mỗi endpoint tự cache.

### G-03 Branding (logo, favicon, tên)

- **Trạng thái:** Có một phần.
- **Hiện có:** V66 thêm `logo_*`, `favicon_*` vào `hotel_setting`; `HotelBrandingService` kiểm tra MIME/kích thước qua `ImageInspector`/`ImageProcessor`; `HotelBrandingController` phục vụ `/branding/logo`, `/branding/favicon`, `/favicon.ico`, **`permitAll` trong `SecurityConfig`**, ETag, `Cache-Control: private`, `immutable` khi có tham số `v` khớp.
- **Thiếu:** không có JSON công khai cho website (tên, URL logo/favicon có phiên bản, alt text). Không có hợp đồng ổn định: [docs/development/api-integration.md](../development/api-integration.md) nêu rõ route branding của PMS "chưa xác lập hợp đồng website".
- **Cần bổ sung:** đưa branding vào DTO công bố của site config; quyết định website tải ảnh trực tiếp từ PMS (cần allowlist `next/image`) hay qua proxy của Next.js; xử lý `Cache-Control: private` khi qua CDN.
- **Ưu tiên/dependency:** P1; phụ thuộc G-06.
- **Rủi ro:** thấp. Lưu ý các route `/branding/*` hiện đã public trên PMS; cần xác nhận đó là ý định khi PMS nằm sau Caddy cùng host.

### G-04 Room Catalog

- **Trạng thái:** Có một phần.
- **Hiện có:** `room_type` (V1, V57): `code` (UNIQUE, in hoa), `name` (UNIQUE không phân biệt hoa thường), `description`, `capacity` (**nullable**, CHECK ≥ 1), `base_price` (nullable), `active`. `RoomTypeManagementService` tạo/sửa/ngừng/kích hoạt; ngừng chỉ được khi không còn phòng active. Màn hình quản trị `RoomTypePageController`.
- **Thiếu:** `show_on_website`, `display_order`, `slug`, mô tả công khai (cột `description` tồn tại nhưng `RoomTypeCreateRequest` chỉ có `code`, `name`, `capacity`; `updateProfile(name, capacity)`), tiện nghi, điều kiện hiển thị (active ∧ bật bán ∧ published). Capacity nullable nên không thể bán nếu chưa cấu hình.
- **Cần bổ sung:** cột/bảng mới cho cấu hình website, quy tắc đủ điều kiện công bố, màn hình quản trị, truy vấn công khai chỉ trả Room Type đủ điều kiện, và **kiểm tra lại ở backend khi tạo booking** (AC-02).
- **Ưu tiên/dependency:** P1; phụ thuộc G-01, G-02.
- **Rủi ro:** slug phải duy nhất và ổn định; đổi tên Room Type không được làm đổi URL công khai âm thầm. Tắt bán không được ảnh hưởng reservation đang có (đặc tả §4).

### G-05 Room Type Gallery

- **Trạng thái:** Chưa có (mô hình hiện tại khác đặc tả).
- **Hiện có:** `room_image` (V38) gắn với **`room_id` (phòng vật lý)**, tối đa 10 ảnh/phòng (`RoomImageService.MAX_IMAGES_PER_ROOM`), chỉ có cờ `is_primary` (unique một ảnh chính/phòng), **không có thứ tự, không có alt text**. Hạ tầng xử lý ảnh tốt, tái sử dụng được: `ImageProcessor`, `ImageVariant`, `ImageAssetStore` (ghi nguyên tử), `ImageInspector`.
- **Cần bổ sung:** bảng ảnh theo Room Type (cover, thứ tự, alt text), upload/xóa/sắp xếp/đổi cover, quy tắc khi xóa cover, phục vụ ảnh công khai (variant, cache). Không migrate `room_image` (đặc tả: không dùng gallery theo Room vật lý).
- **Ưu tiên/dependency:** P1; phụ thuộc G-04.
- **Rủi ro:** hai nguồn ảnh (theo Room cho nhân viên, theo Room Type cho website) dễ gây nhầm lẫn UX; cần đặt tên và vị trí màn hình rõ. Giới hạn dung lượng/định dạng: `RoomImageService` chỉ cho JPEG/PNG, 5 MB; cần chốt có cho WebP không. Khớp V1 spec mục 77 đã hoãn "public hotel-website media serving" tới V2 nên đây là phạm vi V2 hợp lệ.

### G-06 Public Booking API — bề mặt và xác thực

- **Trạng thái:** Chưa có.
- **Hiện có:** hai `SecurityFilterChain` (`SecurityConfig`): `/api/**` là JWT stateless, chỉ `/api/auth/login` mở, `anyRequest().authenticated()`; mọi route còn lại là session + form login. Không có `/api/public`, không có CORS, không có API key/service account, không có DTO công khai. `/api/reservations` (POST tạo, POST `/{id}/confirm`, `/cancel`…) là API **quản trị**, yêu cầu `PERM_MANAGE_BOOKING`.
- **Cần bổ sung:** chain mới `@Order` thấp hơn chain `/api/**` với `securityMatcher("/api/public/v1/**")` (nếu không, chain JWT hiện tại sẽ bắt trước); mô hình xác thực service-to-service; DTO allowlist; lỗi có mã máy đọc; request-id; `no-store` cho giao dịch; hợp đồng OpenAPI hai repo cùng duyệt.
- **Ưu tiên/dependency:** P0; chặn G-03, G-04, G-08, G-16, G-18…
- **Rủi ro:** **không được dùng lại `CreateRequest` quản trị** cho đường công khai: nó nhận `guestId`, `roomId` vật lý, `source`, `currency` và **`nightlyRate` do client gửi** (`dto/booking/request/CreateRequest.java`, `RoomRequest.java`). Dùng lại sẽ cho phép khách tự đặt giá (F-21).

### G-07 Multi-Room Booking

- **Trạng thái:** Có một phần.
- **Hiện có:** một `reservation` có nhiều `reservation_room`, UNIQUE `(reservation_id, room_id)`. Khóa phòng theo thứ tự id, rồi mới khóa reservation (`lockRoomsThenReservation`). `ReservationRoomReassignmentService` cho phép nhân viên đổi phòng trước check-in.
- **Thiếu:** đặt theo **Room Type × quantity**: `reservation_room.room_id` là `NOT NULL` và `RoomRequest.roomId` bắt buộc, tức đặt luôn là phòng vật lý cụ thể; chưa có bộ gán phòng tự động. `reservation.guest_id` `NOT NULL` và `GuestService.create` không khử trùng; `created_by`/`updated_by` `NOT NULL REFERENCES app_user`, `CurrentActor.id()` ném 401 nếu không đăng nhập, `audit_log.user_id NOT NULL`.
- **Cần bổ sung:** bộ gán phòng trong cùng transaction (khóa tất cả phòng của các Room Type được chọn theo **một thứ tự id toàn cục** để không deadlock với `confirm/check-in`), tạo Guest cho booking công khai, **actor hệ thống** cho audit, nguồn đặt `HOTEL_WEBSITE`.
- **Ưu tiên/dependency:** P1; phụ thuộc G-06, G-09, G-10, G-14.
- **Rủi ro:** gán phòng sớm làm mất linh hoạt xếp phòng (phân mảnh); giảm nhẹ nhờ Room Reassignment có sẵn. Lựa chọn thay thế (giữ tồn theo loại, gán muộn) cần đổi schema lõi và nhiều truy vấn (F-08, quyết định D-04).

### G-08 Availability theo Room Type, Sold Out

- **Trạng thái:** Có một phần.
- **Hiện có:** `RoomAvailabilityService.conflictedRoomIds` → `RoomRepository.findRoomIdsWithInventoryConflict` (một truy vấn): phòng bị chiếm bởi `reservation_room` của reservation `CONFIRMED` giao với `[in, out)` hoặc bởi `stay_room_assignment` của stay `CHECKED_IN`. `Room.isBookableInventory()` = `active` và status khác `MAINTENANCE`/`OUT_OF_ORDER`. `RoomAvailabilityDashboardService` (tối đa 30 đêm) phục vụ giao diện nhân viên.
- **Thiếu:** không có đếm **theo Room Type** cho khoảng ngày, không có endpoint tìm kiếm công khai, không có cờ Sold Out theo ngày đã chọn, không có lọc theo sức chứa. Các lookup hiện có (`/api/rooms/lookup`, `/reservations/new/rooms`) là theo phòng và yêu cầu đăng nhập.
- **Cần bổ sung:** dịch vụ tính tồn theo Room Type (tái dùng truy vấn xung đột, không viết lại), điều kiện loại trừ Room Type không đủ điều kiện, trả số lượng tối đa chọn được.
- **Ưu tiên/dependency:** P1; phụ thuộc G-04, G-06.
- **Rủi ro:** `DRAFT` **không giữ tồn** (comment trong `ReservationService.requireBookableRooms`), nên request chờ duyệt không làm giảm số phòng hiển thị (F-02). `isBookableInventory()` dùng trạng thái **hiện tại** của phòng; truy vấn xung đột **không dùng `room_inventory_period`** (V28), nên phòng sắp bảo trì trong tương lai có thể vẫn bán được. Chưa xác minh đây là chủ ý nghiệp vụ hay thiếu sót (F-12).

### G-09 Capacity Validation

- **Trạng thái:** Có một phần, **mâu thuẫn đặc tả**.
- **Hiện có:** `AdultCapacityRules`: `adultCount <= SUM(RoomType.capacity)` trên các phòng gán cho reservation; **trẻ em không tham gia**; cộng dồn toàn reservation; `capacity = null` là lỗi cấu hình. Được dùng chung bởi `confirm`, Arrival Readiness, `checkIn`, Room Reassignment, và quản lý Room Type khi giảm sức chứa. Đây là bất biến V1 (`technical-spec-v1.md` mục 66.2: "Trẻ em không chiếm sức chứa người lớn").
- **Đặc tả client:** `adults + children` cùng tính vào sức chứa (§2, §5.1, AC-05).
- **Cần bổ sung:** quy tắc `adults + children ≤ Σ capacity` và `adults ≥ 1` cho đường đặt công khai (xem F-06 về phạm vi áp dụng).
- **Ưu tiên/dependency:** P1; cần quyết định D-03.
- **Rủi ro:** đổi `AdultCapacityRules` toàn cục làm đổi hành vi check-in, đổi phòng, và chặn giảm capacity của Room Type; phải sửa `technical-spec-v1` mục 66.2 và hồi quy rộng. Đường công khai nghiêm ngặt hơn quy tắc nhân viên là an toàn về hướng (không có reservation công khai nào vi phạm quy tắc check-in).

### G-10 Chống overbooking

- **Trạng thái:** Có một phần (lõi mạnh cho luồng nhân viên).
- **Hiện có:** `confirm()` khóa `PESSIMISTIC_WRITE` các `Room` (sắp theo id) rồi `Reservation`, sau đó gọi `conflictedRoomIds` và `requireAdultCapacity` trong cùng transaction. Có `lockAllByRoomType` (`RoomRepository`, khóa mọi phòng của một loại theo id). Không có ràng buộc DB loại trừ chồng lấn (không `EXCLUDE`/btree_gist), nên **tính đúng dựa hoàn toàn vào khóa ứng dụng**.
- **Thiếu:** đường công khai phải tạo + gán + xác nhận trong **một** transaction (hiện là hai bước `create` rồi `confirm`). `confirm` **đã có** test đồng thời với Testcontainers (`ReservationConfirmConcurrencyIntegrationTest`: hai reservation chồng lấn cùng phòng chỉ một cái được xác nhận, hai phòng khác nhau thì cả hai thành công; `ReservationLifecycleLockOrderIntegrationTest`: thứ tự khóa không deadlock giữa check-out/check-in). Chưa có test cho đường đặt công khai, bộ gán phòng nhiều Room Type, hay tranh chấp giữa đặt công khai và thao tác nhân viên (chưa chạy các test này trong audit).
- **Cần bổ sung:** hàm đặt công khai dùng cùng thứ tự khóa; test đồng thời nhiều luồng tranh phòng cuối (AC-13).
- **Ưu tiên/dependency:** P1; phụ thuộc G-07.
- **Rủi ro:** khóa nhiều Room Type có thể gây deadlock nếu mỗi loại khóa theo thứ tự riêng; phải khóa một lần theo thứ tự id toàn cục. Khóa cả loại phòng làm tăng tranh chấp khi lưu lượng cao (chấp nhận được với một khách sạn/instance).

### G-11 Base Rate

- **Trạng thái:** Có một phần.
- **Hiện có:** cột `room_type.base_price NUMERIC(19,6)` nullable từ V1. Getter `getBasePrice()` **không được dùng ở đâu** (grep), `RoomType.create/updateProfile` không đặt nó, không có màn hình sửa. Giá trên reservation do nhân viên nhập từng phòng (`RoomRequest.nightlyRate`).
- **Cần bổ sung:** quyết định dùng lại cột `base_price` hay bảng giá mới (có tiền tệ), quản trị (quyền, audit), validation theo `SupportedCurrency` (VND scale 0), dịch vụ tính giá hiệu lực.
- **Ưu tiên/dependency:** P1; phụ thuộc D-02.
- **Rủi ro:** `base_price` hiện toàn NULL ở dữ liệu hiện hữu; Room Type không có giá không được hiển thị giá, không được bán qua web.

### G-12 Daily Rate Override

- **Trạng thái:** Chưa có (không có bảng, entity hay dịch vụ).
- **Cần bổ sung:** bảng `(room_type, ngày) → giá` duy nhất một giá hiệu lực mỗi cặp (đặc tả §6: không xây rule engine nhiều tầng), quy tắc xung đột xác định, audit.
- **Ưu tiên/dependency:** P1; phụ thuộc G-11, **F-01**.
- **Rủi ro:** xem F-01. Không xây bảng nếu chưa chốt cách lưu snapshot theo đêm.

### G-13 Room Rate Calendar

- **Trạng thái:** Chưa có.
- **Cần bổ sung:** màn hình lịch giá nhân viên, sửa một ngày, bulk theo khoảng ngày (gồm hai đầu) + lọc thứ, **preview** số ngày và giá cũ/mới, xác nhận, lưu lịch sử (actor/time/thay đổi). Không hồi tố lên reservation đã xác nhận (AC-09).
- **Ưu tiên/dependency:** P2; phụ thuộc G-12. Tái dùng `AuditLog`/mẫu màn hình danh mục.
- **Rủi ro:** thao tác bulk lớn (khoảng ngày dài) cần giới hạn và transaction; hai nhân viên cùng sửa cần khóa lạc quan để không ghi đè lặng lẽ.

### G-14 Snapshot giá, tiền tệ, thuế/phí

- **Trạng thái:** Có một phần.
- **Hiện có:** `reservation_room.nightly_rate` + `total_amount = nightlyRate × số đêm` (`ReservationRoom` constructor), `reservation.total_amount`, `reservation.currency` (CHECK `VND`/`USD`, nhưng `ReservationService.validateDraftData` ép **chỉ VND**). Khi check-in, `createRoomCharges` tạo **một Charge cho mỗi `ReservationRoom`** với số lượng = số đêm, đơn giá = `nightlyRate`. `getNightlyRate()` được dùng bởi `Charge`, `StayExtensionService`, `RoomChangeService`, `ReservationRoomReassignmentService`, `CheckInService`, `StayRoomAssignmentQueryService`, mapper.
- **Thiếu:** không biểu diễn được giá khác nhau theo đêm; không có thuế/phí; tiền tệ cố định VND, không có cấu hình tiền tệ khách sạn.
- **Cần bổ sung:** snapshot theo đêm (xem F-01), hiển thị tổng/tiền tệ do PMS trả; mô hình thuế/phí **không suy đoán** — V2A cần quyết định hiển thị (đã gồm hay chưa gồm thuế) trước khi làm UI.
- **Ưu tiên/dependency:** P1; **BLOCKER F-01**.
- **Rủi ro:** thay đổi lõi folio/charge/reconciliation (V33) ảnh hưởng báo cáo tài chính; yêu cầu hồi quy rộng.

### G-15 Quote

- **Trạng thái:** Chưa có.
- **Cần bổ sung:** báo giá có định danh/thời hạn hoặc cơ chế kiểm tra lại tương đương; PMS tính lại tại điểm ghi và trả `PRICE_CHANGED` kèm giá mới. Khuyến nghị token ký (HMAC) chứa tiêu chí + giá + chế độ + phiên bản chính sách + hạn, để tránh tạo bảng quote và vẫn phát hiện thay đổi; chi tiết cần chốt ở hợp đồng.
- **Ưu tiên/dependency:** P1; phụ thuộc G-08, G-11–G-14.
- **Rủi ro:** quote hết hạn giữa review và submit; mode đổi giữa chừng (§7.1) phải được mã hóa vào quote để phát hiện.

### G-16 Instant Booking

- **Trạng thái:** Có một phần.
- **Hiện có:** `create` → `DRAFT`; `confirm` → `CONFIRMED` với kiểm tra tồn + sức chứa dưới khóa; audit `CREATE`/`CONFIRM`; số reservation sinh tự động (`ReservationNumberGenerator`).
- **Thiếu:** đường công khai nguyên tử (create+gán+confirm một transaction), chế độ Instant theo cấu hình, kết quả trả theo union, outbox email, snapshot chính sách hủy.
- **Ưu tiên/dependency:** P1; phụ thuộc G-06, G-07, G-09, G-10, G-14, G-15, G-21, G-22.
- **Rủi ro:** ngày check-in quá khứ **không được kiểm tra** ở `create/confirm` (Issue #90 mở); đường công khai phải tự chặn ngày quá khứ theo ngày khách sạn. Số reservation lấy ngày theo `CURRENT_DATE` của phiên CSDL (Issue #89 mở), có thể lệch ngày so với giờ khách sạn; ảnh hưởng mã hiển thị cho khách.

### G-17 Booking Request lifecycle

- **Trạng thái:** Chưa có.
- **Hiện có:** `ReservationStatus` chỉ gồm `DRAFT, CONFIRMED, CANCELLED, NO_SHOW, CHECKED_IN, CHECKED_OUT`. `DRAFT` không giữ tồn, không hết hạn, và là trạng thái nhân viên dùng cho bản nháp bỏ dở. Nhân viên "duyệt" = `confirm()`, "từ chối" = `cancel()` với mã lý do; không có thông báo, hết hạn, hay màn hình hàng đợi. `ReservationStatus` được tham chiếu ở 47 file Java, `ReservationStatus.DRAFT` ở 15 chỗ; đặc tả danh sách V2 §1.2 có test bảo vệ buộc mỗi trạng thái mới phải gán vào một tab.
- **Đặc tả:** "không mặc định đồng nhất `DRAFT` với pending request" (§7.1, §12.1).
- **Cần bổ sung:** quyết định trạng thái (D-01), giữ tồn hay không, hết hạn, duyệt/từ chối, hàng đợi cho nhân viên, email, hiệu ứng lên danh sách/board/báo cáo.
- **Ưu tiên/dependency:** P2; **BLOCKER F-02**.
- **Rủi ro:** thêm enum value ảnh hưởng các `switch` và báo cáo; dùng `DRAFT` tái sử dụng thì lẫn với nháp của nhân viên và Walk-in.

### G-18 Manage Booking + xác thực email

- **Trạng thái:** Chưa có.
- **Hiện có:** `guest.email`, `reservation.booking_contact_email` (V36), `GuestEmail` validator. Không có bảng token, không có tra cứu theo (mã tham chiếu, email), không có phản hồi chống enumeration. Số reservation dạng `R yyyyMMdd-NNNNNN` (xem Issue #89), **đoán được**, nên không đủ làm bằng chứng truy cập (đúng như [tài liệu bảo mật](../security/README.md) yêu cầu).
- **Cần bổ sung:** yêu cầu truy cập (phản hồi giống nhau dù khớp hay không, thời gian xấp xỉ nhau), token ngẫu nhiên ≥128 bit lưu hash, TTL ngắn, dùng một lần, trao đổi lấy phiên quản lý ngắn hạn, rate limit theo IP/mã/email, `noindex`, token không vào log/analytics/referrer.
- **Ưu tiên/dependency:** P2; phụ thuộc G-21 (email), G-23.
- **Rủi ro:** spam email tới địa chỉ người khác; brute force mã tham chiếu; rò rỉ token qua Referer. Cần `Referrer-Policy` và chuyển token sang POST/fragment.

### G-19 Chính sách hủy

- **Trạng thái:** Chưa có.
- **Hiện có:** `hotel_setting.check_in_time` (14:00) và `time_zone`. Không có cửa sổ hủy, không có snapshot chính sách trên reservation (`grep cancellationPolicy` rỗng), pms-0040 loại trừ rõ "chính sách hủy".
- **Cần bổ sung:** cấu hình 24/48/72h (mặc định 24h), snapshot chính sách + văn bản áp dụng tại thời điểm tạo reservation (để thay đổi sau không bất lợi cho booking cũ — AC-18), nội dung trang Cancellation Policy.
- **Ưu tiên/dependency:** P2; phụ thuộc G-01, F-09.
- **Rủi ro:** cutoff phụ thuộc múi giờ: xem F-09.

### G-20 Giao dịch tự hủy của khách

- **Trạng thái:** Có một phần.
- **Hiện có:** `ReservationService.cancel` (khóa reservation, chỉ từ `DRAFT`/`CONFIRMED`, yêu cầu lý do, chặn nếu còn prepayment active, audit). Inventory giải phóng ngầm vì tồn tính từ trạng thái `CONFIRMED`. Mã lý do `GUEST_REQUEST` đã có.
- **Thiếu:** cổng công khai, kiểm tra cutoff, **idempotency** (gọi hủy lần hai hiện ném 409 `Invalid reservation state transition`), actor hệ thống, email xác nhận, quy tắc rút Request đang chờ.
- **Ưu tiên/dependency:** P2; phụ thuộc G-18, G-19, G-21.
- **Rủi ro:** nhân viên có thể thu prepayment (Payment nội bộ) cho booking web; hủy phải bị chặn và hướng dẫn liên hệ khách sạn (đặc tả: không có hoàn tiền tự động).

### G-21 Email giao dịch

- **Trạng thái:** Chưa có.
- **Hiện có:** `pom.xml` không có `spring-boot-starter-mail`; không `JavaMailSender`, không `spring.mail`, không `@Scheduled`/`@EnableScheduling`, không bảng outbox. Chỉ có địa chỉ email lưu trữ. Issue #33 (PMS-0037B Email) là PROPOSED, chưa Ready — có thể chồng lấn phạm vi.
- **Cần bổ sung:** nhà cung cấp + sender đã xác minh, outbox trong cùng transaction, dispatcher có retry/backoff, template (vi/en), sự kiện (Confirmed, Request received, Approved/Rejected, Cancelled, Manage link), theo dõi lỗi gửi, không log PII.
- **Ưu tiên/dependency:** P1 (tối thiểu cho Confirmed + Manage link); cần quyết định D-08.
- **Rủi ro:** lỗi gửi mail không được đảo ngược reservation (đặc tả §10); trùng nỗ lực với #33; địa chỉ gửi/SPF/DKIM thuộc vận hành.

### G-22 Idempotency và đối soát

- **Trạng thái:** Có một phần.
- **Hiện có:** mẫu đã được kiểm chứng ở Payment (V65 `payment_idempotency_record`: khóa 16–64 ký tự, fingerprint SHA-256, TTL 30 ngày, `UNIQUE (reservation_id, request_key)`) và Charge (V63). `PaymentIdempotencyService.findEarlierPayment` phân biệt replay (cùng fingerprint) với conflict (khác fingerprint).
- **Thiếu:** không có cho tạo reservation (chưa có reservation để gắn khóa nên không dùng được nguyên bảng cũ), không có trạng thái "đang xử lý", không có thao tác đối soát khi timeout.
- **Cần bổ sung:** bảng ghi nhận yêu cầu công khai theo khóa, `UNIQUE(key)`, lưu fingerprint + kết quả, truy vấn đối soát kết quả theo khóa với đầu ra tối thiểu, hết hạn/dọn dẹp.
- **Ưu tiên/dependency:** P1; phụ thuộc G-06.
- **Rủi ro:** khóa đối soát là bí mật tương đương mật khẩu ngắn hạn: cần entropy đủ cao và không lộ PII khi tra theo khóa; race hai request cùng khóa phải được `UNIQUE` hóa ở DB.

### G-23 Bảo mật API công khai

- **Trạng thái:** Có một phần.
- **Hiện có:** `LoginFailureTracker`/`ThrottledAuthenticationProvider` (trong bộ nhớ, một instance), `ClientIpResolver` (tin `X-Forwarded-For` chỉ từ proxy cấu hình), `LoginRequestSizeFilter`, `AuditLogRedactionPolicy`, `use-relative-redirects`, cookie session `HttpOnly/SameSite=lax`, chỉ thị không tin `X-Forwarded-*` cho redirect.
- **Thiếu:** rate limit cho tìm kiếm/quote/submit/tra cứu, CORS (không cần nếu chỉ Next.js server gọi), phản hồi chống enumeration, giới hạn body cho API công khai, mã lỗi có cấu trúc, request-id/correlation, quy tắc log PII cho đường mới.
- **Cần bổ sung:** filter giới hạn tần suất tái dùng `ClientIpResolver`; khóa giới hạn theo IP đầu cuối **do Next.js chuyển tiếp** (PMS chỉ thấy IP của Next.js) — cần chốt cách truyền IP khách tin cậy giữa hai tầng.
- **Ưu tiên/dependency:** P1; phụ thuộc G-06.
- **Rủi ro:** rate limit trong bộ nhớ mất khi restart (chấp nhận được ở V2A một instance, theo tiền lệ `LoginFailureTracker`); nếu PMS chạy nhiều instance cần lưu trữ chung.

### G-24 Domain, TLS, canonical

- **Trạng thái:** Chưa có (phía PMS/vận hành).
- **Hiện có:** `docs/operations/production-deployment.md` mô tả VPS với Docker Compose + Caddy, host staging `pmshotelvietnam.com`, `PMS_PUBLIC_URL`. Không có mô hình dữ liệu domain, không có xác minh quyền sở hữu domain, không có cấu hình website trong Compose. Đặc tả nêu ví dụ `hotelabc.pmshotelvietnam.com` là "subdomain PMS mặc định" nhưng không rõ là host của website hay của PMS (F-19).
- **Cần bổ sung:** runbook cấp subdomain/custom domain, xác minh DNS, cấp TLS, canonical origin cấu hình ở website (`CANONICAL_ORIGIN`), alias redirect; PMS lưu trạng thái domain nếu muốn hiển thị trên Website Settings.
- **Ưu tiên/dependency:** P3; phụ thuộc quyết định vận hành (D-10).
- **Rủi ro:** cấp nhầm chứng chỉ/trỏ nhầm instance; host header giả mạo (đã có quy tắc: chỉ đối chiếu allowlist).

### G-25 Múi giờ, giờ check-in, quy tắc đặt

- **Trạng thái:** Có một phần.
- **Hiện có:** `hotel_setting.time_zone` được lưu và validate, nhưng **không được áp dụng**: `DashboardClockConfiguration` cố định `Asia/Ho_Chi_Minh`; `pms-0040` PO-2 và nhãn UI nói rõ "stored and validated but not yet applied". Giờ check-in tiêu chuẩn chỉ mang tính thông tin trong V1 (không ảnh hưởng deadline).
- **Thiếu:** quy tắc đặt phòng do PMS cung cấp (đặc tả §7 bước 1): không có min/max stay, cửa sổ đặt trước, số phòng/khách tối đa mỗi lần đặt, cut-off cùng ngày. `ReservationService.validateDraftData` chỉ kiểm tra `checkOut > checkIn`.
- **Cần bổ sung:** hàm thời gian khách sạn dùng cho cutoff hủy và kiểm tra ngày quá khứ (AC-19); cấu hình/giá trị mặc định cho giới hạn đặt (D-11).
- **Ưu tiên/dependency:** P1; phụ thuộc D-06, D-11.
- **Rủi ro:** nếu một khách sạn đặt múi giờ khác `Asia/Ho_Chi_Minh`, cutoff và "ngày hôm nay" sẽ lệch mà không có cảnh báo (F-09).

### G-26 Ứng dụng frontend

- **Trạng thái:** Chưa có.
- **Hiện có:** chỉ tài liệu và cấu hình agent. Chưa có Next.js app, TypeScript, Tailwind, ESLint, test runner, Docker, biến môi trường thật ([environment-variables.md](../development/environment-variables.md) nói rõ repo chưa có ứng dụng).
- **Cần bổ sung:** toàn bộ ứng dụng theo [client-architecture.md](../coding-rules/client-architecture.md): `app/`, `features/{website-config,rooms,booking}`, `templates/modern-tropical`, `components/ui`, `lib/pms` (server-only), Route Handler cùng origin, schema runtime, fixture, test, CI.
- **Ưu tiên/dependency:** P0 (khởi tạo), các phần tích hợp phụ thuộc hợp đồng G-06.
- **Rủi ro:** các lệnh lint/typecheck/build/test chưa tồn tại; các quy tắc "lấy lệnh từ cấu hình thực tế" chưa áp dụng được cho tới khi scaffold.

### G-27 Mockup và asset thiết kế

- **Trạng thái:** Chưa có.
- **Hiện có:** `docs/designs/{mockups,references,screenshots}` chỉ chứa `.gitkeep`. Đặc tả tham chiếu "hình ảnh thiết kế đã duyệt" và Sunset House nhưng không có file.
- **Cần bổ sung:** mockup đã duyệt theo màn hình/viewport (home, catalog, detail, search/select, guest info, review, result, manage booking, các trạng thái rỗng/lỗi), tài liệu ghi nguồn/quyền.
- **Ưu tiên/dependency:** P0 cho phần giao diện; không chặn phần logic/adapter.
- **Rủi ro:** dựng giao diện không có chuẩn đối chiếu → làm lại; [design README](../designs/README.md) yêu cầu đối chiếu mockup đã duyệt.

### G-28 Governance

- **Trạng thái:** Chưa có.
- **Hiện có:** Issue #30 (Backlog, OPEN) ghi "Cần làm rõ" tiêu chí chấp nhận và "KHÔNG mặc định luồng checkout/thanh toán hoàn chỉnh"; `technical-spec-v1` là nguồn nghiệp vụ PMS và `AGENTS.md` PMS cấm suy diễn hành vi không có trong đặc tả; `docs/requirements/technical-spec-v2.md` hiện chỉ có mục #52 và #36; workflow PMS: "no implementation starts on an Issue the Product Owner has not approved", một Issue hoàn tất là một commit tập trung; `architecture.md` PMS cấm tạo business domain/package mới (chỉ `customer`, `booking`, `room`; `reservation`, `payment`… bị nêu cụ thể là cần phê duyệt).
- **Cần bổ sung:** xem F-04 và mục 7.
- **Ưu tiên/dependency:** P0.
- **Rủi ro:** làm backend trước khi có đặc tả/Issue Ready vi phạm quy trình PMS và có nguy cơ bị yêu cầu làm lại.

## 5. Ma trận phủ tiêu chí nghiệm thu AC-01…AC-22

| AC | Tình huống | Gap chính cần đóng |
| --- | --- | --- |
| AC-01 | Domain/branding/template/HTTPS/canonical | G-01, G-03, G-24, G-26 |
| AC-02 | Room Type ẩn/inactive không hiện, không book được | G-04 (+ kiểm tra backend ở G-16) |
| AC-03 | Thứ tự phòng/ảnh, đổi cover | G-02, G-04, G-05 |
| AC-04 | Hết tồn → Sold Out | G-08 |
| AC-05 | 3 adults + 1 child, 2 Double | G-09 (**F-06**) |
| AC-06 | Nhiều Room Type, một reservation | G-07 |
| AC-07 | Một loại thiếu tồn → không tạo một phần | G-07, G-10 |
| AC-08 | Base + override, checkout không tính thêm đêm | G-11, G-12, G-14 (**F-01**) |
| AC-09 | Bulk Fri/Sat, preview, audit, không đổi giá cũ | G-13, G-14 |
| AC-10 | Instant: chỉ Confirmed khi PMS xác nhận, gửi email | G-16, G-21 |
| AC-11 | Request: Pending, không khẳng định confirmed | G-17 (**F-02**) |
| AC-12 | Mode/giá đổi lúc submit → xác nhận lại | G-15, G-16, G-17 |
| AC-13 | Hai khách tranh phòng cuối | G-10 |
| AC-14 | Submit lặp/timeout | G-22 |
| AC-15 | Tra cứu chưa xác thực không lộ dữ liệu | G-18, G-23 |
| AC-16 | Hủy trước cutoff đúng một lần, có email | G-19, G-20, G-21 |
| AC-17 | Quá cutoff/trạng thái sai bị chặn | G-19, G-20 |
| AC-18 | Đổi chính sách không bất lợi cho booking cũ | G-19 (snapshot) |
| AC-19 | Trình duyệt khác múi giờ | G-25 (**F-09**) |
| AC-20 | PMS lỗi/mất ảnh/thiếu dữ liệu tùy chọn | G-03, G-05, G-26 |
| AC-21 | Mobile/tablet/desktop/bàn phím | G-26, G-27 |
| AC-22 | Không thanh toán online | Giữ nguyên: không thêm payment vào đường công khai; ràng buộc ở G-20 (prepayment nhân viên) |

Lưu ý: [acceptance-criteria.md](../requirements/acceptance-criteria.md) dùng mã `WEB-01…10`, còn đặc tả dùng `AC-01…22`; hai bộ chưa được ánh xạ (F-15).

## 6. Findings

### 6.1 BLOCKER

#### F-01 Giá theo đêm không biểu diễn được trong schema và folio hiện tại

- **Bằng chứng:** `reservation_room.nightly_rate` (một giá) và `total_amount` (V1); `ReservationRoom` constructor tính `nightlyRate × số đêm`; `ReservationService.createRoomCharges` tạo một `Charge` mỗi `ReservationRoom` với `quantity = số đêm`; `getNightlyRate()` được dùng ở 11 file (Charge, StayExtension, RoomChange, Reassignment, CheckIn, mapper…). UNIQUE `(reservation_id, room_id)` chặn việc tách một phòng thành nhiều dòng theo đoạn giá.
- **Xung đột đặc tả:** §6 yêu cầu giá hiệu lực từng đêm = override hoặc base và snapshot "từng đêm và reservation"; AC-08/09.
- **Phương án:**
  - **A (đề xuất):** thêm bảng snapshot theo đêm cho mỗi `reservation_room` (ngày, giá); `total_amount` = tổng; `nightly_rate` giữ nghĩa "giá cơ sở/đại diện" cho dữ liệu cũ. `createRoomCharges` gộp các đêm liên tiếp cùng giá thành một Charge. Phải rà `StayExtension`/`RoomChange`/`Reassignment`/báo cáo.
  - **B:** lưu `nightly_rate` = trung bình có làm tròn. Rẻ nhưng sai số làm tổng ≠ tổng giá PMS đã báo (mâu thuẫn AC-08), **không khuyến nghị**.
  - **C:** bỏ override theo đêm trong V2A (chỉ base rate). Thay đổi phạm vi đã chốt, **không được tự quyết**.
- **Cần:** quyết định D-02 và một spike thiết kế trước khi code.

#### F-02 Vòng đời Request chưa định nghĩa

- **Bằng chứng:** `ReservationStatus` (6 giá trị), `DRAFT` không giữ tồn và không hết hạn (`ReservationService.requireBookableRooms` comment), không có từ chối/hết hạn/thông báo; Issue #30 ghi vòng đời cần chi tiết riêng.
- **Phương án:** (A, đề xuất) thêm trạng thái riêng, không giữ tồn trong V2A, có hạn xử lý, duyệt = kiểm tra lại tồn/sức chứa dưới khóa rồi `CONFIRMED`, từ chối/hết hạn = trạng thái cuối; (B) tái dùng `DRAFT` kèm nguồn `HOTEL_WEBSITE` và cờ phụ — nhanh nhưng lẫn với nháp nhân viên và vi phạm khuyến cáo của đặc tả; (C) Request giữ tồn trong thời hạn — chính xác hơn nhưng đổi truy vấn tồn lõi.
- **Cần:** D-01.

#### F-03 Chưa có hợp đồng public API và xác thực service-to-service

- **Bằng chứng:** `SecurityConfig` (chain `/api/**` chỉ JWT người dùng, đăng nhập mật khẩu, hạn 8h); không có API key/service account; không có `/api/public`; [api-integration.md](../development/api-integration.md) xác nhận hợp đồng chưa chốt.
- **Phương án xác thực:** (A) API key riêng có phạm vi hẹp, gửi qua header, lưu hash, quay vòng được; (B) tài khoản dịch vụ dùng JWT hiện có (tái dùng RolePermissionPolicy nhưng dựa mật khẩu + throttle đăng nhập, hạn 8h); (C) mTLS/IP allowlist ở Caddy làm lớp bổ sung. Đề xuất A + C (PMS không công khai trực tiếp ra Internet cho phần public API, chỉ Next.js truy cập qua mạng riêng).
- **Cần:** D-05, rồi GOV-04 (hợp đồng).

#### F-04 Governance và quy trình PMS

- **Bằng chứng:** Issue #30 Backlog, tiêu chí "chưa xác định"; `technical-spec-v2.md` chưa có PMS-0034; `AGENTS.md` PMS: không suy diễn nghiệp vụ ngoài đặc tả; `architecture.md` PMS mục 14–15: không tạo business domain/package mới khi chưa phê duyệt; workflow: một Issue một commit, chỉ PO chuyển Ready.
- **Phương án:** PO đưa tiêu chí AC-01…22 vào Issue; thêm mục PMS-0034 vào `technical-spec-v2.md` (hoặc tài liệu đặc tả riêng) làm nguồn cho PMS; sửa `pms-0040` mục "Ngoài phạm vi" nếu mở rộng Hotel Settings; chọn đặt gói mới trong các domain sẵn có (`room`, `booking`, `common`) hoặc xin phê duyệt domain mới; tách Issue #30 thành các Issue con theo kế hoạch.
- **Cần:** D-07.

#### F-05 Không có mockup đã duyệt

- **Bằng chứng:** `docs/designs/{mockups,references,screenshots}` chỉ `.gitkeep`; [design README](../designs/README.md) quy định đối chiếu mockup đã duyệt.
- **Phương án:** PO cung cấp/duyệt mockup và đặt vào `docs/designs/mockups/`; phần adapter/logic có thể chạy trước bằng fixture.
- **Cần:** D-09.

### 6.2 MAJOR

#### F-06 Quy tắc sức chứa mâu thuẫn

- **PMS V1:** sức chứa là **người lớn**, trẻ em không chiếm chỗ (`technical-spec-v1` mục 66.2; `AdultCapacityRules`). **Client spec:** `adults + children` tính vào sức chứa.
- **Phương án:** (A, đề xuất) quy tắc mới chỉ cho đường công khai: `adults ≥ 1` và `adults + children ≤ Σ capacity` (không ràng buộc bổ sung theo phòng, đúng §5.1); giữ nguyên `AdultCapacityRules` cho nhân viên. (B) đổi quy tắc toàn hệ thống — phải sửa V1 spec 66.2 và hồi quy check-in/đổi phòng/quản lý Room Type.
- **Lưu ý:** khác biệt dẫn tới hai định nghĩa "sức chứa" trong cùng sản phẩm; cần ghi rõ trong đặc tả PMS. **Cần:** D-03.

#### F-07 Gallery gắn theo Room vật lý, không theo Room Type

- **Bằng chứng:** V38 `room_image.room_id`; `RoomImageService`. Đặc tả §4: ảnh theo Room Type, không theo Room vật lý.
- **Phương án:** bảng mới theo Room Type, tái dùng hạ tầng xử lý ảnh, không migrate dữ liệu cũ; giữ `room_image` cho giao diện nhân viên.
- **Rủi ro:** tên gần giống gây nhầm; cần UX tách bạch.

#### F-08 Reservation đòi phòng vật lý, Guest và người dùng

- **Bằng chứng:** `reservation_room.room_id NOT NULL`; `reservation.guest_id NOT NULL`; `created_by/updated_by NOT NULL REFERENCES app_user`; `audit_log.user_id NOT NULL`; `CurrentActor.id()` 401 nếu chưa đăng nhập; `GuestService` không khử trùng.
- **Phương án gán phòng:** (A, đề xuất) gán tại thời điểm đặt (tự động, trong khóa), nhân viên đổi lại được trước check-in; (B) giữ tồn theo loại, gán muộn — cần `room_id` nullable + `room_type_id`, đổi truy vấn tồn, board, check-in, charge.
- **Phương án Guest:** luôn tạo Guest mới (đơn giản, trùng lặp) hoặc khớp theo email+SĐT chuẩn hóa (giảm trùng, có nguy cơ gộp nhầm/lộ dữ liệu). Mặc định đề xuất luôn tạo mới và để nhân viên gộp (chưa có công cụ gộp: ngoài phạm vi).
- **Phương án actor:** tài khoản hệ thống chuyên dụng (`app_user`) không đăng nhập được; audit ghi rõ nguồn `HOTEL_WEBSITE`. **Cần:** D-04 và D-05.

#### F-09 Múi giờ khách sạn chưa được áp dụng

- **Bằng chứng:** `DashboardClockConfiguration` cứng `Asia/Ho_Chi_Minh`; `pms-0040` PO-2/mục 17 và nhãn UI xác nhận. Đặc tả client: ngày lưu trú, giá đêm và cutoff theo múi giờ khách sạn, không theo trình duyệt (§7, §8.2, AC-19).
- **Phương án:** (A, đề xuất cho V2A) dùng đúng `Clock` hiện tại làm "giờ khách sạn", thêm kiểm tra khởi động/vận hành cảnh báo nếu `hotel_setting.time_zone` khác `Clock` zone, và ghi nhận giới hạn; (B) áp dụng `hotel_setting.time_zone` cho `Clock` — đổi hành vi V1 (PO-2 cấm), cần quyết định riêng.
- **Cần:** D-06.

#### F-10 Không có hạ tầng email, outbox, scheduler

- **Bằng chứng:** grep rỗng; `pom.xml` không có mail starter; Issue #33 (0037B) chưa Ready.
- **Phương án:** xây outbox tối thiểu cho #30 và để #33 mở rộng sau; hoặc đợi/hợp nhất với #33. Cần nhà cung cấp, sender đã xác minh, SPF/DKIM.
- **Cần:** D-08.

#### F-11 Chưa có rate limit, chống enumeration; mã tham chiếu đoán được

- **Bằng chứng:** không có `RateLimit`/`Cors`/`Bucket`; số reservation tuần tự theo ngày (`reservation_number_sequence`, 6 chữ số); Issue #89 (ngày lấy theo `CURRENT_DATE` của DB).
- **Phương án:** rate limit ở PMS (theo IP đầu cuối do Next.js chuyển tiếp qua kênh tin cậy) và ở Next.js; phản hồi đồng nhất cho tra cứu; coi số reservation chỉ là mã hiển thị, quyền truy cập dựa token email. Cân nhắc có sửa #89 trong phạm vi hay chỉ ghi nhận.

#### F-12 Tồn phòng: DRAFT không giữ, không theo loại, bỏ qua kỳ bảo trì tương lai, thiếu quy tắc ngày

- **Bằng chứng:** `findRoomIdsWithInventoryConflict` (chỉ `reservation_room` của `CONFIRMED` và `stay_room_assignment` của stay `CHECKED_IN`); `Room.isBookableInventory()` theo trạng thái hiện tại; `room_inventory_period` (V28) không nằm trong truy vấn; Issue #90 (ngày nhận quá khứ không được kiểm tra).
- **Phương án:** dịch vụ tồn theo loại tái dùng truy vấn hiện có; đường công khai tự kiểm tra ngày quá khứ theo ngày khách sạn; **xác minh với PO** ý định về kỳ bảo trì tương lai trước khi coi là lỗi.

#### F-13 Phạm vi Hotel Settings bị loại trừ trong đặc tả PMS

- **Bằng chứng:** `pms-0040-hotel-settings.md` mục "Ngoài phạm vi": chính sách hủy, API REST cho cài đặt, màu chủ đề, multi-tenant, đổi `Clock`.
- **Phương án:** coi nội dung website là module mới (không sửa hành vi PMS-0040), cập nhật các ghi chú loại trừ có liên quan sau khi PO duyệt.

#### F-14 Mô hình xuất bản và làm mới cache chưa có

- **Bằng chứng:** G-02. Kiến trúc client nói "cache có thời hạn hoặc cơ chế làm mới" nhưng chưa chọn; PMS chưa có tín hiệu.
- **Phương án:** `contentVersion` hoặc ETag trên mọi DTO công bố + webhook revalidate có bí mật; nhược điểm webhook: cần PMS biết URL Next.js (cấu hình theo instance).

### 6.3 MINOR

| ID | Nội dung | Đề xuất |
| --- | --- | --- |
| F-15 | Tài liệu website lệch nhau: yêu cầu trỏ `docs/architecture/client-architecture.md` (thực tế `docs/coding-rules/`); [acceptance-criteria.md](../requirements/acceptance-criteria.md) dùng `WEB-xx` và liên kết tới anchor không còn (`#tiêu-chí-nghiệm-thu-đề-xuất`, `#quyết-định-đã-chốt`); nhiều tài liệu vẫn mô tả hợp đồng "chưa chốt" dù đặc tả đã duyệt; tài liệu gọi PMS là `hotel-system` nhưng thư mục local là `hotel`; `README.md` chỉ một dòng | Cập nhật khi viết hợp đồng; thêm bảng ánh xạ AC↔WEB; không sửa trong audit này |
| F-16 | `booking_source` chỉ seed `DIRECT, AGODA, BOOKING_COM, AIRBNB`; `HOTEL_WEBSITE` có trong catalog (V58) nhưng không active mặc định; category `OTHER` (không phải `DIRECT`) ảnh hưởng báo cáo theo nguồn | Migration seed nguồn `HOTEL_WEBSITE`; PO xác nhận category báo cáo |
| F-17 | `Reservation.cancel` ném 409 khi đã `CANCELLED`, trái yêu cầu idempotent của đặc tả §8.2 | Đường tự hủy xử lý "đã hủy" như thành công idempotent, không đổi `cancel()` của nhân viên |
| F-18 | Chỉ VND; không có thuế/phí; đặc tả cấm tự giả định | V2A hiển thị tổng do PMS trả kèm nhãn do PO/pháp lý chọn (D-11); không tự tính |
| F-19 | Ví dụ `hotelabc.pmshotelvietnam.com` là "subdomain PMS mặc định": không rõ host của website hay PMS | Chốt: website một subdomain/instance, PMS ở host riêng nội bộ (D-10) |
| F-20 | Route `/branding/*` public, `Cache-Control: private` | Website lấy qua server hoặc `next/image` với allowlist; giữ tách khỏi hợp đồng JSON |
| F-21 | `CreateRequest`/`RoomRequest` nhận `nightlyRate`, `source`, `currency`, `roomId` từ client | DTO công khai riêng; giá luôn do PMS tính |

## 7. Quyết định cần xác nhận

Chia hai nhóm: **chặn bắt đầu** (cần trước khi code vùng liên quan) và **có mặc định đề xuất** (không chặn, có thể xác nhận sau).

### 7.1 Chặn bắt đầu

| ID | Câu hỏi | Đề xuất | Chặn |
| --- | --- | --- | --- |
| D-01 | Vòng đời Request: trạng thái riêng hay tái dùng `DRAFT`; có giữ tồn không; hạn xử lý; ai duyệt/từ chối; rút yêu cầu | Trạng thái riêng, không giữ tồn, có hạn xử lý do PO chọn; duyệt kiểm tra lại dưới khóa | G-17, AC-11/12 |
| D-02 | Cách lưu giá theo đêm (F-01) | Snapshot theo đêm cho từng phòng + gộp Charge theo đoạn giá | G-12–G-14, AC-08/09 |
| D-03 | Phạm vi quy tắc sức chứa mới (F-06) | Chỉ đường công khai; giữ quy tắc người lớn của nhân viên | G-09, AC-05 |
| D-04 | Thời điểm gán phòng vật lý (F-08) | Gán tự động lúc đặt | G-07 |
| D-05 | Xác thực Next.js → PMS và cách phơi bày API công khai (F-03) | API key phạm vi hẹp + mạng riêng/Caddy; IP khách chuyển tiếp qua kênh tin cậy | G-06 |
| D-06 | Múi giờ/cutoff (F-09) | Dùng `Clock` hiện tại; cảnh báo nếu `time_zone` khác | G-19, G-25, AC-19 |
| D-07 | Cách tách Issue #30, đặc tả PMS-0034, vị trí gói mã (F-04) | Epic + Issue con theo kế hoạch; đặc tả trong `technical-spec-v2.md`; đặt mã trong `room`/`booking`/`common`, xin duyệt cho tên gói mới nếu cần | toàn bộ BE |
| D-08 | Email: tự xây outbox tối thiểu hay hợp nhất với #33; nhà cung cấp, sender | Outbox tối thiểu trong #30, mở rộng ở #33 | G-21, AC-10/16 |
| D-09 | Mockup đã duyệt (F-05) | PO cung cấp trước FE giao diện | G-27 |

**Cập nhật 2026-11-10 — D-01…D-09 đã được Product Owner chốt.** Nội dung quyết định và hệ quả ghi tại `hotel-website/docs/decisions/004`…`012` (ADR) và tại `hotel-system/docs/specs/PMS-0034-hotel-website-direct-booking-v2a.md` mục 2; xem [mục 10](#10-cập-nhật-m0--quyết-định-đã-chốt-2026-11-10) của tài liệu này để biết chi tiết từng ID. D-06 được chốt với phạm vi hẹp hơn đề xuất ban đầu ở bảng trên (áp dụng `hotel_setting.time_zone` cho quy tắc mới của Issue #30, không đổi `Clock` dùng chung của PMS V1/Admin — xem ADR 009); các D khác khớp với đề xuất đã ghi ở cột "Đề xuất". D-07 chốt vị trí đặc tả là `hotel-system/docs/specs/` (khác đề xuất ban đầu `technical-spec-v2.md`).

### 7.2 Có mặc định, không chặn

| ID | Chủ đề | Mặc định đề xuất |
| --- | --- | --- |
| D-10 | Domain/custom domain do ai cấp, host website vs PMS (F-19, G-24) | Vận hành thủ công theo runbook ở V2A |
| D-11 | Giới hạn đặt: min/max stay, đặt trước, số phòng/khách tối đa mỗi lần, cut-off cùng ngày; nhãn thuế/phí (F-18) | Giá trị cấu hình hóa, mặc định do PO chốt; chưa chốt thì UI không hiển thị nhãn thuế |
| D-12 | Khử trùng Guest (F-08) | Luôn tạo mới |
| D-13 | Định dạng ảnh Room Type (JPEG/PNG như Room hay thêm WebP), số ảnh tối đa | Như Room hiện tại (JPEG/PNG, 5 MB), tối đa theo PO |
| D-14 | Kỳ bảo trì tương lai có chặn bán web không (F-12) | Xác minh ý định PO trước; chưa chặn |

## 8. Phần chưa xác minh

1. Hành vi đồng thời thực tế của `confirm` và đường đặt mới: chưa chạy test nào. Test đồng thời của `confirm` đã tồn tại nhưng chưa được chạy lại trong audit; đường đặt công khai chưa có test.
2. Múi giờ phiên JDBC/JVM của container (Issue #89); ảnh hưởng mã tham chiếu hiển thị.
3. Cấu hình Caddy, `forward-headers-strategy`, DNS/TLS thực tế ở staging (tài liệu ops ghi nhận khác biệt giữa quan sát và cấu hình).
4. Phần `technical-spec-v1.md` ngoài các đoạn đã grep: có thể còn ràng buộc liên quan reservation công khai chưa được thấy.
5. Mức tái sử dụng được của `RoomAvailabilityDashboardService` cho đếm theo loại (chưa đọc kỹ).
6. Hành vi báo cáo/Excel/PDF khi có nguồn `HOTEL_WEBSITE` và khi giá theo đêm thay đổi (phụ thuộc D-02).
7. Cấu hình hiện tại của `application-dev.yml`, seed demo, Playwright E2E của PMS.

## 9. Phụ lục: bằng chứng chính

| Chủ đề | Vị trí |
| --- | --- |
| Trạng thái reservation | `hotel:…/entity/booking/ReservationStatus.java` |
| Tạo/confirm/cancel | `hotel:…/service/booking/ReservationService.java` (`create` ~168, `validateDraftData` ~436, `confirm` ~546, `cancel` ~894, `createRoomCharges` ~1109) |
| Khóa phòng | `hotel:…/repository/room/RoomRepository.java` (`lockAllByRoomType` ~74, `findRoomIdsWithInventoryConflict` ~138) |
| Phòng bán được | `hotel:…/entity/room/Room.java` `isBookableInventory` ~150 |
| Sức chứa | `hotel:…/service/booking/AdultCapacityRules.java`; `docs/requirements/technical-spec-v1.md` mục 66.2 |
| Schema lõi | `hotel:src/main/resources/db/migration/V1, V5, V22, V28, V30, V36, V37, V38, V41, V42, V57, V58, V63–V66` |
| Idempotency mẫu | `V65__payment_idempotency_record.sql`; `…/service/booking/PaymentIdempotencyService.java` |
| Bảo mật | `hotel:…/config/SecurityConfig.java`; `…/security/{CurrentActor,ClientIpResolver,LoginFailureTracker}.java` |
| Branding | `hotel:…/controller/common/HotelBrandingController.java`; `V66` |
| Cài đặt khách sạn | `V64`; `docs/requirements/pms-0040-hotel-settings.md` |
| Múi giờ | `hotel:…/config/DashboardClockConfiguration.java` |
| Issue liên quan | #30, #33 (email), #35, #55 (CLOSED), #64 (Done), #89, #90 |

## 10. Cập nhật M0 — Quyết định đã chốt (2026-11-10)

Product Owner đã chốt D-01…D-09 (mục 7.1) theo yêu cầu của chủ repo, thực hiện M0 — Governance & Technical Design của Issue #30. Status/priority/release của PMS-0034 và phạm vi V2A không đổi (xem `hotel-system/docs/requirements/master-backlog.md` mục 6F). Việc chốt **không tự chuyển GOV-01 (Backlog → Ready)** — đó vẫn là quyết định riêng của Product Owner.

| ID | Quyết định đã chốt | ADR | Đặc tả kỹ thuật thực hiện |
| --- | --- | --- | --- |
| D-01 | Trạng thái Booking Request riêng, không giữ tồn, duyệt kiểm tra lại availability, hết hạn mặc định 24h, UI nêu rõ chưa đảm bảo. | [ADR 004](../decisions/004-booking-request-lifecycle.md) | `PMS-0034-hotel-website-direct-booking-v2a.md` mục 2, 4, 10; `PMS-0034-data-design.md` V73 |
| D-02 | Snapshot giá theo đêm; không hồi tố theo Rate Calendar; migration tương thích V1. | [ADR 005](../decisions/005-nightly-pricing-snapshot.md) | `PMS-0034-data-design.md` V70, V72 |
| D-03 | Public API tính sức chứa `adults+children`; không đổi hành vi V1. | [ADR 006](../decisions/006-public-occupancy-capacity.md) | `PMS-0034-hotel-website-direct-booking-v2a.md` mục 8 |
| D-04 | Instant gán phòng khi xác nhận; Request gán khi duyệt; transaction/locking/chống overbooking. | [ADR 007](../decisions/007-room-assignment-timing.md) | `PMS-0034-data-design.md` V71 |
| D-05 | Next.js→PMS qua server-side API client; xác thực service-to-service; không lộ credential/API nội bộ. | [ADR 008](../decisions/008-public-api-security.md) | `PMS-0034-public-api-contract.md` mục 1, 6, 7 |
| D-06 | `hotel_setting.time_zone` áp dụng cho quy tắc thời gian MỚI của Issue #30; không đổi `Clock` chung của PMS V1/Admin (phạm vi hẹp hơn đề xuất ban đầu — xem ADR 009 mục Hệ quả). | [ADR 009](../decisions/009-hotel-time-zone.md) | `PMS-0034-hotel-website-direct-booking-v2a.md` mục 10 |
| D-07 | Issue #30 là Epic, mốc M0–M5; đặc tả PMS-0034 tại `hotel-system/docs/specs/` (khác đề xuất `technical-spec-v2.md`); không domain mới. | [ADR 010](../decisions/010-issue-30-governance-architecture.md) | `PMS-0034-hotel-website-direct-booking-v2a.md` toàn bộ; mục 5 (không domain mới) |
| D-08 | Outbox email tối thiểu trong PMS, không phụ thuộc #33; provider/sender qua môi trường. | [ADR 011](../decisions/011-transactional-email-outbox.md) | `PMS-0034-data-design.md` V75 |
| D-09 | Modern Tropical – Bright & Fresh là template; mockup website + PMS Admin cần duyệt trước khi implement UI. | [ADR 012](../decisions/012-mockup-gate.md) | [`docs/designs/mockup-requirements-issue-30.md`](../designs/mockup-requirements-issue-30.md) |

Các đường dẫn `PMS-0034-*.md` nằm trong repo `hotel-system` tại `docs/specs/`. Nội dung schema/API/class cụ thể trong các tài liệu đó vẫn là **đề xuất kỹ thuật**, chưa qua Codex review — khác với các quyết định nghiệp vụ D-01…D-09 ở trên, vốn đã **Đã chốt**. Điểm còn mở cần xác nhận thêm trước khi code: xem `PMS-0034-hotel-website-direct-booking-v2a.md` mục 15 và `PMS-0034-public-api-contract.md` mục 7.

## 11. Cập nhật Codex Review 2026-11-11 — REQUEST CHANGES (2 BLOCKER, 9 MAJOR, 2 MINOR)

Codex review độc lập đối chiếu thiết kế mục 10 với source code thực tế, verdict **REQUEST CHANGES** (200/200 baseline test PASS tại thời điểm review). Tổng hợp đầy đủ finding → fix → tài liệu → GitHub Issue: [`PMS-0034-codex-review-resolution-2026-11-11.md`](../../../hotel/docs/specs/PMS-0034-codex-review-resolution-2026-11-11.md) (repo `hotel-system`).

**Tóm tắt BLOCKER (cả hai đã có thiết kế thay thế, RESOLVED IN DESIGN):**
- **BLOCKER-01** (tương thích tài chính): thiết kế D-02/ADR 005 bản đầu (`total_amount` = tổng theo đêm) vi phạm migration **V33** (`ux_charge_source_reservation_room`, một `reservation_room` chỉ một Charge gốc) và invariant `MonthlyFinancialReportService.verifySnapshot`. ~~Thiết kế Round 2: `nightly_rate` là giá trung bình phái sinh~~ — **bị loại ở Round 3** (làm tròn phá `verifySnapshot`, phân bổ sai khi vắt tháng/`CHECKED_IN`). Thiết kế hiện hành: chế độ giá `FLAT` (V1 nguyên vẹn)/`DAILY` (bảng `room_night_rate` là nguồn sự thật), không sửa V33 — xem mục 12.
- **BLOCKER-02** (Booking Request chưa gán phòng): thiết kế D-04/ADR 007 bản đầu (`reservation_room.room_id` nullable) được xác nhận sẽ gây NPE ở ≥4 điểm (`ReservationMapper`, `Reservation.findRoomLine`, `ReservationService.updateDraft`/`lockRoomsThenReservation`) và cần thêm cột `room_type_id`. Chọn phương án thay thế: bảng `booking_request_line` riêng, không tạo `ReservationRoom` nào trước khi duyệt — giữ nguyên 100% invariant `ReservationRoom` V1.

**9 MAJOR, 2 MINOR**: đã RESOLVED IN DESIGN hoặc đánh dấu NEEDS VALIDATION/BLOCKED theo đúng mức độ — chi tiết từng finding ở tài liệu resolution mục 4-5. Không finding nào được tự ý tuyên bố "đã giải quyết" nếu thiếu thiết kế cụ thể và bằng chứng đối chiếu source.

**4 quyết định Product Owner bổ sung (PO-01…PO-04)**: ghi tại ADR 013-016 (`docs/decisions/`), tóm tắt ở tài liệu resolution mục 1.

## 12. Cập nhật Codex Re-review #2 (Round 3) — 2026-11-11

Verdict re-review #2: REQUEST CHANGES (BLOCKER-01 nightly financial integrity còn mở, cùng các điểm lifecycle, API contract, idempotency, Manage Booking, chính sách hủy, outbox, Rate Calendar, migration/nguồn đặt phòng, truy vết). Thiết kế cuối và bằng chứng source: [`PMS-0034-codex-round3-remediation-2026-11-11.md`](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md); ADR đề xuất [017](../decisions/017-nightly-pricing-flat-and-daily-modes.md); plan mục 15. Nguyên văn báo cáo Codex #2 không có trong repo (xem lưu ý ở round 3 §0). Không finding nào ghi PASS; trạng thái là RESOLVED IN DESIGN hoặc chờ PO (PO-Q1…PO-Q10).

## 13. Cập nhật Codex Re-review #3 (Round 4) — 2026-11-11

Bốn MAJOR còn lại và bốn MINOR được sửa ở mức thiết kế: [PMS-0034-codex-round4-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md). Ảnh hưởng tới các gap: G-06 (bảo mật S2S — fail-closed nay là **không khởi động**), G-15 (quote — transaction `REPEATABLE_READ` cho phép ghi, không `readOnly`), G-21 (outbox — SQL, ràng buộc và giới hạn lượt thử viết lại; at-least-once), G-22 (idempotency — ràng buộc theo trạng thái, reconcile theo `state`, replay ở header). Không gap mới. Trạng thái: `RESOLVED IN DESIGN — Claude assessment`, chưa Codex xác nhận.
