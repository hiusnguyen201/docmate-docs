# DocMate — Acceptance Criteria (Given–When–Then)

| | |
|---|---|
| **Phiên bản** | 1.3 — 2026-07-06 |
| **Tham chiếu** | `02-product-backlog.md` |

> Mỗi story liệt kê các scenario chính (happy path + biên/lỗi quan trọng). Test step đầy đủ nằm ở tài liệu Test Cases (Nhóm 4).

---

## E0 — Platform & Foundation

### US-0.1 — Môi trường dev Docker Compose
- **Given** máy dev đã cài Docker, **When** chạy `docker compose up`, **Then** Golang, Spring Boot, FastAPI, PostgreSQL, Qdrant, MinIO đều khởi động và healthcheck cả 6 container OK.
- **Given** stack đang chạy, **When** các service gọi nhau qua mạng nội bộ Docker (Golang→Java, Java→Python, Java→PostgreSQL/Qdrant, Golang→MinIO), **Then** kết nối hoạt động không cần cấu hình thêm; chỉ Nginx expose ra ngoài.

### US-0.2 — Extraction Service `/extract`
- **Given** một `s3Key` trỏ tới file PDF hợp lệ (hoặc một URL), **When** gọi `POST /extract`, **Then** nhận HTTP 200 với Markdown + metadata (định dạng, kích thước/số trang, thời gian xử lý).
- **Given** file không hỗ trợ hoặc hỏng, **When** gọi `/extract`, **Then** nhận 4xx với mã lỗi phân loại (`UNSUPPORTED_FORMAT` / `CORRUPTED_FILE`), không bao giờ 500 không kiểm soát.

### US-0.3 — Schema PostgreSQL + Qdrant collection
- **Given** schema PostgreSQL với FK cascade (Session → Source → Chunk/Message) và Qdrant collection có payload index, **When** insert chunk (Postgres) + point (Qdrant) rồi search với filter `userId` + `sessionChatId` cho phiên X của user A, **Then** kết quả đúng thứ tự similarity và **chỉ** chứa point của đúng phiên X, đúng user A.
- **Given** một point tồn tại trên Qdrant nhưng chunk tương ứng đã bị xóa khỏi PostgreSQL (vector mồ côi), **When** search và join lại PostgreSQL theo `chunkId`, **Then** point đó bị loại khỏi kết quả cuối.
- **Given** xóa một Source trong PostgreSQL, **When** cascade chạy, **Then** chunk/message liên quan biến mất khỏi PostgreSQL và job xóa point theo payload `sourceId` trên Qdrant được thực thi.

### US-0.4 — CI/CD GitHub Actions
- **Given** PR merge vào `main`, **When** workflow chạy, **Then** build + test → build 3 Docker image → deploy VPS → healthcheck sau deploy OK; bước nào fail thì dừng, bản cũ trên VPS vẫn chạy.

### US-0.5 — [SPIKE] MarkItDown tiếng Việt
- **Given** bộ file mẫu tiếng Việt (PDF văn bản, PDF scan, DOCX, XLSX, PPTX, ảnh, audio), **When** chạy qua MarkItDown, **Then** có báo cáo tỉ lệ đọc đúng theo loại + kết luận Go / Go-with-limits / No-Go, ghi thành ADR.

### US-0.6 — Bộ test vàng + eval harness
- **Given** tập tài liệu mẫu cố định (Việt + Anh) và ≥ 40 câu hỏi thuộc **4 nhóm** (có đáp án / không có đáp án / mơ hồ / có các đoạn mâu thuẫn — cả cùng nguồn lẫn khác nguồn), **When** chạy script eval, **Then** xuất báo cáo: % đúng + nguồn đúng, % từ chối đúng, % nêu-đủ-các-bên với nhóm mâu thuẫn; chạy so sánh được giữa 2 cấu hình (ví dụ có/không contextual embedding).
- **Given** case hỏi tiếng Việt trên nguồn tiếng Anh (và ngược lại), **When** chạy eval, **Then** báo cáo đo riêng độ chính xác truy xuất hai chiều để phát hiện lệch điểm similarity theo ngôn ngữ.

### US-0.7 — Upload Service (Golang)
- **Given** request multipart hợp lệ kèm JWT + `sessionChatId`, **When** Golang xử lý, **Then** file được stream lên S3 (không buffer toàn bộ vào RAM), callback Java `{s3Key, sessionChatId, metadata}` với xác thực service-to-service thành công, client nhận response kèm `sourceId`.
- **Given** JWT sai/hết hạn, **When** gọi upload, **Then** 401 — không có gì lên S3.
- **Given** file sai định dạng hoặc > 50MB, **When** upload, **Then** Golang từ chối ngay với mã lỗi rõ — không có gì lên S3.
- **Given** callback sang Java thất bại, **When** retry hết N lần, **Then** file rác trên S3 được dọn (hoặc đánh dấu orphan để job dọn định kỳ) và client nhận thông báo lỗi.

