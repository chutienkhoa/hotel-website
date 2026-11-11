# Implementation Plan — Issue #30 (PMS-0034): Hotel Website & Direct Booking V2A

**Trạng thái:** đề xuất để PO và Codex review. **Chưa phải phê duyệt triển khai**; không task nào dưới đây được bắt đầu cho tới khi qua cổng M0.
**Ngày lập:** 2026-11-10
**Đồng bộ Epic GitHub:** 2026-11-11 — thêm GOV-06 ([hotel-system#127](https://github.com/chutienkhoa/hotel-system/issues/127)) và BE-10a ([hotel-system#150](https://github.com/chutienkhoa/hotel-system/issues/150)); tổng **40 task** (xem [mục 4.1](#41-đồng-bộ-với-epic-github-hotel-system30)).
**Căn cứ:** [Gap Analysis Issue #30](../audits/issue-30-gap-analysis.md) (mã gap `G-xx`, finding `F-xx`, quyết định `D-xx`), [Client Specification V2A](../specs/client-specs.md) (mã `AC-xx`), [kiến trúc client](../coding-rules/client-architecture.md), [quy tắc frontend](../coding-rules/frontend.md), [bảo mật](../security/README.md).
**Không đổi:** phạm vi nghiệp vụ V2A đã chốt. Plan không thêm tài khoản khách, thanh toán online, hoàn tiền tự động, gallery theo phòng vật lý, rate plan nâng cao, OTA/Channel Manager, multi-tenant hay chọn phòng vật lý.

## 1. Nguyên tắc triển khai

1. **Tái sử dụng trước khi viết mới.** Giữ nguyên khóa phòng + truy vấn xung đột tồn của PMS, `AuditLog`, hạ tầng ảnh (`ImageProcessor`, `ImageAssetStore`), mẫu idempotency của Payment, `ClientIpResolver`, mẫu màn hình danh mục. Không đổi hành vi các luồng nhân viên hiện có (`AdultCapacityRules`, `cancel()`, check-in, folio) trừ khi một quyết định ghi rõ.
2. **Đường công khai tách khỏi đường quản trị.** DTO, controller, security chain, mã lỗi riêng. Không dùng lại `CreateRequest`/`RoomRequest` (nhận giá và phòng từ client).
3. **PMS là nguồn sự thật** cho giá, tồn, chế độ, chính sách và kết quả ([ADR 003](../decisions/003-pms-booking-authority.md)). Frontend không tính giá, không suy ra tồn.
4. **Một thao tác ghi = một transaction** ở PMS: kiểm tra published/active, sức chứa, khóa, gán phòng, giá, tạo reservation, snapshot, idempotency, outbox.
5. **Hợp đồng trước, mã sau.** Hai repo cùng duyệt hợp đồng và fixture ở M0; frontend làm với fixture song song với backend.
6. **Không suy đoán** endpoint, schema, trạng thái. Mọi tên trong plan là đề xuất cho tới khi hợp đồng M0 được duyệt.
7. **Quy trình PMS** (`hotel:docs/project-management/workflow.md`): chỉ làm Issue đã Ready, mỗi Issue một commit tập trung trên `develop-v2`, bàn giao local cho Codex review, chỉ commit/push khi PO cho phép riêng. Phía website theo [CLAUDE.md](../../CLAUDE.md): bàn giao diff local, không stage/commit/push khi chưa được chỉ thị.

## 2. Bức tranh tổng thể

```text
Trình duyệt ──HTTPS──► Next.js (hotel-website, 1 instance/khách sạn)
                         │  Server Components + Route Handler cùng origin
                         │  lib/pms (server-only): schema runtime, timeout, ánh xạ lỗi
                         ▼  REST /api/public/v1 (đề xuất), xác thực service (D-05)
                       Spring Boot PMS (hotel-system)
                         │  chain bảo mật riêng · DTO công khai · rate limit
                         │  Giao dịch đặt phòng nguyên tử: khóa → gán → giá → ghi → outbox
                         ▼
                       PostgreSQL        (outbox ► email)
```

Thành phần mới ở PMS (đề xuất, chờ D-07 về vị trí gói): chain `/api/public/v1/**`; nội dung website + xuất bản; trường catalog cho Room Type; gallery Room Type; giá base + override + snapshot theo đêm; bộ gán phòng + quy tắc sức chứa khách; tìm kiếm/quote; ghi nhận idempotency; chính sách hủy + snapshot; outbox email; Manage Booking; (tùy D-01) trạng thái Request.

## 3. Cổng và thứ tự

### 3.1 Milestones

| Mốc | Mục tiêu | Task | Ra khỏi mốc khi |
| --- | --- | --- | --- |
| **M0** Cổng | Quy trình, quyết định, hợp đồng, thiết kế | GOV-01…GOV-06 | Issue #30 Ready (hoặc Issue con), D-01…D-09 chốt, hợp đồng v1 + fixture được hai repo duyệt, mockup website (GOV-05) đã duyệt. GOV-06 (mockup PMS Admin) thuộc M0 theo Issue nhưng **chỉ chặn phần UI quản trị liên quan**, không chặn backend/API và không chặn các task website |
| **M1** Site đọc | Website hiển thị nội dung công bố và catalog, chưa đặt phòng | BE-01, 02, 04, 05, 06, 07, 08 · FE-01…FE-05 | AC-01, 02, 03, 20 (phần nội dung) đạt trên PMS test + staging |
| **M2** Giá và tìm phòng | Lịch giá, snapshot theo đêm, tìm phòng, quote | BE-03, 09, **10a**, 10, 11, 12, 13 · FE-06 | AC-04, 05, 08, 09, 19 đạt |
| **M3** Instant | Đặt phòng Instant đầu-cuối + email xác nhận | BE-14, 15, 16, 17 · FE-07 | AC-06, 07, 10, 12, 13, 14, 22 đạt (kèm test đồng thời) |
| **M4** Request, Manage, Hủy | Request, tra cứu an toàn, tự hủy | BE-18, 19, 20 · FE-08 | AC-11, 15, 16, 17, 18 đạt |
| **M5** Phát hành | Hardening, domain, staging, nghiệm thu | BE-21, 22 · FE-09, 10, 11 | AC-01…22 + WEB-01…10 có bằng chứng; PO nghiệm thu |

### 3.2 Đường găng

`GOV-01 → GOV-03 (D-02, D-05) → GOV-04 → BE-09 → BE-10a → BE-10 → BE-13 → BE-17 → BE-19 → BE-20 → BE-21` (BE-10a là spike thiết kế phải được PO duyệt trước khi BE-10 viết migration; BE-18 nối sau BE-17 theo [mục 14.3](#143-đường-găng-cập-nhật)).
Frontend không nằm trên đường găng nếu dùng fixture: FE-01/02 bắt đầu ngay sau GOV-04; FE-03 sau GOV-05. GOV-06 không nằm trên đường găng vì chỉ chặn phần UI quản trị PMS.

### 3.3 Chạy song song

| Làn | Tuần tự trong làn | Chạy song song với |
| --- | --- | --- |
| PMS-nội dung | BE-04 → 05 → 06 → 07 → 08 | PMS-giá (sau BE-02) |
| PMS-giá | BE-09 → 10a → 10; BE-11 (sau BE-09, không chờ BE-10a) | PMS-nội dung; BE-12 |
| Thiết kế | GOV-05 (website) ‖ GOV-06 (PMS Admin) | Mọi task backend/API; GOV-06 chỉ gate phần UI quản trị của BE-04, 06, 07, 11, 16, 18 |
| PMS-đặt phòng | BE-12 → 13 → 14 → 17 | BE-15, 16 |
| PMS-hậu đặt phòng | BE-18 → 19 → 20 | FE-08 (fixture) |
| Frontend | FE-01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 | Toàn bộ làn PMS (dùng fixture đến khi tích hợp) |

### 3.4 Đồ thị phụ thuộc quanh hai task bổ sung

Mũi tên `A → B` nghĩa là B phụ thuộc A. Chỉ vẽ các cạnh liên quan GOV-06 và BE-10a; các cạnh còn lại xem cột "Phụ thuộc" ở mục 4 và [mục 14.2](#142-cập-nhật-dependencies-task-catalog-mục-4).

```text
BE-09 ──► BE-10a ──(PO duyệt spike)──► BE-10 (BE-10b) ──► BE-16 ──► BE-17
  │                                         │                        ▲
  └────► BE-11                              └────────────────────────┘

GOV-06 ··(chỉ phần UI quản trị)··► BE-04 · BE-06 · BE-07 · BE-11 · BE-16 · BE-18
GOV-05 ··(UI website)·············► FE-03 … FE-08
```

Nét liền là phụ thuộc bắt buộc theo Issue GitHub. Nét chấm (GOV-05, GOV-06) là cổng thiết kế chỉ chặn phần giao diện; cạnh GOV-06 đã được ghi vào Dependencies của các Issue BE tương ứng (mục 4.1).

## 4. Danh mục task

Cỡ: **S** (≤ vài ngày), **M** (~1 tuần), **L** (1–2 tuần), **XL** (> 2 tuần hoặc cần spike). Cỡ chỉ để so sánh tương đối, không phải cam kết thời gian.

| ID | Repo | Tên | Gap | Phụ thuộc | Mốc | Cỡ |
| --- | --- | --- | --- | --- | --- | --- |
| GOV-01 | hotel-system | PO duyệt phạm vi và tiêu chí, tách Issue con, chuyển Ready | G-28 | — | M0 | S |
| GOV-02 | hotel-system | Đặc tả PMS-0034 trong PMS, cập nhật loại trừ pms-0040, quy ước gói mã | G-28, G-01 | GOV-01 | M0 | M |
| GOV-03 | cả hai | Chốt quyết định D-01…D-09, ghi ADR | F-01…F-09 | GOV-01 | M0 | M |
| GOV-04 | cả hai | Hợp đồng Public API v1, mã lỗi, fixture | G-06 | GOV-03 | M0 | L |
| GOV-05 | hotel-website | Mockup Modern Tropical đã duyệt vào repo | G-27 | — | M0 | M |
| GOV-06 | hotel-system | Mockup PMS Admin cho PMS-0034 được PO duyệt (Room Catalog, Gallery, Rate Calendar, Booking Request, Website Settings) | G-27 (bổ sung — PMS Admin) | — (gate UI cho BE-04, 06, 07, 11, 16, 18) | M0 | M |
| BE-01 | hotel-system | Khung API công khai: chain bảo mật, xác thực service, lỗi, rate limit | G-06, G-23 | GOV-03, GOV-04, **BE-02** (actor hệ thống cho kiểm tra khởi động; Round 5) | M1 | L |
| BE-02 | hotel-system | Migration nền: actor hệ thống, nguồn `HOTEL_WEBSITE`, quyền mới | F-08, F-16 | GOV-02, GOV-03 | M1 | M |
| BE-03 | hotel-system | Giờ khách sạn và quy tắc đặt (min/max stay, đặt trước, ngày quá khứ) | G-25 | GOV-03 | M2 | M |
| BE-04 | hotel-system | Mô hình nội dung website và màn hình quản trị | G-01 | BE-02; GOV-06 (UI) | M1 | L |
| BE-05 | hotel-system | Xuất bản: draft/published, preview, version/ETag, tín hiệu làm mới | G-02 | BE-04 | M1 | L |
| BE-06 | hotel-system | Catalog Room Type: bật bán, thứ tự, slug, nội dung, tiện nghi | G-04 | BE-04; GOV-06 (UI) | M1 | L |
| BE-07 | hotel-system | Gallery Room Type: cover, thứ tự, alt text | G-05 | BE-06; GOV-06 (UI) | M1 | L |
| BE-08 | hotel-system | API đọc công khai: site config, branding, catalog, gallery, chính sách | G-03, G-04, G-05 | BE-01, 05, 06, 07 | M1 | M |
| BE-09 | hotel-system | Mô hình giá: base rate, daily override, bộ tính giá hiệu lực | G-11, G-12 | GOV-03 (D-02), BE-02 | M2 | L |
| BE-10a | hotel-system | Spike thiết kế tương thích tài chính cho nightly price snapshot (financial report, folio/charge) | G-14, F-01 (BLOCKER-01) | BE-09 | M2 | M |
| BE-10 | hotel-system | Snapshot giá theo đêm và tương thích folio/charge | G-14 | BE-09, **BE-10a** (PO duyệt trước BE-10b) | M2 | XL |
| BE-11 | hotel-system | Room Rate Calendar: sửa 1 ngày, bulk, preview, lịch sử | G-13 | BE-09; GOV-06 (UI) | M2 | L |
| BE-12 | hotel-system | Quy tắc sức chứa khách và bộ gán phòng | G-09, G-07, G-10 | GOV-03 (D-03, D-04, **PO-02**), BE-06 | M2 | L |
| BE-13 | hotel-system | Tìm phòng theo Room Type, Sold Out, quote | G-08, G-15 | BE-03, 06, 09, 12 | M2 | L |
| BE-14 | hotel-system | Kho idempotency và đối soát kết quả chưa rõ | G-22 | BE-01, **BE-02** (actor hệ thống cho `created_by`, migration M01 trước M09) | M3 | M |
| BE-15 | hotel-system | Outbox email và dispatcher (xác nhận, link quản lý) | G-21 | GOV-03 (D-08) | M3 | L |
| BE-16 | hotel-system | Cấu hình chính sách hủy và snapshot trên reservation | G-19 | BE-03, BE-04, **BE-10** (PO-03); GOV-06 (UI) | M3 | M |
| BE-17 | hotel-system | Đặt phòng công khai Instant (nguyên tử) | G-16, G-07, G-10 | BE-01, 02, 10, 12, 13, 14, 15, 16 | M3 | XL |
| BE-18 | hotel-system | Chế độ Request: vòng đời, duyệt/từ chối, hết hạn | G-17 | GOV-03 (D-01, **PO-01, PO-02**), BE-17, **BE-02**; GOV-06 (UI) | M4 | XL |
| BE-19 | hotel-system | Manage Booking: yêu cầu truy cập, token, đọc chi tiết | G-18 | BE-15, BE-17 | M4 | L |
| BE-20 | hotel-system | Tự hủy: cutoff, idempotent, audit, email | G-20 | BE-16 (**+PO-04**), BE-19 | M4 | L |
| BE-21 | hotel-system | Bộ test đồng thời, hợp đồng, bảo mật (xuyên suốt) | G-10, G-22, G-23 | tăng dần từ BE-17 | M3→M5 | L |
| BE-22 | hotel-system | Tài liệu vận hành, cấu hình bí mật, domain, runbook | G-24 | BE-17 | M5 | M |
| FE-01 | hotel-website | Khởi tạo Next.js App Router, TypeScript strict, Tailwind, lint/test | G-26 | GOV-03 | M1 | M |
| FE-02 | hotel-website | Client PMS server-only, schema runtime, fixture, mock server, cấu hình môi trường | G-26 | GOV-04, FE-01 | M1 | L |
| FE-03 | hotel-website | Registry template, vỏ Modern Tropical, token theme, UI dùng chung | G-26, G-27 | GOV-05, FE-01 | M1 | L |
| FE-04 | hotel-website | Trang chủ và hạ tầng SEO | G-01, G-03 | FE-02, FE-03 | M1 | M |
| FE-05 | hotel-website | Danh mục phòng, chi tiết, gallery, liên hệ, chính sách | G-04, G-05 | FE-04 | M1 | L |
| FE-06 | hotel-website | Booking Engine: tìm, chọn phòng, báo giá | G-08, G-15 | FE-02, FE-05 | M2 | L |
| FE-07 | hotel-website | Booking Engine: thông tin khách, review, submit, kết quả, chưa rõ kết quả | G-16, G-17, G-22 | FE-06; BE-17, BE-18 (tích hợp) | M3 | XL |
| FE-08 | hotel-website | Manage Booking: yêu cầu truy cập, xem, hủy | G-18, G-20 | FE-07; BE-19, BE-20 (tích hợp) | M4 | L |
| FE-09 | hotel-website | Bảo mật và vận hành phía Next.js | G-23 | FE-07 | M5 | M |
| FE-10 | hotel-website | Bộ test E2E, accessibility, responsive, bằng chứng | G-26 | FE-07, FE-08 | M5 | L |
| FE-11 | hotel-website | Triển khai: Docker, Caddy, staging, domain, canonical, làm mới cache | G-24 | FE-09, BE-22 | M5 | M |

**Quy ước cột Phụ thuộc:** "GOV-06 (UI)" là phụ thuộc có điều kiện, chỉ chặn phần giao diện PMS Admin; backend/API/DB/test của task vẫn tiến hành độc lập và task không bị coi là BLOCKED chỉ vì GOV-06 chưa xong. "(tích hợp)" ở FE-07/FE-08 nghĩa là task website bắt đầu và làm bằng fixture sau khi phụ thuộc frontend xong, nhưng chỉ hoàn tất khi BE tương ứng sẵn sàng. In đậm là thay đổi từ mục 14.2 (Codex review, PO-01…PO-04). Bảng này khớp mục Dependencies của 40 Issue GitHub (đối chiếu 2026-11-11).

### 4.1 Đồng bộ với Epic GitHub `hotel-system#30`

Đối chiếu ngày 2026-11-11 với danh sách sub-issue của `chutienkhoa/hotel-system#30` (40 Issue, đều Open). Cột "Repo" ở mục 4 là nơi thực hiện theo kế hoạch; GOV-03 và GOV-04 là task hai repo nhưng Issue GitHub nằm ở `hotel-system`.

| Nhóm | Số task | Task |
| --- | --- | --- |
| GOV | **6** | GOV-01…GOV-06 |
| BE (PMS) | **23** | BE-01…BE-22 và **BE-10a** |
| FE (website) | **11** | FE-01…FE-11 |
| **Tổng** | **40** | trước đồng bộ: 38 (5 GOV + 22 BE + 11 FE) |

Theo repo của Issue GitHub: `hotel-system` 28 Issue (GOV-01…04, GOV-06, BE-01…22, BE-10a), `hotel-website` 12 Issue (GOV-05, FE-01…11).

| Task | Issue GitHub |
| --- | --- |
| GOV-01…GOV-04 | hotel-system#123, #124, #125, #126 |
| **GOV-06** | **hotel-system#127** |
| BE-01…BE-09 | hotel-system#128…#136 |
| BE-10 | hotel-system#137 |
| BE-11…BE-20 | hotel-system#138…#147 |
| BE-22, BE-21 | hotel-system#148, #149 |
| **BE-10a** | **hotel-system#150** |
| GOV-05 | hotel-website#1 |
| FE-01…FE-11 | hotel-website#2…#12 |

Truy vết hai task bổ sung:

| Task | Gap / Finding | Nguồn | Quyết định liên quan | Chặn | Bị chặn bởi |
| --- | --- | --- | --- | --- | --- |
| GOV-06 | G-27 (bổ sung PMS Admin) | [mockup-requirements-issue-30.md](../designs/mockup-requirements-issue-30.md) mục 2; GOV-05 chỉ phủ website công khai | D-01, D-02, D-04, D-07 | Phần **UI quản trị** của BE-04, 06, 07, 11, 16, 18 | — |
| BE-10a | G-14, F-01 (BLOCKER-01, Codex review 2026-11-11) | [mục 14.1](#141-task-mới--be-10a-tách-khỏi-be-10-bắt-buộc-trước-be-10b); [ADR 005](../decisions/005-nightly-pricing-snapshot.md), [ADR 015](../decisions/015-po03-extension-date-change-pricing.md) | D-02, PO-03 | BE-10 (phần BE-10b), và gián tiếp BE-16, BE-17 | BE-09 |

**Đã đồng bộ 2026-11-11:** Dependencies của BE-04, 06, 07, 11, 16, 18 trên GitHub đã thêm GOV-06 (hotel-system#127) dưới dạng phụ thuộc có điều kiện cho phần UI PMS Admin; BE-20 (hotel-system#147) đã ghi PO-04. Không tạo Issue riêng cho BE-10b: BE-10b là phần code của Issue BE-10 (hotel-system#137), chặn bởi BE-10a (hotel-system#150).

## 5. Task cổng (GOV)

### GOV-01 Duyệt phạm vi và tách Issue #30

- **Việc:** PO đưa tiêu chí AC-01…AC-22 vào Issue #30, duyệt phạm vi, xác nhận #30 là epic và tạo Issue con ánh xạ theo bảng task BE (workflow PMS: một Issue một commit; Issue/comment bằng tiếng Việt). Đưa Issue ra Backlog → Ready. Chỉ PO duyệt; Claude chỉ chuyển Backlog → Ready theo chỉ thị.
- **Acceptance:** mỗi Issue con có phạm vi, tiêu chí truy được về `client-specs.md`, phụ thuộc, và trạng thái Ready; #30 liệt kê Issue con; #33 (email) và #89/#90 được ghi quan hệ.
- **Ghi chú:** điều kiện mở khóa V2 đã thỏa (PMS-0028 Done, PMS-0053 CLOSED); snapshot `project-status.json` đang cũ.

### GOV-02 Đặc tả PMS-0034 trong PMS

- **Việc:** thêm mục PMS-0034 vào `hotel:docs/requirements/technical-spec-v2.md` (hoặc tài liệu đặc tả riêng được trỏ từ đó), tóm tắt hành vi từ `client-specs.md` §2–§10 làm nguồn cho PMS; sửa các câu loại trừ liên quan trong `pms-0040-hotel-settings.md` (chính sách hủy, API cho cài đặt, theme) sau khi PO duyệt; quy ước vị trí mã (miền `room`/`booking`/`common`, hoặc xin duyệt miền mới theo `architecture.md` mục 14–15).
- **Acceptance:** PO duyệt đặc tả; ghi rõ hai định nghĩa sức chứa (nhân viên so với công khai) nếu D-03 = chỉ đường công khai; không sửa hành vi V1 nào ngoài những điều đã duyệt.

### GOV-03 Chốt quyết định và ghi ADR

- **Việc:** PO xác nhận D-01…D-09 ([gap analysis mục 7](../audits/issue-30-gap-analysis.md#7-quyết-định-cần-xác-nhận)). Kết quả ghi thành ADR mới trong `docs/decisions/` (004 trở đi) và vào đặc tả PMS.
- **Acceptance:** mỗi D có quyết định, lý do, phương án đã loại, hệ quả; task bị chặn được mở.

### GOV-04 Hợp đồng Public API v1

- **Việc:** viết hợp đồng cho các khả năng: site config và branding, catalog và chi tiết, gallery, chính sách, tìm phòng (giá tham khảo) + quote lưu server-side, tạo booking (Instant/Request), đối soát theo khóa idempotency, yêu cầu truy cập Manage, đọc booking, hủy. Mỗi operation: method/path (`/api/public/v1`), xác thực, request/response, lỗi có mã máy đọc, header (`Idempotency-Key`, request-id), cache (`no-store` cho giao dịch), ví dụ và **fixture đã lọc** cho cả thành công lẫn lỗi.
- **Round 3 (2026-11-11):** phong bì lỗi chuẩn duy nhất (lỗi không nằm trong body 200), bảng 24 mã lỗi có `details` theo mã, quote ràng buộc basket + occupancy + booking mode + revision giá + revision chính sách, gallery qua proxy Next.js. **Đã có** JSON Schema (18) và fixture (52, gồm một fixture cho mỗi mã lỗi, đã kiểm hợp lệ) tại `docs/development/contracts/public-api-v1/` — xem `README.md` ở đó. Mô tả hành vi: `hotel-system/docs/specs/PMS-0034-public-api-contract.md`. Còn thiếu để đạt Acceptance: review đồng thuận hai repo; quyết định có cần OpenAPI YAML hay không; cập nhật [api-integration.md](api-integration.md) khi chốt.
- **Acceptance:** hai repo cùng duyệt; mọi trường DTO là allowlist; không có trường PII ngoài những gì cần cho thao tác; fixture khớp schema (kiểm tự động trong FE-02/BE-21); điều kiện tương thích khi triển khai/rollback được ghi lại.
- **Chặn bởi:** D-01 (Request), D-02, D-05.
- **Round 4 (F4, M1, M3):** `roomTypeCode` = mã V1 nguyên văn (schema 1–64 ký tự, không khoảng trắng, tập cha của quy tắc V1); reconcile union theo `state`; status `201` thống nhất và replay ở header; thêm `negative/negative-controls.json` (31), `corpus/room-type-code.json` (26 mã), 55 tệp fixture. **AC bổ sung:** mọi mã V1 hợp lệ qua schema; mọi negative control bị từ chối; fixture dùng `DBL-SEA` và mã 40 ký tự. [round 4 §5–§6](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### GOV-05 Mockup đã duyệt

- **Việc:** PO/nhà thiết kế cung cấp mockup đã duyệt (home, catalog, detail, search/select, guest info, review, result Instant/Request, unknown outcome, manage booking, hủy; mobile 390, tablet 768, desktop 1440; trạng thái loading/empty/error/sold-out) đặt trong `docs/designs/mockups/` theo quy tắc đặt tên; ghi nguồn/quyền ảnh.
- **Acceptance:** [designs/README.md](../designs/README.md) được tuân thủ; mỗi mockup ghi trạng thái "đã duyệt".

### GOV-06 Mockup PMS Admin đã duyệt

- **Repo thực hiện:** `hotel-system` ([hotel-system#127](https://github.com/chutienkhoa/hotel-system/issues/127), mốc M0, cỡ M). Danh mục màn hình nằm trong repo này ([mockup-requirements-issue-30.md mục 2](../designs/mockup-requirements-issue-30.md)) vì đây là nơi quản lý thiết kế chung của Issue #30.
- **Việc:** PO duyệt mockup cho các màn hình PMS Admin mới: quản lý Room Type Gallery (BE-07); chọn Room Type bán trên website và sắp xếp (BE-06); Room Rate Calendar (BE-11); cấu hình Instant Booking/Booking Request và chính sách hủy (BE-04, BE-16); hàng đợi duyệt Booking Request (BE-18); Website Settings mở rộng (BE-04). Dùng khuôn Thymeleaf hiện có của PMS Admin, không cần design system riêng trừ khi PO yêu cầu. Dữ liệu giả.
- **Điều kiện:** mockup được PO phê duyệt **trước khi implement** các màn hình PMS Admin tương ứng. Chỉ chặn phần UI; phần backend/API/migration vẫn làm được bằng fixture theo nguyên tắc hợp đồng trước, mã sau (không phải chờ GOV-06).
- **Phân biệt với GOV-05:** GOV-05 chỉ phủ website công khai (Modern Tropical); GOV-06 chỉ phủ PMS Admin. Hai task độc lập, chạy song song.
- **Acceptance:** mỗi màn hình ở mục 2 của `mockup-requirements-issue-30.md` có mockup đã duyệt; mockup bám khuôn UI PMS hiện có; không có dữ liệu khách thật.

## 6. Task PMS (`hotel-system`)

Mọi task PMS: Flyway migration an toàn theo `database-standards.md`; Javadoc theo `java.md`; test Testcontainers theo tiền lệ `*MigrationTest`/`*IntegrationTest`; `./mvnw` chạy trong phạm vi tối thiểu, ghi lệnh và kết quả; không đổi hành vi các endpoint `/api/**` hiện có.

### BE-01 Khung API công khai

- **Gap:** G-06, G-23 · **F-03, F-21** · **Phụ thuộc:** GOV-03 (D-05), GOV-04, **BE-02** (Round 5, R4-02: kiểm tra khởi động cần actor `system.public-website` do M01 tạo).
- **Nội dung:**
  - `SecurityFilterChain` mới với thứ tự **trước** chain `/api/**` hiện tại, `securityMatcher` của không gian công khai; stateless, không CORS (chỉ Next.js server gọi), giới hạn kích thước body, không cookie/session.
  - Xác thực service theo D-05 (đề xuất: API key phạm vi hẹp, lưu hash, quay vòng được, ghi nhận ở cấu hình bí mật theo `docs/operations/configuration-and-secrets.md`), quyền chỉ các operation công khai.
  - Phong bì lỗi chuẩn (mã máy đọc, thông điệp an toàn, `requestId`), bộ lọc request-id, `Cache-Control: no-store` mặc định cho giao dịch.
  - Rate limit theo IP khách đầu cuối do Next.js chuyển tiếp qua kênh tin cậy (dùng `ClientIpResolver`/proxy tin cậy), và theo khóa dịch vụ; nơi lưu trong bộ nhớ như `LoginFailureTracker`, ghi nhận giới hạn một instance.
  - Quy tắc log: chỉ tên thao tác, mã lỗi, request-id, thời gian; không PII/token.
- **Acceptance:**
  - Không có khóa → 401; khóa sai phạm vi → 403; body quá lớn → 413; vượt tần suất → 429 với phản hồi an toàn.
  - Giả `Host`, `X-Forwarded-For`, `X-Forwarded-Host` không đổi IP tính giới hạn khi không đến từ proxy tin cậy.
  - Các endpoint `/api/**` JWT hiện có và luồng đăng nhập form **không đổi hành vi** (test hồi quy `SecurityConfig`).
  - Không có log chứa email, điện thoại, tên, token (kiểm tra bằng test bắt log).
- **Kiểm thử:** integration (MockMvc + Testcontainers), unit cho bộ giới hạn, test hồi quy hai chain.
- **Round 4 (M2):** cấu hình S2S bắt buộc thiếu/sai (`PMS_PUBLIC_SERVICE_KEY` thiếu, rỗng, < 32 ký tự, placeholder; actor `system.public-website` thiếu/active) ⇒ **ứng dụng không khởi động** (không chỉ "tắt chain"); thông điệp không in giá trị khóa; không cờ tắt; chain `@Order(0)` luôn chiếm tiền tố nên không rơi xuống chain JWT. Test: khởi động thất bại từng trường hợp, không rò khóa, `401` từ chain công khai, bearer JWT vào tiền tố công khai ⇒ `401`. Chi tiết: [round 4 §6.2](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).
- **Round 5 (R4-02) — thứ tự triển khai:** (1) BE-02/M01 và khởi tạo permission/actor hoàn tất; (2) cấp `PMS_PUBLIC_SERVICE_KEY` an toàn cho từng môi trường (không ghi giá trị thật vào tài liệu, fixture hay Issue); (3) triển khai BE-01 với kiểm tra khởi động bật; (4) xác minh khởi động hợp lệ và các trường hợp âm (khóa thiếu/rỗng/ngắn/placeholder, actor thiếu, actor active ⇒ không khởi động, thông điệp không in khóa). Trình kiểm tra chạy **sau Flyway**; test Testcontainers cơ sở dữ liệu trống. [round 5 §5](../../../hotel/docs/specs/PMS-0034-codex-round5-remediation-2026-11-11.md#5-r4-02--be-01-phụ-thuộc-be-02m01).

### BE-02 Migration nền

- **Gap:** F-08, F-16 · **Phụ thuộc:** GOV-02, GOV-03. **Round 3:** là migration **M01 — đứng đầu** thứ tự migration của Issue #30 ([PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §10).
- **Nội dung:** (1) actor hệ thống `system.public-website` (`app_user` + cột `system_account`, `CHECK` chặn kích hoạt, không `user_role`, không đăng nhập được qua form lẫn JWT — **phải có test cho cả hai đường**; `UserService` ẩn/từ chối sửa); (2) permission mới và `role_permission`: `MANAGE_BOOKING_REQUEST` (PO-01, PO-Q3: MANAGER có mặc định, ADMIN thu hồi được, STAFF_NEVER) và **một** permission `MANAGE_WEBSITE_CONFIG` (PO-Q4: ADMIN mặc định, vai trò khác chỉ khi cấp tường minh; không `ADMIN_ONLY`; không `VIEW_*` riêng), kèm thêm vào `RolePermissionPolicy.APPLICATION_PERMISSIONS` + test ma trận (code, không chỉ SQL); backend kiểm permission hiệu lực, không theo tên vai trò; (3) **không** seed hay kích hoạt nguồn `HOTEL_WEBSITE` (PO-Q5): đường public kiểm tra nguồn tồn tại ∧ active ∧ hợp lệ và fail-closed (`BOOKING_UNAVAILABLE`/`SOURCE_NOT_CONFIGURED`), không fallback `DIRECT`, không kích hoạt lại nguồn inactive, không ghi đè cấu hình, không tạo nguồn như tác dụng phụ; chẩn đoán `SOURCE_MISSING/INACTIVE/INVALID` hiện ở Website Settings; ADMIN tạo và kích hoạt nguồn trước go-live.
- **Acceptance:** migration đạt `Flyway Migration Integrity`; không đổi quyền/vai trò/`booking_source` hiện hữu (test so dữ liệu trước/sau); actor không đăng nhập được; ma trận quyền khớp quyết định PO; migration fail rõ (dữ liệu không đổi) khi username actor đã bị chiếm; bản ghi audit của đường public truy được về actor.
- **Quyết định PO đã duyệt:** PO-Q3, PO-Q4, PO-Q5 (không còn chặn). Còn diễn giải O1 (STAFF_NEVER), O2 (Rate Calendar/Gallery giữ `MANAGE_ROOM_TYPE`) ở round 3 §13.2.
- **Round 4 (F1):** đồng bộ catalog quyền đủ tám điểm: `APPLICATION_PERMISSIONS` (25→27), `STAFF_NEVER_PERMISSIONS` (chỉ `MANAGE_BOOKING_REQUEST`), `RolePermissionService.GROUPS` (nếu lệch, context không khởi động), `STAFF_NEVER_REJECTIONS`, seed M01, thông điệp VI/EN, làm mới authority (đã xác minh), cập nhật test cứng (`RolePermissionPolicyTest` 25→27, `RolePermissionIntegrationTest` 26/25→28/27). **AC bổ sung:** Spring context khởi động; ADMIN thu hồi/cấp lại `MANAGE_BOOKING_REQUEST` cho MANAGER có hiệu lực ở request kế tiếp qua `ActiveUserSessionFilter` thật và `JwtFilter`; STAFF không nhận được; `MANAGE_WEBSITE_CONFIG` mặc định ADMIN. Test P01–P14: [round 4 §2.4](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### BE-03 Giờ khách sạn và quy tắc đặt

- **Gap:** G-25 · **F-09, F-12** · **Phụ thuộc:** GOV-03 (D-06, D-11).
- **Nội dung:** thành phần "giờ khách sạn" dùng cho đường công khai: ngày hiện tại, cutoff (check-in date + giờ check-in tiêu chuẩn − cửa sổ hủy) theo D-06 (đề xuất dùng đúng `Clock` hiện có; cảnh báo vận hành nếu `hotel_setting.time_zone` khác zone của `Clock`); quy tắc đặt cấu hình được: ngày nhận không ở quá khứ, min/max số đêm, cửa sổ đặt trước, tối đa phòng/khách mỗi lần; mã lỗi tương ứng. Không đổi `Clock`, `DisplayFormats`, hay Confirm/Create nhân viên (Issue #90 vẫn mở riêng).
- **Acceptance:** AC-19: cùng đầu vào cho cùng kết quả khi chạy JVM/trình duyệt ở múi giờ khác; kiểm tra biên 23:59/00:00 theo giờ khách sạn; đặt ngày quá khứ bị từ chối ở đường công khai nhưng không đổi hành vi nhân viên.

### BE-04 Nội dung website và màn hình quản trị

- **Gap:** G-01 · **F-13** · **Phụ thuộc:** BE-02; GOV-06 (có điều kiện, chỉ UI quản trị).
- **Nội dung:** (sở hữu migration **M07a** `cancellation_policy_revision`: giờ hủy 24/48/72 + văn bản theo phiên bản, append-only) bảng singleton có `version` (khóa lạc quan) cho nội dung website: tagline, giới thiệu, hero (ảnh qua kho ảnh sẵn có), điểm nổi bật, liên hệ/bản đồ (liên kết), kênh liên hệ, mã template (trong danh sách hỗ trợ), màu thương hiệu (token cho phép), **chế độ Instant/Request (mặc định Instant)**, cửa sổ hủy 24/48/72h (mặc định 24h), thông tin domain hiển thị, nội dung Privacy/Terms/Cancellation dạng text. Màn hình Thymeleaf theo `frontend.md`/`responsive-design.md` của PMS (mục Website Settings: General, Appearance, Homepage, Contact & Location, Booking & Publishing, Domain). Chỉ text thuần, không HTML/JS tùy ý.
- **Acceptance:** validate và giới hạn độ dài; chỉ người có permission `MANAGE_WEBSITE_CONFIG` (PO-Q4; ADMIN mặc định, vai trò khác cần cấp tường minh; backend kiểm permission hiệu lực, kể cả gọi trực tiếp); audit mọi thay đổi; không ảnh hưởng `hotel_setting` và PMS-0040 (test `HotelSettingBusinessRulesUnchangedIntegrationTest` vẫn đạt); responsive 320/390/768/1440.

### BE-05 Xuất bản, preview, làm mới cache

- **Gap:** G-02 · **F-14** · **Phụ thuộc:** BE-04.
- **Nội dung:** tách bản nháp và bản đã xuất bản; hành động Publish có quyền riêng và audit; preview có token ngắn hạn, phản hồi `noindex`; `contentVersion`/ETag trên mọi DTO công bố; cơ chế làm mới cache (webhook revalidate có bí mật tới Next.js, hoặc Next.js hỏi version; cách làm mới chốt ở GOV-04). Cache chỉ cho nội dung công bố; dữ liệu giao dịch không bao giờ cache.
- **Acceptance:** AC-03: đổi thứ tự phòng/cover sau Publish phản ánh trên website sau tối đa một chu kỳ làm mới đã định; nội dung nháp không rò qua API công khai; preview không dùng được khi hết hạn/không token; Publish ghi audit actor/time.

### BE-06 Catalog Room Type

- **Gap:** G-04 · **F-12** · **Phụ thuộc:** BE-04; GOV-06 (có điều kiện, chỉ UI quản trị).
- **Nội dung:** cấu hình website cho Room Type: bật bán, thứ tự hiển thị, slug duy nhất ổn định, mô tả công khai, tiện nghi (danh mục khóa cố định); điều kiện hiển thị = active ∧ bật bán ∧ published ∧ có capacity. Màn hình quản trị trong khu vực Room Type. Quy tắc tắt bán không đổi reservation hiện có hay bán nội bộ.
- **Acceptance:** AC-02: Room Type ẩn/inactive không xuất hiện ở catalog/detail và **bị từ chối khi tạo booking qua API công khai** (test cả khi client giữ dữ liệu cũ); slug trùng bị chặn; đổi tên không đổi slug âm thầm; Room Type thiếu capacity không thể công bố.

### BE-07 Gallery Room Type

- **Gap:** G-05 · **F-07** · **Phụ thuộc:** BE-06; GOV-06 (có điều kiện, chỉ UI quản trị).
- **Nội dung:** bảng ảnh theo Room Type (cover duy nhất, thứ tự, alt text), upload/xóa/sắp xếp/đổi cover; tái dùng `ImageInspector`/`ImageProcessor`/`ImageAssetStore`/variant; giới hạn số ảnh và dung lượng theo D-13; xóa cover thì chọn cover thay thế hoặc fallback rõ; phục vụ ảnh công khai qua variant với cache có version. **Không** migrate hay thay đổi `room_image`.
- **Acceptance:** MIME/kích thước/nội dung được kiểm tra, file nguy hiểm bị từ chối; không mất ảnh khi lỗi giữa chừng (nguyên tử); alt text hoặc fallback; thứ tự ổn định; `room_image` của nhân viên không đổi.

### BE-08 API đọc công khai

- **Gap:** G-03, G-04, G-05 · **Phụ thuộc:** BE-01, 05, 06, 07.
- **Nội dung:** operation đọc: site config + branding (tên, logo/favicon có phiên bản, tagline, liên hệ, template, chế độ, chính sách hiển thị), catalog, chi tiết theo slug, gallery, văn bản chính sách; ETag/Cache-Control cho nội dung công bố; 404 không tiết lộ Room Type chưa công bố.
- **Acceptance:** chỉ trả trường allowlist; test hợp đồng khớp fixture GOV-04; Room Type không đủ điều kiện → 404; không có trường nội bộ, giá chưa công bố hay cấu hình bí mật.

### BE-09 Mô hình giá

- **Gap:** G-11, G-12 · **F-01** · **Phụ thuộc:** GOV-03 (D-02), BE-02.
- **Nội dung:** base rate theo Room Type (quyết định dùng `room_type.base_price` hay bảng mới có tiền tệ), daily override duy nhất một giá hiệu lực cho `(Room Type, ngày)`, dịch vụ giá hiệu lực: đêm `D` = override(D) nếu có, ngược lại base; tính từ check-in đến **trước** check-out; validation theo `SupportedCurrency` (VND scale 0); audit; Room Type không có giá hợp lệ ⇒ không bán/không hiển thị giá.
- **Acceptance:** AC-08: một đêm base + một đêm override cho tổng đúng, checkout không tính thêm đêm, currency đúng; ràng buộc DB chống trùng; không có logic giá trong frontend.
- **Round 4 (F3):** hàng `room_type_pricing_revision` tạo ngay khi Room Type tồn tại (trigger `AFTER INSERT ON room_type`, `revision = 0`), thay cách "tạo lười biếng"; AC: Room Type mới có hàng revision, quote và `FOR SHARE` không gặp hàng thiếu. [round 4 §4.4](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### BE-10a Spike thiết kế tương thích tài chính (nightly pricing)

- **Repo thực hiện:** `hotel-system` ([hotel-system#150](https://github.com/chutienkhoa/hotel-system/issues/150), mốc M2, cỡ M, loại: design spike) · **Gap:** G-14 · **F-01 / BLOCKER-01** · **Phụ thuộc:** BE-09. (`#149` là BE-21, không phải BE-10a.)
- **Điều kiện:** thiết kế tương thích **financial report, folio/charge và nightly pricing** được PO duyệt **trước BE-10b và migration M06**. Không implement code hay migration thật.
- **Kết quả Round 3 (thiết kế đã viết, chờ review/duyệt):** giả định "`nightly_rate` là giá trung bình phái sinh" **bị loại bỏ** (làm tròn phá `verifySnapshot`; phân bổ sai khi vắt tháng, sai với `CHECKED_IN` và dòng Excel theo đêm). Thay bằng hai chế độ giá `FLAT` (V1 nguyên vẹn) và `DAILY` (bảng `room_night_rate` là nguồn sự thật; `total = Σ` giá đêm chính xác; báo cáo cộng đúng giá từng đêm; Charge gốc không đổi; gia hạn lưu đêm riêng; `UNIQUE (reservation_room_id, stay_date)` chặn trùng doanh thu). Gồm: schema, 10 bất biến N1–N10, thuật toán theo luồng, danh sách mọi nơi đọc `nightlyRate` kèm hành động, 21 test (T01–T21): [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §2 và [ADR 017](../decisions/017-nightly-pricing-flat-and-daily-modes.md).
- **Acceptance:** (1) danh sách đầy đủ nơi dùng `nightly_rate`/`total_amount` (đã có: round 3 §2.4); (2) schema `room_night_rate` + chế độ `pricing_mode` được PO xác nhận; (3) **PO-Q1 đã duyệt:** `technical-spec-v1` **không** bị sửa; quy tắc `DAILY` nằm trong đặc tả PMS-0034 và ghi đè V1 chỉ cho booking `DAILY` ([v2a mục 17](../../../hotel/docs/specs/PMS-0034-hotel-website-direct-booking-v2a.md#17-tương-thích-v1-và-thứ-tự-ưu-tiên-đặc-tả-po-q1-đã-duyệt)); câu hỏi "chọn (a) hay (b)" đã đóng; (4) danh sách test hiện có phải giữ nguyên kết quả (báo cáo tháng, đối soát folio V33, gia hạn, đổi phòng, Charge); (5) PO duyệt thiết kế **và** Codex re-review #3 xác nhận trước khi BE-10b bắt đầu.

### BE-10 Snapshot giá theo đêm

- **Gap:** G-14 · **F-01 (BLOCKER)** · **Phụ thuộc:** BE-09, **BE-10a** (PO duyệt trước khi code BE-10b).
- **Phạm vi:** Issue này (hotel-system#137) là **BE-10b**: migration **M06** + entity + code. BE-10a (#150) là thiết kế. **Đây cũng là chủ sở hữu PO-03** (Extension/Date Change) và bộ test hồi quy tương ứng.
- **Nội dung BE-10b:** M06 (`pricing_mode`, `priced_room_type_id`, `room_night_rate`, constraint trigger); `NightlyPriceCalculator` + `NightlyBreakdownVerifier`; ghi breakdown khi tạo dòng `DAILY`; rẽ nhánh theo `pricing_mode` ở `MonthlyFinancialReportService`, `StayExtensionService` (extend + preview), `ReservationService/Reservation/ReservationRoom.changeConfirmedDates`, `Charge.createOriginalRoomCharge` (quantity/unit_price NULL cho `DAILY`); `IssueType.NIGHTLY_BREAKDOWN_MISMATCH` ở `FolioReconciliationService`; constructor tương thích cho `ReservationRoomRevenueRow`/`StayExtensionRevenueRow`; guard test allowlist `getNightlyRate()`.
- **Acceptance:** AC-08/09: cập nhật lịch giá **không** đổi giá reservation đã xác nhận; mọi bất biến N1–N10 và test T01–T21 ([PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §2.7) đạt; test V1 (`MonthlyFinancialReportServiceTest`, `FolioRevenueReconciliationIntegrationTest`, `StayExtensionIntegrationTest`, `StayExtensionRevenueRegressionIntegrationTest`, `RoomChangeServiceTest`, `ChargeServiceTest`…) giữ **nguyên assertion**; gia hạn không đổi Charge gốc và không ghi nhận trùng doanh thu; đổi ngày theo calendar date; đối soát folio V33 đạt.
- **Rủi ro:** cao nhất của Issue (lõi tài chính). Chỉ bắt đầu BE-10b sau khi #150 được PO duyệt **và** Codex re-review #3 xác nhận (PO-Q1 đã duyệt).
- **Round 4 (M4):** với `DAILY`, ước tính gia hạn hiển thị phải từ preview backend theo từng đêm (response preview thêm `pricingMode` và `nightlyBreakdown`); JS không ước tính phẳng cho `DAILY`; `FLAT` giữ nguyên; thêm `stay-extension.js:41` và `stay-extension.html:123` vào reader inventory. **AC bổ sung:** DAILY 1.000.000/1.100.000/1.200.000 ⇒ preview 3.300.000; hỗn hợp FLAT/DAILY đúng theo dòng. [round 4 §6.4](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### BE-11 Room Rate Calendar

- **Gap:** G-13 · **Phụ thuộc:** BE-09 (sở hữu M04 `room_type_pricing_revision` + override + trigger revision); BE-11 sở hữu M05 (lịch sử).
- **Nội dung:** lịch giá theo Room Type; sửa một ngày; bulk trên khoảng ngày (gồm hai đầu) với bộ lọc thứ trong tuần; **preview** trả danh sách ngày tác động (giá cũ → mới) + `expectedRevision`; **confirm trong một transaction**: khóa `room_type_pricing_revision` `FOR UPDATE`, so `expectedRevision` (lệch ⇒ `409 RATE_CALENDAR_CHANGED`), kiểm giá (`> 0`, `SupportedCurrency.hasValidPrecision`, giới hạn kích thước bulk), upsert/xóa cả tập, ghi `room_type_rate_history` (từng ngày cũ/mới, `operation_id`) — tất cả hoặc không dòng nào. Revision tăng bởi trigger khi đổi `base_price`, tạo/sửa/xóa override (kể cả xóa rồi tạo lại và ngày chưa có override). Quote đọc trong snapshot `REPEATABLE READ`; submit `FOR SHARE` hàng revision theo id tăng dần. Thứ tự khóa và lý do không deadlock: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §9.
- **Acceptance:** AC-09: chọn Thứ Sáu/Thứ Bảy ⇒ preview đúng danh sách ngày; xác nhận cập nhật đúng; lịch sử đầy đủ; hai người sửa cùng lúc ⇒ người thứ hai nhận 409; bulk lỗi một dòng ⇒ không ngày nào đổi; revision tăng ở mọi nhánh trên (kể cả đường V1 `RoomTypeService` sửa `base_price`, không đổi hành vi V1); reservation xác nhận không đổi; khoảng ngày vượt giới hạn bị từ chối; responsive và bàn phím.

### BE-12 Sức chứa khách và bộ gán phòng

- **Gap:** G-09, G-07, G-10 · **F-06, F-08** · **Phụ thuộc:** GOV-03 (D-03, D-04), BE-06.
- **Nội dung:**
  - `GuestCapacityRules` thuần: `adults ≥ 1` và `adults + children ≤ Σ capacity` của các phòng được gán; `capacity = null` ⇒ lỗi cấu hình. **Không sửa `AdultCapacityRules`** (D-03 = chỉ đường công khai).
  - Bộ gán phòng: nhận danh sách (Room Type, quantity); khóa mọi phòng của các Room Type liên quan theo **một thứ tự id toàn cục** (cùng thứ tự với `confirm`/check-in để không deadlock); chọn phòng trống theo chiến lược xác định (số phòng rồi id), tất cả hoặc không; dùng lại truy vấn xung đột tồn hiện có. Được dùng bởi BE-17 (Instant) và BE-18 (duyệt Request).
  - **Round 3:** Issue này **không còn schema** và **không yêu cầu `room_id` nullable** (bản trước ghi "theo data-design V71 (room_id nullable, room_type_id)" — đã loại bỏ). `reservation_room` không đổi; dữ liệu Request nằm ở `booking_request_line*` do **BE-18 (#145)** sở hữu (M12).
- **Acceptance:** AC-05/06/07: 3 adults + 1 child với 2 Double (capacity 2) hợp lệ nếu đủ tồn; thiếu tồn ở một loại ⇒ từ chối toàn bộ, không tạo một phần; unit test thuần cho quy tắc; test đồng thời bộ gán không deadlock với `confirm`; hành vi nhân viên không đổi.

### BE-13 Tìm phòng, Sold Out, quote

- **Gap:** G-08, G-15 · **Phụ thuộc:** BE-03, 06, 09, 12 (sở hữu migration M10 `public_booking_quote`).
- **Nội dung:** `availability/search` trả giá tham khảo từng đêm (không có `quoteId`, không giá trung bình). `POST /quotes` tính giá bằng `NightlyPriceCalculator` trong snapshot `REPEATABLE READ` và lưu **quote server-side bất biến** ràng buộc: ngày, `adults`, `children`, basket, `bookingMode`, currency, **revision giá từng Room Type**, **revision chính sách hủy**, giá từng đêm, tổng, hạn (mặc định 15 phút). Submit so từng trường (lệch ⇒ `QUOTE_INVALID`), hạn (`QUOTE_EXPIRED`), mode (`MODE_CHANGED`), chính sách (`POLICY_CHANGED`), giá (`PRICE_CHANGED`). Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §4.
- **Acceptance:** AC-04/08: số lượng không vượt tồn; phân biệt Sold Out với lỗi tải; mỗi nhánh lỗi quote ở trên có test và fixture; đáp ứng `no-store`; rate limit; test với nhiều Room Type và quantity; quote cũ không dùng được sau khi Rate Calendar đổi (revision khác ⇒ tính lại, đêm khác ⇒ `PRICE_CHANGED`).
- **Round 4 (F3, F4):** quote chạy trong **một transaction `REPEATABLE_READ` cho phép ghi** (không `readOnly`; `availability/search` vẫn `readOnly = true`), retry `40001`/`40P01` tối đa 2 lần, trigger toàn vẹn (tổng, đêm, revision của snapshot, `READ COMMITTED` bị chặn), TTL ≤ 1 giờ, cấm `UPDATE`. DTO công khai áp quy tắc V1 của `roomTypeCode` (không trống, ≤ 64, không khoảng trắng, không chữ thường) + tra cứu khớp tuyệt đối + **test trôi lệch** với `RoomTypeCreateRequest`; loại Room Type có mã không hợp lệ khỏi catalog kèm cảnh báo readiness; không dùng mã trong URL. **AC bổ sung:** `SHOW transaction_isolation`/`transaction_read_only` trong quote service; `DBL-SEA`, mã 40 và 64 ký tự đi qua đủ luồng. [round 4 §4–§5](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).
- **Round 5 (R4-01):** trigger toàn vẹn quote viết lại để so **tập chính xác**: Room Type của basket = khóa `pricing_revisions` = khóa `nightly_prices` (hai chiều, không chỉ đếm); revision `NULL`/JSON `null`/sai kiểu/khác snapshot (`IS DISTINCT FROM`) bị từ chối; mỗi đêm có `rate` chuỗi thập phân hợp lệ (đêm thiếu/`NULL` bị từ chối dù tổng khớp tổng rút gọn); ngày parse `DATE` phải đúng bằng `[check_in, check_out)` mỗi ngày một lần; `quantity` nguyên ≥ 1. `rate` là chuỗi JSON, `revision` là số nguyên JSON. **AC bổ sung:** probe PostgreSQL R5-01…R5-120 thành test migration/integration; quote service tuần tự hóa `BigDecimal` thành chuỗi (test serialize) và kiểm trước bằng `NightlyBreakdownVerifier`; biên hợp lệ (1 đêm, 366 đêm, 29/2, qua năm, đơn giá 13 chữ số) được chấp nhận. [round 5 §3](../../../hotel/docs/specs/PMS-0034-codex-round5-remediation-2026-11-11.md#3-r4-01--toàn-vẹn-quote).

### BE-14 Idempotency và đối soát

- **Gap:** G-22 · **Phụ thuộc:** BE-01, **BE-02** (actor cho `created_by`; migration M09 sau M01).
- **Nội dung (viết lại Round 3):** bảng `public_idempotency_record` dùng chung cho `BOOKING_SUBMIT` và `BOOKING_CANCEL`; **hai pha**: (1) claim trong transaction riêng bằng `INSERT … ON CONFLICT (operation_type, idempotency_key) DO NOTHING` (không exception ⇒ không transaction aborted bởi 23505; đọc bản ghi trong transaction hợp lệ, READ COMMITTED); (2) business transaction riêng, đầu transaction khóa bản ghi và kiểm `owner_token`, cập nhật `SUCCEEDED` + response **trong cùng transaction** tạo Reservation. Trạng thái `PROCESSING/SUCCEEDED/REJECTED/FAILED_RETRYABLE`; lease + recovery bằng `UPDATE` có điều kiện; replay nguyên response; retention 48 giờ + job dọn (retention > quote TTL); `GET /bookings/reconcile`. Học từ `PaymentIdempotencyService` (V65) nhưng **không** dùng lại bảng/transaction `MANDATORY` của nó. Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §5.
- **Acceptance:** AC-14: gửi lặp hoặc đồng thời cùng khóa ⇒ tối đa một reservation (test đồng thời trên PostgreSQL thật, không có "current transaction is aborted"); crash giả lập trước/sau commit nghiệp vụ; owner cũ bị chặn sau khi lease bị chiếm; replay `SUCCEEDED` và `REJECTED`; fingerprint khác ⇒ `IDEMPOTENCY_KEY_REUSED`; dọn không xóa `PROCESSING` còn lease; không lộ PII ở đối soát.
- **Round 4 (M1, M3):** ràng buộc DB theo trạng thái (`ck_public_idempotency_open/succeeded/rejected`; lỗi hạ tầng/cấu hình là `FAILED_RETRYABLE`, không bao giờ `REJECTED`); reconcile là union theo `state`, chỉ cho `BOOKING_SUBMIT`; replay trả cùng status + body (tương đương ngữ nghĩa, **không** byte-for-byte) kèm header `Idempotency-Replayed: true`. [round 4 §6.1, §6.3](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).
- **Round 5 (R4-03):** `CHECK` theo trạng thái dùng `IS [NOT] NULL` tường minh và `(…) IS TRUE` (Round 4 để `NULL` lọt: `SUCCEEDED` không `response_status`; `REJECTED` không `response_status` hoặc không `error_code`). `SUCCEEDED` bắt buộc `response_status`, body là JSON object, `reservation_id`, không `error_code`; `REJECTED` bắt buộc `response_status` (`409`/`422`), body object, `error_code` hợp lệ; `access_token_id` có giá trị khi và chỉ khi `BOOKING_CANCEL`. Ma trận 256 tổ hợp có test migration (thay AC "16 trường hợp" của Round 4). [round 5 §4](../../../hotel/docs/specs/PMS-0034-codex-round5-remediation-2026-11-11.md#4-r4-03--check-của-idempotency).

### BE-15 Outbox email

- **Gap:** G-21 · **F-10** · **Phụ thuộc:** GOV-03 (D-08). Sở hữu migration M08.
- **Nội dung:** `spring-boot-starter-mail` hoặc nhà cung cấp theo D-08; bảng `outbox_event` ghi trong cùng transaction nghiệp vụ (`dedupe_key` UNIQUE); **claim nguyên tử** bằng một câu `UPDATE … WHERE id IN (SELECT … FOR UPDATE SKIP LOCKED) RETURNING` (owner `claimed_by` + lease `claim_expires_at`); kết quả ghi có hàng rào owner; retry với backoff, `FAILED` sau N lần; phục hồi bằng lease hết hạn; **at-least-once** (Message-ID xác định từ event id). Hạ tầng scheduler dùng chung cho dispatcher, `expireDueRequests` (BE-18) và job dọn (idempotency, quote, bí mật trong payload). Template vi/en cho: xác nhận, đã nhận yêu cầu, duyệt/từ chối/hết hạn (BE-18), hủy (BE-20), liên kết quản lý (BE-19). Không log nội dung/địa chỉ; payload liên kết một lần dùng bị xóa khi `SENT`/`FAILED` và dọn theo TTL. Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §8.
- **Acceptance:** AC-10/16: outbox chỉ ghi khi giao dịch thành công; hai dispatcher song song ⇒ mỗi dòng gửi một lần; crash sau claim ⇒ claim lại sau khi lease hết; worker mất lease không ghi đè kết quả; lỗi gửi không đảo ngược reservation; backoff và `FAILED`; enqueue lặp vô hại; test dispatcher với sender giả; cảnh báo khi nhà cung cấp lỗi kéo dài.
- **Ghi chú:** làm tối thiểu cho #30, thiết kế để #33 (PMS-0037B) mở rộng.
- **Round 4 (F2):** thay SQL/ràng buộc Round 3 (ba lỗi đã tái hiện): `SENDING` ⇔ đủ chủ + hạn lease, trạng thái khác ⇒ cả hai NULL; fence `(id, claimed_by, attempts)`; `fail_exhausted` giới hạn lượt thử cả đường phục hồi; bảng chuyển trạng thái T1–T9; at-least-once giải thích rủi ro gửi trùng (không exactly-once). **AC bổ sung:** T-OB-01…08 qua repository thật (gửi thường, lỗi tạm, worker chết, lease hết hạn, cạn lượt, đồng thời, owner lỗi thời, ràng buộc); chết giữa gửi và ghi `SENT` ⇒ gửi hai lần được ghi nhận. [round 4 §3](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### BE-16 Chính sách hủy và snapshot

- **Gap:** G-19 · **Phụ thuộc:** BE-03, BE-04. Sở hữu migration **M07b** (snapshot trên `reservation`; bảng `cancellation_policy_revision` M07a do BE-04 sở hữu). (Bản trước thêm phụ thuộc BE-10/PO-03 — **đã gỡ**: BE-16 không đổi giá; PO-03 thuộc BE-10b.)
- **Nội dung:** `cancellation_policy_revision` **append-only** (giờ 24/48/72 + văn bản, theo phiên bản; sửa = chèn revision mới; mặc định 24h); snapshot ghi **lúc submit** (Instant và Request như nhau) trên `reservation`: `cancellation_policy_revision_id`, `cancellation_free_until` (check-in + giờ check-in tiêu chuẩn tại múi giờ khách sạn − số giờ), `cancellation_policy_accepted_at`; ràng buộc all-or-none; **duyệt Request không đọc lại, không ghi lại**; nhân viên đổi ngày không đổi cutoff. Booking V1/nhân viên tạo: cả ba cột NULL, không backfill, không tự hủy qua web (PO-04), không có job nào chạm. Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §7.
- **Acceptance:** AC-18: đổi chính sách hoặc `check_in_time` sau khi đặt không làm đổi booking cũ; tái dựng điều khoản khách chấp nhận (giờ + văn bản + cutoff + thời điểm) từ ba cột; Request duyệt sau khi đổi chính sách vẫn giữ snapshot lúc gửi; unit test công thức cutoff qua nhiều múi giờ; `POLICY_CHANGED` khi policy đổi giữa quote và submit; reservation legacy không tự hủy được; PMS Admin hủy booking legacy như cũ.

### BE-17 Đặt phòng công khai Instant

- **Gap:** G-16, G-07, G-10 · **Phụ thuộc:** BE-01, 02, 10, 12, 13, 14, 15, 16.
- **Nội dung:** một transaction: xác thực khóa idempotency → kiểm tra published/active/bật bán của từng Room Type (backend) → kiểm tra ngày/quy tắc đặt (BE-03) → kiểm tra sức chứa khách (BE-12) → khóa + gán phòng → tính lại giá và so quote (`PRICE_CHANGED`) → so chế độ hiệu lực (`MODE_CHANGED`) → tạo Guest (chính sách D-12) → tạo Reservation nguồn `HOTEL_WEBSITE` + các `ReservationRoom` + snapshot theo đêm + snapshot chính sách hủy → xác nhận (`CONFIRMED`) → audit (actor hệ thống) → outbox → hoàn tất bản ghi idempotency. Phản hồi là union rõ: `CONFIRMED` | lỗi có mã (`SOLD_OUT`, `PRICE_CHANGED`, `MODE_CHANGED`, `INVALID`, …). Không thu thông tin thanh toán. Mã tham chiếu công khai chỉ khi PMS cho phép. Cân nhắc cơ chế ngừng nhận booking ([deployment-guide](../deployment/deployment-guide.md) mục 4) chốt ở GOV-04.
- **Acceptance:** AC-06, 07, 10, 12, 13, 14, 22: một lần gọi tạo đúng một reservation và đúng số `ReservationRoom`; hai khách tranh phòng cuối ⇒ chỉ một thành công (test đồng thời nhiều luồng); Room Type ẩn ⇒ từ chối dù client gửi dữ liệu cũ; không có `DRAFT` sót lại khi lỗi; audit và outbox nhất quán với kết quả; Guest/reservation không chứa dữ liệu thẻ; hồi quy staff `create/confirm/cancel/check-in` đạt.
- **Round 3 (2026-11-11):** sở hữu migration **M11** (`reservation.public_reference`, mã công khai ngẫu nhiên không tuần tự, không phải UUID nội bộ/`reservation_number`, một mình không cấp quyền quản lý — PO-Q6 đã duyệt). Luồng dùng claim idempotency hai pha (BE-14), so quote theo thứ tự `QUOTE_INVALID → QUOTE_EXPIRED → MODE_CHANGED → POLICY_CHANGED → PRICE_CHANGED` (BE-13), `FOR SHARE` revision giá rồi khóa phòng (thứ tự khóa: idempotency → revision → phòng → reservation), nguồn `HOTEL_WEBSITE` phải tồn tại ∧ active (nếu không `BOOKING_UNAVAILABLE`, không fallback `DIRECT`), snapshot chính sách hủy từ quote (BE-16), tạo `reservation_room` `DAILY` + `room_night_rate` (BE-10b), `FOR UPDATE` record idempotency và cập nhật `SUCCEEDED` cùng transaction. Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §2–§5, §10.
- **Round 4 (M3):** `201 Created` cho thành công; replay cùng status + body + header `Idempotency-Replayed: true`. [round 4 §6.3](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### BE-18 Chế độ Request

- **Gap:** G-17 · **F-02 (BLOCKER)** · **Phụ thuộc:** GOV-03 (D-01, PO-01, PO-02), BE-17, BE-02 (permission `MANAGE_BOOKING_REQUEST`). Sở hữu migration **M12** (`booking_request_line*`, cột Request trên `reservation`).
- **Nội dung (viết lại Round 3):** ba trạng thái `PENDING_APPROVAL`, `REJECTED`, `EXPIRED` (riêng); không giữ tồn; **duyệt**: khóa phòng (id tăng dần) → khóa reservation → đọc lại status → **kiểm hạn dưới khóa (duyệt không bao giờ thành công sau hạn; hạn thắng)** → kiểm tồn + sức chứa → tạo `reservation_room` DAILY (sao chép breakdown, `Σ` bằng giá đã báo) → `CONFIRMED`; **hết phòng/sức chứa ⇒ `REJECTED` trong cùng transaction** (không "giữ chờ"); scheduler `expireDueRequests` chỉ khóa một dòng reservation; ma trận đua tranh duyệt/từ chối/hết hạn/hai nhân viên; hàng đợi cho nhân viên (quyền `MANAGE_BOOKING_REQUEST`); email; hiệu ứng lên danh sách/board/báo cáo/test bảo vệ trạng thái. **Khách không tự rút hay tự hủy Request chờ duyệt** (PO-Q8 đã duyệt; bản trước ghi "rút yêu cầu của khách" — đã bỏ khỏi phạm vi); `EXPIRED` là trạng thái cuối riêng, Request hết hạn không thể được duyệt lại (PO-Q9); lúc duyệt Room Type phải còn `active` và đủ điều kiện công bố, giá giữ như đã đảm bảo lúc submit, không thay Room Type khác, thất bại ⇒ `REJECTED` `ROOM_TYPE_UNAVAILABLE` (PO-Q10, diễn giải O3); duyệt kiểm permission hiệu lực `MANAGE_BOOKING_REQUEST`. Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §3.
- **Acceptance:** AC-11/12: Request thành công ⇒ UI chờ duyệt, không khẳng định confirmed; mode đổi giữa review và submit ⇒ `MODE_CHANGED`; duyệt sau hạn (chưa quét) ⇒ `REQUEST_EXPIRED`, trạng thái `EXPIRED`; duyệt khi hết phòng ⇒ `REJECTED`, không overbooking; hai nhân viên duyệt đồng thời ⇒ một thành công; STAFF có `MANAGE_BOOKING` không duyệt được; không deadlock với `confirm()` (mở rộng `ReservationLifecycleLockOrderIntegrationTest`); hết hạn xử lý đúng một lần; mọi `switch`/báo cáo theo `ReservationStatus` được rà (test bảo vệ).
- **Round 4 (M3):** Request được tạo trả `201 Created` (không `202`), cùng endpoint và schema với Instant; website phân nhánh theo `outcome`. [round 4 §6.3](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### BE-19 Manage Booking

- **Gap:** G-18 · **F-11** · **Phụ thuộc:** BE-15, BE-17. Sở hữu migration M13.
- **Nội dung:** lookup với (`public_reference`, email): phản hồi và thời gian xấp xỉ giống nhau dù khớp hay không; nếu khớp tạo token ngẫu nhiên ≥128 bit, lưu **hash**, TTL ngắn, dùng một lần (`FOR UPDATE` + `used_at`), giới hạn số lần yêu cầu, gửi qua outbox; `exchange` đổi LOOKUP lấy SESSION ngắn hạn (lưu hash, ràng buộc `reservation_id`); đọc chi tiết với DTO tối thiểu; `noindex`/no-store; token trong body/header, không URL, không log. **Round 3:** định danh phiên (`access_token_id`) được ghi vào `public_idempotency_record` và audit của thao tác hủy. Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §6.
- **Acceptance:** AC-15: không xác thực ⇒ không lộ dữ liệu hay sự tồn tại booking; brute force bị giới hạn; token hết hạn/đã dùng bị từ chối (`MANAGE_TOKEN_INVALID`); không dùng riêng mã + email làm bằng chứng quyền hủy; test timing/enumeration; không PII/token trong log.

### BE-20 Tự hủy

- **Gap:** G-20 · **F-17** · **Phụ thuộc:** BE-16, BE-19, BE-14 (idempotency `BOOKING_CANCEL`).
- **Nội dung (viết lại Round 3):** thứ tự bắt buộc: session hợp lệ → claim idempotency và **replay trước mọi đánh giá cutoff** → khóa reservation → replay theo reservation (hủy thành công trước đó của chính khách; hủy bởi nhân viên không được nhận công) → **guard V1**: `CONFIRMED` + không prepayment active (`requireNoActivePrepayments`) → `cancellation_free_until IS NOT NULL` (PO-04) → cutoff → `Reservation.cancel(GUEST_REQUEST, …)` (entity V1 không đổi) → record `SUCCEEDED` cùng transaction; audit actor hệ thống ghi `operationId` + định danh phiên; email qua outbox. Một lần hủy thành công **không bao giờ** bị retry sau cutoff biến thành lỗi. Request `PENDING_APPROVAL` không tự hủy được (PO-Q8 đã duyệt: khách liên hệ khách sạn). Snapshot chính sách đã chấp nhận bất biến, nhân viên đổi ngày không thay (PO-Q7). Chi tiết: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) §6.
- **Acceptance:** AC-16/17/18: hủy trước cutoff ⇒ `CANCELLED`, tồn được giải phóng, audit đúng một lần, có email; retry cùng/khác khóa cùng phiên sau cutoff ⇒ `200` cùng body kèm header `Idempotency-Replayed: true`; quá cutoff, `CHECKED_IN`, `CHECKED_OUT`, `NO_SHOW`, nhân viên đã hủy ⇒ không tự hủy; booking không snapshot hoặc có prepayment ⇒ `CANCELLATION_NOT_AVAILABLE`; hủy đồng thời hai khóa ⇒ một lần; chính sách áp dụng là snapshot lúc submit.
- **Round 4 (M3):** thành công `200 { "outcome": "CANCELLED" }`; replay (cùng khóa hoặc khóa khác cùng phiên) trả cùng `200` + body và header `Idempotency-Replayed: true`; **không** có trường `replayed` trong body. [round 4 §6.3](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### BE-21 Bộ test xuyên suốt

- **Gap:** G-10, G-22, G-23 · **Phụ thuộc:** tăng dần từ BE-17.
- **Nội dung:** mở rộng bộ test đồng thời hiện có (`ReservationConfirmConcurrencyIntegrationTest`, `ReservationLifecycleLockOrderIntegrationTest`) cho đường công khai: nhiều luồng tranh phòng cuối, nhiều Room Type, đặt cùng lúc với `confirm`/hủy của nhân viên, đặt lặp cùng khóa; thêm test hợp đồng đối chiếu fixture GOV-04, test bảo mật (rate limit, header giả, enumeration, kích thước), test hồi quy chain `/api/**`, test folio/báo cáo cho dữ liệu snapshot.
- **Acceptance:** chạy được trong CI hiện có (`Build and Test`); không flaky khi lặp; kết quả ghi lệnh và phạm vi.

### BE-22 Vận hành và tài liệu

- **Gap:** G-24 · **F-19** · **Phụ thuộc:** BE-17.
- **Nội dung:** cập nhật `docs/operations/configuration-and-secrets.md` (khóa dịch vụ, nguồn email, cấu hình phơi bày API), runbook xử lý booking chưa rõ kết quả, quay vòng khóa, tắt nhận booking; mô hình lưu thông tin domain nếu D-10 yêu cầu; cập nhật đặc tả.
- **Acceptance:** runbook diễn tập được trên staging; không có bí mật trong repo.
- **Round 4 (M2, F4):** runbook: `PMS_PUBLIC_SERVICE_KEY` phải có ở mọi môi trường **trước** khi triển khai bản có BE-01 (thiếu ⇒ toàn bộ PMS không khởi động); truy vấn chẩn đoán chỉ-đọc mã Room Type (`code = '' OR code <> UPPER(code) OR code ~ '\s' OR char_length(code) > 64`) phải trả 0 hàng trước go-live; email có thể gửi trùng (at-least-once). [round 4 §6.2, §5.3, §3.5](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).
- **Round 5 (R4-02):** runbook thứ tự triển khai BE-01: M01 đã áp → cấp `PMS_PUBLIC_SERVICE_KEY` (không ghi giá trị thật) → triển khai với kiểm tra khởi động bật → xác minh hợp lệ và các trường hợp âm. [round 5 §5.3](../../../hotel/docs/specs/PMS-0034-codex-round5-remediation-2026-11-11.md#53-thứ-tự-triển-khai-mọi-môi-trường).

## 7. Task website (`hotel-website`)

Mọi task: strict TypeScript, không `any`, Server Components mặc định, `lib/pms` `server-only`, validate bằng schema runtime cả phản hồi lỗi, không log PII, ngày dạng calendar date, định dạng tiền qua hàm chung từ amount/currency của PMS, lệnh lint/typecheck/build/test lấy từ cấu hình thực tế sau FE-01, giao diện kiểm ở 320/390/768/1440 và Chromium + WebKit khi chạm luồng booking ([frontend.md](../coding-rules/frontend.md#kiểm-tra-khi-thay-đổi-mã)).

### FE-01 Khởi tạo ứng dụng

- **Nội dung:** Next.js App Router, TypeScript `strict`, Tailwind CSS, ESLint (cấm `any`), công cụ test (unit/component, E2E), cấu trúc thư mục theo [client-architecture.md](../coding-rules/client-architecture.md) (tạo thư mục khi có mã), `.env.example` chỉ chứa tên biến thật, schema cấu hình khởi động (PMS origin, canonical origin, allowlist host/origin, nguồn ảnh) báo lỗi theo tên biến không lộ giá trị, `README` với lệnh thật. Chọn thư viện schema runtime với ghi chú nhu cầu theo [frontend.md](../coding-rules/frontend.md#giữ-triển-khai-gọn).
- **Acceptance:** `lint`, `typecheck`, `build`, `test` chạy xanh trên app rỗng; cập nhật [environment-variables.md](environment-variables.md) và [test-strategy.md](../testing/test-strategy.md) với lệnh/biến thật; không có `NEXT_PUBLIC_*` chứa bí mật.

### FE-02 Client PMS, schema, fixture

- **Phụ thuộc:** GOV-04.
- **Nội dung:** `lib/pms` (HTTP client, timeout có hủy, kiểm tra status), adapter theo khả năng, schema runtime cho response thành công và lỗi, ánh xạ lỗi có kiểu (dữ liệu sai, lỗi nghiệp vụ, timeout, lỗi kết nối, phản hồi không hợp lệ), union kết quả booking (đã xác nhận, đã nhận yêu cầu, cần báo giá lại, chưa rõ kết quả), chính sách cache theo operation (nội dung công bố có thể cache; tồn/quote/submit/kết quả `no-store`), fixture từ hợp đồng, mock server cho dev/test/E2E.
- **Acceptance:** unit test cho phản hồi hợp lệ, lỗi PMS, sai schema, timeout, giá trị lạ (xử lý như không hợp lệ); khóa idempotency và credential chỉ ở server; không có import `lib/pms` từ Client Component (kiểm tra bằng lint/test).
- **Round 4 (F4, M1, M3):** adapter/mock dùng schema chung, **không** áp regex `roomTypeCode` chặt hơn (không `[A-Z0-9_]{1,32}`), không đặt mã vào URL, chỉ render qua escape của React; mock theo `manifest.json` (gồm biến thể `REPLAY` + `responseHeaders`); xử lý reconcile theo `state`; dùng `negative-controls.json` và `corpus/room-type-code.json` trong test hợp đồng. [round 4 §5–§6](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### FE-03 Template và vỏ giao diện

- **Phụ thuộc:** GOV-05.
- **Nội dung:** `templates/registry.ts` (mã template đã biết → module tĩnh, fallback Modern Tropical với cảnh báo vận hành, trạng thái tạm gián đoạn khi dữ liệu thiết yếu sai), Modern Tropical (token màu/chữ/spacing/focus, `next/font` có kiểm tra tiếng Việt), header/footer/nav, thành phần dùng chung (nút, trường nhập, thông báo trạng thái, khung ảnh), loading/empty/error/not-found.
- **Acceptance:** khớp mockup đã duyệt ở các viewport; không hardcode tên, giá, địa chỉ, chính sách; focus nhìn thấy, bàn phím điều hướng được; ảnh chụp lưu theo [designs/README.md](../designs/README.md).

### FE-04 Trang chủ và SEO

- **Nội dung:** trang chủ từ site config (hero, giới thiệu, điểm nổi bật, CTA tìm phòng, room highlights, tiện nghi, gallery, vị trí/liên hệ; ẩn section tùy chọn rỗng), `metadata`/`generateMetadata`, `metadataBase` từ canonical origin, `alternates.canonical`, `app/sitemap.ts`, `app/robots.ts`, Open Graph, `noindex` cho preview/manage/checkout, favicon và logo từ branding.
- **Acceptance:** AC-01/20: xem HTML thực tế: canonical, sitemap, robots đúng origin cấu hình; sitemap không chứa URL giao dịch; PMS lỗi ⇒ dùng nội dung công bố đã cache hợp lệ hoặc trạng thái tạm gián đoạn, không lộ lỗi nội bộ; thiếu ảnh/dữ liệu tùy chọn ⇒ fallback đúng.

### FE-05 Danh mục, chi tiết, gallery, chính sách

- **Nội dung:** catalog theo thứ tự cấu hình, cover, mô tả, tiện nghi, sức chứa, giá tham khảo khi hợp lệ; chi tiết theo slug (404 khi không công bố); gallery có alt/fallback; trang Gallery, Contact & Location, Privacy/Terms/Cancellation Policy; `next/image` với allowlist nguồn.
- **Acceptance:** AC-02/03/20: Room Type chưa công bố ⇒ 404 ở chi tiết và vắng ở danh sách; thứ tự/cover phản ánh sau làm mới; không tiết lộ dữ liệu không công bố; `next/image` từ chối nguồn ngoài allowlist.

### FE-06 Booking Engine: tìm, chọn phòng, báo giá

- **Nội dung:** form `checkIn`, `checkOut`, `adults ≥ 1`, `children ≥ 0` (ngày là calendar date theo giờ khách sạn); một nguồn trạng thái tiêu chí; **hủy/bỏ qua phản hồi cũ** khi đổi tiêu chí; kết quả theo Room Type với Sold Out, quantity không vượt tồn, giá và tổng do PMS trả; Server/Client Component tối thiểu; chặn tiếp tục khi chưa có giá khớp tiêu chí hiện hành.
- **Acceptance:** AC-04/19, WEB-03/04: đổi tiêu chí khi đang tải ⇒ không hiển thị giá của lượt cũ đến sau; Sold Out khác với lỗi tải; label, lỗi cạnh trường, focus chuyển tới lỗi đầu tiên, thông báo trạng thái đọc được; chạy đúng khi múi giờ trình duyệt khác; mọi phản hồi tồn/quote `no-store`.

### FE-07 Booking Engine: khách, review, submit, kết quả

- **Phụ thuộc:** FE-06, BE-17 (tích hợp), BE-18 (Request).
- **Nội dung:** Guest Info (trường tối thiểu PMS yêu cầu, privacy notice/consent, **không thu dữ liệu thẻ**), Review (ngày, adults/children, Room Type × quantity, từng khoản giá nếu API có, tổng, tiền tệ, chế độ, chính sách hủy hiện hành), Submit qua Route Handler cùng origin (kiểm tra origin, giới hạn body, khóa idempotency tạo một lần cho mỗi ý định), nút theo chế độ (`Book now`/`Send request`) vô hiệu khi đang xử lý, kết quả: Instant `Confirmed` chỉ khi PMS xác nhận, Request `Request received/Pending approval`, nhánh `PRICE_CHANGED` (hiện giá mới, đồng ý lại), `MODE_CHANGED` (tải chế độ mới, xác nhận lại), `SOLD_OUT` (chọn lại toàn bộ lựa chọn không hợp lệ), **chưa rõ kết quả** (giữ trạng thái, đối soát bằng cùng khóa, không báo thất bại chắc chắn), PMS không khả dụng (lỗi thân thiện, không xác nhận giả). Trang kết quả `noindex`, ngoài sitemap.
- **Acceptance:** AC-10/11/12/14/22, WEB-05…08: gửi lặp, gửi đồng thời hay mất phản hồi sau khi PMS đã ghi ⇒ tối đa một reservation, UI nhất quán; không hiển thị xác nhận khi chưa có kết quả đúng; thông tin khách không nằm trong URL, log, analytics, localStorage hay cache chia sẻ; mọi phản hồi giao dịch `Cache-Control: no-store`; E2E cho Instant, Request, hết phòng, giá đổi, mode đổi, timeout; hoàn tất được bằng bàn phím.
- **Round 4 (M3):** `POST /bookings` trả `201` cho cả `CONFIRMED` và `PENDING_APPROVAL`; phân nhánh theo `outcome`, không theo status; replay có header `Idempotency-Replayed: true` nhưng cùng body. [round 4 §6.3](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### FE-08 Manage Booking

- **Phụ thuộc:** FE-07, BE-19, BE-20.
- **Nội dung:** form (mã tham chiếu + email) với thông báo trung tính không lộ booking tồn tại; trang trao đổi token và xem chi tiết; hủy có bước xác nhận, hiển thị chính sách và cutoff theo giờ khách sạn; trạng thái không hủy được kèm hướng dẫn liên hệ khách sạn; `noindex`, `Referrer-Policy` chặt, token không vào analytics/log.
- **Acceptance:** AC-15/16/17: lỗi đồng nhất cho mọi đầu vào; token hết hạn/đã dùng ⇒ thông báo an toàn; hủy thành công/quá hạn/trạng thái sai hiển thị đúng; khôi phục khi mạng lỗi không hủy hai lần; accessibility đạt.
- **Round 4 (M3):** response hủy chỉ có `outcome`; không đọc `replayed` từ body (replay ở header). [round 4 §6.3](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md).

### FE-09 Bảo mật và vận hành phía Next.js

- **Nội dung:** kiểm tra origin cho thao tác ghi, giới hạn body và tần suất theo operation, chuyển tiếp IP khách tin cậy sang PMS, header an toàn (CSP theo nguồn thực dùng, `frame-ancestors`, `nosniff`, `Referrer-Policy`), rà script bên thứ ba trên luồng booking, kiểm tra Host/forwarded headers chỉ để đối chiếu allowlist, đảm bảo không bí mật trong bundle/HTML/props, cấu hình CORS không mở.
- **Acceptance:** theo [kiểm tra trước khi mở booking](../security/README.md#kiểm-tra-trước-khi-mở-booking): gửi từ origin lạ, giả `Host`/`X-Forwarded-Host`, body quá lớn, URL ảnh ngoài allowlist đều bị từ chối hoặc giới hạn; WEB-10 có bằng chứng (header, sitemap, log, cache).

### FE-10 Bộ test và bằng chứng

- **Nội dung:** unit (schema, ánh xạ lỗi, định dạng), component (form, label, bàn phím, lỗi trường), E2E (tìm phòng → quote → submit; Instant, Request, hết phòng, giá đổi, mode đổi, timeout, chống gửi trùng; Manage Booking), accessibility, responsive 320/390/768/1440, Chromium + WebKit, thu ảnh nghiệm thu vào `docs/designs/screenshots/` theo quy tắc đặt tên.
- **Acceptance:** mỗi mã WEB-01…10 và AC-01…22 liên quan có kết quả **Đạt/Lỗi/Chưa xác minh** kèm bằng chứng, revision, môi trường, PMS test hay fixture ([test-strategy.md](../testing/test-strategy.md)). Kiểm thử bằng mock không được dùng thay bằng chứng PMS test cho hành vi ghi.

### FE-11 Triển khai

- **Phụ thuộc:** FE-09, BE-22.
- **Nội dung:** Dockerfile, cấu hình runtime Node cho SSR + Route Handler (không static export), biến môi trường theo instance, Caddy/HTTPS, canonical và alias redirect, làm mới cache nội dung công bố (khớp BE-05), cô lập staging (kiểm soát truy cập), kiểm tra `no-store` qua proxy/CDN, kế hoạch ngừng nhận booking và rollback, ghi tương thích phiên bản website↔PMS ([deployment-guide.md](../deployment/deployment-guide.md)).
- **Acceptance:** AC-01: mở subdomain/domain đã cấu hình ⇒ đúng khách sạn, branding, HTTPS, canonical; `Host` ngoài allowlist bị từ chối; smoke production chỉ đọc; rollback diễn tập được trên staging.

## 8. Ma trận AC → task

| AC | Task chính (PMS) | Task chính (website) |
| --- | --- | --- |
| AC-01 | BE-04, BE-08, BE-22 | FE-03, FE-04, FE-11 |
| AC-02 | BE-06, BE-08, BE-17 | FE-05 |
| AC-03 | BE-05, BE-06, BE-07 | FE-05 |
| AC-04 | BE-13 | FE-06 |
| AC-05 | BE-12 | FE-06, FE-07 |
| AC-06 | BE-12, BE-17 | FE-07 |
| AC-07 | BE-12, BE-17 | FE-07 |
| AC-08 | BE-09, BE-10a, BE-10, BE-13 | FE-06, FE-07 |
| AC-09 | BE-10a, BE-10, BE-11 | — |
| AC-10 | BE-15, BE-17 | FE-07 |
| AC-11 | BE-18 | FE-07 |
| AC-12 | BE-13, BE-17, BE-18 | FE-07 |
| AC-13 | BE-12, BE-17, BE-21 | — |
| AC-14 | BE-14, BE-17, BE-21 | FE-07 |
| AC-15 | BE-19 | FE-08 |
| AC-16 | BE-16, BE-20 | FE-08 |
| AC-17 | BE-20 | FE-08 |
| AC-18 | BE-16, BE-20 | FE-08 |
| AC-19 | BE-03, BE-16 | FE-06, FE-08 |
| AC-20 | BE-08 | FE-04, FE-05 |
| AC-21 | — | FE-03…FE-08, FE-10 |
| AC-22 | BE-17 (không có thanh toán) | FE-07 |

GOV-05 và GOV-06 là cổng thiết kế, không ánh xạ AC trực tiếp: GOV-05 là điều kiện cho FE-03…FE-08, GOV-06 là điều kiện cho phần UI quản trị PMS của BE-04, 06, 07, 11, 16, 18 (AC-21 vẫn áp dụng cho UI website). BE-10a là task thiết kế nên không có ca kiểm thử AC riêng; AC-08/09 được chứng minh ở BE-10 trên nền thiết kế đã duyệt.

Mỗi AC phải có ít nhất một ca tự động hoặc bằng chứng thủ công trước khi PO nghiệm thu; bảng ánh xạ AC↔WEB-xx của [acceptance-criteria.md](../requirements/acceptance-criteria.md) cần được lập khi viết hợp đồng (F-15).

## 9. Kiểm thử và bằng chứng

| Tầng | PMS | Website |
| --- | --- | --- |
| Đơn vị | quy tắc sức chứa, bộ gán, giá hiệu lực, cutoff, fingerprint, token | schema, ánh xạ lỗi, định dạng ngày/tiền, reducer tiêu chí |
| Tích hợp | Testcontainers: migration, repository, service, security chain, outbox | adapter với mock server, Route Handler |
| Đồng thời | tranh phòng cuối, đặt lặp cùng khóa, đặt với `confirm`/hủy của nhân viên | gửi lặp từ UI, race phản hồi tìm kiếm |
| Hợp đồng | response khớp fixture GOV-04 | fixture dùng chung là nguồn cho schema |
| Bảo mật | rate limit, enumeration, header giả, PII trong log | CSP, origin, no-store, bundle không bí mật |
| E2E / UI | màn hình quản trị theo `responsive-design.md` | Playwright: Chromium + WebKit, 320/390/768/1440, bàn phím |
| Staging | build thật, PMS test, dữ liệu giả | canonical, sitemap, domain, cache giao dịch qua proxy |

Báo cáo theo [test-strategy.md](../testing/test-strategy.md): mã ca, revision website và PMS, môi trường, loại bằng chứng, lệnh/bước, kết quả **Đạt/Lỗi/Chưa xác minh**. Hành vi ghi chỉ coi là đạt khi có PMS test (không chỉ mock).

## 10. Bàn giao và Definition of Done

- **Definition of Ready:** Issue/Task đã Ready (PMS) hoặc được giao (website); quyết định chặn đã chốt; hợp đồng liên quan đã duyệt; tiêu chí truy được về `client-specs.md`. Task có UI: mockup tương ứng đã được PO duyệt (website: GOV-05; màn hình quản trị PMS: GOV-06). Task BE-10 (BE-10b): spike BE-10a đã được PO duyệt.
- **Definition of Done (mỗi task):**
  1. Đáp ứng acceptance của task; không đổi phạm vi V2A.
  2. Lệnh lint/typecheck/build/test của repo đã chạy; ghi lệnh, kết quả, phạm vi; phần chưa xác minh ghi rõ lý do.
  3. UI: ghi màn hình, trạng thái, viewport, browser đã xem; ảnh nghiệm thu theo quy ước.
  4. Tích hợp PMS: ghi hợp đồng đã đối chiếu và ca nào dùng fixture hay PMS test.
  5. Tự review (`/review-frontend` cho website); gọi `frontend-architect`/`ui-ux-reviewer`/`frontend-test-reviewer` khi đúng góc nhìn.
  6. Bàn giao diff local cho Codex review độc lập; sửa lỗi quay lại Claude. Stage/commit/push/PR/deploy chỉ khi PO chỉ thị riêng cho từng thao tác. Phía PMS: mỗi Issue một commit tập trung sau phê duyệt local của PO.
- **Mẫu bàn giao cho Codex:** phạm vi + tiêu chí, base/trạng thái diff, file liên quan, kết quả kiểm tra, mục còn mở ([CLAUDE.md](../../CLAUDE.md#bàn-giao-cho-codex)).

## 11. Rủi ro và giảm thiểu

| Rủi ro | Mức | Giảm thiểu |
| --- | --- | --- |
| Đổi lõi tài chính khi thêm snapshot giá theo đêm (BE-10) | Cao | Spike thiết kế BE-10a được PO duyệt trước BE-10b; dữ liệu cũ giữ giá phẳng; bộ hồi quy folio/báo cáo bắt buộc |
| Overbooking hoặc deadlock do khóa nhiều Room Type | Cao | Một thứ tự khóa toàn cục; tái dùng truy vấn xung đột; test đồng thời trong CI (BE-21) |
| Lộ PII/token (log, URL, referrer, analytics, cache) | Cao | Quy tắc log, `no-store`, `Referrer-Policy`, test bắt log, kiểm tra bundle/HTML ở staging |
| Enumeration và spam email qua Manage Booking | Trung bình | Phản hồi đồng nhất, rate limit nhiều khóa, TTL ngắn, token hash |
| Trùng/lệch phạm vi với #33 (email), #89, #90 | Trung bình | Ghi quan hệ ở GOV-01; outbox tối thiểu có thể mở rộng; đường công khai tự kiểm tra ngày |
| Lẫn `DRAFT` với Request | Trung bình | Quyết định D-01 trước BE-18; test bảo vệ trạng thái |
| Hai định nghĩa sức chứa gây nhầm | Trung bình | Ghi trong đặc tả PMS; tên lớp phân biệt; test riêng |
| Cache nội dung cũ sau Publish | Trung bình | `contentVersion` + tín hiệu làm mới; tách nội dung công bố khỏi giao dịch |
| Thiếu mockup, làm lại giao diện | Trung bình | GOV-05 trước FE-03; logic/adapter chạy trước bằng fixture |
| Thiếu mockup PMS Admin, làm lại màn hình quản trị | Trung bình | GOV-06 trước phần UI của BE-04, 06, 07, 11, 16, 18; backend/API chạy trước bằng fixture |
| Múi giờ cấu hình lệch `Clock` | Trung bình | Cảnh báo vận hành; ghi giới hạn; D-06 |
| Khối lượng lớn, nhiều Issue con | Trung bình | Tách theo bảng task, mỗi Issue một commit, ưu tiên lát cắt Instant (M3) |

## 12. Điều cần xác nhận trước khi bắt đầu

Chặn bắt đầu (chi tiết và phương án ở [gap analysis mục 7](../audits/issue-30-gap-analysis.md#7-quyết-định-cần-xác-nhận)):

| ID | Quyết định | Task bị chặn |
| --- | --- | --- |
| D-01 | Vòng đời Request | BE-18, GOV-04, FE-07 (nhánh Request) |
| D-02 | Biểu diễn giá theo đêm | BE-09, BE-10a, BE-10, BE-11 |
| D-03 | Phạm vi quy tắc sức chứa | BE-12 |
| D-04 | Thời điểm gán phòng vật lý | BE-12, BE-17 |
| D-05 | Xác thực Next.js → PMS và phơi bày API | BE-01, GOV-04 |
| D-06 | Múi giờ/cutoff | BE-03, BE-16 |
| D-07 | Tách Issue, đặc tả PMS-0034, vị trí gói mã | GOV-01, GOV-02, mọi task PMS |
| D-08 | Email: outbox riêng hay gộp #33; nhà cung cấp, sender | BE-15 |
| D-09 | Mockup đã duyệt | FE-03 và các màn hình |

Có mặc định, không chặn: D-10 (domain), D-11 (giới hạn đặt, nhãn thuế/phí), D-12 (khử trùng Guest), D-13 (định dạng và số ảnh Room Type), D-14 (kỳ bảo trì tương lai).

Cho tới khi D-01 chốt, nhánh Request chỉ được mô tả trong hợp đồng và UI, không triển khai ghi.

## 13. Cập nhật M0 — Quyết định đã chốt (2026-11-10)

D-01…D-09 ở mục 12 đã được Product Owner chốt (chi tiết: [gap analysis mục 10](../audits/issue-30-gap-analysis.md#10-cập-nhật-m0--quyết-định-đã-chốt-2026-11-10); ADR: `docs/decisions/004`…`012`). Đây thực hiện GOV-03. Trạng thái các task cổng khác:

- **GOV-01** (tách Issue con, Backlog → Ready): **chưa thực hiện** — vẫn là quyết định riêng của Product Owner, không tự động theo việc chốt D-01…D-09.
- **GOV-02** (đặc tả PMS-0034 trong PMS): đã có bản đầu tại `hotel-system/docs/specs/PMS-0034-hotel-website-direct-booking-v2a.md`, **vị trí khác đề xuất ban đầu của task này** (`technical-spec-v2.md`) — D-07 chốt `docs/specs/` thay vào đó (xem ADR 010). Nội dung chưa qua Codex review; không coi là "PO duyệt đặc tả" theo tiêu chí Acceptance gốc của GOV-02 cho tới khi có review đó.
- **GOV-03** (chốt quyết định, ghi ADR): **hoàn tất phần ghi ADR** (004–012). Mỗi ADR có quyết định, bối cảnh và hệ quả; phương án đã loại không được ghi riêng vì D-01…D-09 được PO chốt trực tiếp theo đề xuất có sẵn ở gap analysis (D-06 thu hẹp phạm vi so với đề xuất, lý do ghi trong ADR 009), không qua so sánh nhiều phương án mới.
- **GOV-04** (hợp đồng Public API v1): đã có bản đầu tại `hotel-system/docs/specs/PMS-0034-public-api-contract.md` (OpenAPI/fixture thật **chưa có**, chỉ mô tả operation/schema dạng văn bản + ví dụ JSON). **Chưa đạt Acceptance của GOV-04**: thiếu fixture đã lọc cho các nhánh lỗi, chưa có review đồng thuận hai repo, chưa cập nhật `api-integration.md` với trạng thái "đã chốt" (chỉ cập nhật tham chiếu — xem file đó).
- **GOV-05** (mockup đã duyệt): **chưa thực hiện** — [`docs/designs/mockup-requirements-issue-30.md`](../designs/mockup-requirements-issue-30.md) chỉ là danh mục yêu cầu (ADR 012), không phải mockup thật.

**Task vẫn bị chặn sau cập nhật này:** mọi task BE/FE phụ thuộc GOV-01 (Ready) và review Codex cho GOV-02/GOV-04 chưa thể coi là "mở khóa" theo đúng nghĩa Definition of Ready ở mục 10 — việc chốt D-01…D-09 mở khóa phần THIẾT KẾ (không còn phải chờ quyết định nghiệp vụ), nhưng chưa mở khóa phần THỰC THI (code), vì Issue chưa Ready và đặc tả/hợp đồng chưa qua review độc lập.

## 14. Cập nhật Codex Review 2026-11-11 — REQUEST CHANGES

> **Round 3 (2026-11-11) đã thay thế một phần mục này.** Codex re-review #2 trả REQUEST CHANGES lần nữa. Nội dung 14.1 về "`nightly_rate` là giá trung bình phái sinh" và các quyết định (a)/(b) **không còn hiệu lực**; bảng 14.2 được hiệu chỉnh ở [mục 15](#15-cập-nhật-codex-re-review-2-round-3--2026-11-11). Khi hai mục mâu thuẫn, mục 15 thắng.

Codex review độc lập đặc tả PMS-0034 (mục 13), verdict **REQUEST CHANGES** (2 BLOCKER, 9 MAJOR, 2 MINOR; 200/200 baseline test PASS). Chi tiết đầy đủ: [`PMS-0034-codex-review-resolution-2026-11-11.md`](../../../hotel/docs/specs/PMS-0034-codex-review-resolution-2026-11-11.md) (repo `hotel-system`). Bốn quyết định Product Owner bổ sung PO-01…PO-04 ghi tại ADR 013-016.

### 14.1 Task mới — BE-10a (tách khỏi BE-10, bắt buộc trước BE-10b)

| ID | Repo | Tên | Phụ thuộc | Mốc | Cỡ |
| --- | --- | --- | --- | --- | --- |
| **BE-10a** | hotel-system | Spike thiết kế tương thích tài chính cho nightly price snapshot (BLOCKER-01) | BE-09 | M2 | M |

**[ĐÃ THAY THẾ bởi mục 15 và round 3 §2 — thiết kế "trung bình" bị loại]** **Nội dung BE-10a:** xác nhận bằng chứng source code đã thực hiện qua Codex review (`MonthlyFinancialReportService.verifySnapshot`/`verifyExtension`, migration V33 `ux_charge_source_reservation_room`, `StayExtensionService.extend`) và thiết kế đã chốt (`nightly_rate` là giá trung bình phái sinh, breakdown thật ở bảng riêng — [resolution mục 2](../../../hotel/docs/specs/PMS-0034-codex-review-resolution-2026-11-11.md#2-blocker-01--nightly-pricing--financial-compatibility)). Việc còn lại của BE-10a: Product Owner duyệt phương án (a) hay (b) cho ghi nhận doanh thu vắt tháng (mục 2.3 tài liệu resolution) trước khi BE-10b viết migration thật.
**Acceptance BE-10a:** Product Owner duyệt phương án ghi nhận doanh thu; không bắt đầu BE-10b trước khi có phê duyệt này (giữ nguyên yêu cầu gốc của BE-10).

GitHub Issue: [hotel-system#150](https://github.com/chutienkhoa/hotel-system/issues/150), sub-issue của #30; Issue BE-10 ([hotel-system#137](https://github.com/chutienkhoa/hotel-system/issues/137)) đã trỏ Dependencies vào đó. Task đã được đưa vào bảng danh mục (mục 4), đồ thị (mục 3.4) và ma trận AC (mục 8).

### 14.2 Cập nhật Dependencies (task catalog mục 4)

| ID | Dependencies cũ | Dependencies mới |
| --- | --- | --- |
| BE-10 | BE-09 (+ spike) | BE-09, **BE-10a** |
| BE-12 | GOV-03 (D-03, D-04), BE-06 | GOV-03 (D-03, D-04, **PO-02**), BE-06 |
| BE-16 | BE-03, BE-04 | BE-03, BE-04, **BE-10 (PO-03 — đối chiếu breakdown theo calendar date khi đổi ngày/gia hạn)** |
| BE-17 | BE-01, 02, 10, 12, 13, 14, 15, 16 | Không đổi (đã bao gồm BE-10, nay gián tiếp qua BE-10a) |
| BE-18 | GOV-03 (D-01), BE-17 | GOV-03 (D-01, **PO-01, PO-02**), BE-17, **BE-02 (permission `MANAGE_BOOKING_REQUEST`)** |
| BE-19 | BE-15, BE-17 | BE-15, BE-17 — nội dung mở rộng theo MAJOR-05 (token hash, atomic exchange) |
| BE-20 | BE-16, BE-19 | BE-16 (**+PO-04**, kiểm tra `cancellation_free_until IS NOT NULL`), BE-19 |
| BE-01 | GOV-03 (D-05), GOV-04 | Không đổi — nội dung mở rộng theo MAJOR-02 (fail-closed, scope, rotation) |
| BE-02 | GOV-02, GOV-03 | GOV-02, GOV-03 — nội dung mở rộng: thêm permission `MANAGE_BOOKING_REQUEST` (PO-01) + kích hoạt `booking_source = HOTEL_WEBSITE` (cần PO xác nhận) |
| BE-03 | GOV-03 (D-06, D-11) | Không đổi — nội dung mở rộng theo MAJOR-06: `HotelTimeProvider` riêng, KHÔNG chạm `DashboardClockConfiguration` |
| BE-09, BE-11 | — | Không đổi phụ thuộc; BE-11 nội dung mở rộng theo MAJOR-09 (`version` optimistic lock) |
| BE-14 | BE-01 | Không đổi — nội dung mở rộng theo MAJOR-04: bảng riêng `public_booking_idempotency_record`, KHÔNG tái dùng `PaymentIdempotencyService` |

### 14.3 Đường găng cập nhật

`GOV-01 → GOV-03 → GOV-04 → BE-09 → BE-10a → BE-10b → BE-12 → BE-13 → BE-17 → BE-18 → BE-19 → BE-20 → BE-21` (chèn **BE-10a** trước BE-10, **BE-18** nối trực tiếp sau BE-17 trên đường găng do phụ thuộc PO-01/PO-02 mới).

## 15. Cập nhật Codex Re-review #2 (Round 3) — 2026-11-11

Tài liệu thiết kế: [PMS-0034-codex-round3-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round3-remediation-2026-11-11.md) (bằng chứng source, thiết kế cuối, test, ma trận truy vết). Schema: `hotel-system/docs/specs/PMS-0034-data-design.md` (mã logic M01–M13). Hợp đồng: `hotel-system/docs/specs/PMS-0034-public-api-contract.md` + `docs/development/contracts/public-api-v1/`. ADR mới: [017](../decisions/017-nightly-pricing-flat-and-daily-modes.md). **Lưu ý trung thực:** báo cáo Codex re-review #2 nguyên văn không có trong repo; các finding được ánh xạ từ yêu cầu của Product Owner, mã `R2-B01`/`R2-M01…M09` là mã nội bộ của phiên này.

### 15.1 Thứ tự migration cuối (mã logic; số Flyway vật lý cấp lúc merge)

Quy tắc: một migration chỉ được merge sau khi các migration nó phụ thuộc đã merge; **không hardcode V-number** trong Issue hay mã (CI guard `check_migration_integrity.py` đòi version cao hơn baseline; V59–V61 reserved vĩnh viễn; hiện cao nhất V67).

| Logical | Provisional | Nội dung | Task sở hữu | Issue |
| --- | --- | --- | --- | --- |
| M01 | V68 | Foundation: actor hệ thống, permission | BE-02 | hotel-system#129 |
| M02 | V69 | `room_type` trường website | BE-06 | #133 |
| M03 | V70 | `room_type_image` | BE-07 | #134 |
| M04 | V71 | pricing revision + `room_type_rate_override` + trigger | BE-09 | #136 |
| M05 | V72 | `room_type_rate_history` | BE-11 | #138 |
| M06 | V73 | Nightly pricing (`pricing_mode`, `room_night_rate`) | BE-10 (BE-10b) | #137 |
| M07a | V74 | `cancellation_policy_revision` | BE-04 | #131 |
| M07b | V75 | Snapshot chính sách trên `reservation` (sau M07a) | BE-16 | #143 |
| M08 | V76 | `outbox_event` | BE-15 | #142 |
| M09 | V77 | `public_idempotency_record` (sau M01) | BE-14 | #141 |
| M10 | V78 | `public_booking_quote` (sau M04, M07a) | BE-13 | #140 |
| M11 | V79 | `reservation.public_reference` | BE-17 | #144 |
| M12 | V80 | Booking Request | BE-18 | #145 |
| M13 | V81 | `manage_booking_access_token` (sau M09) | BE-19 | #146 |

### 15.2 Dependencies hiệu chỉnh (thay bảng 14.2 ở các dòng liên quan)

| ID | Dependencies hiện hành | Thay đổi so với 14.2 |
| --- | --- | --- |
| BE-12 | GOV-03 (D-03, D-04), BE-06 | Bỏ "PO-02" (BE-12 không còn sở hữu schema Request) |
| BE-14 | BE-01, **BE-02** | Thêm BE-02 (actor, M01 trước M09) |
| BE-16 | BE-03, BE-04 | **Gỡ BE-10/PO-03** (BE-16 không đổi giá; PO-03 do BE-10b sở hữu) |
| BE-18 | GOV-03 (D-01, PO-01, PO-02), BE-17, BE-02 | Sở hữu M12; bỏ "rút yêu cầu của khách" |
| BE-20 | BE-16 (+PO-04), BE-19, BE-14 | Thêm BE-14 (`BOOKING_CANCEL`) |
| BE-10 | BE-09, BE-10a | Chủ sở hữu PO-03 và test hồi quy |

### 15.3 Truy vết sửa

- **BE-10a = hotel-system#150**, không phải #149 (#149 là BE-21).
- #139 (BE-12) không còn yêu cầu `room_id` nullable; #137/#142/#143 dùng mã logic M06/M08/M07b thay `V72`/`V75`/`V74`.
- Chủ sở hữu PO-03: #137 (BE-10b). Chủ sở hữu bảng Request: #145 (BE-18).
- Tham chiếu `reservationReference` sai ở API contract đã sửa (trỏ về round 3 §4.5; PO-Q6 đã duyệt).

### 15.4 Quyết định PO đã duyệt (2026-11-11) và điểm còn mở

**Đã duyệt:** PO-Q1 (FLAT/DAILY; không sửa `technical-spec-v1`; quy tắc ưu tiên có phạm vi — v2a mục 17), PO-Q3 (quyền duyệt Request hiệu lực, ADMIN thu hồi được), PO-Q4 (`MANAGE_WEBSITE_CONFIG`), PO-Q5 (`HOTEL_WEBSITE` do ADMIN cấu hình, fail-closed), PO-Q6 (`public_reference`), PO-Q7 (snapshot hủy bất biến + quy trình chấp nhận định nghĩa), PO-Q8 (không tự hủy/rút Request), PO-Q9 (`EXPIRED` riêng), PO-Q10 (Room Type đủ điều kiện lúc duyệt, giữ giá). Ghi tại [ADR 017](../decisions/017-nightly-pricing-flat-and-daily-modes.md) và [ADR 018](../decisions/018-po-decisions-round3-q3-q10.md); thiết kế: round 3 §13.1.

**Còn mở (không chặn Codex re-review #3; round 3 §13.2):** O1 `STAFF_NEVER` so với "by default"; O2 quyền Rate Calendar/Gallery; O3 định nghĩa "đủ điều kiện"; O4 đổi ngày sớm hơn cutoff đã chấp nhận; O5 VND; O6 CLAUDE.md chưa trỏ tới v2a mục 17; O7 tham số D-11 và audit `ReservationBoardQueryService` trước BE-18.

**Cổng BE-10b:** #150 được PO duyệt **và** Codex re-review #3 xác nhận.

## 16. Cập nhật Codex Re-review #3 (Round 4) — 2026-11-11

Codex re-review #3 trả REQUEST CHANGES (F1–F4 và bốn MINOR). Thiết kế, bằng chứng chạy thật và test: [PMS-0034-codex-round4-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round4-remediation-2026-11-11.md). Mọi mục `RESOLVED IN DESIGN — Claude assessment`, **chưa** Codex xác nhận.

| Mã | Nội dung | Issue chủ |
| --- | --- | --- |
| F1 | Đồng bộ catalog quyền (`GROUPS`, policy, thông điệp, test, làm mới authority đã xác minh) | BE-02 (#129) |
| F2 | Outbox: ràng buộc, fence, giới hạn lượt thử cả đường phục hồi, at-least-once | BE-15 (#142) |
| F3 | Quote: một transaction `REPEATABLE_READ` cho phép ghi; trigger toàn vẹn | BE-13 (#140), BE-09 (#136) |
| F4 | `roomTypeCode` tương thích mã V1 | GOV-04 (#126), FE-02 (website#3), BE-13 (#140) |
| M1 | Reconcile union theo `state` + ràng buộc DB | GOV-04, BE-14 (#141), FE-02 |
| M2 | Fail-closed S2S: không khởi động | BE-01 (#128) |
| M3 | `201` thống nhất; replay ở header | GOV-04, BE-14, BE-17 (#144), BE-18 (#145), BE-20 (#147), FE-07 (website#8), FE-08 (website#9) |
| M4 | Ước tính gia hạn DAILY theo preview backend | BE-10 (#137), BE-10a (#150) |

Đồ thị phụ thuộc không đổi ở Round 4 (không cạnh mới, không Issue mới); 40 sub-issue của Epic #30 giữ nguyên. Round 5 (mục 17) thêm đúng một cạnh BE-01 → BE-02.

## 17. Cập nhật Codex Re-review #4 (Round 5) — 2026-11-11

Codex re-review #4: R4-01 (MAJOR, trigger quote), R4-02 (MAJOR, phụ thuộc triển khai BE-01), R4-03 (MINOR, CHECK idempotency). Thiết kế, bằng chứng và test: [PMS-0034-codex-round5-remediation-2026-11-11.md](../../../hotel/docs/specs/PMS-0034-codex-round5-remediation-2026-11-11.md). Mọi mục `RESOLVED IN DESIGN — Claude assessment` (chưa Codex xác nhận).

| Mã | Nội dung | Issue chủ |
| --- | --- | --- |
| R4-01 | Trigger quote: so tập chính xác Room Type/revision/ngày, `NULL`-safe, giá thiếu bị từ chối | BE-13 (#140) |
| R4-02 | BE-01 phụ thuộc BE-02/M01; thứ tự triển khai | BE-01 (#128), BE-02 (#129), BE-22 (#148) |
| R4-03 | `CHECK` idempotency theo trạng thái, không để `NULL` lọt | BE-14 (#141) |

**Cập nhật Dependencies (thay dòng BE-01 ở các bảng 14.2 và 15.2):**

| Task | Trước | Sau |
| --- | --- | --- |
| BE-01 | GOV-03 (D-05), GOV-04 | GOV-03 (D-05), GOV-04, **BE-02** |

BE-02 chỉ phụ thuộc GOV-02, GOV-03 ⇒ chạy song song với GOV-04; không có đường nào từ BE-02 về BE-01 ⇒ đồ thị 40 task vẫn không chu trình (đã kiểm trên thân 40 Issue: 79 cạnh, sắp xếp tô-pô phủ 40/40). Độ sâu của BE-01 vẫn là 4 (BE-02 và GOV-04 cùng độ sâu 3) nên đường găng không đổi. 40 sub-issue của Epic #30 giữ nguyên; không Issue mới.
