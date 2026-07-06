# ADR-001 — PostgreSQL (data) + Qdrant (vector) thay MongoDB

| | |
|---|---|
| **Trạng thái** | ✅ Accepted — 2026-07-06 |
| **Phạm vi ảnh hưởng** | Toàn bộ tầng lưu trữ DocMate; PRD v1.2 → v1.3; US-0.1, US-0.3 và mọi story chạm dữ liệu |
| **Tham chiếu** | `PRD-docmate-v1.3.md` §5.1, §7, §8 · `07-architecture.md` §5 · `08-erd.md` |

---

## 1. Context

PRD tới v1.2 chọn **MongoDB làm DB duy nhất** (Atlas cho prod, Community 8.2 cho dev), bao gồm cả vector search — hấp dẫn vì "một hệ lưu trữ cho tất cả". Khi rà lại trước lúc chốt schema (US-0.3), các điểm sau nổi lên:

1. **Dữ liệu DocMate có tính quan hệ mạnh:** User → ChatSession → Source → Chunk/Message là cây sở hữu chặt, và NFR yêu cầu **xóa triệt để theo cascade** (xóa phiên = xóa nguồn = xóa chunk/message; GDPR-ready). MongoDB không có FK — cascade phải tự viết ở tầng ứng dụng, mỗi đường xóa là một nguồn bug cô lập dữ liệu tiềm tàng. Với sản phẩm mà "dữ liệu của ai người đó thấy, xóa là xóa triệt để" là nguyên tắc bất biến, cascade tự chế là rủi ro sai chỗ đau nhất.
2. **Vector search của MongoDB gắn với Atlas:** Atlas Vector Search là dịch vụ cloud; bản Community self-host không có năng lực tương đương ổn định — trong khi hạ tầng đã chốt là **VPS + Docker Compose tự vận hành**. Dev (Community) và prod (Atlas) lệch nhau đúng ở tính năng quan trọng nhất của sản phẩm (retrieval).
3. **Chi phí & khóa vendor:** Atlas là chi phí thuê bao theo cluster; định hướng dự án là self-host tối đa trên VPS.

Quyết định cần đưa ra **trước** US-0.3 (schema) vì mọi story sau xây trên nó — đổi sau Sprint 0 là đập móng.

## 2. Decision

Dùng **hai hệ lưu trữ chuyên trách**:

- **PostgreSQL** — toàn bộ dữ liệu nghiệp vụ (User, ChatSession, Source, Chunk, Message, IngestJob): quan hệ + transaction + **FK `ON DELETE CASCADE`** làm cơ chế xóa triệt để ở mức DB, không phụ thuộc code ứng dụng.
- **Qdrant** (self-host, 1 container trong Compose) — vector embedding + payload filter (`userId`, `sessionChatId`, `sourceId`, `chunkId`): HNSW, filter mạnh trong lúc search, delete theo payload phục vụ cascade.
- Truy cập Qdrant qua abstraction **Spring AI `VectorStore`** — không khóa cứng vào một vendor vector DB.

## 3. Alternatives considered

| Phương án | Ưu | Nhược | Kết luận |
|---|---|---|---|
| **A. Giữ MongoDB (Atlas prod + Community dev)** | Một DB duy nhất; đã quen từ v1.2; document model linh hoạt | Không FK/cascade → tự viết xóa triệt để; vector search chỉ đủ mạnh trên Atlas (thuê bao, lệch dev/prod); trái định hướng self-host | ❌ Loại — điểm yếu rơi đúng vào 2 nguyên tắc bất biến (xóa triệt để, cô lập dữ liệu) |
| **B. PostgreSQL + pgvector (một DB duy nhất)** | Một container; cascade FK ăn luôn cả vector (vector là cột trong bảng); không có bài toán dual-write | Hiệu năng filter-trong-search (multi-tenant theo user/phiên/nguồn enabled) kém hơn engine chuyên dụng khi dữ liệu lớn; index HNSW của pgvector cạnh tranh RAM/vacuum với workload OLTP cùng instance | ⚠️ Ứng viên mạnh — bị xếp sau vì filter payload là thao tác nóng nhất của DocMate (mọi câu hỏi đều filter 3 chiều). Là **đường lùi số 1** nếu vận hành 2 DB quá nặng |
| **C. PostgreSQL + Qdrant (chọn)** | Mỗi hệ làm đúng sở trường: Postgres cascade/transaction, Qdrant HNSW + payload filter + delete-theo-payload; cả hai self-host nhẹ, 2 container trong Compose | **Dual-write:** ghi/xóa trên 2 hệ, một bên fail → nguy cơ vector mồ côi; thêm một hệ để vận hành | ✅ Chọn — nhược điểm dual-write kiểm soát được bằng thiết kế (§4), và hướng fail được ép về phía an toàn |

