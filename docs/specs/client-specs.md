# Hotel Website & Direct Booking — Client Specification V2A

**Status:** Business scope approved · 2026-11-10  
**Issue:** #30 / PMS-0034  
**Primary repository:** `hotel-website`  
**Backend of record:** `hotel-system` (Spring Boot PMS)

> Tài liệu hợp nhất đặc tả client ban đầu với các quyết định nghiệp vụ đã duyệt. **Approved** nghĩa là chốt hành vi sản phẩm; **Verify with PMS** nghĩa là cần đối chiếu schema, API hoặc vòng đời hiện có trước khi code. Không coi endpoint hay cột dữ liệu minh họa là API đã tồn tại.

## 1. Mục tiêu và phạm vi

Website riêng theo thương hiệu của từng khách sạn, giúp khách xem thông tin, tìm phòng, nhận báo giá, đặt phòng trực tiếp, tra cứu và hủy đặt phòng. Một codebase tái sử dụng cho nhiều khách sạn bằng các instance triển khai riêng; **mỗi instance phục vụ đúng một khách sạn**, không triển khai multi-tenant V2A.

- Frontend: **Next.js App Router, TypeScript, Tailwind CSS**, mobile-first, SEO, accessibility.
- Backend: **Spring Boot PMS** (`hotel-system`) quản lý dữ liệu, trạng thái đặt phòng, giá, tồn phòng, chính sách hủy và giao dịch, đồng thời cung cấp REST API cho website. Website chỉ giao tiếp qua REST API; không truy cập PostgreSQL trực tiếp.
- Quản trị: toàn bộ cấu hình và vận hành dành cho nhân viên nằm trong **PMS Admin** (Spring Boot + Thymeleaf). `hotel-website` không có Admin Dashboard.
- V2A: **Modern Tropical — Bright & Fresh** là template đầu tiên; kiến trúc hỗ trợ template khác về sau, cùng một booking flow.
- Triển khai Next.js và PMS độc lập qua Docker/Caddy, HTTPS.
- **Không thuộc V2A:** tài khoản khách, thanh toán online, thu thập thẻ, phí hủy/hoàn tiền tự động, room-level gallery, rate plan nâng cao, OTA/Channel Manager, multi-tenant, chọn phòng vật lý cụ thể, Admin Dashboard hoặc màn hình quản trị trong website.

### 1.1 Ranh giới hệ thống (quyết định kiến trúc chính thức)

| | `hotel-website` | `hotel-system` (PMS) |
| --- | --- | --- |
| Công nghệ | Next.js, TypeScript, Tailwind CSS | Spring Boot, Thymeleaf, PostgreSQL |
| Người dùng | Khách hàng (public) | Nhân viên khách sạn (PMS Admin) |
| Khách / nhân viên làm được | Xem thông tin khách sạn và Room Type; tìm phòng, kiểm tra giá và availability; đặt nhiều Room Type trong một reservation; nhập thông tin khách và xác nhận booking; tra cứu và tự hủy booking sau khi xác thực | Quản lý Room Type, gallery, tiện nghi; chọn Room Type bán trên website và sắp xếp thứ tự; quản lý Base Rate và Room Rate Calendar; quản lý Website Settings (branding, homepage, template, gallery, contact, domain); cấu hình Instant Booking / Booking Request; cấu hình Cancellation Policy; duyệt Booking Request; quản lý Reservation và các nghiệp vụ khách sạn |
| Không có | Admin Dashboard; màn hình quản lý giá; màn hình cấu hình khách sạn | — |

Nguyên tắc:

1. **PMS là nguồn dữ liệu và business logic duy nhất:** PostgreSQL; tính giá; kiểm tra availability/capacity; xác nhận booking và chống overbooking; vòng đời reservation; xác thực Manage Booking và xử lý cancellation; REST API cho website.
2. **Website không tự quyết định nghiệp vụ:** không truy cập PostgreSQL trực tiếp, không tự tính giá cuối cùng, availability hoặc trạng thái booking. Giá, tồn phòng, chế độ và kết quả hiển thị đều lấy từ phản hồi PMS.
3. **Cấu hình chỉ đi một chiều:** nhân viên thay đổi trong PMS Admin; website nhận nội dung đã **published** qua REST. Website không có đường ghi cấu hình khách sạn.
4. **Triển khai độc lập, một instance cho một khách sạn:** hai repository phát hành riêng; V2A chưa có multi-tenant.
5. **Không phát minh hợp đồng:** endpoint, schema và trạng thái reservation chỉ được dùng khi đã xác nhận với PMS (mục 12). Tài liệu này không định nghĩa chúng.

