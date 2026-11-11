# ADR 015 — Giá Extension/Date Change mapping theo calendar date, không theo vị trí đêm

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-11, phản hồi Codex review REQUEST CHANGES — BLOCKER-01), giải quyết PO-03. [Thiết kế nightly pricing](../../../hotel/docs/specs/PMS-0034-codex-review-resolution-2026-11-11.md#24-po-03--extension--date-change-thiết-kế-mapping-theo-calendar-date)

## Bối cảnh

Xác nhận từ source: `StayExtensionService.extend` (dòng 382) tính giá đêm gia hạn bằng `ReservationRoom.getNightlyRate()` — một giá PHẲNG duy nhất cho toàn bộ kỳ ở, không phân biệt đêm cũ/mới. `Reservation.changeConfirmedDates`/`ReservationRoom.changeConfirmedDates` cũng tính lại `totalAmount = nightlyRate (không đổi) × số đêm mới` cho toàn bộ khoảng. Với nightly pricing có daily override (D-02/ADR 005), hành vi này sẽ tính SAI giá cho đêm mới thêm vào.

## Quyết định

Khi nhân viên gia hạn hoặc thay đổi ngày lưu trú: giữ giá đã chốt của những đêm vẫn thuộc booking cũ; các đêm mới được bổ sung sử dụng giá hiện hành tại thời điểm thao tác; các đêm bị loại bỏ không tiếp tục tính tiền; không tính lại toàn bộ booking theo giá hiện hành. Mapping giá theo calendar date cụ thể, không theo vị trí đêm đầu/cuối.

## Hệ quả

Đây là một THAY ĐỔI LOGIC NGHIỆP VỤ bắt buộc khi implement (không chỉ thiết kế schema): `StayExtensionService.extend` và `Reservation.changeConfirmedDates`/`ReservationRoom.changeConfirmedDates` phải đối chiếu breakdown `reservation_room_nightly_rate` theo `stay_date`: đêm còn trong giao của khoảng cũ/mới → giữ; đêm mới → chèn giá hiện hành (tra base rate + daily override tại thời điểm thao tác, dùng bộ tính giá BE-09); đêm bị loại → xoá. `total_amount`/`nightly_rate` (trung bình phái sinh, ADR 005) tính lại từ breakdown còn lại. Không ảnh hưởng reservation V1 cũ (không có breakdown, vẫn dùng công thức phẳng hiện tại). `FolioReconciliationService`/`MonthlyFinancialReportService.verifyExtension` không cần sửa vì đẳng thức vẫn đúng bằng cấu trúc (như ADR 005). Chi tiết đầy đủ: `hotel-system/docs/specs/PMS-0034-codex-review-resolution-2026-11-11.md` mục 2.4.

Mở ADR mới nếu Product Owner muốn một cách tính khác cho trường hợp đổi cả Room Type khi gia hạn (ngoài phạm vi V2A hiện tại).

## Cập nhật Round 3 (2026-11-11)

Phần "Hệ quả" nhắc `reservation_room_nightly_rate` và `total_amount`/`nightly_rate` (trung bình phái sinh) **không còn hiệu lực**. Thực hiện PO-03 theo [ADR 017](017-nightly-pricing-flat-and-daily-modes.md): áp dụng cho dòng `DAILY` (booking tạo từ website); gia hạn lưu đêm mới riêng theo `StayExtensionRoom` với giá hiện hành tra bằng `priced_room_type_id` của lineage (giá đi theo booking, không theo phòng thực tế) và **không chạm** `ReservationRoom` hay Charge gốc; đổi ngày đối chiếu theo `stay_date` (giữ đêm chung giá đóng băng, thêm đêm mới giá hiện hành, bỏ đêm bị loại). Dòng `FLAT` (V1, PMS Admin) không đổi hành vi. Chủ sở hữu: hotel-system#137 (BE-10b). Cần PO-Q1 (sửa `technical-spec-v1`, vì spec V1 hiện ghi gia hạn "không dùng giá động"). Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §2.4.

## Cập nhật Round 4 (2026-11-11) — ước tính gia hạn

Với dòng `DAILY`, số tiền gia hạn hiển thị phải lấy từ preview có thẩm quyền của backend theo **giá từng đêm**; không nhân số đêm với giá phẳng. Ước tính phẳng của giao diện hiện tại (`stay-extension.js`) chỉ đúng cho `FLAT` và giữ nguyên cho `FLAT`. Quyết định của ADR không đổi. Chi tiết: [../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md) §6.4.
