# Chiến lược kiểm thử V2A

**Đề xuất áp dụng khi ứng dụng được khởi tạo.** Phạm vi và tầng kiểm tra dựa trên quy tắc frontend. Dùng công cụ, lockfile và script thực tế của repo khi có. [Quy tắc frontend](../coding-rules/frontend.md#kiểm-tra-khi-thay-đổi-mã)

## Chọn kiểm tra theo rủi ro

| Thay đổi | Kiểm tra trước khi bàn giao | Khi nào mở rộng |
| --- | --- | --- |
| Template Modern Tropical hoặc component trình bày | Mở trực tiếp màn hình và trạng thái liên quan trên mobile, tablet và desktop. So với mockup đã duyệt, kiểm tra nội dung dài, bàn phím, focus và CTA. | Kiểm tra các route dùng chung layout hoặc token Tailwind bị đổi. Nếu thay đổi metadata, xem HTML và URL canonical thực tế. |
| Form hoặc điều phối booking | Component test kiểm tra thao tác, lỗi trường và cập nhật báo giá. E2E với fixture cho các nhánh booking bị ảnh hưởng. | Với thay đổi dùng chung, chạy lại hành trình từ chọn ngày đến kết quả trên Chromium và WebKit. |
| Adapter PMS hoặc Route Handler | Unit test cho decode, ánh xạ lỗi và timeout. Kiểm tra tích hợp biên HTTP, schema, giới hạn dữ liệu công khai và cache theo thao tác. | Chạy ca đối chiếu trên PMS test khi hợp đồng, quyền, báo giá, trạng thái hoặc chống ghi trùng thay đổi. |
| Domain, cache, secrets hoặc cấu hình build | Kiểm tra bản build phục vụ thật trong staging, gồm HTTPS, host, canonical, sitemap, staging access và cache giao dịch. | Rà lại các đường đi của PII và bí mật qua response, log, trình duyệt và proxy/CDN. |

Kèm lint, typecheck và build theo script của repo cho thay đổi mã. Với UI, dùng các mốc viewport đề xuất trong quy tắc frontend và ghi rõ ảnh nào đã mở. Ảnh kiểm tra lưu theo hướng dẫn thiết kế khi cần bàn giao. [Tư liệu thiết kế](../designs/README.md)

## Dữ liệu và mức bằng chứng

- **Fixture và mock** dùng cho test nhanh và ca khó tái hiện. Giữ fixture có thể lặp lại, bám schema PMS đã xác nhận khi có và dùng dữ liệu khách giả lập. Đối với tìm phòng, tạo ca phản hồi cũ đến sau khi khách đổi tiêu chí.
- **PMS test** dùng để xác minh quyền và hành vi tại biên thực. Ghi version hợp đồng, cấu hình Instant hoặc Request, ngày, hạng phòng và giá test. Với timeout sau khi PMS đã ghi hoặc hai lượt gửi đồng thời, kiểm tra số bản ghi qua cách đối soát được cấp quyền. Kết quả mock chỉ chứng minh hành vi của website.
- **Staging** dùng để kiểm tra đúng bản build, cấu hình domain, proxy/CDN, SEO và hành vi trình duyệt. Dùng upstream test và dữ liệu giả lập, giới hạn truy cập và tránh đụng dữ liệu của phiên kiểm tra khác.

## Báo cáo và điều kiện nghiệm thu

Ghi mã ca, phiên bản website và PMS, môi trường, loại bằng chứng, lệnh hoặc bước chạy, browser/viewport khi áp dụng và kết quả thực tế. Tách **Đạt**, **Lỗi** và **Chưa xác minh**, kèm lý do và bước tái hiện. Khi test hồi quy, dùng lại mã ca liên quan và bổ sung bằng chứng cho bản mới.

Trước khi mở booking thật, cần bằng chứng PMS test cho luồng gửi, kết quả Instant/Request và các ca đối soát, cùng bằng chứng staging cho domain, cache giao dịch và bảo mật dữ liệu khách. [Tiêu chí nghiệm thu](../requirements/acceptance-criteria.md), [Kiểm tra bảo mật](../security/README.md#kiểm-tra-trước-khi-mở-booking), [Hướng dẫn triển khai](../deployment/deployment-guide.md)
