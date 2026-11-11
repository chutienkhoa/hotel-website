# Bảo mật Hotel Website V2A

Các kiểm soát dưới đây là **đề xuất kỹ thuật** cần rà soát trước khi mở V2A công khai. Website nhận request ẩn danh và thông tin liên hệ khách đặt phòng. PMS là nơi xác thực nghiệp vụ và lưu dữ liệu đặt phòng.

## Ranh giới truy cập

| Điểm tiếp nhận | Kiểm soát |
| --- | --- |
| Trình duyệt → Next.js | Dùng HTTPS. Kiểm tra origin của thao tác ghi từ trình duyệt theo origin đã cấu hình. Nếu dùng cookie phiên, thiết lập `Secure`, `HttpOnly`, `SameSite` phù hợp và bổ sung cơ chế CSRF khi cần. Giới hạn kích thước body và tần suất theo thao tác. |
| Next.js → PMS | Chỉ gọi các operation website đã thống nhất qua upstream cố định. Giới hạn quyền thông tin xác thực ở các operation cần thiết. Tách bí mật theo môi trường, lưu phía server và có cách thu hồi/đổi khóa. |
| PMS | Xác thực caller theo hợp đồng; kiểm tra schema, cấu hình booking, giá, tồn phòng và khóa chống trùng ngay cả khi Next.js đã kiểm tra. Chỉ trả các trường được phép công bố. |
| Domain và proxy | Xác minh quyền kiểm soát custom domain trước khi cấp chứng chỉ và mở định tuyến. Từ chối host ngoài cấu hình. Chỉ tin forwarded headers từ proxy tin cậy và tạo URL tuyệt đối từ canonical origin đã cấu hình. |

Giữ token PMS, cookie quản trị và bí mật ở ranh giới server tương ứng. Bundle, props Client Component và biến `NEXT_PUBLIC_*` chỉ chứa dữ liệu được phép gửi đến trình duyệt. PMS origin, đích chuyển hướng và đích tải phía server phải lấy từ cấu hình tin cậy hoặc qua allowlist đã định nghĩa.

## Dữ liệu khách và nội dung công khai

- Chỉ gửi trường liên hệ PMS cần theo hợp đồng. Tách dữ liệu khách khỏi URL, log, analytics, session replay, local storage và cache chia sẻ. Bản ghi kỹ thuật chỉ giữ mã theo dõi nội bộ đã lọc, nhóm lỗi và thời gian xử lý. Kiểm tra cả log của reverse proxy và dịch vụ giám sát.
- Gửi kết quả booking với `Cache-Control: no-store` và kiểm tra CDN tôn trọng chỉ thị. Nếu có màn hình tra cứu kết quả, yêu cầu cơ chế truy cập giới hạn do PMS và website thống nhất. Mã reservation nội bộ hoặc mã có thể đoán không đủ để truy cập.
- Kiểm tra kích thước và định dạng các trường ngày, số khách, định danh công khai và liên hệ tại biên. Trả thông báo an toàn. Giữ stack trace, SQL, thông tin xác thực và phản hồi nội bộ trong vùng vận hành được phân quyền.
- Render nội dung PMS dưới dạng text mặc định. Nếu có rich text đã duyệt, lọc theo allowlist ở biên trước khi render. Giới hạn nguồn ảnh và đường dẫn tối ưu ảnh. Từ chối URL tự do, đích mạng nội bộ ngoài cấu hình hoặc redirect không tin cậy ở các luồng tải ảnh phía server.
- Cấu hình CSP theo nguồn thực sự sử dụng, `frame-ancestors` để kiểm soát nhúng, `X-Content-Type-Options: nosniff` và `Referrer-Policy` phù hợp. Rà soát tải script bên thứ ba trên luồng booking trước khi bật.

## Gửi lặp và chống lạm dụng

- Áp dụng giới hạn tần suất và chi phí xử lý cho tìm phòng, báo giá và submit tại các biên công khai tương ứng. PMS giữ lớp kiểm soát tại điểm ghi dữ liệu. Ngưỡng giới hạn và phương án xử lý khách hợp lệ bị giới hạn cần chốt theo môi trường.
- Khóa chống trùng do luồng tạo theo hợp đồng phải được PMS thực thi tại điểm ghi. Với timeout sau khi gửi, dùng cùng khóa và cùng nội dung nếu hợp đồng cho phép gửi lại an toàn, hoặc đối soát kết quả qua thao tác được cấp quyền. UI giữ trạng thái chưa rõ kết quả đến khi có phản hồi đáng tin cậy.
- Tách môi trường và dùng dữ liệu khách giả lập cho staging. Giới hạn truy cập staging. Chốt thời hạn lưu và quy trình xử lý dữ liệu khách với người phụ trách vận hành trước khi tiếp nhận booking thật.

## Kiểm tra trước khi mở booking

- Gửi từ origin lạ, giả `Host` hoặc `X-Forwarded-Host`, gọi vượt tần suất, gửi body quá lớn và thử URL ảnh ngoài allowlist. Xác minh request bị từ chối hoặc bị giới hạn tại biên phù hợp.
- Sửa giá, chế độ, ngày hoặc hạng phòng trong request. Kiểm tra PMS kiểm tra lại và trả lỗi an toàn khi dữ liệu sai, đã thay đổi hoặc không còn phòng.
- Gửi đồng thời cùng một ý định và mô phỏng timeout sau khi PMS đã ghi. Kiểm tra tối đa một bản ghi, trạng thái client phù hợp và phản hồi lặp có kết quả nhất quán.
- Kiểm tra HTML, phản hồi API, bundle, URL, log và CDN ở staging để xác minh dữ liệu khách và bí mật chỉ xuất hiện ở những vị trí được phép.
