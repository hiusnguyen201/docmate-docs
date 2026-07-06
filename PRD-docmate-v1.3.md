# PRD — DocMate: Trợ lý Hỏi đáp Tài liệu Cá nhân (RAG Chatbot)

| | |
|---|---|
| **Tên sản phẩm** | **DocMate** |
| **Phiên bản** | 1.3 |
| **Ngày** | 2026-07-06 |
| **Trạng thái** | Draft để review |
| **Loại sản phẩm** | Web App / SaaS |
| **Người sở hữu tài liệu** | _(điền tên PO/PM)_ |

> Bản 1.3: đổi tầng lưu trữ sang **PostgreSQL (data) + Qdrant (vector)** — mô hình 2 DB, thay MongoDB (lý do sẽ ghi ADR-001); thêm **quy tắc xử lý mâu thuẫn giữa các đoạn truy xuất** (cùng hoặc khác nguồn) và mở rộng bộ test vàng tương ứng. Bản 1.2: bỏ mục "Ngoài phạm vi"; đăng nhập còn Google + email/mật khẩu; làm rõ prompt caching (prefix, không phải semantic). Bản 1.1: session-centric, Upload Service (Golang) + S3, contextual embeddings, panel nguồn realtime.

---

## 1. Tóm tắt điều hành (Executive Summary)

**DocMate là một web app SaaS cho phép người dùng trò chuyện với AI trên chính dữ liệu của mình: trong mỗi phiên chat, người dùng vừa hỏi đáp vừa thêm nguồn (file/URL) ngay tại chỗ — mỗi phiên là một "notebook" với bộ nguồn riêng.**

Ba nguyên tắc sản phẩm:

1. **Chỉ trả lời từ dữ liệu người dùng cung cấp.** Chatbot không dùng kiến thức chung của LLM để trả lời câu hỏi nghiệp vụ; mọi câu trả lời phải được truy xuất (retrieve) từ các nguồn đã nạp vào phiên.
2. **Không bịa.** Nếu không tìm thấy căn cứ đủ liên quan trong nguồn, chatbot trả lời rõ ràng: *"Tôi không tìm thấy thông tin trong tài liệu của bạn"* — tuyệt đối không suy diễn.
3. **Luôn có trích dẫn nguồn.** Mỗi câu trả lời chỉ rõ nội dung được lấy từ nguồn nào, đoạn nào.

**Đối tượng:** cá nhân có tập tài liệu riêng cần tra cứu nhanh — chuyên viên, freelancer, sinh viên, nhà nghiên cứu.

Mỗi người dùng chỉ nhìn thấy và hỏi đáp trên dữ liệu của chính mình.

---

## 2. Vấn đề & Đối tượng

### 2.1. Vấn đề
- Tài liệu cá nhân nằm rải rác ở nhiều định dạng (PDF, Word, Excel, slide, ảnh, trang web, YouTube…) — muốn tìm một chi tiết phải mở từng file đọc lại thủ công.
- Các chatbot AI phổ thông trả lời trơn tru nhưng **hay bịa** khi không có dữ liệu — người dùng không phân biệt được đâu là thông tin từ tài liệu của họ, đâu là AI tự nghĩ ra.
- Tìm kiếm từ khóa (Ctrl+F) không hiểu ngữ nghĩa: hỏi "quy trình bàn giao thế nào" không tìm ra đoạn viết "các bước chuyển giao".

### 2.2. Personas & Jobs-to-be-Done

| Persona | Job cần làm ("Khi… tôi muốn… để…") |
|---|---|
| **Chuyên viên văn phòng** | Khi có bộ tài liệu quy trình/hợp đồng/báo cáo, tôi muốn gom vào một phiên và hỏi bằng ngôn ngữ tự nhiên, để khỏi đọc lại từng file. |
| **Sinh viên / nhà nghiên cứu** | Khi có tập paper/giáo trình, tôi muốn hỏi đáp và đối chiếu nhanh trong một notebook, để học và trích dẫn chính xác. |
| **Freelancer / consultant** | Khi nhận brief + tài liệu từ khách, tôi muốn mỗi khách một phiên riêng, nạp tài liệu và tra cứu tức thì, để trả lời khách nhanh và đúng. |

---