> Các phương án vector DB chuyên dụng khác (Milvus, Weaviate) không được xét sâu: nặng vận hành hơn Qdrant trên một VPS solo-dev, trong khi Qdrant đủ và đã là fallback quen thuộc từ trước qua Spring AI VectorStore.

## 4. Consequences

### Tích cực
- Xóa triệt để trong Postgres là **bảo đảm của DB engine** (FK cascade), không phải kỷ luật của code — đúng chỗ cần cứng nhất.
- Dev và prod **cùng một stack** (2 container), hết lệch Atlas/Community.
- Retrieval filter 3 chiều (`userId`+`sessionChatId`+`sourceId ∈ enabled`) chạy trên engine sinh ra để làm việc đó.

### Tiêu cực + giảm thiểu (đây là phần trả giá)

| Hệ quả xấu | Giảm thiểu (đã ghi thành yêu cầu cứng ở PRD §8 + 07 §5) |
|---|---|
| Vector mồ côi khi xóa fail giữa chừng | (1) **Join bắt buộc**: mọi kết quả search join lại Postgres theo `chunkId` — mồ côi tự bị loại; text không nằm trong payload để không có đường tắt; (2) **job đối soát định kỳ** dọn mồ côi + re-embed chunk thiếu vector; (3) thứ tự thao tác cố định: ghi Postgres→Qdrant, xóa Postgres→Qdrant→S3 — mọi trạng thái lỡ dở đều nghiêng về "không truy xuất được" |
| Thêm một hệ để vận hành (backup, upgrade, monitor) | Qdrant 1 container ít cấu hình; dữ liệu Qdrant **tái tạo được 100%** từ Postgres + re-embed → backup nghiêm túc chỉ cần cho Postgres + S3, Qdrant hỏng là re-index |
| Không có transaction xuyên 2 hệ | Chấp nhận — thiết kế mọi luồng để trạng thái giữa chừng an toàn (trên) thay vì mô phỏng distributed transaction |
| DoD nặng thêm | Mọi story chạm xóa dữ liệu bắt buộc test "xóa rồi hỏi lại" + test vector mồ côi (`04-dor-dod.md` §2.2) |

### Ảnh hưởng tài liệu/story
- PRD v1.3, vision/backlog/AC v1.3 đã cập nhật theo quyết định này.
- US-0.1 (compose thêm Postgres + Qdrant), US-0.3 (schema + collection + 3 test nền) hiện thực hóa quyết định.

## 5. Điều kiện xem xét lại (revisit triggers)

Mở lại ADR này nếu một trong các điều sau xảy ra:

- Job đối soát phát hiện vector mồ côi **lặp lại có hệ thống** (không phải sự cố đơn lẻ) → cân nhắc phương án B (pgvector) để khử dual-write.
- Vận hành 2 DB trên VPS chứng minh quá tải cho solo dev (đo bằng thời gian vá sự cố lưu trữ qua các sprint) → phương án B.
- Qdrant self-host không đáp ứng hiệu năng/độ ổn định → nhờ Spring AI VectorStore, thay engine vector khác mà không đổi mô hình 2 hệ.

---

*Format theo Nygard ADR. ADR kế tiếp: ADR-002 — kết luận spike MarkItDown tiếng Việt (US-0.5, viết trong Sprint 0).*
