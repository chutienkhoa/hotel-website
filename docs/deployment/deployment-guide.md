# Hướng dẫn triển khai Hotel Website V2A

**Runbook đề xuất.** Cần chốt hosting, Next.js runtime, cách build và quyền vận hành trước khi chạy. Kiến trúc yêu cầu môi trường hỗ trợ render phía server và Route Handler cho giao dịch booking. Static export chỉ cung cấp tài nguyên tĩnh và không hỗ trợ những phần này. [Kiến trúc client](../coding-rules/client-architecture.md), [Triển khai Next.js](https://nextjs.org/docs/app/getting-started/deploying#static-export)

## 1. Chuẩn bị bản phát hành và đích triển khai

- Ghi revision và định danh bản build, phiên bản hợp đồng PMS tương thích, môi trường đích, người phê duyệt và bản đã xác minh để quay lui. Dùng lệnh build và kiểm tra từ repo sau khi chúng được thiết lập.
- Đối chiếu bản đồ **khách sạn → canonical domain → instance Next.js → PMS upstream**. Xác minh quyền sở hữu custom domain, HTTPS, alias và danh sách host hợp lệ. Kiểm tra kết nối từ runtime Next.js tới đúng PMS qua cấu hình tin cậy. [Cấu hình môi trường](../development/environment-variables.md)
- Xác nhận cơ chế giới hạn truy cập staging và cô lập staging với production. Xác nhận cách cấp bí mật, xoay khóa, làm mới cache nội dung công bố và kiểm tra cache giao dịch. [Bảo mật](../security/README.md)

## 2. Kiểm tra bản phát hành trên staging

1. Triển khai bản dự kiến phát hành với cấu hình staging. Kiểm tra khả năng phục vụ trang công khai, đúng thương hiệu và kết nối tới PMS test. Kiểm tra log ứng dụng và proxy theo nhóm lỗi đã lọc.
2. Chạy các ca nghiệm thu liên quan với dữ liệu giả lập. Kiểm tra canonical và sitemap theo cấu hình đích. Luồng gửi và kết quả booking nằm ngoài sitemap. Trang kết quả dùng `noindex` nếu có. Kiểm tra các phản hồi giao dịch có `Cache-Control: no-store` và CDN tôn trọng chỉ thị.
3. Xác minh trước cơ chế ngừng nhận booking và quay lui website. Ghi các lỗi hoặc ca chưa xác minh còn chặn việc mở booking. Xin phê duyệt cho bản build, cấu hình đích và phạm vi phát hành trước khi chuyển traffic. [Tiêu chí nghiệm thu](../requirements/acceptance-criteria.md)

## 3. Đưa vào phục vụ và theo dõi

- Sau phê duyệt, triển khai bản đã xác minh với cấu hình production. Nếu phải build riêng cho production, ghi định danh bản build từ revision đã duyệt và kiểm tra phần bị ảnh hưởng bởi cấu hình build trước khi chuyển traffic. Kiểm tra DNS/TLS, domain chính và alias, branding, danh mục công bố, canonical, sitemap và khả năng kết nối đúng PMS. Thực hiện smoke production bằng thao tác chỉ đọc.
- Trước khi mở booking thật, đối chiếu bằng chứng tích hợp PMS test, biện pháp chống lạm dụng và quy trình hỗ trợ khách khi kết quả chưa rõ. Theo dõi lỗi, timeout và khả năng kết nối theo nhóm thao tác. Chốt ngưỡng và kênh cảnh báo cùng bên vận hành.

## 4. Dừng mở booking hoặc quay lui

1. Nếu sai khách sạn/upstream, sai trạng thái booking, có dấu hiệu lộ dữ liệu hoặc lỗi giao dịch nghiêm trọng, dùng biện pháp ngừng nhận booking đã duyệt. Ghi thời điểm, revision và nhóm lỗi đã lọc.
2. Quay về bản website và cấu hình tương thích đã xác minh theo quy trình của hạ tầng. Giữ khóa hiện hành hợp lệ. Kiểm tra lại domain, upstream, cache và smoke test. Việc quay lui website chỉ tác động đến bản website đang phục vụ. PMS quản lý các giao dịch đã ghi.
3. Phối hợp với PMS để đối soát các lượt gửi chưa rõ kết quả theo hợp đồng. Dùng bằng chứng sau quay lui để quyết định khi nào mở lại booking. [Tích hợp PMS](../development/api-integration.md)

**Cần chốt để vận hành** — Nền tảng hosting và runtime, quy trình build/phát hành, DNS/TLS, lưu bí mật, cơ chế làm mới cache, kiểm tra trạng thái ứng dụng, cảnh báo, ngừng nhận booking và rollback.