## 3. Phạm vi sản phẩm (Scope)

### 3.1. Mô hình sản phẩm: phiên chat = notebook

Đơn vị làm việc trung tâm là **phiên chat (ChatSession)**. Mỗi phiên có **bộ nguồn riêng** — nguồn thuộc phiên, không có thư viện chung; **xóa phiên là xóa toàn bộ nguồn + dữ liệu phái sinh của phiên đó**.

**Bố cục giao diện:** như một chat AI quen thuộc (khung hội thoại giữa, danh sách phiên bên trái) cộng thêm **panel bên phải** của mỗi phiên:

| Khu vực panel phải | Nội dung |
|---|---|
| **Nguồn (Sources)** | Danh sách file/URL của phiên kèm trạng thái (PROCESSING/READY/FAILED); nút thêm file / dán URL ngay trong lúc chat; bật/tắt từng nguồn khỏi phạm vi truy xuất; xóa nguồn; xem preview Markdown đã trích xuất |
| **Tổng quan (Overview)** | Số nguồn, tổng dung lượng, tổng số chunk đã index, ngôn ngữ phát hiện |
| **Token & hiệu năng** | Token in/out của request gần nhất (hiển thị realtime ngay khi có), tổng token cả phiên, thời gian phản hồi |
| **Trích dẫn** | Click citation trong câu trả lời → panel nhảy tới đoạn gốc trong nguồn tương ứng |

### 3.2. Ba khối chức năng chính

| # | Khối | Mô tả |
|---|---|---|
| 1 | **Nạp nguồn vào phiên (Ingestion)** | Upload file (qua Upload Service Golang → S3) hoặc dán URL ngay trong phiên chat; pipeline xử lý bất đồng bộ |
| 2 | **Panel nguồn (Sources Panel)** | Quản lý nguồn của phiên, tổng quan, token usage realtime |
| 3 | **Chat hỏi đáp (RAG Chat)** | Trả lời grounded + trích dẫn; từ chối khi thiếu dữ liệu; đa phiên; song ngữ Việt–Anh; streaming |

### 3.3. Định dạng đầu vào hỗ trợ
Engine trích xuất là **MarkItDown** (Microsoft) — mọi định dạng được convert về **Markdown**:

- **Tài liệu văn phòng:** PDF, Word (.docx), Excel (.xlsx), PowerPoint (.pptx)
- **Ảnh:** metadata EXIF + OCR · **Audio:** metadata + speech transcription
- **Web:** HTML, URL trang web, **YouTube URL** (transcript)
- **Text:** CSV, JSON, XML, Markdown, plain text · **Khác:** ZIP (duyệt từng file), EPub

> Chất lượng OCR/transcription tiếng Việt phụ thuộc MarkItDown — kiểm chứng bằng spike trước khi cam kết (xem Rủi ro).

---

## 4. Yêu cầu Chức năng (Functional Requirements)

### 4.1. Xác thực & Tài khoản (Auth)

**US-0.1** — Là người dùng mới, tôi muốn đăng ký/đăng nhập bằng Google hoặc email/mật khẩu, để bắt đầu dùng nhanh.

**Tiêu chí chấp nhận:**
- Hỗ trợ 2 phương thức: **OAuth Google**, **email + mật khẩu**.
- Email/mật khẩu: xác minh email; băm bcrypt/argon2; luồng quên mật khẩu.
- Email OAuth trùng tài khoản email/pass hiện có → **account linking** tự động kèm thông báo (một hành vi, nhất quán).
- Phiên đăng nhập bằng JWT có thời hạn; đăng xuất được. JWT được **cả Core Backend (Java) lẫn Upload Service (Golang) xác thực** bằng cùng cơ chế khóa.
- Xóa tài khoản → toàn bộ phiên, nguồn, embedding, hội thoại bị xóa (§8).

### 4.2. Nạp nguồn vào phiên (Ingestion)

**US-1.1** — Là người dùng, tôi muốn thêm file vào phiên chat ngay trong lúc chat (không rời khỏi hội thoại), để bổ sung nguồn khi cần.
**US-1.2** — Là người dùng, tôi muốn dán URL (trang web, YouTube) vào panel nguồn, để nạp nội dung không cần file.
**US-1.3** — Là người dùng, tôi muốn thấy trạng thái từng nguồn cập nhật realtime trong panel, để biết khi nào hỏi được trên nguồn đó.

