# ADR 002 — Booking Engine dùng chung giữa các template

**Trạng thái** — Đã chốt cho V2A theo đặc tả client. [Quyết định đã chốt](../specs/client-specs.md#quyết-định-đã-chốt)

## Bối cảnh

Modern Tropical là template đầu tiên. Codebase cần cho phép đổi cách trình bày mà giữ cùng hành vi đặt phòng.

## Quyết định

Dùng chung Booking Engine cho tìm phòng, báo giá, gửi và hiển thị kết quả. Template chịu trách nhiệm bố cục và theme. Kiến trúc client mô tả cách truyền dữ liệu đã chuẩn hóa và các khối UI vào template. [Kiến trúc client](../coding-rules/client-architecture.md#luồng-dữ-liệu-và-ranh-giới)

## Hệ quả

Kiểm tra hành vi booking ở feature dùng chung. Kiểm tra việc gắn các khối UI và khả năng sử dụng ở từng template được triển khai. Khi đổi template, các ca nghiệp vụ dùng cùng tiêu chí nghiệm thu.
