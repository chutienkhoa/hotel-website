# ADR 017 — Hai chế độ giá `FLAT`/`DAILY`; giá theo đêm là nguồn phân bổ doanh thu chính xác

**Trạng thái** — **Đã duyệt bởi Product Owner (PO-Q1, 2026-11-11).** Chi tiết thiết kế vẫn chờ Codex re-review #3 trước BE-10b. Làm rõ cách thực hiện [ADR 005](005-nightly-pricing-snapshot.md) (D-02, đã chốt) và [ADR 015](015-po03-extension-date-change-pricing.md) (PO-03, đã chốt); không thay đổi hai quyết định đó, chỉ thay phần "Hệ quả" kỹ thuật bị Codex re-review #2 bác bỏ.

## Bối cảnh

Hệ quả kỹ thuật cũ của ADR 005/015 (Round 2): giữ `reservation_room.nightly_rate` làm "giá trung bình phái sinh" `total_amount / số đêm` và không sửa `MonthlyFinancialReportService`. Codex re-review #2 (REQUEST CHANGES) và đối chiếu source cho thấy thiết kế này sai:

1. Trung bình không chia hết, lưu `NUMERIC(19,6)`, làm `verifySnapshot` (`nightlyRate × nights` phải bằng `totalAmount`, `compareTo`) ném `ReportDataIntegrityException`.
2. `addSegment` nhân một giá phẳng nên doanh thu từng tháng sai khi booking vắt tháng; tổng toàn kỳ đúng nhưng tháng nào cũng sai.
3. `CHECKED_IN` chỉ ghi nhận đêm đã bắt đầu; trung bình × số đêm đã bắt đầu ≠ tổng giá các đêm đã bắt đầu.
4. Dòng Excel theo đêm mang giá trung bình thay vì giá của đêm đó.
5. V33 giữ một Charge gốc cho mỗi `reservation_room`; không thể biểu diễn breakdown bằng nhiều Charge.

Quy tắc V1 trong `technical-spec-v1` (§61.2, §61.6 gồm "Toàn vẹn snapshot", §68 gia hạn "luôn là `nightlyRate` của lineage", §69, §73 đổi ngày "giữ `nightlyRate`") mô tả đúng cho giá phẳng, và vẫn đúng cho dữ liệu hiện có.

## Quyết định (đã duyệt, PO-Q1)

Mỗi dòng `ReservationRoom` và `StayExtensionRoom` có `pricing_mode`:

- **`FLAT`** — mọi dữ liệu V1 và mọi luồng PMS Admin. Logic, dữ liệu và test giữ nguyên tuyệt đối. Không backfill.
- **`DAILY`** — chỉ luồng đặt phòng công khai của Issue #30. Bảng giá theo đêm `room_night_rate` là nguồn sự thật; `total_amount = Σ giá các đêm` chính xác (giá chỉ được **nhập** đúng độ chính xác tiền tệ nên không có làm tròn nào); báo cáo tháng cộng đúng giá từng đêm trong khoảng overlap và đêm đã bắt đầu; Charge gốc vẫn một dòng/`ReservationRoom` với `amount = total_amount`; gia hạn lưu đêm riêng theo `StayExtensionRoom` và không chạm Charge gốc; `UNIQUE (reservation_room_id, stay_date)` chặn ghi nhận trùng giữa booking gốc và extension.

Giá trung bình **không còn là một lựa chọn** ở bất kỳ nơi nào (câu hỏi "giữ xấp xỉ hay sửa `addSegment`" của Round 2 đóng lại).

## Hệ quả

- **PO-Q1 đã duyệt: `technical-spec-v1` KHÔNG bị sửa.** Hành vi `DAILY` nằm trong đặc tả PMS-0034 và chỉ áp dụng cho booking Hotel Website V2A đủ điều kiện; với booking `DAILY`, quy tắc PMS-0034 ghi đè các điều khoản FLAT tương ứng của V1 (§61.2, §61.6, §68, §69, §73) và chỉ cho dòng đó. Bảng ghi đè, quy tắc ưu tiên, bất biến "không chuyển đổi chế độ, không backfill": [PMS-0034 v2a mục 17](../../../hotel/docs/specs/PMS-0034-hotel-website-direct-booking-v2a.md#17-tương-thích-v1-và-thứ-tự-ưu-tiên-đặc-tả-po-q1-đã-duyệt). Gia hạn giữ nguyên breakdown gốc và lưu đêm thêm riêng.
- Cổng của BE-10b: #150 được PO duyệt **và** Codex re-review #3 xác nhận.
- Báo cáo, gia hạn, đổi ngày và `Charge.createOriginalRoomCharge` rẽ nhánh theo `pricing_mode`; nhánh `FLAT` không đổi. Danh sách đầy đủ các nơi đọc `nightlyRate` và hành động cho từng nơi: `hotel-system/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md` §2.4.
- Test V1 giữ nguyên assertion; record `ReservationRoomRevenueRow`/`StayExtensionRevenueRow` giữ constructor tương thích.

Mở ADR mới nếu PO muốn chuyển dữ liệu V1 sang `DAILY` (backfill) hoặc cho PMS Admin tạo booking `DAILY`.

## Cập nhật Round 4 (2026-11-11) — ước tính gia hạn DAILY

Bảng "reader inventory" thêm hai điểm hiển thị: `static/js/reservation/stay-extension.js` (ước tính phẳng `nights × Σ data-nightly-rate`) và `templates/reservation/stay-extension.html` (`data-nightly-rate`). Với `DAILY`, preview backend theo từng đêm là nguồn duy nhất; JS không ước tính cục bộ cho `DAILY`; `FLAT` giữ nguyên. Việc sửa JS/template/DTO thuộc BE-10b (#137). Chi tiết: [../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md) §6.4.