## 2. Quyết định nghiệp vụ được duyệt

| Chủ đề | Quyết định V2A |
| --- | --- |
| Room Catalog | Nhân viên chọn Room Type nào được bán trên website và sắp xếp thứ tự hiển thị, thao tác trong PMS Admin |
| Gallery | Ảnh theo **Room Type**, một cover và nhiều ảnh có thứ tự; không theo Room vật lý |
| Capacity | `adults + children` cùng tính vào sức chứa; không có giá trẻ em riêng |
| Guest allocation | Khách chỉ nhập tổng adults/children; PMS kiểm tra phân bổ hợp lệ vào phòng đã chọn |
| Multi-room | Một reservation chứa nhiều Room Type, mỗi loại có quantity; chung check-in/check-out |
| Booking mode | Khách sạn cấu hình **Instant** hoặc **Request** trong PMS Admin; mặc định Instant |
| Pricing | Base nightly rate theo Room Type + override cho từng ngày; PMS tính và trả giá cuối |
| Rate Calendar | Chỉ trong PMS Admin: chỉnh một ngày hoặc hàng loạt theo khoảng ngày, lọc thứ trong tuần, xem trước và ghi lịch sử |
| Manage Booking | Khách không cần tài khoản; xác thực email để xem chi tiết và tự hủy |
| Cancellation | Cửa sổ hủy miễn phí theo khách sạn, cấu hình trong PMS Admin; mặc định 24h, tùy chọn 24/48/72h trước giờ check-in; quá hạn liên hệ khách sạn |
| Domain | Subdomain PMS mặc định và custom domain đã xác minh, quản lý trong Website Settings của PMS; HTTPS và canonical |
| Payment | Không thanh toán online trong V2A |

## 3. Website Pages và Template

### 3.1 Các trang public

1. **Homepage:** tên/logo/tagline, hero image, giới thiệu, điểm nổi bật, CTA tìm phòng, room highlights, tiện nghi, gallery, vị trí/liên hệ.
2. **Room Catalog:** danh sách Room Type được công bố, thứ tự cấu hình, ảnh cover, mô tả, tiện nghi, sức chứa, giá tham khảo khi có dữ liệu hợp lệ; tìm theo ngày/số khách.
3. **Room Type Detail:** gallery, mô tả, sức chứa, tiện nghi, chính sách và CTA đặt phòng. URL/slug ổn định, không tiết lộ Room Type chưa công bố.
4. **Search / Select Rooms:** check-in, check-out, adults, children; kết quả giá và tồn phòng theo tiêu chí; quantity theo từng Room Type.
5. **Guest Info → Review & Confirm → Booking Result:** thông tin liên hệ, tổng hợp chi tiết, giá từ PMS, chính sách, nút xác nhận theo booking mode.
6. **Manage Booking:** yêu cầu truy cập bằng mã đặt phòng + email; xác thực bằng liên kết/token email; xem thông tin và hủy nếu đủ điều kiện.
7. **Gallery, Contact & Location:** chỉ hiển thị nội dung đã công bố; ẩn section không có dữ liệu tùy chọn.
8. **Privacy / Terms / Cancellation Policy:** nội dung chính sách công bố theo cấu hình và yêu cầu pháp lý của khách sạn.

### 3.2 Modern Tropical

Nền sáng ivory/cream, màu nhấn xanh lá tự nhiên, typography tiêu đề thanh lịch, hero ảnh lớn, nhiều khoảng trắng, room cards, amenity icons, gallery và CTA dễ thấy trên mobile. Tham chiếu hình ảnh thiết kế đã duyệt; không hardcode tên khách sạn, ảnh, giá, địa chỉ hoặc chính sách trong template. Các template dùng chung dữ liệu và booking components.

