---
name: review-pr
description: >-
  Review độc lập diff PR hoặc thay đổi local của Hotel Website. Dùng để tìm lỗi có thể hành động trong phần thay đổi và đánh giá rủi ro hồi quy.
---

# Review diff độc lập

1. Đọc `AGENTS.md` và tài liệu theo phạm vi. Xác định diff cần review và ghi base/head hoặc phạm vi local. Với PR, đọc đúng diff và metadata qua quyền đọc. Với local, kiểm tra cả thay đổi staged, unstaged và file mới liên quan. Giữ nguyên checkout và trạng thái PR. Nêu phạm vi thiếu nếu chưa đọc được đúng diff.
2. Theo diff qua caller, route, feature và test bị ảnh hưởng. Ưu tiên lỗi có thể tái hiện hoặc chứng minh từ mã. Với booking, rà dữ liệu và trạng thái do PMS quyết định, phản hồi cũ, retry, schema, cache, bí mật và PII. Với route hoặc UI công khai, rà SEO, accessibility và responsive theo thay đổi.
3. Dùng test hiện có khi cần kiểm chứng một nhận định và khi bản đang chạy khớp bản review. Với yêu cầu nghiệm thu luồng theo đặc tả, dùng `verify-feature` để lập bằng chứng riêng.
4. Trả findings theo mức ảnh hưởng. Mỗi finding có vị trí file/dòng, điều kiện kích hoạt và tác động. Nếu chưa thấy lỗi có thể hành động, ghi phạm vi đã đọc và các kiểm tra chưa thực hiện. Giữ kết quả ở lượt review và chuyển phần sửa về Claude.