**Luồng upload file (kiến trúc đã chốt):**
1. Client gửi file → **Upload Service (Golang)** kèm JWT + `sessionChatId`.
2. Golang validate (JWT, định dạng, kích thước ≤ 50MB) → stream file lên **S3** → gọi API nội bộ sang **Core Backend (Java)**: `{fileKey, sessionChatId, metadata}` (xác thực service-to-service).
3. Java tạo Source (status=`PROCESSING`), đẩy job: tải file từ S3 → **Extraction Service (Python/MarkItDown)** → Markdown → chunk → **contextual embedding** → lưu PostgreSQL + upsert vector Qdrant → `READY`.

**Tiêu chí chấp nhận:**
- Thêm nguồn không làm gián đoạn hội thoại đang diễn ra; nguồn mới `READY` là dùng được ngay cho câu hỏi kế tiếp trong cùng phiên.
- Validate định dạng (extension + MIME) và kích thước **tại Golang, trước khi** đẩy S3; lỗi báo ngay trên UI.
- URL: fetch snapshot tại thời điểm nạp; **chống SSRF** (chặn private IP/metadata endpoint, kiểm sau DNS resolve và sau mỗi redirect; chỉ http/https; giới hạn kích thước + whitelist content-type; fetcher cô lập trong Extraction Service).
- Xử lý bất đồng bộ với trạng thái `PROCESSING` → `READY` / `FAILED` (kèm lý do dễ hiểu, phân loại theo nguyên nhân); retry với backoff cho lỗi tạm thời.
- Nguồn văn bản ≤ 20MB đạt `READY` ≤ 2 phút sau upload (mục tiêu — lưu ý contextual embedding thêm thời gian, xem NFR).
- ZIP: mỗi file hợp lệ bên trong thành một Source riêng; file bỏ qua được liệt kê.
- Mỗi Source lưu metadata: tên, loại nguồn (file/URL), định dạng, kích thước, `sessionChatId`, `userId`, thời điểm, trạng thái, `s3Key`.

### 4.3. Panel nguồn của phiên (Sources Panel)

**US-2.1** — Là người dùng, tôi muốn xem danh sách nguồn của phiên kèm trạng thái và tổng quan (số nguồn, dung lượng, số chunk), để nắm được "notebook" của mình có gì.
**US-2.2** — Là người dùng, tôi muốn bật/tắt từng nguồn khỏi phạm vi truy xuất, để câu trả lời tập trung đúng nguồn tôi muốn.
**US-2.3** — Là người dùng, tôi muốn xóa một nguồn khỏi phiên và mọi dữ liệu phái sinh bị xóa triệt để, để kiểm soát dữ liệu.
**US-2.4** — Là người dùng, tôi muốn xem preview Markdown đã trích xuất của từng nguồn, để kiểm tra hệ thống "đọc" đúng chưa.
**US-2.5** — Là người dùng, tôi muốn thấy token in/out + thời gian phản hồi của request gần nhất (realtime) và tổng token cả phiên, để hiểu mức tiêu thụ của mình.

**Tiêu chí chấp nhận:**
- Trạng thái nguồn cập nhật không cần refresh (SSE/WebSocket/polling).
- Bật/tắt nguồn có hiệu lực ngay ở câu hỏi kế tiếp (filter ở tầng retrieval).
- **Xóa triệt để (cascade):** xóa Source → xóa file trên S3 + Markdown + toàn bộ chunk/embedding; nguồn đã xóa không bao giờ xuất hiện trong retrieval sau đó (test bắt buộc). Mục tiêu hoàn tất tức thời trong job, tối đa ≤ 24h.
- Token usage của mỗi request (embedding + LLM, in/out) được lưu theo Message và đẩy về client ngay khi có.

### 4.4. Chat hỏi đáp RAG

**US-3.1** — Là người dùng, tôi muốn hỏi bằng tiếng Việt hoặc tiếng Anh trên các nguồn của phiên và nhận câu trả lời kèm trích dẫn click được.
**US-3.2** — Là người dùng, khi nguồn không chứa đáp án, tôi muốn hệ thống nói thẳng "không tìm thấy", để không bị đánh lừa.
**US-3.3** — Là người dùng, tôi muốn tạo/đổi tên/xóa phiên và xem lại lịch sử; xóa phiên đồng nghĩa xóa toàn bộ nguồn của phiên (có cảnh báo rõ).

