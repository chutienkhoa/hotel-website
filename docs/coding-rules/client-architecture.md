# Kiến trúc client Hotel Website

Hai repo triển khai độc lập. Một instance website kết nối với một PMS của khách sạn qua REST. PMS sở hữu PostgreSQL, cấu hình có hiệu lực, tồn phòng, tính giá và ghi nhận đặt phòng. Các mục sau là **đề xuất kỹ thuật** cho V2A.

## Ranh giới giữa hai hệ thống

Quyết định kiến trúc chính thức, đồng bộ với [đặc tả client mục 1.1](../specs/client-specs.md#11-ranh-giới-hệ-thống-quyết-định-kiến-trúc-chính-thức).

| | `hotel-website` | `hotel-system` |
| --- | --- | --- |
| Vai trò | Website public cho khách hàng | PMS và giao diện quản trị cho nhân viên |
| Công nghệ | Next.js, TypeScript, Tailwind CSS | Spring Boot, Thymeleaf, PostgreSQL |
| Sở hữu | Trình bày, SEO, luồng đặt phòng phía khách, Manage Booking phía khách | Dữ liệu và business logic: Room Type, gallery, tiện nghi, Room Type bán trên website và thứ tự, Base Rate, Room Rate Calendar, Website Settings, Instant/Request, Cancellation Policy, duyệt Booking Request, Reservation |
| Không sở hữu | Admin Dashboard, quản lý giá, cấu hình khách sạn, PostgreSQL, giá cuối cùng, availability, quyết định trạng thái booking | Giao diện public của khách |

- Website không có route, feature hay Route Handler quản trị. Thay đổi cấu hình thực hiện trong PMS Admin và đến website dưới dạng nội dung đã công bố qua REST.
- Website không truy cập PostgreSQL trực tiếp và không tự tính giá cuối cùng, availability hay trạng thái booking. Giá, tồn, chế độ và kết quả hiển thị lấy từ phản hồi PMS. Phần tính toán ở client chỉ phục vụ định dạng hiển thị.
- PMS xác thực Manage Booking, kiểm tra điều kiện hủy và thực hiện cancellation. Website thu thập đầu vào, gửi yêu cầu và hiển thị kết quả.
- Hai repo phát hành độc lập, nên mọi thay đổi hợp đồng cần xác định phiên bản hai bên cùng chạy. V2A dùng một instance cho một khách sạn, chưa có multi-tenant.
- Endpoint, schema và trạng thái reservation chỉ dùng sau khi đã thống nhất với PMS. Tài liệu này không xác lập chúng.

## Luồng dữ liệu và ranh giới

```text
Trình duyệt → Next.js (trang công khai, thao tác đặt phòng)
                   ↓ REST qua kết nối tin cậy
              Spring Boot PMS → PostgreSQL
```

Trang server-rendered gọi trực tiếp module tích hợp PMS trên server Next.js. Tương tác từ trình duyệt đi qua thao tác cùng origin của Next.js. Dùng Route Handler cho tương tác cần HTTP từ trình duyệt. Mỗi handler chỉ nhận đầu vào của một thao tác được định nghĩa, kiểm tra dữ liệu và trả về DTO công khai. Lớp này chuyển tiếp nghiệp vụ sang PMS.

App Router giữ route và metadata. Feature xây dựng dữ liệu cho màn hình và điều phối UI theo nghiệp vụ. Template nhận dữ liệu đã chuẩn hóa và các khối giao diện dùng chung để bố trí trang. API adapter giữ HTTP, timeout, giải mã DTO và ánh xạ lỗi. Truy cập PMS, thông tin xác thực và cấu hình kết nối thuộc module chỉ chạy trên server.

## Cấu trúc source dự kiến

```text
src/
├── app/                         # Layout, routes và metadata
│   ├── page.tsx                 # Trang chủ
│   ├── rooms/
│   │   ├── page.tsx
│   │   └── [slug]/page.tsx
│   ├── gallery/page.tsx
│   ├── booking/page.tsx
│   └── api/                     # Route Handler cho thao tác cùng origin
├── features/
│   ├── website-config/         # Thương hiệu và cấu hình công bố
│   ├── rooms/                   # Danh mục và chi tiết hạng phòng
│   ├── booking/                 # Tìm phòng, xem giá, gửi và hiển thị kết quả
│   └── manage-booking/          # Tra cứu, xác thực email và yêu cầu hủy phía khách
├── templates/
│   ├── registry.ts              # Mã template được hỗ trợ
│   └── modern-tropical/        # Bố cục, thành phần trình bày và token theme
├── components/ui/               # Thành phần dùng chung giữa các feature
└── lib/
    └── pms/                     # HTTP client và adapter REST chỉ chạy trên server
```

Giữ kiểu dữ liệu cạnh feature hoặc adapter sở hữu nó. Tách component dùng chung khi có nhu cầu tái sử dụng rõ ràng. Thêm thư mục khi có mã tương ứng.

Route dùng feature để tải dữ liệu và registry để chọn template. Template nhận view model và các khối UI được truyền vào, ví dụ form tìm phòng. Registry ánh xạ mã template đã biết sang module tĩnh. Với mã không hỗ trợ, dùng Modern Tropical nếu có cấu hình fallback hợp lệ của cùng khách sạn, đồng thời ghi cảnh báo vận hành. Nếu dữ liệu website thiết yếu không hợp lệ, hiển thị trạng thái tạm gián đoạn.

## Hợp đồng REST cần thống nhất với PMS

Một namespace có phiên bản, ví dụ `/api/public/v1`, là đề xuất. Bảng dưới đây mô tả các khả năng cần có. Đường dẫn và schema cần thống nhất với PMS. Tích hợp website chỉ dùng các operation trong hợp đồng riêng, tách khỏi route quản trị như `/api/reservations`.

| Khả năng | Dữ liệu và cam kết cần thống nhất |
| --- | --- |
| Đọc cấu hình công bố | Nội dung do nhân viên cấu hình trong PMS Admin và đã published: tên, logo/favicon, nội dung, liên hệ, mã template, chế độ booking, chính sách hủy hiển thị và dữ liệu địa phương hóa nếu được công bố. |
| Đọc danh mục phòng | Hạng phòng đã công bố, mã hoặc slug công khai, ảnh và thông tin hiển thị. Dùng mã công khai mà PMS công nhận để đặt phòng. |
| Tìm phòng và xem giá | Tiêu chí ngày, số khách, hạng phòng khi cần. Kết quả availability, tổng giá, tiền tệ và chi tiết giá được phép công bố. Chốt cách biểu diễn báo giá và thời hạn hiệu lực. |
| Gửi đặt phòng/yêu cầu | Tiêu chí đã chọn, tham chiếu báo giá hoặc dữ liệu đối chiếu theo hợp đồng, trường liên hệ cần thiết và khóa chống trùng. PMS kiểm tra giá, cấu hình và tồn phòng trong quá trình ghi, kể cả khi nhiều khách gửi đồng thời. |
| Nhận hoặc đối soát kết quả | Kết quả công khai phân biệt đã xác nhận, đã nhận yêu cầu, hết phòng, giá/chế độ thay đổi, dữ liệu sai và lỗi tạm thời. Chốt cách trả lại kết quả theo cùng khóa hoặc cơ chế tra cứu được kiểm soát. |
| Tra cứu và xác thực Manage Booking | Cách khách yêu cầu truy cập, xác thực qua email, thời hạn và phạm vi quyền, phản hồi chống dò booking, dữ liệu chi tiết được phép công bố. PMS thực hiện xác thực. |
| Hủy booking | Điều kiện hủy do PMS quyết định, cách gửi yêu cầu hủy an toàn khi lặp, kết quả phân biệt đã hủy, quá hạn và trạng thái không hợp lệ. PMS thực hiện cancellation. |

Dùng hợp đồng có schema và ví dụ phản hồi cho cả lỗi. Phân biệt DTO công khai với DTO của PMS Admin; website không gọi operation của PMS Admin. Khóa chống trùng gắn với một ý định đặt phòng và nội dung gửi. Cùng khóa và cùng nội dung phải trả cùng kết quả nếu PMS đã xử lý. Cùng khóa với nội dung khác phải bị từ chối. Chỉ bắt đầu ý định mới sau khi kết quả cũ đã rõ hoặc theo quy tắc nghiệp vụ đã chốt.

## Cache, domain và triển khai

- **Cache nội dung công bố.** Cho phép cache có thời hạn hoặc cơ chế làm mới khi đổi thương hiệu, cấu hình hay danh mục. Nếu cấu hình hiển thị và chế độ giao dịch có thời điểm cập nhật khác nhau, PMS kiểm tra chế độ hiện hành khi gửi.
- **Dữ liệu giao dịch.** Tồn phòng, báo giá theo lượt tìm, gửi và kết quả đặt phòng đi qua đường dẫn không dùng cache chia sẻ. Áp dụng chính sách nhất quán tại fetch, route và proxy/CDN. Khi PMS gián đoạn, trang giới thiệu có thể phục vụ nội dung công bố hợp lệ đã cache. Luồng đặt phòng báo lỗi tạm thời.
- **Cấu hình mỗi instance.** Website dùng một PMS origin đã cấu hình và một canonical origin. Edge định tuyến subdomain hoặc custom domain đã xác minh vào đúng instance qua HTTPS. Alias hợp lệ chuyển về canonical nếu cấu hình yêu cầu. Host từ request chỉ dùng để đối chiếu allowlist, không chọn khách sạn hoặc PMS upstream.
- **SEO.** Metadata, sitemap và canonical dựa trên URL cấu hình và nội dung đã công bố. Luồng gửi và kết quả đặt phòng không thuộc sitemap. Chi tiết biện pháp bảo vệ ở [tài liệu bảo mật](../security/README.md).
- **Môi trường.** Cấu hình phát triển, staging và production dùng upstream và bí mật riêng. Chốt nơi chạy Next.js, chính sách làm mới cache và quy trình DNS/TLS khi triển khai. Phiên bản công cụ và cấu hình build được chọn khi khởi tạo ứng dụng.
