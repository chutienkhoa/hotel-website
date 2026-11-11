---
name: verify-feature
description: >-
  Xác minh độc lập một tính năng Hotel Website theo tiêu chí nghiệm thu bằng test hiện có hoặc kiểm tra trình duyệt trên môi trường an toàn.
---

# Xác minh tính năng

1. Đọc `AGENTS.md` và đặc tả liên quan. Ghi bản đang kiểm tra, môi trường, điều kiện ban đầu và kết quả mong đợi cho từng tiêu chí. Dùng package manager và script có trong repo.
2. Chạy các kiểm tra phù hợp với thay đổi. Với UI, xem trực tiếp màn hình và tương tác ở viewport và browser phù hợp, gồm bàn phím hoặc focus khi liên quan. Với booking, chọn các ca Instant, Request, hết phòng, đổi giá hoặc chế độ, timeout và gửi lặp mà thay đổi tác động tới. Dùng fixture giả lập hoặc PMS test với dữ liệu tổng hợp.
3. Kiểm tra tác động theo tiêu chí. Ví dụ, nhánh timeout cần giữ trạng thái chưa rõ cho khách, còn chống gửi trùng cần bằng chứng tại biên ghi PMS test để xác minh số bản ghi. Phân biệt kiểm tra UI với mock, tích hợp với PMS test và phần chỉ đọc mã. Giữ kết quả kiểm tra ngoài các file được theo dõi và giữ nguyên tiến trình của phiên khác.
4. Báo từng tiêu chí là đạt, lỗi hoặc chưa xác minh, kèm lệnh hoặc bước tái hiện, kết quả thực tế và giới hạn môi trường. Gửi các sai lệch cho Claude xử lý.