**Luồng hỏi đáp:** client gửi `{câu hỏi, sessionChatId}` → Java embedding câu hỏi → vector search (filter `sessionChatId` + `userId` + nguồn đang bật) → gate ngưỡng similarity → ghép ngữ cảnh → Claude sinh câu trả lời grounded → streaming về client kèm trích dẫn + token usage.

**Tiêu chí chấp nhận:**
- **Grounding bắt buộc:** câu trả lời chỉ từ chunk truy xuất của đúng phiên đó (và đúng user).
- **Trích dẫn:** mỗi câu trả lời liệt kê nguồn + vị trí; click mở đúng đoạn gốc trong panel.
- **Mâu thuẫn giữa các đoạn (cùng hoặc khác nguồn):** nêu rõ sự không thống nhất kèm trích dẫn từng đoạn, không tự phân xử hay trộn (chi tiết §6.3).
- **Từ chối khi thiếu dữ liệu (2 lớp):** (Gate 1) không chunk nào vượt ngưỡng similarity; (Gate 2) LLM xác định ngữ cảnh không đủ → cùng thông điệp chuẩn *"Tôi không tìm thấy thông tin trong tài liệu của bạn."* + gợi ý. Không trả lời một phần bằng suy diễn.
- **Câu hỏi ngoài phạm vi** (small talk, kiến thức chung): lịch sự giải thích chỉ trả lời từ nguồn của phiên.
- **Song ngữ:** hỏi Việt trên nguồn Anh (và ngược lại) vẫn truy xuất được; trả lời theo ngôn ngữ câu hỏi.
- Hội thoại nhiều lượt hiểu ngữ cảnh trước đó; lịch sử lưu bền vững; streaming (first-token mục tiêu ≤ 3s); đánh giá 👍/👎 từng câu trả lời.

---

## 5. Kiến trúc & Data Pipeline

### 5.1. Kiến trúc microservice

```
[React + TS] ──upload file──► [Upload Service · Golang] ──put──► [S3]
     │                                 │ callback {fileKey, sessionChatId}
     │ chat / API                      ▼
     └────────────────────► [Core Backend · Java 21 Spring Boot]
                                   │            │           │
                        [Extraction · Python    │      [Claude API]
                         FastAPI + MarkItDown]  │      [Voyage API]
                                   │            ▼
                                   └──► [PostgreSQL · data] + [Qdrant · vector]
```

| Thành phần | Công nghệ | Trách nhiệm |
|---|---|---|
| **Frontend** | React + TypeScript | UI chat + panel nguồn; upload trực tiếp tới Upload Service |
| **Upload Service** | **Golang** | Một việc duy nhất: xác thực JWT, validate file, stream lên S3, callback Java. Không chứa logic nghiệp vụ |
| **Core Backend** | Java 21 + Spring Boot | Auth, API chat/phiên/nguồn, orchestration pipeline, RAG, token accounting |
| **Extraction Service** | Python (FastAPI) + MarkItDown | File/URL → Markdown + metadata; fetcher URL (chống SSRF) chạy tại đây |
| **Storage** | S3-compatible (AWS S3 / MinIO trên VPS) | File gốc + bản Markdown |
| **Database (data)** | **PostgreSQL** | User, ChatSession, Source, Chunk (text gốc + contextualized), Message — quan hệ + `ON DELETE CASCADE` |
| **Vector DB** | **Qdrant** (self-host, 1 container) | Vector embedding + payload filter (`userId`, `sessionChatId`, `sourceId`, `chunkId`) |
| **AI** | Claude API (LLM + sinh context cho chunk) · Voyage `voyage-4-lite` (embedding, dự phòng OpenAI) | Sinh câu trả lời grounded; contextual embeddings |

**Giao tiếp nội bộ:** Golang → Java qua API nội bộ có xác thực service-to-service (shared secret/mTLS); Java → Python qua HTTP nội bộ; tất cả trong mạng Docker Compose, chỉ Nginx public.