---

## E1 — Auth & Account

### US-1.1 — Đăng ký email/mật khẩu + xác minh
- **Given** email chưa tồn tại, **When** đăng ký với mật khẩu ≥ 8 ký tự, **Then** tài khoản tạo ở trạng thái chưa xác minh, email xác minh được gửi, mật khẩu băm bcrypt/argon2.
- **Given** tài khoản chưa xác minh, **When** click link xác minh còn hạn, **Then** tài khoản kích hoạt và đăng nhập được.
- **Given** email đã tồn tại, **When** đăng ký lại, **Then** báo lỗi rõ, không tạo trùng.

### US-1.2 — JWT dùng chung Java/Golang
- **Given** đăng nhập đúng, **When** nhận JWT, **Then** token dùng được cho cả API Java (chat/phiên) lẫn API Golang (upload) — cùng cơ chế ký/verify.
- **Given** sai mật khẩu, **When** đăng nhập, **Then** lỗi chung chung (không lộ email tồn tại hay không).
- **Given** JWT hết hạn, **When** gọi bất kỳ API nào (Java hoặc Golang), **Then** 401 và frontend điều hướng về đăng nhập.

### US-1.3 — OAuth Google
- **Given** người dùng chưa có tài khoản, **When** đăng nhập Google thành công, **Then** tài khoản tạo mới (email từ Google, đã xác minh) và vào thẳng app.
- **Given** email Google trùng tài khoản email/pass sẵn có, **When** đăng nhập Google, **Then** account linking tự động + thông báo — không tạo tài khoản thứ hai.

### US-1.4 — Quên mật khẩu
- **Given** email tồn tại, **When** yêu cầu reset, **Then** email chứa link reset (≤ 30 phút, 1 lần); **Given** email không tồn tại, **Then** UI hiển thị thông điệp giống hệt.
- **Given** link hợp lệ, **When** đặt mật khẩu mới, **Then** đăng nhập được bằng mật khẩu mới, link cũ vô hiệu.

### UC-A5 — Xem hồ sơ người dùng (Get Profile)
- **Given** người dùng đã đăng nhập, **When** mở trang hồ sơ/tài khoản, **Then** thấy: Email, Tên, Avatar, phương thức auth (Google / email+password), ngày tạo tài khoản.
- **Given** người dùng đã đăng nhập, **When** gọi API `GET /api/user/profile` với JWT hợp lệ, **Then** nhận HTTP 200 kèm dữ liệu JSON: `{email, name, avatar, googleId, createdAt}` (googleId = null nếu auth bằng email/password).
- **Given** JWT không hợp lệ hoặc hết hạn, **When** gọi `/api/user/profile`, **Then** 401 không lộ dữ liệu cá nhân.

---

## E2 — Ingestion (session-scoped)

### US-2.1 — Thêm file vào phiên
- **Given** đang trong một phiên chat, **When** kéo-thả/chọn file hợp lệ ở panel phải, **Then** file đi qua Golang → S3, Source xuất hiện ngay trong panel với trạng thái `PROCESSING`, hội thoại đang gõ không bị gián đoạn.
- **Given** file sai định dạng hoặc > 50MB, **When** chọn file, **Then** bị chặn ngay trên UI (validate cả extension lẫn MIME) kèm thông báo định dạng hỗ trợ.

### US-2.2 — Pipeline contextual embedding
- **Given** Java nhận callback `{s3Key, sessionChatId}`, **When** pipeline chạy, **Then** tuần tự: tải file từ S3 → Extraction → Markdown → chunk theo heading (~500–1000 token, overlap 10–15%) → **với mỗi chunk, Claude sinh đoạn ngữ cảnh ~50–100 token (dùng prompt caching cho toàn văn)** → ghép context + chunk → Voyage embed → lưu chunk (text gốc + `contextualizedText` + `embeddingModel`/`dimension`) vào PostgreSQL và upsert vector vào Qdrant (payload `userId`, `sessionChatId`, `sourceId`, `chunkId`) → Source = `READY`.
- **Given** nguồn văn bản ≤ 20MB, **When** upload xong, **Then** `READY` trong ≤ 2 phút ở điều kiện thường; nguồn nhiều chunk hiển thị tiến trình (% chunk đã xử lý) thay vì treo.
- **Given** một bước lỗi tạm thời (timeout/rate-limit Claude/Voyage), **When** retry với backoff hết N lần, **Then** Source = `FAILED` kèm lý do phân loại; token đã tiêu cho contextual được log.
- **Given** Source thuộc phiên X, **When** kiểm tra chunk trong DB, **Then** mọi chunk gắn đúng `sessionChatId` = X.

