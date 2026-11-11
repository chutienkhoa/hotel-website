# Cấu hình môi trường Hotel Website

**Trạng thái V2A** — Bảng này kiểm kê cấu hình cần có cho mỗi instance. Repo chưa có ứng dụng Next.js, schema cấu hình hoặc tên biến dự án đã xác nhận. Tên, định dạng và cách cấp từng giá trị sẽ được ghi khi khởi tạo ứng dụng và chốt hạ tầng.

## Kiểm kê cấu hình

| Cấu hình cần quản lý | Phạm vi sử dụng | Điều cần kiểm tra hoặc chốt |
| --- | --- | --- |
| PMS upstream | Server Next.js | Origin cố định, đúng môi trường và khách sạn. Xác nhận kết nối tin cậy. Host do trình duyệt gửi chỉ dùng để đối chiếu allowlist. |
| Thông tin xác thực PMS | Bí mật phía server | Cấp riêng theo môi trường, đúng quyền của từng operation đã duyệt. Chốt cơ chế cấp, luân chuyển và thu hồi theo hợp đồng. |
| Canonical origin | Cấu hình website được phép công bố | Origin chính dùng cho metadata, canonical và sitemap. Xác nhận domain staging riêng và HTTPS cho domain công khai. |
| Host và origin được chấp nhận | Website và lớp định tuyến theo thiết kế triển khai | Đồng bộ host chính, alias đã xác minh, origin hợp lệ cho thao tác ghi và quy tắc tin cậy forwarded headers. |
| Nguồn ảnh công khai được phép | Cấu hình Next.js và đường tải ảnh phía server | Chỉ cấp host/path cần dùng. Chốt danh sách dựa trên nguồn ảnh được công bố. |
| Kiểm soát vận hành | Website hoặc hạ tầng theo thiết kế triển khai | Chốt nơi đặt giới hạn tần suất, kích thước body, timeout và cơ chế làm mới cache nội dung công bố. |

Chế độ đặt phòng và dữ liệu thương hiệu công bố đi theo hợp đồng PMS. Giá trị hạ tầng cho một instance gắn với đúng khách sạn, từ local/test đến staging và production. [Kiến trúc client](../coding-rules/client-architecture.md#cache-domain-và-triển-khai), [Tích hợp PMS](api-integration.md)

## Cấp và xác minh cấu hình

1. Xác định giá trị nào được đọc lúc build, lúc server chạy hoặc khi trang được render trước. Với tham chiếu trực tiếp tới biến `NEXT_PUBLIC_*`, Next.js có thể đưa giá trị vào JavaScript khi build. Kiểm tra lại HTML, metadata và bundle khi dùng cùng một bản build qua nhiều môi trường. Bí mật và cấu hình kết nối PMS chỉ được đọc phía server. [Biến môi trường trong Next.js](https://nextjs.org/docs/app/guides/environment-variables)
2. Khai báo cấu hình bắt buộc khi schema thực tế có mặt. Kiểm tra trường thiếu, định dạng sai và origin không khớp trước khi đưa instance phục vụ traffic. Báo tên hoặc nhóm cấu hình lỗi và giữ giá trị bí mật ngoài thông báo.
3. Dùng fixture hoặc PMS test ở local/test. Cấp cấu hình và bí mật riêng cho staging và production qua cơ chế đã được duyệt. Kiểm tra cấu hình đích trước mỗi lần triển khai và sau khi xoay khóa. Dùng dữ liệu giả lập để kiểm tra thao tác ghi trên môi trường test. [Bảo mật](../security/README.md), [Triển khai](../deployment/deployment-guide.md)

Khi có cấu hình thực tế, bổ sung tên biến, kiểu, tính bắt buộc, nơi cấp, thời điểm đọc và giá trị mặc định nếu đã duyệt. Một file mẫu trong repo chỉ chứa tên biến đã triển khai và giá trị minh họa an toàn.