**PostgreSQL + Qdrant (mô hình 2 DB — sẽ ghi ADR-001):** PostgreSQL giữ toàn bộ dữ liệu nghiệp vụ với transaction và cascade bằng foreign key; Qdrant là engine vector chuyên dụng (HNSW, payload filter mạnh, self-host 1 container). **Nguyên tắc nhất quán dual-write:** (1) sau mỗi lần search Qdrant, Java luôn join lại PostgreSQL theo `chunkId` để lấy text — vector "mồ côi" (Postgres đã xóa, Qdrant chưa) tự bị loại khỏi kết quả; (2) job đối soát định kỳ dọn vector mồ côi trên Qdrant; (3) truy cập Qdrant qua abstraction (Spring AI `VectorStore`) để không khóa cứng vào một vendor.

### 5.2. Luồng ingest chi tiết

```
[Chọn file trong panel] → Golang: validate JWT/định dạng/size → S3 (s3Key)
    → callback Java {s3Key, sessionChatId} → tạo Source (PROCESSING) → job queue
    → tải từ S3 → Extraction (MarkItDown) → Markdown
    → chunk theo heading (~500–1000 token, overlap 10–15%)
    → CONTEXTUAL EMBEDDING (§6.2): Claude sinh ngữ cảnh cho từng chunk → Voyage embed
    → lưu chunk (text gốc + contextualized) vào PostgreSQL · upsert vector vào Qdrant
      (payload: userId, sessionChatId, sourceId, chunkId) → Source = READY → báo panel realtime
```

### 5.3. Độ bền & quan sát
- Job queue retry với backoff; quá N lần → `FAILED` kèm lý do phân loại.
- Log mỗi ingest (thời điểm, kích thước, thời gian từng bước, token tiêu thụ cho contextual, lỗi).
- Metric: thời gian xử lý p50/p95 theo định dạng, tỉ lệ lỗi, token usage theo phiên/request.
- Health check cả 4 service; tracing xuyên Golang ↔ Java ↔ Python.

---

## 6. Thiết kế AI / RAG

### 6.1. Luồng trả lời câu hỏi

```
[Câu hỏi + sessionChatId] → embed câu hỏi (Voyage)
    → Qdrant search, filter payload: userId + sessionChatId + sourceId ∈ [nguồn đang bật]
    → Top-K điểm + similarity → join PostgreSQL theo chunkId lấy text (vector mồ côi tự bị loại)
    → [Gate 1] không chunk nào ≥ ngưỡng? → "Tôi không tìm thấy thông tin..."
    → ghép ngữ cảnh (chunk + metadata nguồn) vào prompt Claude
    → [Gate 2] system prompt: CHỈ dùng ngữ cảnh; không đủ → tín hiệu từ chối
    → streaming câu trả lời + trích dẫn [luận điểm → chunk] + token usage
```

### 6.2. Contextual Embeddings (áp dụng đầy đủ từ v1)
Theo kỹ thuật *contextual retrieval*: trước khi embed, **mỗi chunk được Claude sinh một đoạn ngữ cảnh ngắn (~50–100 token)** mô tả chunk đó nằm ở đâu/nói về gì trong toàn tài liệu; đoạn ngữ cảnh được ghép vào đầu chunk rồi mới embed.

- **Model sinh context:** dùng model nhỏ/rẻ của Claude (cấu hình được); **prompt caching** cho phần toàn văn tài liệu để giảm mạnh chi phí khi lặp qua các chunk. *Lưu ý: đây là cache theo prefix chính xác (exact-prefix) của Anthropic — phần toàn văn tài liệu lặp lại 100% giữa các call sinh context nên exact match là tối ưu; không nhầm với semantic cache (cache câu trả lời theo độ tương đồng câu hỏi — không áp dụng ở v1 vì nguồn theo phiên bật/tắt được và hội thoại nhiều lượt khiến câu trả lời cache dễ sai).*
- **Lưu trữ:** giữ cả text gốc lẫn text đã contextual hóa (`contextualizedText` dùng để embed; text gốc dùng để hiển thị trích dẫn).
- **Đánh đổi chấp nhận:** ingest chậm hơn và tốn thêm chi phí Claude theo số chunk — đổi lấy độ chính xác retrieval cao hơn rõ rệt. Token tiêu thụ cho bước này được log riêng.

