# ADR 004 — Vòng đời Booking Request dùng trạng thái riêng

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-10, M0), giải quyết [D-01](../audits/issue-30-gap-analysis.md#71-chặn-bắt-đầu). [Booking Flow — Instant vs Request](../specs/client-specs.md#71-instant-vs-request)

## Bối cảnh

`Reservation` hiện chỉ có 6 trạng thái: `DRAFT`, `CONFIRMED`, `CANCELLED`, `NO_SHOW`, `CHECKED_IN`, `CHECKED_OUT`. `DRAFT` là trạng thái nhân viên đang tạo đặt phòng nội bộ trong PMS: nó không giữ tồn phòng, không có hạn xử lý và không phân biệt được một yêu cầu công khai đang chờ khách sạn duyệt với một đặt phòng nhân viên đang soạn (gap analysis G-17, finding F-02, BLOCKER).

## Quyết định

Booking Request dùng một trạng thái nghiệp vụ riêng, không tái sử dụng `DRAFT`. Request không giữ tồn phòng tại thời điểm gửi. Khi nhân viên duyệt, PMS bắt buộc kiểm tra lại availability trước khi xác nhận. Request mặc định hết hạn sau 24 giờ nếu chưa được duyệt hoặc từ chối. Giao diện công khai phải thông báo rõ ràng rằng phòng chưa được đảm bảo cho tới khi Request được xác nhận.

## Hệ quả

`hotel-system/docs/specs/PMS-0034-hotel-website-direct-booking-v2a.md` đề xuất kỹ thuật cho quyết định này: thêm trạng thái mới vào `ReservationStatus` (tên đề xuất `PENDING_APPROVAL`, cùng các nhánh từ chối/hết hạn), cột hạn xử lý và dữ liệu quyết định duyệt — đây là thiết kế đang chờ Codex và Product Owner xác nhận trước khi tạo migration thật, không phải một phần của quyết định nghiệp vụ đã chốt ở trên. Việc chốt D-01 mở khóa GOV-03, BE-10, BE-17, BE-18 trong [implementation plan](../development/issue-30-implementation-plan.md) và gap G-17.

Mở ADR mới nếu phạm vi Request thay đổi, ví dụ cho phép khách tự rút yêu cầu trước khi khách sạn duyệt.

## Cập nhật Round 3 (2026-11-11) — làm rõ thiết kế, không đổi quyết định

Trạng thái Request chốt về mặt kỹ thuật: `PENDING_APPROVAL`, `REJECTED`, `EXPIRED` (`EXPIRED` là giá trị enum riêng, không phải `REJECTED` kèm lý do). Duyệt kiểm tra hạn xử lý **sau khi** đã lấy đủ khóa và không bao giờ thành công sau hạn; hết phòng/sức chứa lúc duyệt chuyển Request sang `REJECTED` trong cùng transaction (không "giữ chờ"), khớp [ADR 014](014-po02-booking-request-price-guarantee.md). Khách không tự rút Request trong V2A (cần ADR mới nếu thay đổi). Ma trận chuyển trạng thái, thứ tự khóa và đua tranh: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §3.

## Cập nhật theo PO-Q8/Q9/Q10 (2026-11-11)

Đã duyệt: khách **không** tự rút/tự hủy Request chờ duyệt trong V2A (phải liên hệ khách sạn); `EXPIRED` là trạng thái cuối riêng, Request đã `EXPIRED` không thể được duyệt lại; lúc duyệt Room Type phải còn active và đủ điều kiện, giá giữ như đã đảm bảo lúc submit, không thay Room Type khác, thất bại ⇒ `REJECTED`. Xem [ADR 018](018-po-decisions-round3-q3-q10.md).
