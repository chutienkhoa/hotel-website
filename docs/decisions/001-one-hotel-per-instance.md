# ADR 001 — Một khách sạn cho mỗi instance

**Trạng thái** — Đã chốt cho V2A theo đặc tả client. [Quyết định đã chốt](../specs/client-specs.md#quyết-định-đã-chốt)

## Bối cảnh

Cùng codebase cần phục vụ được nhiều khách sạn qua các lần triển khai riêng. Mỗi khách sạn có thương hiệu, domain công khai và PMS tương ứng.

## Quyết định

Mỗi instance Next.js phục vụ một khách sạn. Subdomain hoặc custom domain đã xác minh trỏ tới instance tương ứng. V2A chưa triển khai multi-tenant.

## Hệ quả

Khi triển khai cho khách sạn khác, cấp instance và cấu hình kết nối riêng. Kiểm tra domain, canonical và PMS upstream như một bộ cấu hình của instance. Host từ request được đối chiếu với cấu hình tin cậy theo thiết kế hiện tại. [Kiến trúc client](../coding-rules/client-architecture.md#cache-domain-và-triển-khai)

Mở ADR mới nếu phạm vi sản phẩm yêu cầu một instance phục vụ nhiều khách sạn. Quyết định đó cần đánh giá lại ranh giới dữ liệu, domain, cache và thông tin xác thực.