### 6.3. Chống ảo giác & xử lý mâu thuẫn — yêu cầu cứng
- Hai lớp chặn (Gate 1 similarity threshold cấu hình được, Gate 2 chỉ thị từ chối trong prompt) → cùng thông điệp "không tìm thấy".
- System prompt cấm dùng kiến thức ngoài ngữ cảnh; mọi luận điểm bám chunk cụ thể; chunk không dùng không xuất hiện trong trích dẫn.
- **Mâu thuẫn giữa các đoạn truy xuất — dù cùng hay khác nguồn:** không tự phân xử, không trộn, không im lặng chọn một bên. Câu trả lời phải nêu rõ sự không thống nhất kèm trích dẫn vị trí từng đoạn (ví dụ: *"Mục 2 của [hop-dong.pdf] ghi 500 triệu, nhưng phụ lục điều chỉnh ghi 450 triệu"*) để người dùng tự quyết; `contextualizedText` được dùng để cung cấp ngữ cảnh giúp người dùng phán đoán (đoạn nào thuộc phiên bản/phần nào của tài liệu). Người dùng có thể tắt bớt nguồn gây nhiễu bằng tính năng bật/tắt nguồn.
- Ngưỡng tinh chỉnh bằng bộ test vàng, không bằng cảm tính.

### 6.4. EmbeddingProvider abstraction
- Interface `embed(texts, inputType) → vectors`; provider chọn qua config.
- **Mặc định: Voyage `voyage-4-lite`** (free 200M token, tiếng Việt tốt); **dự phòng: OpenAI `text-embedding-3-small`**.
- Đổi provider ⇒ re-index toàn bộ; mỗi chunk lưu `embeddingModel` + `dimension`.

### 6.5. LLM
- **Claude API**: model trả lời và model sinh context cấu hình độc lập (mặc định: model tầm trung cho trả lời, model nhỏ cho context).
- Streaming; giới hạn top-K + budget token ngữ cảnh.

### 6.6. Đánh giá chất lượng
- **Bộ test vàng song ngữ** trên tập tài liệu mẫu cố định, **4 nhóm câu hỏi**: có đáp án / không có đáp án / mơ hồ / **có các đoạn mâu thuẫn (cùng và khác nguồn)** — nhóm 4 đo việc nêu đủ các bên thay vì chọn một; kèm case đo **lệch điểm similarity giữa ngôn ngữ** (hỏi tiếng Việt trên nguồn tiếng Anh và ngược lại). Chạy trước mỗi thay đổi lớn (model, ngưỡng, chunking, bật/tắt contextual).
- So sánh có/không contextual embeddings trên cùng bộ test để định lượng lợi ích.

---

## 7. Mô hình Dữ liệu Cốt lõi

- **User** — tài khoản (email, phương thức auth, trạng thái xác minh).
- **ChatSession** — phiên chat = notebook (userId, tên, thời điểm tạo/cập nhật, tổng token tích lũy). **Gốc sở hữu của nguồn:** xóa phiên → cascade xóa Source/Chunk/Message của phiên.
- **Source** — nguồn của phiên (sessionChatId, userId, tên, loại file/URL, định dạng, kích thước, `s3Key`, tham chiếu Markdown, trạng thái PROCESSING/READY/FAILED + lý do, cờ `enabled` bật/tắt truy xuất, metadata thời gian).
- **Chunk** — (sourceId, sessionChatId, userId, text gốc, `contextualizedText`, vị trí, embeddingModel + dimension) — bản ghi PostgreSQL; vector tương ứng nằm trên Qdrant, nối bằng `chunkId`.
- **Message** — (sessionId, vai trò, nội dung, trích dẫn [chunkId → sourceId], **tokenUsage {embedIn, llmIn, llmOut, latencyMs}**, đánh giá 👍/👎, timestamp).
- **IngestJob** — (sourceId, trạng thái, retry, log lỗi, token tiêu thụ contextual).

Dữ liệu nghiệp vụ trong **PostgreSQL** (quan hệ User → ChatSession → Source → Chunk/Message với `ON DELETE CASCADE`); vector trong **Qdrant** — mỗi point mang payload `{userId, sessionChatId, sourceId, chunkId}` để filter và để xóa theo điều kiện khi cascade.

