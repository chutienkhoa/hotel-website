# Ca nghiệm thu Hotel Website V2A

Các ca dưới đây chuyển tiêu chí đề xuất thành các bước kiểm tra và bằng chứng. Hành vi mong đợi theo đặc tả client, kiến trúc và bảo mật. Ca phụ thuộc hợp đồng hoặc cấu hình chưa duyệt được ghi **Chưa xác minh** cho đến khi có điều kiện kiểm tra. [Đặc tả client](../specs/client-specs.md#tiêu-chí-nghiệm-thu-đề-xuất), [Chiến lược kiểm thử](../testing/test-strategy.md)

## Chuẩn bị

Dùng instance và domain test đã cấu hình, mockup đã duyệt nếu có, và dữ liệu giả lập. Chuẩn bị hạng phòng đã công bố và chưa công bố, một kỳ lưu trú gồm đêm Base Rate và đêm có override, cùng hai cấu hình PMS Instant và Request. Với ca ghi, dùng PMS test có cách đối chiếu kết quả được cấp quyền.

| Mã | Thao tác | Đối chiếu và bằng chứng |
| --- | --- | --- |
| WEB-01 | Mở domain chính và alias đã cấu hình nếu có. Xem trang chủ và metadata. | Đúng khách sạn, Modern Tropical, tên, logo và favicon theo dữ liệu công bố. Canonical và URL trong sitemap theo origin chính. Ghi HTML hoặc URL đã kiểm tra. |
| WEB-02 | Mở danh mục có hạng phòng thiếu ảnh tùy chọn. Thử truy cập URL dự kiến của hạng phòng chưa công bố. | Ghi ảnh trạng thái thiếu ảnh và response của trang chi tiết. Xác nhận hạng phòng chưa công bố không xuất hiện ở cả danh sách và trang chi tiết. |
| WEB-03 | Tìm kỳ lưu trú có Base Rate và override. Lặp lại với múi giờ trình duyệt khác. | Đối chiếu ngày tìm, amount và currency hiển thị với phản hồi PMS test đã lọc. Ghi cùng ngày lịch ở cả hai lần chạy. |
| WEB-04 | Đổi ngày, số khách hoặc hạng phòng khi đang tải báo giá. Mô phỏng phản hồi cho lượt cũ đến sau. | Giá khớp tiêu chí hiện hành. Luồng chặn gửi khi chưa có giá tương ứng. Ghi tiêu chí, thứ tự phản hồi và trạng thái nút gửi. |
| WEB-05 | Tìm phòng và gửi với cấu hình Instant. | Ghi phản hồi PMS test và thông báo xác nhận tương ứng. Chỉ hiển thị mã tham chiếu nếu PMS cấp mã công khai. Kiểm tra luồng V2A không thu thông tin thanh toán. |
| WEB-06 | Gửi với cấu hình Request. | Đối chiếu phản hồi tiếp nhận của PMS test với thông báo chờ khách sạn xử lý. Ghi bằng chứng riêng với ca Instant. |
| WEB-07 | Thử hết phòng, PMS báo giá đổi, chế độ đổi và PMS gián đoạn. | Ghi UI và bước tiếp theo ở từng nhánh. Phân biệt hết phòng với lỗi tải, yêu cầu đồng ý giá mới hoặc xác nhận lại thao tác theo chế độ mới khi phù hợp. |
| WEB-08 | Gửi lặp cùng một ý định và mô phỏng mất phản hồi sau khi PMS đã ghi. | Đối chiếu số bản ghi tại PMS test để xác nhận tối đa một bản ghi. Ghi trạng thái chưa rõ kết quả, sau đó ghi kết quả đối soát hoặc retry an toàn theo hợp đồng nếu được hỗ trợ. |
| WEB-09 | Tìm phòng, đọc giá, sửa lỗi và gửi trên mobile, tablet, desktop và bằng bàn phím. | Ghi màn hình, browser và viewport. Kiểm tra label, focus, thông báo trạng thái và CTA có thể tiếp cận. |
| WEB-10 | Kiểm tra response giao dịch, URL, sitemap, analytics, log và cache của lần gửi giả lập. | Ghi header đã lọc và kết quả kiểm tra trước khi mở booking. Xác nhận dữ liệu khách ở đúng ranh giới, response giao dịch dùng `no-store`, route gửi/kết quả ngoài sitemap và trang kết quả riêng dùng `noindex` nếu có. [Bảo mật](../security/README.md#kiểm-tra-trước-khi-mở-booking) |

## Ghi nhận kết quả

Ghi một dòng kết quả cho mỗi mã ca với revision/bản build, môi trường, PMS test hay fixture, kết quả thực tế, trạng thái và bằng chứng đã lọc. Trạng thái dùng **Đạt**, **Lỗi** hoặc **Chưa xác minh**. Với ca có nhiều nhánh, ghi nhánh chưa chạy. Ca kiểm thử bằng mock cần kết quả PMS test riêng khi nghiệm thu hành vi ghi booking. Tập hợp các ca còn chặn và người phụ trách quyết định trước khi mở booking thật. [Hướng dẫn triển khai](../deployment/deployment-guide.md)
