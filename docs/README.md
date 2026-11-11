# Tài liệu Hotel Website

Bộ tài liệu V2A dành cho website công khai của khách sạn. Đọc đặc tả trước khi thiết kế màn hình hoặc tích hợp PMS.

| Tài liệu | Dùng khi |
| --- | --- |
| [Đặc tả client](specs/client-specs.md) | Xác định phạm vi, hành vi đặt phòng, trạng thái giao diện và tiêu chí nghiệm thu. |
| [Kiến trúc client](coding-rules/client-architecture.md) | Chọn vị trí đặt mã, xác định trách nhiệm của template và feature, thiết kế hợp đồng API và triển khai theo domain. |
| [Quy tắc frontend](coding-rules/frontend.md) | Viết hoặc review mã Next.js, TypeScript và Tailwind CSS. |
| [Thiết kế giao diện](designs/README.md) | Tìm và lưu ảnh tham khảo, mockup và screenshot của website. |
| [Bảo mật](security/README.md) | Thiết kế ranh giới truy cập PMS, bảo vệ thông tin khách và kiểm tra trước khi mở website công khai. |
| [Quy trình phát triển](development/workflow.md) | Áp dụng chính sách Git branch, vòng đời thay đổi, cổng phê duyệt và bàn giao review độc lập. |
| [Tích hợp PMS API](development/api-integration.md) | Chốt hợp đồng theo từng thao tác và đối chiếu phản hồi trên PMS test. |
| [Cấu hình môi trường](development/environment-variables.md) | Kiểm kê cấu hình cho từng instance, phân biệt cấu hình công khai và bí mật phía server. |
| [Chiến lược kiểm thử](testing/test-strategy.md) | Chọn phạm vi và mức bằng chứng theo rủi ro thay đổi. |
| [Ca nghiệm thu V2A](requirements/acceptance-criteria.md) | Chạy ca theo mã và ghi kết quả đối chiếu với đặc tả. |
| [Hướng dẫn triển khai](deployment/deployment-guide.md) | Chuẩn bị bản phát hành, kiểm tra staging, mở phục vụ và quay lui. |
| [Quyết định kiến trúc](decisions/) | Tra bối cảnh và hệ quả của các quyết định V2A. |

## Cách đọc trạng thái

- **Đã chốt** ghi các quyết định đã thống nhất cho V2A.
- **Đề xuất** ghi thiết kế hoặc quy ước kỹ thuật dùng để review trước khi triển khai.
- **Cần chốt** ghi nội dung còn thiếu trong đặc tả hoặc hợp đồng giữa hai repo. Khi tính năng phụ thuộc vào một mục này, ghi rõ phần bị chặn trong công việc triển khai.

Dùng đặc tả client cho hành vi khách nhìn thấy, tài liệu bảo mật cho các giới hạn dữ liệu và tài liệu kiến trúc cho cách tổ chức mã. PMS là nguồn có thẩm quyền cho giá, khả năng nhận đặt phòng và kết quả đặt phòng.

Mỗi ADR ghi trạng thái, bối cảnh, quyết định, hệ quả và nguồn. Nếu một quyết định được thay thế, tạo ADR mới và liên kết hai bản để giữ lịch sử.

## Tham chiếu từ hotel-system

Đối chiếu checkout PMS tại `60821ff`. Các đường dẫn bên dưới tính từ gốc repo `hotel-system`.

- `docs/coding-rules/architecture.md` và `docs/coding-rules/frontend.md` quy định tổ chức frontend theo nghiệp vụ, tái sử dụng UI và ánh xạ thao tác sang backend.
- `docs/coding-rules/responsive-design.md` và `docs/coding-rules/testing-standards.md` hướng dẫn kiểm tra theo phạm vi ảnh hưởng và xem trực tiếp ảnh màn hình.
- `docs/coding-rules/api-standards.md` và `docs/coding-rules/security-standards.md` quy định xác thực dữ liệu tại biên, dùng DTO rõ ràng và giới hạn quyền truy cập.
- `docs/requirements/master-backlog.md`, mục PMS-0034, ghi phạm vi Website & Direct Booking. `docs/requirements/pms-0040-hotel-settings.md`, mục 21, mô tả tên, logo và favicon của PMS. Hai tài liệu cung cấp bối cảnh nghiệp vụ. Hợp đồng dữ liệu công khai dành cho website cần được thống nhất riêng.
- Cách tách tài liệu phát triển, kiểm thử và triển khai tham khảo các thư mục `docs/development/`, `docs/testing/` và `docs/deployment/`. Cách ghi ca, bằng chứng và các bước quay lui tham khảo checklist và runbook staging trong `docs/release/`.