---

## 8. Yêu cầu Phi chức năng (NFR)

- **Cô lập dữ liệu:** mọi truy vấn (CRUD + vector search) bắt buộc filter `userId` (+ `sessionChatId` với dữ liệu phiên) ở tầng thấp nhất; test tự động xác minh user A không truy xuất được dữ liệu user B, và phiên X không "nhìn" được nguồn phiên Y.
- **Xóa dữ liệu:** xóa Source/Session/tài khoản → cascade xóa mọi dữ liệu phái sinh: PostgreSQL bằng `ON DELETE CASCADE`, Qdrant bằng delete-theo-payload, S3/Markdown trong cùng job — ≤ 24h (mục tiêu tức thời); sẵn sàng yêu cầu xóa GDPR.
- **Nhất quán 2 hệ lưu trữ (dual-write):** kết quả search Qdrant luôn được join lại PostgreSQL trước khi dùng (vector mồ côi tự bị loại); job đối soát định kỳ dọn vector mồ côi; thứ tự ghi/xóa được thiết kế để trạng thái lỗi giữa chừng luôn nghiêng về phía an toàn (thà thiếu vector — không truy xuất được — còn hơn thừa).
- **Bảo mật:** mật khẩu băm chuẩn; API key (Claude/Voyage/OpenAI/S3) trong secret manager, không hardcode; TLS + encryption at rest; chống SSRF (§4.2); xác thực service-to-service giữa Golang↔Java; rate limiting kỹ thuật chống abuse; JWT xác thực thống nhất giữa Java và Golang.
- **Hiệu năng (mục tiêu ban đầu):** hỏi đáp p95 ≤ 10s, p50 ≤ 5s, first-token ≤ 3s; nguồn văn bản ≤ 20MB `READY` ≤ 2 phút *(nguồn lớn nhiều chunk có thể lâu hơn do contextual embedding — hiển thị tiến trình thay vì cam kết cứng)*; token usage hiển thị ngay khi request hoàn tất.
- **Độ chính xác AI (mục tiêu):** bộ test vàng — ≥ 90% trả lời đúng + nguồn đúng; **≥ 95% từ chối đúng khi không có đáp án (chỉ tiêu quan trọng nhất)**; đo riêng tiếng Việt/tiếng Anh.
- **Chi phí AI:** log token theo request/phiên/user từ ngày đầu (LLM trả lời + contextual + embedding) — phục vụ hiển thị minh bạch và theo dõi vận hành.
- **Khả năng mở rộng:** Upload Service và Extraction Service stateless, scale ngang; job queue tách tải ingest khỏi chat.
- **Observability:** logging có cấu trúc, tracing xuyên 3 service, cảnh báo khi tỉ lệ lỗi/độ trễ vượt ngưỡng.

---

## 9. Chỉ số Thành công (KPIs)

**Chất lượng AI (quan trọng nhất):** ≥ 95% từ chối đúng khi không có đáp án; ≥ 90% trả lời đúng + nguồn đúng (đo riêng Việt/Anh); tỉ lệ 👍 ≥ 80%.

**Sử dụng:** Activation (% đăng ký → tạo phiên + nạp ≥ 1 nguồn + hỏi ≥ 1 câu trong 24h); WAU; số câu hỏi/tuần/người; số nguồn/phiên; Retention tuần 4.

**Vận hành:** ingest thành công ≥ 95% theo định dạng; p95 độ trễ trong mục tiêu; token usage trung bình/request và /phiên (theo dõi từ ngày đầu).

---

## 10. Rủi ro & Giả định