### US-2.3 — Trạng thái nguồn realtime
- **Given** nguồn đang xử lý, **When** nhìn panel, **Then** trạng thái `PROCESSING` → `READY` cập nhật không cần F5 (SSE/WebSocket/polling).
- **Given** nguồn lỗi, **When** xem chi tiết, **Then** lý do bằng ngôn ngữ dễ hiểu ("File PDF có mật khẩu", "Link cần đăng nhập"…), không phải stack trace.
- **Given** nguồn vừa `READY`, **When** hỏi câu tiếp theo trong phiên, **Then** nội dung nguồn mới đã nằm trong phạm vi truy xuất.

### US-2.4 — URL ingestion + chống SSRF
- **Given** URL http(s) công khai, **When** dán vào panel, **Then** nội dung fetch + snapshot, đi qua pipeline như file; Source ghi rõ nguồn URL kèm link gốc.
- **Given** URL trỏ private IP / localhost / metadata endpoint (kể cả qua DNS rebinding hoặc redirect), **When** kiểm tra sau resolve và sau mỗi redirect, **Then** từ chối "URL không được hỗ trợ", ghi log bảo mật.
- **Given** content-type ngoài whitelist hoặc vượt giới hạn kích thước, **When** fetch, **Then** dừng, báo lỗi cụ thể.
- **Given** link chết / cần đăng nhập, **When** fetch thất bại, **Then** `FAILED` với lý do phân biệt được hai trường hợp.

### US-2.5 — YouTube URL
- **Given** video công khai có transcript, **When** dán URL, **Then** transcript thành Source (metadata: tiêu đề, kênh, link).
- **Given** video không transcript / bị chặn, **When** nạp, **Then** lỗi nêu đúng nguyên nhân.

### US-2.6 — ZIP
- **Given** ZIP chứa hỗn hợp file hỗ trợ và không, **When** upload, **Then** mỗi file hợp lệ thành một Source riêng của phiên; danh sách file bị bỏ qua hiển thị sau xử lý.

### US-2.7 — Audio
- **Given** file audio định dạng hỗ trợ, **When** upload, **Then** transcript thành nội dung Source; giới hạn tiếng Việt (nếu có, theo spike US-0.5) ghi rõ trong UI.

---

## E3 — Sources Panel

### US-3.1 — Danh sách nguồn + xóa cascade
- **Given** phiên có nhiều nguồn, **When** mở panel, **Then** thấy danh sách nguồn của **riêng phiên đó** (tên, loại, trạng thái, thời điểm).
- **Given** một nguồn bất kỳ, **When** xóa và xác nhận, **Then** Source + file S3 + Markdown + chunk (PostgreSQL, qua cascade) + vector (Qdrant, xóa theo payload `sourceId`) bị xóa (trong job, tối đa ≤ 24h).
- **Given** nguồn vừa xóa từng là căn cứ trả lời, **When** hỏi lại đúng câu cũ, **Then** không còn truy xuất được nội dung đó (trả "không tìm thấy" nếu không còn nguồn khác) — test bắt buộc.

### US-3.2 — Preview Markdown
- **Given** nguồn `READY`, **When** mở preview, **Then** Markdown đã trích xuất được render; nguồn `PROCESSING`/`FAILED` hiển thị trạng thái thay vì nội dung.

### US-3.3 — Bật/tắt nguồn
- **Given** phiên có nguồn A (bật) và B (tắt), **When** hỏi nội dung chỉ có trong B, **Then** trả lời "không tìm thấy"; **When** bật lại B và hỏi lại, **Then** trả lời được kèm trích dẫn từ B — filter có hiệu lực ngay câu hỏi kế tiếp.

### US-3.4 — Tổng quan phiên
- **Given** phiên có nguồn, **When** xem khu tổng quan, **Then** thấy đúng: số nguồn, tổng dung lượng, tổng chunk đã index, ngôn ngữ phát hiện; số liệu cập nhật khi thêm/xóa nguồn.

