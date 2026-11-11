# ADR 008 — Next.js gọi PMS qua server-side API client với xác thực service-to-service

**Trạng thái** — Đã chốt cho V2A theo quyết định Product Owner (2026-11-10, M0), giải quyết [D-05](../audits/issue-30-gap-analysis.md#71-chặn-bắt-đầu). [Hợp đồng REST cần thống nhất với PMS](../coding-rules/client-architecture.md#hợp-đồng-rest-cần-thống-nhất-với-pms)

## Bối cảnh

PMS hiện chỉ có hai security chain: `/api/**` yêu cầu JWT theo người dùng đã đăng nhập, phần còn lại là session/form-login nội bộ. Không có cơ chế API key hay tài khoản service nào cho một caller công khai, ẩn danh như website (gap analysis G-06, finding F-03, BLOCKER). Một số DTO admin hiện tại (`CreateRequest`/`RoomRequest`) còn nhận `nightlyRate`, `roomId`, `currency` từ client — tái dùng cho đường công khai sẽ cho khách tự đặt giá (finding F-21).

## Quyết định

Next.js giao tiếp PMS bằng một server-side API client (không gọi PMS trực tiếp từ browser). Giao tiếp Next.js↔PMS dùng xác thực service-to-service. Không đưa credentials hoặc bất kỳ API nội bộ (admin) của PMS ra browser.

## Hệ quả

`PMS-0034-hotel-website-direct-booking-v2a.md` và `PMS-0034-public-api-contract.md` (repo `hotel-system`) đề xuất kỹ thuật cho quyết định này: một security chain riêng cho `/api/public/v1/**`, đặt trước chain `/api/**` hiện có, xác thực bằng một khóa service (ví dụ header dạng API key tĩnh cấu hình qua biến môi trường, phạm vi chỉ gọi được các operation công khai), cùng một tài khoản/actor hệ thống riêng cho các trường `created_by`/`updated_by`/`audit_log` mà các luồng public tạo ra. DTO công khai hoàn toàn mới, tách biệt khỏi `CreateRequest`/`RoomRequest` nội bộ; PMS luôn tự tính giá, không nhận giá từ client. Thiết kế cụ thể (cơ chế khóa, allowlist IP/mạng riêng) còn chờ Codex và Product Owner xác nhận trước khi hiện thực. Việc chốt D-05 mở khóa gap G-06/G-23 và GOV-04.

Mở ADR mới nếu V2A cần nâng cấp sang OAuth2 client-credentials hoặc mTLS giữa hai hệ thống.

## Cập nhật Round 4 (2026-11-11) — fail-closed khi khởi động

Cấu hình service-to-service bắt buộc (`PMS_PUBLIC_SERVICE_KEY`, tài khoản hệ thống `system.public-website`) thiếu hoặc sai ⇒ **ứng dụng PMS không khởi động**. Không chỉ "tắt chain": khi đó route `/api/public/v1/**` rơi xuống chain JWT `/api/**`. Không có cờ tắt API công khai trong V2A; chain riêng luôn chiếm tiền tố. Thông điệp lỗi nêu tên thuộc tính và quy tắc, không in giá trị khóa. Quyết định của ADR này không đổi. Chi tiết và test: [../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md) §6.2.

## Cập nhật Round 5 (2026-11-11) — phụ thuộc triển khai

Kiểm tra khởi động đọc tài khoản hệ thống `system.public-website`, do BE-02/M01 tạo; vì vậy BE-01 phụ thuộc BE-02. Thứ tự: M01 đã áp → cấp `PMS_PUBLIC_SERVICE_KEY` an toàn cho từng môi trường (không ghi giá trị thật vào ADR, tài liệu, fixture hay Issue) → triển khai BE-01 → xác minh khởi động hợp lệ và các trường hợp âm. [round 5 §5](../../../hotel/docs/specs/PMS-0034-codex-round5-remediation-2026-11-11.md#5-r4-02--be-01-phụ-thuộc-be-02m01).