| Rủi ro / Giả định | Mô tả | Giảm thiểu |
|---|---|---|
| **Chất lượng MarkItDown tiếng Việt** | OCR ảnh, transcription audio, PDF scan tiếng Việt có thể kém | Spike Sprint 0 với bộ file tiếng Việt thật; kém → giới hạn định dạng hoặc cắm engine thay thế sau Extraction Service |
| **Chi phí + độ trễ contextual embedding** | Mỗi chunk thêm một Claude call khi ingest; tài liệu lớn → chậm & tốn | Prompt caching toàn văn; model nhỏ; log token riêng; hiển thị tiến trình; benchmark có/không contextual để chứng minh đáng giá |
| **3 ngôn ngữ backend (Go + Java + Python)** | Chi phí vận hành/deploy cao cho solo dev | Golang & Python giữ tối giản tuyệt đối (mỗi service 1 nhiệm vụ, stateless, Docker hóa); mọi logic nghiệp vụ ở Java |
| **Nhất quán Postgres ↔ Qdrant (dual-write)** | Ghi/xóa trên 2 hệ, một bên fail → vector mồ côi có nguy cơ được truy xuất từ nguồn đã xóa | Join lại Postgres sau mỗi search (mồ côi tự bị loại); job đối soát định kỳ; thứ tự thao tác nghiêng về an toàn; test bắt buộc "xóa rồi hỏi lại" |
| **Free tier embedding cạn** | Voyage free 200M token sẽ hết | Log usage + ngưỡng cảnh báo; abstraction sẵn để chuyển paid/OpenAI (chấp nhận re-index) |
| **Chi phí LLM không kiểm soát** | Không quota → người dùng hỏi/ingest thoải mái | Rate limiting kỹ thuật; log chi phí theo user/phiên; ngưỡng cảnh báo vận hành |
| **SSRF qua URL** | Link trỏ hạ tầng nội bộ | Biện pháp §4.2; fetcher cô lập; test bảo mật |
| **Ngưỡng similarity khó chỉnh** | Cao quá → từ chối oan; thấp quá → nhiễu, tăng nguy cơ bịa | Bộ test vàng có nhóm "không có đáp án"; chỉnh bằng dữ liệu |
| **Kỳ vọng chatbot đa năng** | Người quen ChatGPT thấy "không tìm thấy" là khó chịu | Onboarding nói rõ định vị; thông điệp từ chối kèm gợi ý hành động |
| Giả định giá trị cốt lõi | Trung thực được đánh giá cao hơn trả lời trơn tru nhưng bịa | Kiểm chứng bằng 👍/👎 và phỏng vấn người dùng sớm |

---

## Phụ lục A — Quyết định kỹ thuật đã chốt

| Hạng mục | Quyết định |
|---|---|
| Kiến trúc | Microservices: **Golang** (upload → S3) · **Java 21 Spring Boot** (core logic, RAG) · **Python FastAPI + MarkItDown** (extraction) |
| Storage | **S3-compatible** (AWS S3 / MinIO trên VPS) |
| Database | **PostgreSQL** (data — JPA, transaction, `ON DELETE CASCADE`) + **Qdrant** (vector — self-host 1 container, payload filter; qua Spring AI VectorStore) — ADR-001 |
| Mô hình dữ liệu | **Session-centric**: nguồn thuộc phiên chat, xóa phiên = xóa nguồn |
| RAG | **Contextual embeddings** đầy đủ từ v1 (Claude sinh context + prompt caching) |
| LLM | **Claude API** · Embedding: **Voyage `voyage-4-lite`** (dự phòng OpenAI `text-embedding-3-small`) qua interface |
| Extract engine | **MarkItDown** — mọi định dạng → Markdown |
| Frontend | **React + TypeScript** — chat UI + panel nguồn bên phải |
| Auth | Google OAuth · Email/Password · JWT dùng chung Java/Golang |
| Hạ tầng | VPS + Docker Compose + Nginx + GitHub Actions |

## Phụ lục B — Thuật ngữ

- **RAG:** AI trả lời dựa trên tài liệu truy xuất được, kèm nguồn.
- **Contextual embeddings / contextual retrieval:** kỹ thuật sinh ngữ cảnh cho từng chunk trước khi embed để tăng độ chính xác truy xuất.
- **Embedding / Chunk / Grounding / Hallucination / SSRF / OCR / Similarity threshold:** như bản 1.0.
- **Session-centric:** mô hình dữ liệu lấy phiên chat làm trung tâm sở hữu nguồn (kiểu notebook).
- **Prompt caching:** cache phần prompt lặp lại (toàn văn tài liệu) để giảm chi phí khi gọi LLM nhiều lần.
- **Dual-write:** một thao tác nghiệp vụ phải ghi/xóa trên hai hệ lưu trữ (PostgreSQL + Qdrant) — cần cơ chế bảo đảm nhất quán.
