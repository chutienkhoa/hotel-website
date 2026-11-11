---
name: build-template
description: >-
  Xây dựng hoặc chỉnh bố cục, theme và component trình bày của Modern Tropical từ tư liệu thiết kế Hotel Website. Dùng cho giao diện dùng chung Booking Engine.
---

# Xây dựng template

Đọc `AGENTS.md`, hướng dẫn trong `docs/designs/README.md` và các mục frontend liên quan. Xác định màn hình, trạng thái và mockup đã duyệt. Nếu chỉ có tư liệu tham khảo, ghi rõ các quyết định thiết kế đang được giả định.

1. Ánh xạ mockup vào bố cục, typography và token ngữ nghĩa của Modern Tropical. Dùng view model công bố, component sẵn có và các khối Booking Engine được truyền vào. Giữ logic tìm phòng, báo giá và gửi booking ở feature chung.
2. Kiểm tra dữ liệu thiếu ảnh và nội dung dài. Giữ CTA đặt phòng dùng được trên màn hình hẹp, kể cả khi có bàn phím hoặc nút cố định. Kiểm tra ngữ nghĩa heading, alt, contrast, label và focus. Đối chiếu metadata và canonical nếu thay đổi tác động tới route công khai.
3. Mở bản render ở các kích thước phù hợp với trang, gồm mobile, tablet và desktop. So với mockup và kiểm tra các trạng thái liên quan. Chỉ lưu ảnh có dữ liệu giả lập theo quy ước trong `docs/designs/README.md` khi cần bằng chứng nghiệm thu.
4. Dùng `/review-frontend` cho lượt tự rà soát. Khi bàn giao, ghi mockup, các khác biệt có chủ đích và viewport đã kiểm tra.