### 3.3 Website Settings (quản trị trong PMS)

Các mục dưới đây là màn hình của **PMS Admin** dành cho nhân viên. `hotel-website` chỉ hiển thị nội dung đã published do PMS trả về và không có màn hình chỉnh sửa tương ứng.

- **General:** hotel name, tagline, logo, favicon, thông tin giới thiệu.
- **Appearance:** template được hỗ trợ, màu thương hiệu trong phạm vi thiết kế cho phép.
- **Homepage:** hero, sections, thứ tự/nội dung phù hợp template.
- **Rooms & Gallery:** thông tin Room Type công bố, cover, ảnh bổ sung, thứ tự ảnh, thứ tự Room Type.
- **Contact & Location:** địa chỉ, bản đồ/liên kết bản đồ, điện thoại, email, kênh liên hệ.
- **Booking & Publishing:** Instant/Request, chính sách hủy, trạng thái draft/published, preview trước công bố.
- **Domain:** subdomain, custom domain, trạng thái xác minh, canonical.

Chỉ nội dung **published** được public API trả về. Preview phải có bảo vệ truy cập, không index SEO. Không cho nhập HTML/JavaScript tùy ý V2A. Ảnh lưu trong media storage phù hợp, metadata/quyền sở hữu lưu ở PMS; kiểm tra MIME, dung lượng, quyền truy cập và chống file nguy hiểm.

## 4. Room Catalog & Gallery

- PMS Room Type là nguồn sự thật về loại phòng và capacity; mỗi Room Type có cấu hình `showOnWebsite` (tên field chỉ là khái niệm), `displayOrder`, cover và ordered images.
- Chỉ Room Type **active trong PMS + được bật bán trên website + được công bố** mới hiển thị và được phép tạo booking mới qua public API. Chặn ở **backend**, không chỉ ẩn nút frontend.
- Tắt bán không thay đổi reservation đã tồn tại hoặc hoạt động bán nội bộ PMS.
- Room Type hết phòng vẫn có thể hiển thị trong catalog; với ngày đã chọn, hiển thị **Sold Out** và không cho chọn quantity vượt tồn.
- Nhân viên upload/xóa/sắp xếp ảnh, chọn cover trong PMS Admin; ảnh có alt text hoặc fallback thích hợp. Khi xóa ảnh cover, phải chọn cover thay thế hoặc fallback rõ ràng.
- Không cho khách chọn số phòng vật lý; phân bổ Room thực tế thuộc PMS.
- Slug duy nhất trong phạm vi website; redirect/404 hợp lý khi Room Type ngừng công bố; không tiết lộ thông tin không công bố.

## 5. Guest Occupancy & Multi-Room Booking

### 5.1 Input và sức chứa

- Search nhập `checkIn`, `checkOut`, `adults >= 1`, `children >= 0` (giới hạn trên do PMS cấu hình/xác minh).
- **Tổng số khách = adults + children**; trẻ em tính như người lớn về sức chứa V2A, không thu giá riêng theo tuổi.
- Khách chọn nhiều Room Type và quantity mỗi loại; cùng một kỳ lưu trú; một reservation chứa nhiều ReservationRoom (cần đối chiếu mapping schema).
- Không yêu cầu phân bổ số khách hay tên khách cho từng phòng.
- PMS kiểm tra **có thể phân bổ toàn bộ khách vào số phòng đã chọn** và từng phòng không vượt `RoomType.capacity`. Chỉ kiểm tra tổng sức chứa là chưa đủ nếu có ràng buộc bổ sung; V2A không tự đặt quy tắc bắt buộc một người lớn mỗi phòng.
- Không xác nhận booking một phần nếu một Room Type thiếu tồn; yêu cầu khách điều chỉnh lựa chọn và lấy báo giá mới.

### 5.2 Ví dụ

2 Double (capacity 2/phòng) + 1 Family (capacity 4/phòng) = 8 chỗ. Nhóm 5 adults + 2 children có thể đặt nếu PMS xác minh đủ tồn và phân bổ hợp lệ. Giá tính theo từng loại phòng × số phòng × từng đêm.

## 6. Pricing & Room Rate Calendar