### US-3.5 — Token usage realtime
- **Given** một câu hỏi vừa được trả lời, **When** response hoàn tất, **Then** panel hiển thị ngay token in/out (embedding + LLM) và thời gian phản hồi của request đó, cộng dồn vào tổng token phiên.
- **Given** mở lại phiên cũ, **When** xem panel, **Then** tổng token phiên đúng bằng tổng các request lịch sử (số liệu lưu bền vững theo Message).

---

## E4 — RAG Chat

### US-4.1 — Hỏi đáp grounded + trích dẫn
- **Given** phiên có nguồn `READY` chứa đáp án, **When** hỏi, **Then** câu trả lời chỉ dựa trên chunk truy xuất từ nguồn đang bật của đúng phiên, kèm trích dẫn [nguồn + vị trí]; click trích dẫn → panel nhảy tới đoạn gốc.
- **Given** câu trả lời nhiều luận điểm, **When** đối chiếu, **Then** mỗi luận điểm bám ≥ 1 chunk trong trích dẫn; chunk không dùng không xuất hiện.
- **Given** hai nguồn khác nhau trong phiên chứa thông tin trái ngược về cùng câu hỏi (ví dụ hai bản hợp đồng ghi hai mức giá), **When** hỏi, **Then** câu trả lời nêu rõ sự không thống nhất kèm trích dẫn từng nguồn — không im lặng chọn một bên, không trộn thành một con số.
- **Given** hai đoạn **trong cùng một nguồn** mâu thuẫn nhau (ví dụ điều khoản chính và phụ lục điều chỉnh), **When** hỏi, **Then** câu trả lời nêu cả hai kèm vị trí từng đoạn; nếu `contextualizedText` cho biết ngữ cảnh (đoạn nào thuộc phụ lục/phiên bản nào) thì trình bày kèm để người dùng tự phán đoán.
- **Given** user B, **When** B hỏi nội dung chỉ có trong phiên của user A, **Then** "không tìm thấy" — không bao giờ lộ dữ liệu chéo user; tương tự giữa 2 phiên khác nhau của cùng user.

### US-4.2 — Từ chối khi thiếu dữ liệu
- **Given** không chunk nào vượt ngưỡng similarity (Gate 1), **When** hỏi, **Then** đúng thông điệp chuẩn *"Tôi không tìm thấy thông tin trong tài liệu của bạn."* + gợi ý (diễn đạt lại / thêm nguồn), không kèm suy diễn.
- **Given** retrieval có kết quả nhưng LLM thấy ngữ cảnh không đủ (Gate 2), **When** sinh trả lời, **Then** cùng thông điệp từ chối chuẩn.
- **Given** câu hỏi kiến thức chung / small talk, **When** hỏi, **Then** bot lịch sự giải thích chỉ trả lời từ nguồn của phiên.
- **Given** phiên chưa có nguồn nào `READY`, **When** hỏi, **Then** bot hướng dẫn thêm nguồn ở panel phải thay vì cố trả lời.

### US-4.3 — Hội thoại nhiều lượt
- **Given** phiên đã hỏi về "chi phí dự án X", **When** hỏi tiếp "còn tiến độ thì sao?", **Then** hệ thống hiểu là tiến độ dự án X và truy xuất đúng ngữ cảnh.

### US-4.4 — Quản lý phiên (xóa phiên = xóa nguồn)
- **Given** user có nhiều phiên, **When** tạo/đổi tên phiên, **Then** phản ánh ngay; mở lại phiên cũ thấy đủ lịch sử tin nhắn + trích dẫn + token usage.
- **Given** phiên có nguồn, **When** bấm xóa phiên, **Then** hộp thoại cảnh báo nêu rõ *"Xóa phiên sẽ xóa vĩnh viễn N nguồn và toàn bộ hội thoại"*; xác nhận → cascade xóa Source/Chunk/Message (PostgreSQL) + vector (Qdrant, theo payload `sessionChatId`) + file S3 của phiên.

### US-4.5 — Streaming
- **Given** câu hỏi hợp lệ, **When** trả lời, **Then** chữ hiện dần (first-token ≤ 3s mục tiêu); nếu là từ chối (Gate 2) thì thông điệp từ chối hiển thị trọn vẹn, không nửa vời.

### US-4.6 — Đánh giá 👍/👎
- **Given** một câu trả lời, **When** bấm 👍/👎, **Then** lưu vào Message (đổi được một lần) và tính vào số liệu chất lượng.

### US-4.7 — Song ngữ Việt–Anh
- **Given** nguồn tiếng Anh, **When** hỏi tiếng Việt, **Then** truy xuất đúng chunk và trả lời tiếng Việt (chiều ngược lại tương tự); cả hai chiều nằm trong bộ test vàng.
