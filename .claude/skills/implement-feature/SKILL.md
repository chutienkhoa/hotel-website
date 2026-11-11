---
name: implement-feature
description: >-
  Triển khai tính năng V2A đã được giao trong Hotel Website, gồm route, hành vi frontend, Booking Engine và tích hợp REST PMS. Dùng build-template cho công việc chỉ đổi trình bày.
---

# Triển khai tính năng

Đọc `CLAUDE.md` và các tài liệu được dẫn từ `AGENTS.md` theo phạm vi tính năng. Trước khi ghi file, kiểm tra branch là `develop` theo `docs/development/workflow.md`; nếu không, dừng và báo cáo. Không commit, push hay tạo PR khi chưa có chỉ thị riêng.

1. Ánh xạ yêu cầu vào tiêu chí nghiệm thu. Với thao tác PMS, xác định operation, schema, dữ liệu công bố, lỗi và cách đối soát trước khi nối luồng. Tách phần triển khai được nếu hợp đồng còn mở.
2. Kiểm tra route, feature, adapter và test hiện có. Giữ hành vi trong feature sở hữu nó và tái sử dụng Booking Engine cho mọi template. Chỉ bổ sung lớp hoặc dependency khi có nhu cầu cụ thể.
3. Với tương tác đặt phòng, ràng buộc báo giá theo tiêu chí hiện hành và bỏ qua phản hồi cũ. Hiển thị trạng thái từ PMS, giữ nhánh chưa rõ kết quả khi submit timeout và dùng lại khóa chống trùng cùng nội dung nếu hợp đồng cho phép retry.
4. Chạy test theo hành vi thay đổi. Với UI, xem màn hình và các trạng thái bị ảnh hưởng. Dùng `/review-frontend` để tự rà soát diff trước khi bàn giao theo `CLAUDE.md`.