- Mỗi Room Type có **Base Nightly Rate**; mỗi ngày có thể có **Daily Rate Override**. Giá hiệu lực đêm `D` = override(D) nếu tồn tại, ngược lại base rate; tính từ check-in đến **trước** check-out.
- PMS tính subtotal/tổng, tiền tệ, thuế/phí theo cấu hình thực tế; frontend không tự tính giá authoritative. Không tự giả định thuế đã bao gồm hay chưa.
- Rate Calendar (chỉ trong PMS Admin; website không có màn hình quản lý giá): sửa một ngày; bulk update khoảng ngày (inclusive), tùy chọn thứ trong tuần, preview số ngày và giá cũ/mới, xác nhận, lưu audit actor/time/changes. Chỉnh sửa cùng ngày phải có quy tắc xung đột xác định; V2A lưu **một giá override hiệu lực cho mỗi Room Type/ngày**, không xây rule engine nhiều tầng.
- Giá reservation đã xác nhận được **đóng băng theo snapshot tại thời điểm xác nhận**; cập nhật lịch giá không hồi tố.
- Khi thay ngày, số khách, Room Type hoặc quantity phải lấy quote mới. Trước khi submit, PMS kiểm tra lại; nếu giá đổi, hiển thị giá mới và yêu cầu khách đồng ý lại.
- Quote phải có định danh/thời hạn hoặc cơ chế kiểm tra lại tương đương theo hợp đồng PMS; không tự giả định quote còn hiệu lực.

## 7. Booking Flow

1. **Search:** kiểm tra định dạng ngày, check-out > check-in, quy tắc đặt trước/giới hạn lưu trú do PMS cung cấp; ngày là calendar date theo timezone khách sạn, không phụ thuộc timezone trình duyệt.
2. **Select:** hiển thị Room Type được bán, availability, giá từ PMS; quantity không vượt tồn; phân biệt sold-out và lỗi tải dữ liệu.
3. **Guest Info:** thu thông tin tối thiểu cần thiết (tên, email, điện thoại nếu PMS yêu cầu); không thu dữ liệu thẻ. Privacy notice và consent theo yêu cầu pháp lý.
4. **Review:** ngày, adults/children, danh sách Room Type × quantity, từng khoản giá nếu API hỗ trợ, tổng tiền, tiền tệ, booking mode và chính sách hủy hiện hành.
5. **Submit:** nút theo mode (`Book now`/`Send request`); disable khi đang xử lý; dùng idempotency key cho cùng một ý định đặt phòng. PMS **transactionally** revalidate published catalog, capacity, inventory, effective mode, quote/price và tạo booking; tránh overbooking/concurrent double booking.
6. **Result:** Instant chỉ hiển thị **Confirmed** khi PMS thực sự trả xác nhận; Request hiển thị **Request received / Pending approval**, không nói đã xác nhận. Chỉ công bố mã tham chiếu được PMS cho phép. Gửi email tương ứng.

### 7.1 Instant vs Request

- **Instant (mặc định):** thành công → reservation `CONFIRMED` theo lifecycle PMS được xác minh.
- **Request:** thành công → yêu cầu chờ nhân viên duyệt trong PMS Admin (website không có màn hình duyệt); **không mặc định đồng nhất `DRAFT` với pending request** cho tới khi đối chiếu lifecycle. PMS xác định giữ tồn hay không, thời hạn xử lý/expiry, duyệt/từ chối và thông báo. Không hứa có phòng trước khi xác nhận.
- Thay đổi mode chỉ áp dụng cho booking mới; không chuyển trạng thái booking cũ. Nếu mode đổi giữa review/submit, bắt khách xem lại và xác nhận hành vi mới.

### 7.2 Xử lý ngoại lệ

- **Price changed:** hiển thị giá mới và yêu cầu đồng ý lại.
- **Inventory changed:** từ chối toàn bộ lựa chọn không còn hợp lệ; hỗ trợ chọn lại.
- **Network timeout/unknown outcome:** không báo thất bại chắc chắn; đối soát bằng idempotency key hoặc cơ chế PMS tương đương, không tạo reservation thứ hai.
- **Repeated submit:** tối đa một booking cho một ý định đặt phòng.
- **Backend unavailable:** hiển thị lỗi thân thiện, không tạo trạng thái xác nhận giả.
- **Stale/hidden Room Type:** PMS từ chối đặt mới ngay cả khi frontend đang giữ dữ liệu cũ.

