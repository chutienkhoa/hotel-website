# Hotel Website

Repo frontend công khai độc lập, dùng Next.js App Router, TypeScript và Tailwind CSS. V2A dùng một instance cho mỗi khách sạn, chưa có multi-tenant. Modern Tropical là template V2A. Mọi template dùng chung Booking Engine trong feature booking.

## Tài liệu theo phạm vi

- [Mục lục docs](docs/README.md) giải thích trạng thái quyết định và cách dùng tài liệu.
- [Đặc tả client](docs/specs/client-specs.md) quy định phạm vi V2A, hành vi booking và tiêu chí nghiệm thu.
- [Kiến trúc client](docs/coding-rules/client-architecture.md) quy định ranh giới route, feature, template, adapter PMS và cấu hình instance.
- [Quy tắc frontend](docs/coding-rules/frontend.md) hướng dẫn Next.js, TypeScript, Tailwind, SEO, accessibility và kiểm thử.
- [Bảo mật](docs/security/README.md) áp dụng cho API công khai, cache, dữ liệu khách và bí mật.
- [Tư liệu thiết kế](docs/designs/README.md) hướng dẫn chọn mockup đã duyệt và lưu ảnh nghiệm thu.

Đọc đặc tả và kiến trúc trước khi triển khai hoặc review tính năng. Đọc các mục liên quan của quy tắc frontend, bảo mật và thiết kế. Giữ các mục “Cần chốt” ở trạng thái chờ quyết định. Nếu thiếu hợp đồng PMS, nêu operation, schema hoặc hành vi cần xác nhận và tách phần đang bị chặn.

## Ranh giới kỹ thuật

- Ưu tiên Server Components và giới hạn Client Components ở phần tương tác. Template nhận view model, theme và các khối UI dùng chung. Booking Engine điều phối luồng chung. Adapter REST tới PMS chạy phía server và kiểm tra schema tại biên.
- PMS Spring Boot là nguồn sự thật cho giá, tồn phòng, chế độ và kết quả booking. Giao diện phân biệt Instant đã xác nhận, Request đã tiếp nhận và kết quả chưa rõ sau timeout. Retry hoặc đối soát chỉ theo cơ chế chống trùng đã chốt với PMS.
- Chỉ công bố các trường cần thiết qua DTO. Giữ bí mật phía server và PII của khách ngoài URL, log, analytics, bộ nhớ lưu bền phía trình duyệt và cache chia sẻ. Dữ liệu giao dịch booking dùng `no-store`. Dùng fixture giả lập hoặc môi trường test khi kiểm tra thao tác ghi.
- SEO dựa trên canonical origin đã cấu hình và nội dung công bố. URL kết quả booking nằm ngoài sitemap và dùng `noindex` nếu có trang riêng. Kiểm tra responsive, label, focus và thao tác bàn phím cho UI bị ảnh hưởng.

## Vai trò và bàn giao

- Claude Code triển khai thay đổi đã được giao, tự kiểm tra và bàn giao diff local. Codex review độc lập, đối chiếu diff hoặc hành vi với đặc tả và bằng chứng tự thu thập. Trong lượt review hoặc xác minh, Codex chỉ đọc mã và chạy kiểm tra an toàn, mọi sửa lỗi quay lại Claude.
- Codex dùng [review-pr](.agents/skills/review-pr/SKILL.md) để review diff của PR hoặc diff local, và [verify-feature](.agents/skills/verify-feature/SKILL.md) để xác minh tiêu chí nghiệm thu bằng kiểm tra thực tế.
- Kiểm tra trạng thái repo trước khi làm việc. Giữ thay đổi của phiên khác và kiểm tra lại tài liệu nếu chúng đang được cập nhật. Chỉ đọc `hotel-system` để đối chiếu hợp đồng và quy ước.
- Bàn giao local. Stage, commit, push, thao tác ghi trên PR và deploy cần chỉ thị riêng của chủ repo cho từng thao tác.
- Chỉ phát triển trên branch `develop`. Không tạo branch khác, không checkout, merge hay rebase, không force push, không reset hoặc xóa thay đổi chưa commit. Nếu branch hiện tại không phải `develop`, dừng thao tác ghi và báo cáo. Codex không đổi branch hay ghi vào repo trong lượt review. Chi tiết ở [quy trình phát triển](docs/development/workflow.md).
- Lấy lệnh lint, typecheck, build và test từ cấu hình thực tế của repo. Ghi lệnh, kết quả và phạm vi đã chạy. Nếu thiếu mã, script, môi trường hoặc hợp đồng, ghi rõ phần chưa xác minh và lý do.
