# Quy trình phát triển Hotel Website

**Trạng thái:** áp dụng cho repo `hotel-website` theo chỉ thị của chủ repo (Product Owner, PO). Quy trình lấy mô hình từ `hotel-system` (`docs/project-management/workflow.md`, mục 8 và 10) và chỉnh cho Next.js. `hotel-system` chỉ để đối chiếu, không sửa từ repo này.

Tài liệu này là nguồn của quy trình Git và vòng đời thay đổi. `CLAUDE.md` và `AGENTS.md` chỉ tóm tắt và dẫn tới đây.

## 1. Thứ bậc quy tắc

1. Chỉ thị rõ ràng của PO cho thao tác đó.
2. Quy trình trong tài liệu này, cho tới khi PO thay thế bằng quyết định mới.
3. Đặc tả, [kiến trúc](../coding-rules/client-architecture.md), [quy tắc frontend](../coding-rules/frontend.md) và [bảo mật](../security/README.md) vẫn bắt buộc.
4. Khi các chỉ dẫn mâu thuẫn, dừng và hỏi PO. Không âm thầm bỏ qua một cổng phê duyệt.

Quy tắc của `hotel-system` là tham chiếu. Chỉ phần nào được ghi ở mục 8 mới áp dụng cho repo này.

## 2. Chính sách Git branch

Bắt buộc, không có ngoại lệ ngầm.

| Được làm | Không được làm nếu PO chưa chỉ thị riêng cho thao tác đó |
| --- | --- |
| Làm việc trên branch **`develop`** | Tạo feature branch, task branch hoặc release branch |
| Đọc: `git status`, `git diff`, `git log`, `git branch --show-current` | Checkout sang branch khác |
| Sửa file trong phạm vi được giao, để thay đổi ở trạng thái **chưa stage** | Merge hoặc rebase |
| `git fetch` khi cần so sánh với `origin` (không đổi working tree) | Tạo PR |
| | Commit hoặc push |
| | Force push (`--force`, `--force-with-lease`) trên mọi branch |
| | `git reset --hard`, `git clean`, `git checkout -- <path>`, `git restore` đè lên thay đổi, `git stash drop`, xóa branch |
| | Thao tác trực tiếp trên `main` |

**Kiểm tra trước khi ghi file.** Chạy `git branch --show-current` và `git status --short`.

- Nếu branch hiện tại **không phải `develop`**: dừng mọi thao tác ghi, báo branch hiện tại và trạng thái, chờ chỉ thị. Không tự chuyển branch, không tự stash để chuyển.
- Nếu có thay đổi chưa commit mà không phải của lượt làm việc này (của PO hoặc phiên khác): giữ nguyên, không đưa vào bàn giao như thay đổi của mình, không format hoặc ghi đè. Hiện repo có nhiều file chưa track (`.agents/`, `.claude/`, `AGENTS.md`, `CLAUDE.md`, `docs/`) là trạng thái bình thường, không phải lý do để dừng.
- Khi hai phiên cùng sửa một file, đọc lại file trước khi sửa và chỉ sửa phần của mình.

## 3. Vòng đời một thay đổi

| Bước | Người thực hiện | Kết quả |
| --- | --- | --- |
| 1. GitHub Issue và Acceptance Criteria | PO (có thể qua công cụ lập kế hoạch) | Phạm vi và tiêu chí đo được, truy về [đặc tả](../specs/client-specs.md). Thiếu tiêu chí thì báo khoảng trống, không tự bịa hành vi |
| 2. Technical Design / Specification | Claude hoặc PO soạn, PO duyệt | Thiết kế, hợp đồng PMS (mục 6), phần còn chờ quyết định ghi rõ |
| 3. Product Owner approval | PO | Bắt buộc khi có quyết định nghiệp vụ. Mục "Cần chốt" giữ ở trạng thái chờ |
| 4. Mockup approval | PO | Bắt buộc trước khi implement UI. Mockup lưu theo [tư liệu thiết kế](../designs/README.md) |
| 5. Implement | Claude Code, trên `develop` | Thay đổi chưa stage, chỉ trong phạm vi đã duyệt |
| 6. Test và báo kết quả | Claude Code | Lệnh đã chạy, kết quả thật, phần chưa xác minh và lý do (mục 5) |
| 7. Review độc lập | Codex, chỉ đọc | Verdict và findings (mục 7) |
| 8. Sửa findings | Claude Code | Sửa trên `develop`, chạy lại kiểm tra bị ảnh hưởng, vẫn chưa commit |
| 9. Re-review | Codex, khi cần | Áp dụng khi có BLOCKER, MAJOR hoặc khi PO yêu cầu |
| 10. Commit, push, PR, merge | **PO quyết định** | Mỗi thao tác cần chỉ thị riêng (mục 4) |

