# DocMate — Definition of Ready & Definition of Done

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Tham chiếu** | `02-product-backlog.md` · `03-acceptance-criteria.md` |
| **Bối cảnh áp dụng** | Solo dev, Scrum rút gọn, sprint 2 tuần |

> Vì team là 1 người, các cơ chế vốn dựa vào "người khác" (code review, QA riêng) được thay bằng **self-review có checklist + CI bắt buộc**. DoR/DoD là hợp đồng với chính mình — không thỏa mãn thì không kéo story vào sprint / không đóng story.

---

## 1. Definition of Ready (DoR)

Một story **đủ điều kiện kéo vào Sprint Backlog** khi tick đủ:

### 1.1. Checklist DoR

- [ ] **Story rõ ràng:** đúng format "As a…, I want…, so that…" và tự đứng được — đọc story hiểu ngay giá trị, không cần hỏi lại.
- [ ] **Có AC Given-When-Then** trong `03-acceptance-criteria.md`, phủ tối thiểu: happy path + các ca lỗi/biên quan trọng.
- [ ] **Đã ước lượng SP** (Fibonacci 1–8). Story > 8 SP phải tách trước khi vào sprint.
- [ ] **Phụ thuộc đã xác định và không chặn:** story phụ thuộc (kỹ thuật hoặc dữ liệu) đã Done, hoặc phần phụ thuộc nằm cùng sprint và được xếp trước.
- [ ] **Quyết định kỹ thuật liên quan đã chốt** trong PRD/ADR — không kéo story vào sprint khi kiến trúc nền của nó còn đang tranh luận.
- [ ] **Dữ liệu/tài nguyên test sẵn sàng:** nếu AC cần dữ liệu mẫu (file tiếng Việt, tài khoản test, API key) thì đã chuẩn bị xong hoặc có task chuẩn bị nằm trong sprint, xếp trước story.
- [ ] **Điều kiện Done đo được:** biết trước sẽ demo/kiểm chứng story bằng cách nào (chạy lệnh gì, gọi endpoint nào, nhìn thấy gì).

### 1.2. DoR riêng cho SPIKE

Spike (như US-0.5) không tạo tính năng nên áp bộ rút gọn:

- [ ] **Câu hỏi cần trả lời** được phát biểu rõ (1–2 câu).
- [ ] **Timebox** cứng (SP đã ước lượng = trần, không gia hạn).
- [ ] **Đầu ra định trước:** báo cáo + kết luận Go / Go-with-limits / No-Go, ghi thành ADR.

---

## 2. Definition of Done (DoD)

Một story **được đóng** khi tick đủ. DoD chia 2 tầng: **tầng chung** (mọi story) + **tầng bổ sung theo loại** story.

### 2.1. DoD chung (mọi story)

- [ ] **Toàn bộ AC pass:** từng scenario Given-When-Then trong `03-acceptance-criteria.md` được kiểm chứng thực tế (chạy tay hoặc test tự động), không có AC "để sau".
- [ ] **Test tự động cho logic lõi:** unit/integration test cho phần nghiệp vụ chính của story; test chạy trong CI.
- [ ] **CI green:** GitHub Actions build + test pass trên nhánh trước khi merge `main`. CI đỏ = story chưa Done, không ngoại lệ. *(Điều kiện này áp dụng từ khi US-0.4 Done — trước đó thay bằng: toàn bộ test pass khi chạy bằng lệnh local.)*
- [ ] **Self-review theo checklist (§2.3)** thực hiện *sau khi nghỉ tối thiểu vài giờ* kể từ lúc viết xong — đọc lại diff bằng con mắt reviewer, không phải tác giả.
- [ ] **Không hardcode secret:** API key/mật khẩu/token nằm trong env/secret manager; `git log` sạch secret.
- [ ] **Chạy được từ sạch:** `docker compose up` từ máy trắng (hoặc container sạch) dựng lại được stack chứa thay đổi mới — tránh "works on my machine".
- [ ] **Tài liệu đồng bộ:** nếu story làm thay đổi quyết định kỹ thuật/hành vi đã ghi trong PRD, AC, hay ADR → cập nhật tài liệu **trong cùng commit chuỗi**, message `docs(scope): …`, push ngay theo quy ước repo.
- [ ] **Không TODO chặn:** không còn TODO/FIXME thuộc phạm vi story; việc ngoài phạm vi → ghi thành item backlog, không để trong code.

### 2.2. DoD bổ sung theo loại story

| Loại story | Điều kiện thêm |
|---|---|
| **Chạm dữ liệu người dùng** (Source/Chunk/Message/Session) | Test cô lập dữ liệu pass: user A không đọc được dữ liệu user B; phiên X không "nhìn" thấy nguồn phiên Y (filter `userId` + `sessionChatId` ở tầng thấp nhất) |
| **Có xóa dữ liệu** (Source/Session/Account) | Test cascade pass trên **cả 3 nơi**: PostgreSQL (`ON DELETE CASCADE`), Qdrant (delete theo payload), S3. Test bắt buộc "xóa rồi hỏi lại" → không còn truy xuất được |
| **Ghi/đọc vector** (dual-write Postgres ↔ Qdrant) | Search luôn join lại Postgres theo `chunkId`; có test chứng minh vector mồ côi bị loại khỏi kết quả |
| **API public** (qua Nginx) | Có xác thực JWT + trả mã lỗi phân loại (không 500 trần); endpoint mới ghi vào tài liệu API nội bộ |
| **Gọi AI** (Claude/Voyage) | Token usage được log theo request; lỗi tạm thời có retry + backoff; quá N lần → trạng thái FAILED kèm lý do phân loại |
| **Có UI** | Trạng thái loading/lỗi hiển thị dễ hiểu (không stack trace); kiểm tra trên viewport desktop tối thiểu |
| **SPIKE** | Báo cáo kết quả + kết luận ghi thành ADR trong repo docs; **không cần** code production-quality — code spike vứt được |

### 2.3. Checklist self-review (thay code review)

Đọc lại diff và tự hỏi:

- [ ] Tên biến/hàm đọc hiểu được không cần giải thích miệng?
- [ ] Ca lỗi nào chưa xử lý? (input rỗng, timeout, service kia chết, file hỏng)
- [ ] Có logic trùng lặp đáng gộp không? Có gì copy-paste từ chỗ khác mà quên sửa?
- [ ] Query DB nào thiếu filter `userId`/`sessionChatId`? (rà từng query mới)
- [ ] Log đủ để 3 tháng sau debug được không? Log có lộ dữ liệu nhạy cảm không?
- [ ] Nếu là người khác viết đoạn này, mình sẽ comment gì? → tự sửa luôn.

---

## 3. Quy ước vận hành DoR/DoD

| Tình huống | Xử lý |
|---|---|
| Story đang làm mới phát hiện thiếu DoR (phụ thuộc chặn, AC mơ hồ) | Dừng, đưa story về backlog, ghi lý do vào biên bản sprint; kéo story Ready khác vào nếu còn capacity |
| Hết sprint, story dở dang | **Không đóng một phần.** Story quay về backlog nguyên trạng, ước lượng lại phần còn thiếu ở Planning kế tiếp |
| Muốn nới DoD "cho kịp" | Không nới per-story. Muốn đổi DoD → sửa tài liệu này ở Retro, áp dụng từ sprint sau |
| DoD thay đổi | Bump phiên bản tài liệu này + ghi lý do ở phần Changelog dưới |

## Changelog

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 2026-07-06 | Bản đầu, viết cho bối cảnh solo dev |