## 8. Manage Booking & Cancellation

### 8.1 Truy cập an toàn

- Khách nhập **booking reference + email**; không tiết lộ booking có tồn tại hay không qua thông báo lỗi.
- Gửi **liên kết xác thực một lần, thời hạn ngắn** đến email khớp với booking; token ngẫu nhiên, lưu hash hoặc cơ chế bảo vệ tương đương, giới hạn lần dùng/thời hạn, HTTPS, chống brute force/rate limit.
- Chỉ sau xác thực mới xem được chi tiết booking nhạy cảm hoặc yêu cầu hủy; token không xuất hiện trong analytics, referrer hay log; trang quản lý có `noindex`.
- Không dùng mỗi mã booking + email làm bằng chứng đủ quyền hủy.

### 8.2 Chính sách hủy V2A

- Chính sách **theo khách sạn**, do nhân viên cấu hình trong PMS Admin; website chỉ hiển thị chính sách PMS trả về và gửi yêu cầu hủy, còn PMS quyết định kết quả. Mặc định hủy miễn phí đến **24 giờ trước giờ check-in**, tùy chọn **24/48/72 giờ**. Giờ check-in chuẩn và timezone lấy từ cấu hình khách sạn/PMS, không theo giờ máy khách.
- Booking hợp lệ và còn trong cửa sổ hủy: khách xác nhận thao tác hủy; PMS kiểm tra lại trạng thái, thời hạn, quyền token trong transaction rồi chuyển `CANCELLED`, cập nhật inventory và audit.
- Quá thời hạn: **không cho tự hủy**, hiển thị hướng dẫn liên hệ khách sạn; không tự tính phí.
- Booking `CANCELLED`, `CHECKED_IN`, `CHECKED_OUT`, `NO_SHOW` không tự hủy. Trạng thái Request pending có được tự rút yêu cầu hay không phải đối chiếu lifecycle; mặc định không tự giả định là hủy reservation confirmed.
- Thông báo email khi hủy thành công; hành động hủy phải idempotent, không giải phóng tồn hai lần.
- **Không có phí hủy/hoàn tiền tự động** ở V2A. Nếu về sau có deposit/payment, cần nghiệp vụ hoàn tiền riêng.
- Chính sách hiển thị rõ trước khi khách submit. PMS lưu/đối chiếu chính sách áp dụng cho reservation tại thời điểm tạo; thay đổi cấu hình sau đó **không được tự ý làm bất lợi cho booking đã tạo**. Cách snapshot cần xác minh với schema PMS.
- **Booking tạo trước Issue #30 (legacy, không có snapshot chính sách hủy):** không cho tự hủy qua website; hiển thị hướng dẫn liên hệ khách sạn; **không** tự áp cửa sổ 24h mặc định hiện tại cho các booking này (quyết định PO-04, 2026-11-11, phản hồi Codex review — xem [ADR 016](../decisions/016-po04-legacy-booking-cancellation.md)). Không đổi chức năng hủy booking hiện có trong PMS Admin.

## 9. Domain, SEO, Publishing & Localization

- Mỗi instance có subdomain mặc định, ví dụ `hotelabc.pmshotelvietnam.com`; có thể gắn custom domain sau khi xác minh quyền kiểm soát và DNS/TLS. Custom domain active → canonical về domain chính; alias redirect phù hợp, fallback subdomain vẫn có lối vận hành an toàn.
- Caddy/HTTPS, HSTS khi phù hợp, không cấp nhầm chứng chỉ hoặc trỏ nhầm instance. Không dùng hostname để giải quyết tenant động trong V2A.
- SEO: metadata/title/description, canonical, sitemap, robots, Open Graph cho trang public; `noindex` preview, manage booking và bước checkout chứa thông tin cá nhân.
- Nội dung hỗ trợ locale theo cấu hình đã xác minh; không tự giả định ngôn ngữ/tiền tệ mặc định. Hiển thị currency do PMS trả.
- Draft/published: chỉnh nội dung không tự công bố; preview có xác thực, publish có quyền và audit. Quy tắc cache invalidation sau publish phải được thiết kế.