Bước 3 và 4 là cổng: chưa qua thì chưa sang bước 5 cho phần bị ảnh hưởng. Phần không bị chặn vẫn triển khai được, và phần bị chặn được tách rõ.

Thay đổi chỉ sửa tài liệu cũng theo bước 5 đến 10, nhưng bỏ qua bước 4 và rút gọn bước 6.

## 4. Cổng phê duyệt Git

Commit, push, tạo PR và merge là các quyết định **riêng biệt** của PO. Quyết định này không kéo theo quyết định kia.

- Không có chỉ thị thường trực. Mỗi lần commit, push hoặc tạo PR cần chỉ thị cho lần đó.
- Không tính là phê duyệt: test đạt, self-review sạch, Codex PASS, im lặng, "trông ổn", biểu tượng đồng ý. Nếu chỉ thị mơ hồ, hỏi lại và chưa làm gì.
- Phạm vi phê duyệt là diff đã bàn giao ở lần cuối. Nếu working tree khác diff đã review, dừng và hỏi lại.
- Chỉ thị "commit" không bao gồm push. Chỉ thị "push" không bao gồm PR.

Khi PO đã chỉ thị commit:

1. Xác nhận branch là `develop` và diff đúng với bản đã duyệt. Không đưa thay đổi không liên quan, file `.env` hay cấu hình IDE cục bộ vào commit.
2. Stage theo đường dẫn tường minh, không dùng `git add -A` hoặc `git add .`. Xem `git diff --cached` trước khi commit.
3. Một commit tập trung cho mỗi đầu việc. Không gộp việc không liên quan. Không amend hay viết lại lịch sử đã push. Sửa sau review là commit mới.
4. Thông điệp commit tham chiếu Issue dạng `Refs #N`, không dùng từ khóa đóng Issue (`Closes`, `Fixes`, `Resolves`).
5. Chỉ push khi PO cho phép riêng. Trước khi push, `git fetch` và xác nhận `origin/develop` không có commit mà local chưa có. Nếu đã lệch, dừng và báo. Không force push, không push `main`.
6. Không tạo PR, không merge, không đóng Issue.

## 5. Kiểm tra và báo cáo

