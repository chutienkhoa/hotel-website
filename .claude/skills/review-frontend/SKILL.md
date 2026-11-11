---
name: review-frontend
description: >-
  Tự rà soát thay đổi frontend trong phiên triển khai của Claude trước khi bàn giao Hotel Website cho Codex review độc lập.
---

# Tự rà soát frontend

1. Đọc diff và so với tiêu chí đã giao. Tách thay đổi của phiên khác và xác định đường đi thực tế từ route qua UI tới adapter hoặc PMS nếu liên quan.
2. Rà các rủi ro theo phần thay đổi. Với API và booking, ưu tiên sai trạng thái, cache, lộ dữ liệu và ranh giới server. Với UI công khai, kiểm tra responsive, accessibility và metadata liên quan. Gọi agent chuyên biệt trong `CLAUDE.md` khi cần phân tích sâu một phần cụ thể.
3. Đối chiếu bằng chứng kiểm tra với các nhánh có rủi ro. Chạy các script thực tế còn thiếu và xem ảnh hoặc luồng trên trình duyệt khi có thay đổi UI.
4. Báo các điểm cần sửa, bước tái hiện hoặc vị trí mã và phần chưa xác minh. Khi skill được dùng trong lượt triển khai, sửa lỗi thuộc phạm vi và chạy lại kiểm tra bị ảnh hưởng. Khi yêu cầu chỉ rà soát, trả findings để Claude xử lý ở lượt triển khai.
