# ADR 003 — PMS quyết định giá, tồn phòng và kết quả booking

**Trạng thái** — Đã chốt nguyên tắc cho V2A. Hợp đồng REST và ánh xạ trạng thái còn cần xác nhận. [Đặc tả client](../specs/client-specs.md)

## Bối cảnh

Giá và tình trạng còn phòng có thể đổi giữa lúc khách tìm và lúc gửi. Chế độ Instant hoặc Request có thể thay đổi, và timeout có thể xảy ra sau khi PMS đã ghi.

## Quyết định

PMS Spring Boot là nguồn sự thật cho giá, tồn phòng, chế độ hiệu lực và kết quả giao dịch. Website hiển thị dữ liệu PMS trả về. PMS xác thực nghiệp vụ tại điểm ghi, nơi có thông tin để xử lý thay đổi và các lượt gửi đồng thời.

## Hệ quả

UI phải đợi kết quả có ngữ nghĩa đã xác nhận theo hợp đồng trước khi hiển thị xác nhận hoặc tiếp nhận. Nhánh chưa rõ kết quả phụ thuộc cơ chế đối soát hoặc retry an toàn từ PMS. Việc đối chiếu chống ghi trùng cần bằng chứng tại PMS test. [Tích hợp PMS](../development/api-integration.md), [Ca nghiệm thu](../requirements/acceptance-criteria.md)