- Lấy lệnh lint, typecheck, build và test từ cấu hình thực tế của repo (`package.json` khi đã có). Hiện repo chưa có mã nguồn hay script, nên chưa có lệnh nào để ghi.
- Báo cáo: lệnh, kết quả thật, phạm vi đã chạy, phần chưa chạy và lý do. Không mô tả kiểm tra chưa chạy là đã đạt. Không tắt hay bỏ qua test để build qua.
- Thay đổi UI: ghi màn hình, trạng thái, viewport và browser đã xem ([quy tắc frontend](../coding-rules/frontend.md#kiểm-tra-khi-thay-đổi-mã)).
- Thao tác ghi booking chỉ kiểm tra bằng fixture giả lập hoặc PMS test.

## 6. Đối chiếu hợp đồng với hotel-system

Áp dụng cho mọi thay đổi chạm API, adapter PMS, schema, DTO, trạng thái booking hoặc dữ liệu công bố.

1. Đọc phần liên quan của `hotel-system` (read-only) và ghi revision đã đối chiếu.
2. Ghi operation, schema request/response, lỗi, xác thực và cách chống trùng đã đối chiếu. Phân biệt đã xác nhận với PMS test và chỉ kiểm tra bằng fixture ([tích hợp PMS API](api-integration.md)).
3. Nếu thiếu hợp đồng, ghi operation, schema hoặc hành vi cần xác nhận và tách phần bị chặn. Không tự đặt endpoint, schema hay trạng thái reservation.
4. Thay đổi hợp đồng cần xác định phiên bản hai repo phải chạy cùng nhau khi triển khai hoặc quay lui.
5. Mọi sửa đổi bên PMS thuộc workflow của `hotel-system`, không làm từ phiên website.

## 7. Bàn giao và review độc lập

**Gói bàn giao cho Codex** (bổ sung vào mục "Bàn giao cho Codex" của [CLAUDE.md](../../CLAUDE.md)):

1. Tên Issue hoặc đầu việc, phạm vi và tiêu chí nghiệm thu.
2. Branch (`develop`) và trạng thái diff local, gồm file đã stage, chưa stage và file mới.
3. File liên quan, thay đổi không thuộc phạm vi (nếu có).
4. Tài liệu và mục đặc tả đã dựa vào.
5. Lệnh kiểm tra và kết quả thật, phần chưa xác minh.
6. Hợp đồng PMS đã đối chiếu (mục 6) và mục còn chờ.
7. Với UI: màn hình, trạng thái, viewport, browser đã xem.
8. Giới hạn đã biết và cách kiểm tra local.

Kết thúc bàn giao bằng câu sau, nguyên văn:

> Thay đổi sẵn sàng để review local. Chưa có commit, push hay PR nào được tạo. Đang chờ quyết định của Product Owner.

**Codex** chỉ đọc mã và chạy kiểm tra an toàn. Không sửa file, không đổi branch, không commit, push, merge, rebase hay đụng tới PR trừ khi PO cho phép trong cùng yêu cầu. Verdict gồm đúng một trong **PASS**, **PASS WITH MINOR ISSUES**, **FAIL**, kèm findings theo BLOCKER, MAJOR, MINOR (file và dòng, bằng chứng, tác động, hướng sửa).

| Kết quả | Việc tiếp theo |
| --- | --- |
| PASS, hoặc PASS WITH MINOR ISSUES | Chờ quyết định của PO. Báo mục MINOR, chỉ sửa khi PO yêu cầu |
| FAIL, hoặc có BLOCKER hay MAJOR | Claude sửa trên `develop` mà không commit, chạy lại kiểm tra bị ảnh hưởng, bàn giao lại và xin re-review |

Codex PASS không phải phê duyệt của PO.

## 8. So sánh với hotel-system

**Kế thừa**

- Phát triển trên một branch duy nhất, không branch hay PR cho từng đầu việc.
- Không commit, push, PR hay merge nếu PO chưa chỉ thị riêng. Các quyết định này tách biệt.
- Để thay đổi chưa stage trong lúc chờ review để PO xem được `git diff`.
- Sửa sau review theo vòng "sửa, chạy lại, bàn giao lại", chưa commit trước khi PO duyệt.
- Không destructive Git, không force push, không ghi đè công việc của phiên khác.
- Codex là reviewer độc lập chỉ đọc, verdict PASS, PASS WITH MINOR ISSUES, FAIL và mức BLOCKER, MAJOR, MINOR.
- Commit tập trung, `Refs #N`, không từ khóa đóng Issue, stage theo đường dẫn.
- Claude không tự coi test đạt hay Codex PASS là phê duyệt.

**Khác, dành riêng cho Next.js frontend**

| Chủ đề | hotel-system | hotel-website |
| --- | --- | --- |
| Branch phát triển | `develop-v2` | `develop` |
| Cổng thiết kế | Đặc tả nghiệp vụ `technical-spec` | Đặc tả client, kiến trúc client, **mockup đã duyệt trước khi làm UI** |
| Cổng chất lượng | Maven, Flyway, test tích hợp | Lint, typecheck, build, unit, component, E2E, responsive, accessibility theo [quy tắc frontend](../coding-rules/frontend.md) khi script đã có |
| Đối chiếu bên ngoài | Chính repo là nguồn sự thật | Phải đối chiếu hợp đồng với `hotel-system` (mục 6) |
| Bằng chứng UI | Ảnh màn hình theo responsive-design | Ảnh và luồng theo viewport 320, 390, 768, 1440 trên Chromium và WebKit |
| Dữ liệu và bảo mật | Backend sở hữu | Server-only cho bí mật PMS, `no-store` cho giao dịch, không PII trong log hay URL |
| Trạng thái Issue | Tự động hóa Project #1, đồng bộ `project-status.json`, comment bằng `gh` | **Không áp dụng** trong quy trình này |

Phần quản lý GitHub Project và trạng thái Issue của `hotel-system` (mục 1 đến 7, 9) không chuyển sang repo này. Claude không tạo hay sửa Issue, không đổi Project nếu PO chưa chỉ thị.

## 9. Khôi phục ngữ cảnh ở phiên sau

Phiên mới không nhớ phiên cũ. Khôi phục từ repo, không từ cuộc trò chuyện trước.

| Phiên | Đọc theo thứ tự |
| --- | --- |
| Claude Code | `CLAUDE.md` (kéo theo `AGENTS.md`), tài liệu này, Issue và đặc tả liên quan, skill phù hợp (`/implement-feature`, `/build-template`, `/review-frontend`) |
| Codex | `AGENTS.md`, tài liệu này, Issue và tiêu chí nghiệm thu, diff và bằng chứng kiểm tra, skill `review-pr` hoặc `verify-feature` |

Trước khi làm việc, kiểm tra `git branch --show-current` và `git status --short` (mục 2), và kiểm tra lại tài liệu nếu phiên khác đang cập nhật.