## 10. Email, Security & Operations

- Email: booking confirmed, request received, request approved/rejected (nếu lifecycle hỗ trợ), cancellation confirmed, secure manage-booking link. Gửi **sau khi transaction thành công** qua outbox/retry hoặc cơ chế tin cậy tương đương; lỗi gửi email không tự đảo ngược reservation đã tạo.
- API public chỉ trả trường cần thiết; không lộ dữ liệu guest khác, thông tin nội bộ, giá chưa công bố hoặc cấu hình bí mật.
- Validation cả frontend và PMS; PMS là authoritative. Rate limiting tìm kiếm/tra cứu/OTP, anti-enumeration, CSRF/cookie protection nếu sử dụng session, CORS allowlist, audit các thao tác nhạy cảm.
- Không đưa secret vào `NEXT_PUBLIC_*`, không log PII/token; bảo vệ dữ liệu khách theo chính sách retention và quyền truy cập đã duyệt.
- Observability: correlation/request ID, structured error codes, retry an toàn, monitoring lỗi booking/email, quy trình hỗ trợ booking không rõ kết quả.
- Website phải có loading/empty/error states, keyboard navigation, accessible labels, ảnh tối ưu, responsive mobile/tablet/desktop.

## 11. Acceptance Criteria (V2A)

| ID | Tình huống | Kết quả mong đợi |
| --- | --- | --- |
| AC-01 | Mở domain/subdomain đã cấu hình | Đúng hotel instance, branding, Modern Tropical, HTTPS và canonical |
| AC-02 | Room Type ẩn hoặc inactive | Không hiện trong catalog/detail và không thể book qua public API |
| AC-03 | Sắp xếp Room Type/ảnh, thay cover | Website phản ánh đúng thứ tự và cover sau publish/cache refresh |
| AC-04 | Room Type hết tồn | Vẫn có thể thấy trong catalog, search hiển thị Sold Out và không thể chọn đặt |
| AC-05 | 3 adults + 1 child, 2 Double capacity 2 | PMS chấp nhận về sức chứa nếu đủ tồn và không có ràng buộc bổ sung |
| AC-06 | Nhiều Room Type, quantity khác nhau | Tạo đúng một reservation và các reservation-room tương ứng |
| AC-07 | Một Room Type thiếu tồn | Không tạo booking một phần; yêu cầu chọn lại |
| AC-08 | Base Rate một đêm, override đêm khác | Tổng giá PMS đúng, checkout không tính thêm đêm, currency đúng |
| AC-09 | Bulk rate update chọn Fri/Sat | Preview đúng ngày tác động, audit đầy đủ, không thay giá reservation đã confirmed |
| AC-10 | Instant thành công | Chỉ hiển thị Confirmed nếu PMS xác nhận; gửi email phù hợp |
| AC-11 | Request thành công | Hiển thị Pending/Request received; không khẳng định confirmed |
| AC-12 | Mode hoặc giá thay đổi lúc submit | Bắt khách xác nhận lại mode/giá mới trước khi tiếp tục |
| AC-13 | Hai khách tranh phòng cuối | Không overbooking; tối đa số phòng thực sự còn bán |
| AC-14 | Submit lặp/timeout | Một ý định đặt phòng tạo tối đa một booking; có đối soát kết quả chưa rõ |
| AC-15 | Tra cứu booking chưa xác thực | Không lộ thông tin booking; gửi secure link với chống enumeration/rate limit |
| AC-16 | Booking đủ điều kiện, hủy trước cutoff | PMS cập nhật `CANCELLED`, availability và audit đúng một lần, gửi email |
| AC-17 | Quá cutoff hoặc trạng thái không hợp lệ | Chặn tự hủy; hiển thị liên hệ khách sạn khi thích hợp |
| AC-18 | Chỉnh chính sách hủy sau khi đã đặt | Không tự động làm bất lợi booking cũ; policy áp dụng được truy xuất rõ |
| AC-19 | Trình duyệt khác timezone khách sạn | Ngày lưu trú, giá đêm và cutoff không bị lệch |
| AC-20 | PMS lỗi, mất ảnh, dữ liệu tùy chọn thiếu | UI fallback đúng; không xác nhận sai hoặc lộ PII |
| AC-21 | Mobile, tablet, desktop, bàn phím | Hoàn tất search/booking/manage-booking với validation và accessibility |
| AC-22 | Không có online payment | Không thu thẻ, không hiển thị checkout thanh toán online |

