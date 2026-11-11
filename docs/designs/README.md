# Thiết kế giao diện Hotel Website

Thư mục này lưu hình ảnh dùng để thiết kế, triển khai và nghiệm thu giao diện công khai của website. Dùng [đặc tả client](../specs/client-specs.md) để xác định hành vi, và mockup đã duyệt để đối chiếu giao diện. Ảnh dùng để hiển thị trên website đi qua nguồn asset của ứng dụng.

| Thư mục | Nội dung |
| --- | --- |
| [`references/`](references/) | Ảnh hoặc liên kết tham khảo về bố cục, màu sắc, typography và cách trình bày. Với Modern Tropical, lưu tham chiếu Sunset House dưới `references/modern-tropical/` khi có tư liệu được phép lưu. |
| [`mockups/`](mockups/) | Bản thiết kế trước khi triển khai. Ghi rõ bản nháp hay đã duyệt và liên kết tới màn hình hoặc hạng mục liên quan. |
| [`screenshots/`](screenshots/) | Ảnh chụp từ website đang chạy để review hoặc nghiệm thu. Ghi rõ màn hình, trạng thái, kích thước viewport và môi trường hoặc bản build. |

## Đặt tên và ghi nguồn

- Dùng chữ thường và dấu gạch ngang. Với mockup, đặt tên theo màn hình, viewport hoặc thiết bị, phiên bản và trạng thái duyệt, ví dụ `home-mobile-v02-approved.png`. Với screenshot, thêm trạng thái UI và kích thước, ví dụ `booking-empty-390x844.png`.
- Giữ thư mục phẳng khi ít ảnh. Chỉ tách theo template hoặc tính năng khi có tư liệu tương ứng. Với ảnh nhiều phiên bản, liên kết tới bản hiện hành từ đặc tả hoặc ticket để người triển khai chọn đúng ảnh.
- Ghi nguồn và mục đích tham khảo cạnh ảnh hoặc trong tài liệu liên quan. Dùng liên kết nguồn nếu quyền lưu lại ảnh chưa rõ. Dùng PNG khi cần giữ chi tiết chữ và UI, hoặc WebP/JPEG cho ảnh chụp có thể nén.
- Chụp với dữ liệu giả lập. Che hoặc loại bỏ tên, email, số điện thoại, mã đặt phòng thật và thông tin nội bộ trước khi lưu ảnh vào repo.
