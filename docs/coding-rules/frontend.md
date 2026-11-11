# Quy tắc frontend Next.js, TypeScript và Tailwind CSS

Bộ quy tắc **đề xuất** cho lần khởi tạo mã nguồn V2A. Đối chiếu [đặc tả](../specs/client-specs.md), [kiến trúc](client-architecture.md) và [bảo mật](../security/README.md) trước khi thay đổi hành vi.

## Next.js và React

- Dùng App Router. Mặc định dùng Server Components cho trang, layout và phần chỉ hiển thị dữ liệu. Đặt `'use client'` ở component nhỏ nhất cần state, event handler hoặc API trình duyệt, ví dụ bộ chọn ngày. Props qua ranh giới này phải tuần tự hóa được và chỉ chứa dữ liệu cần cho UI.
- Tổ chức mã theo nghiệp vụ như trong [kiến trúc client](client-architecture.md#cấu-trúc-source-dự-kiến). `app/` ghép route, layout và metadata. `features/` giữ UI, state và logic của từng tính năng. `templates/` bố trí nội dung, `components/ui/` chứa UI dùng chung và `lib/pms/` chứa tích hợp PMS. Mọi template dùng cùng luồng trong `features/booking`.
- Server Components gọi trực tiếp adapter PMS trên server. Tương tác từ trình duyệt cần HTTP đi qua Route Handler cùng origin theo hợp đồng trong kiến trúc client.

Tham chiếu [Server và Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components) và [Fetch API của Next.js](https://nextjs.org/docs/app/api-reference/functions/fetch).

## TypeScript và schema API

- Bật `strict` và cấm `any` tường minh trong mã ứng dụng, có thể kiểm tra bằng rule `@typescript-eslint/no-explicit-any` khi thiết lập lint. Dùng `unknown` cho dữ liệu chưa kiểm tra rồi thu hẹp kiểu.
- Validate request ở Route Handler và response từ PMS bằng schema runtime, gồm cả payload lỗi mà ứng dụng đọc. Kiểm tra kiểu, trường bắt buộc và ràng buộc theo hợp đồng trước khi ánh xạ sang DTO. Validation ở form giúp khách sửa sớm. Server vẫn kiểm tra lại tại biên.
- Đặt tên rõ nghĩa cho DTO và view model. Dùng union phân biệt các kết quả như xác nhận, nhận yêu cầu, cần báo giá lại và chưa rõ kết quả. Giá trị lạ từ PMS được xử lý như phản hồi không hợp lệ.
- Xử lý dữ liệu thiếu bằng kiểm tra kiểu hoặc nhánh lỗi rõ ràng. Cấm ép kiểu hay dùng `!` để bỏ qua kiểm tra đó. Dùng PascalCase cho React component, kebab-case cho thư mục nghiệp vụ và tên file route theo quy ước App Router.
- Giữ ngày lưu trú dưới dạng ngày lịch theo API, ví dụ `YYYY-MM-DD` nếu hợp đồng chọn định dạng này. Định dạng ngày, giờ và tiền tệ qua hàm dùng chung. Giá hiển thị dùng amount và currency từ PMS, theo đơn vị hợp đồng quy định.

Tham chiếu [TypeScript strict](https://www.typescriptlang.org/tsconfig/strict).

## API client, bảo mật và booking

- Tập trung HTTP client gọi PMS trong `lib/pms/`. Đặt upstream, xác thực, timeout có hủy request và kiểm tra HTTP status ở đây. Adapter theo nghiệp vụ giải mã schema và trả lỗi có kiểu để feature phân biệt dữ liệu sai, lỗi nghiệp vụ, timeout, lỗi kết nối và phản hồi PMS không hợp lệ.
- Đặt chính sách cache theo từng thao tác. Dữ liệu công bố có thể cache theo kiến trúc. Availability, báo giá, gửi và kết quả booking dùng fetch không cache và phản hồi giao dịch dùng `Cache-Control: no-store`. Kiểm tra cả proxy/CDN.
- Đánh dấu adapter PMS và cấu hình chứa bí mật là `server-only`. `NEXT_PUBLIC_*` chỉ dành cho giá trị được phép công khai. Giữ token, API key và thông tin xác thực PMS trong biến môi trường phía server. Loại các trường nội bộ trước khi đưa DTO vào HTML, props hay response công khai.
- Cấm log PII như tên, email, số điện thoại và ghi chú của khách. Áp dụng cho console, log server, giám sát và báo lỗi. Chỉ ghi tên thao tác, mã theo dõi đã lọc, nhóm lỗi và thời gian xử lý. Dùng thông báo an toàn cho khách theo [quy tắc bảo mật](../security/README.md).
- PMS quyết định giá, tình trạng còn phòng và kết quả đặt phòng. Frontend hiển thị dữ liệu PMS trả về, định dạng theo tiền tệ và đơn vị trong hợp đồng. Cấm tự tính tổng giá từ giá cơ sở hoặc suy ra availability từ dữ liệu đã cache.
- Khi gửi booking bị timeout, hiển thị trạng thái chưa rõ kết quả. Chỉ gửi lại với cùng khóa chống trùng và cùng nội dung nếu hợp đồng cho phép, hoặc đối soát qua cơ chế PMS đã thống nhất.

Tham chiếu [biến môi trường](https://nextjs.org/docs/app/guides/environment-variables) và [bảo mật dữ liệu trong Next.js](https://nextjs.org/docs/app/guides/data-security).

## Ảnh, font và SEO

- Dùng `next/image` cho ảnh nội dung khi phù hợp. Khai báo kích thước hoặc khung tỷ lệ và `sizes` theo bố cục responsive. Giới hạn nguồn ảnh từ xa theo [quy tắc bảo mật](../security/README.md).
- Dùng `next/font` ở layout hoặc phạm vi theme cần font. Chỉ tải font và độ đậm thực sự dùng. Kiểm tra ký tự tiếng Việt và fallback của font.
- Đặt tiêu đề, mô tả và canonical cho từng trang công khai được lập chỉ mục. Dùng `metadata` cho nội dung cố định và `generateMetadata` cho trang động. Lấy `metadataBase` từ canonical origin đã cấu hình và đặt canonical của từng trang qua `alternates.canonical`.
- Dùng `app/sitemap.ts` để liệt kê URL canonical công khai và `app/robots.ts` để khai báo chính sách crawl và địa chỉ sitemap production. Giữ URL kết quả đặt phòng ngoài sitemap và áp dụng `noindex` nếu có trang kết quả. Bảo vệ staging và dữ liệu riêng bằng kiểm soát truy cập, độc lập với cấu hình robots.

Tham chiếu [Image](https://nextjs.org/docs/app/api-reference/components/image), [Font](https://nextjs.org/docs/app/getting-started/fonts), [Metadata](https://nextjs.org/docs/app/api-reference/functions/generate-metadata), [sitemap](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap) và [robots](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/robots).

## Tailwind CSS, responsive và khả năng tiếp cận

- Viết mobile-first, rồi bổ sung biến thể theo các breakpoint dùng chung khi nội dung cần thêm không gian. Dùng grid co giãn hoặc container query khi bố cục phụ thuộc chiều rộng của khối chứa.
- Khai báo token ngữ nghĩa cho nền, chữ, màu nhấn, focus, khoảng cách và typography. Modern Tropical ánh xạ màu và kiểu chữ vào token trong phạm vi theme. Dùng tên class đầy đủ hoặc bảng ánh xạ biến thể tĩnh để Tailwind nhận diện khi build.
- Dùng HTML ngữ nghĩa, label gắn với input, focus nhìn thấy, alt phù hợp, và thông báo trạng thái đọc được bằng công nghệ hỗ trợ. Với menu hoặc dialog, kiểm tra mở/đóng bằng bàn phím và trả focus về nút mở.
- Kiểm tra form ngày, thông báo lỗi, giá dài và nút đặt phòng ở chiều rộng hẹp. Nút cố định phải chừa chỗ cho nội dung và vùng an toàn của thiết bị. Trang không cuộn ngang khi dữ liệu thực tế dài hơn mockup.

Tham chiếu [responsive design](https://tailwindcss.com/docs/responsive-design) và [cách Tailwind nhận diện class](https://tailwindcss.com/docs/detecting-classes-in-source-files).

## Trạng thái giao diện

- Phân biệt loading, empty, error và success. Kết quả tìm kiếm hợp lệ nhưng hết phòng cần gợi ý sửa ngày hoặc tiêu chí. Lỗi kết nối cần thông báo gián đoạn và thao tác thử lại phù hợp. Chỉ hiển thị xác nhận hoặc đã nhận yêu cầu khi PMS trả đúng kết quả tương ứng.
- Dùng `loading.tsx` khi cần fallback lúc tải route, `error.tsx` cho ngoại lệ ngoài dự kiến của segment và `not-found.tsx` cho tài nguyên không tồn tại. Xử lý lỗi dự kiến như trường nhập sai, hết phòng hay báo giá thay đổi ngay trong luồng UI tương ứng.
- Giữ một nguồn trạng thái cho tiêu chí tìm phòng hiện hành. Hủy hoặc bỏ qua phản hồi của lượt tìm cũ. Hiển thị lỗi cạnh trường, giữ dữ liệu nhập và chuyển focus tới trường lỗi đầu tiên hoặc phần tóm tắt lỗi khi cần. Thông báo thay đổi trạng thái bằng cơ chế công nghệ hỗ trợ đọc được.

Tham chiếu [tải dữ liệu](https://nextjs.org/docs/app/getting-started/fetching-data) và [xử lý lỗi](https://nextjs.org/docs/app/getting-started/error-handling).

## Kiểm tra khi thay đổi mã

- Chạy lint, kiểm tra kiểu, build và test liên quan bằng các script của repo sau khi chúng được thiết lập.
- **Unit test** kiểm tra schema, ánh xạ lỗi và dữ liệu hiển thị. Với adapter, bao phủ phản hồi hợp lệ, lỗi PMS, payload sai schema và timeout. Dùng fixture cố định, không chứa PII thật.
- **Component test** kiểm tra form và component tương tác qua hành vi người dùng, gồm label, bàn phím, lỗi trường và các trạng thái hiển thị có liên quan.
- **E2E** kiểm tra tìm phòng, xem báo giá và gửi booking qua UI. Bao phủ kết quả Instant và Request, hết phòng hoặc báo giá đổi, cùng nhánh timeout và chống gửi trùng theo hợp đồng. Đối chiếu [tiêu chí nghiệm thu](../specs/client-specs.md#tiêu-chí-nghiệm-thu-đề-xuất) và xác minh luồng với PMS ở môi trường test trước khi bật booking thật.
- Thay đổi UI cần xem trực tiếp ở điện thoại, tablet và desktop. Mốc kiểm tra đề xuất là 320, 390, 768 và 1440 px. Chạy luồng booking trên Chromium và WebKit, đồng thời kiểm tra bàn phím và trường hợp nội dung dài. Với component dùng chung, kiểm tra các màn hình sử dụng component đó.

Tham chiếu [kiểm thử ứng dụng Next.js](https://nextjs.org/docs/app/guides/testing).

## Giữ triển khai gọn

- Bắt đầu với tính năng và template trong phạm vi V2A. Giữ state ở component hoặc feature sở hữu nó. Chỉ tách abstraction khi có nơi dùng chung cụ thể hoặc cần một ranh giới rõ ràng như adapter PMS. Tránh lớp trung gian chỉ chuyển tiếp dữ liệu và cấu hình dự phòng cho nhu cầu chưa có.
- Ưu tiên tính năng sẵn có của Next.js, React, Web API và dependency repo đã dùng. Trước khi thêm dependency, ghi rõ nhu cầu, lý do giải pháp hiện có chưa đáp ứng và ảnh hưởng tới bundle client hoặc bảo trì. Dùng chung một công cụ cho cùng mục đích khi đáp ứng được yêu cầu.
