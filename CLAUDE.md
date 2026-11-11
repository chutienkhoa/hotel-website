# Claude Code trong Hotel Website

@AGENTS.md

## Chọn quy trình

| Skill | Khi dùng |
| --- | --- |
| [`/implement-feature`](.claude/skills/implement-feature/SKILL.md) | Triển khai hành vi, tích hợp PMS hoặc thay đổi Booking Engine đã có tiêu chí nghiệm thu. |
| [`/build-template`](.claude/skills/build-template/SKILL.md) | Xây dựng bố cục, theme và giao diện Modern Tropical theo tư liệu thiết kế. |
| [`/review-frontend`](.claude/skills/review-frontend/SKILL.md) | Claude tự rà soát thay đổi frontend trước khi bàn giao. |

Với việc vừa đổi hành vi vừa đổi template, dùng `/implement-feature` cho nghiệp vụ và áp dụng phần hướng dẫn về giao diện trong `/build-template` cho phần trình bày.

Chỉ gọi agent chuyên biệt khi phần thay đổi cần góc nhìn đó. [frontend-architect](.claude/agents/frontend-architect.md) kiểm tra ranh giới module hoặc hợp đồng PMS. [ui-ux-reviewer](.claude/agents/ui-ux-reviewer.md) xem bằng chứng giao diện. [frontend-test-reviewer](.claude/agents/frontend-test-reviewer.md) kiểm tra độ bao phủ test. Claude chính chịu trách nhiệm sửa và chạy các kiểm tra.

## Quy trình Git (bắt buộc)

Nguồn đầy đủ: [docs/development/workflow.md](docs/development/workflow.md). Đọc trước khi sửa file.

- Chỉ phát triển trên branch `develop`. Trước khi ghi file, chạy `git branch --show-current` và `git status --short`. Nếu branch không phải `develop`, dừng mọi thao tác ghi và báo cáo.
- Không tạo feature, task hay release branch. Không tự checkout sang branch khác, không tự merge hoặc rebase.
- Không tự commit, push hoặc tạo PR khi chủ repo chưa chỉ thị riêng cho thao tác đó. Test đạt hoặc Codex PASS không phải phê duyệt.
- Không force push. Không reset hoặc xóa thay đổi chưa commit, kể cả của phiên khác.
- Để thay đổi chưa stage. Khi được yêu cầu commit, stage theo đường dẫn tường minh, một commit tập trung, tham chiếu Issue bằng `Refs #N`.
- Vòng đời: Issue và tiêu chí → thiết kế → PO duyệt khi có quyết định nghiệp vụ → mockup duyệt trước khi làm UI → Claude implement và test → Codex review → Claude sửa → Codex re-review khi cần → chủ repo quyết định commit, push, PR, merge.
- Thay đổi chạm API, adapter hoặc dữ liệu PMS phải đối chiếu hợp đồng với `hotel-system` (chỉ đọc) trước khi nối.

## Bàn giao cho Codex

Ghi phạm vi và tiêu chí nghiệm thu, base hoặc trạng thái diff local, file liên quan, kết quả kiểm tra và các mục còn mở. Với thay đổi UI, ghi màn hình, trạng thái, viewport và browser đã xem. Với tích hợp PMS, ghi hợp đồng đã đối chiếu và kiểm tra nào dùng fixture hoặc PMS test. Chủ repo chuyển diff cho lượt review độc lập của Codex.