## 12. Verify with PMS — bắt buộc trước khi triển khai

Các quyết định sản phẩm trên đã duyệt; **những mục dưới đây chưa được xác nhận là có sẵn trong codebase**:

1. **Reservation lifecycle:** trạng thái pending request thực tế, có nên thêm status riêng, ai approve/reject, expiry, hold inventory và xử lý race condition; tránh sử dụng `DRAFT` tùy tiện.
2. **ReservationRoom schema:** biểu diễn nhiều Room Type × quantity, phân bổ Room vật lý khi nào, capacity validation, adults/children persistence.
3. **Pricing:** Room Type base rate, daily override schema, tax/fee/currency, quote expiry, snapshot giá từng đêm và reservation, concurrency/audit calendar.
4. **Inventory:** cách tính room availability chính xác theo stay/date/status, pending requests có giữ phòng không, transaction locking/uniqueness.
5. **Public REST contracts:** endpoint, request/response schema, error codes, authentication, idempotency, reconciliation; **không tự tạo API names khi chưa thiết kế**.
6. **Catalog/Website Settings:** quyền admin/manager, draft/published model, image storage, Room Type active flag, slug và order, cache invalidation.
7. **Cancellation:** hotel check-in time/timezone, policy snapshot, trạng thái nào được tự hủy, token/OTP storage, notification delivery, booking request withdrawal.
8. **Email infrastructure:** provider, verified sender, transactional outbox/retry, templates, delivery failures.
9. **Domain/deployment:** domain ownership verification, TLS issuance, DNS, preview environment, rollback và domain migration.
10. **Compliance & content:** locale, guest fields, consent, privacy/retention, published policy wording, legal/tax obligations.

**Implementation gate:** Claude Code cần audit `hotel-system` và tài liệu hiện hành, lập API/schema gap analysis và ghi lại quyết định kỹ thuật trước khi implement. Mọi thay đổi backend phải được review như một phần của Issue #30; không suy đoán hệ thống đã hỗ trợ nghiệp vụ chỉ vì đặc tả này mô tả nó.

## 13. Phân chia trách nhiệm

Chi tiết ranh giới ở mục 1.1. Bảng dưới tóm tắt theo chức năng.

| `hotel-website` (khách hàng) | `hotel-system` (PMS và PMS Admin cho nhân viên) |
| --- | --- |
| Render template, SEO, responsive UI, forms, booking summary | Cấu hình khách sạn, Website Settings, publish nội dung; quản lý Room Type, gallery, tiện nghi; chọn Room Type bán và thứ tự |
| Chọn ngày/khách/Room Type/quantity và gọi API | Authoritative pricing, capacity, availability, inventory locking; quản lý Base Rate và Rate Calendar |
| Hiển thị Instant/Request và kết quả thực tế | Cấu hình Instant/Request; duyệt Booking Request; reservation lifecycle, persistence, idempotency, reconciliation |
| Yêu cầu xác thực email và hiển thị Manage Booking | Secure token verification, policy check, cancellation transaction, audit; cấu hình Cancellation Policy |
| Hiển thị giá, chính sách, thông báo lỗi từ PMS | Rate snapshot, email events và lịch sử thay đổi; quản lý Reservation và nghiệp vụ khách sạn |
| Không có Admin Dashboard, không quản lý giá hay cấu hình khách sạn, không truy cập PostgreSQL | Cung cấp REST API cho website; là nguồn dữ liệu và business logic duy nhất |

---

**Kết luận:** Business scope V2A được chốt để review/triển khai theo từng slice. Các mục `Verify with PMS` là điều kiện xác minh kỹ thuật, không phải lời mời tự mở rộng scope hoặc thay đổi quyết định nghiệp vụ.
