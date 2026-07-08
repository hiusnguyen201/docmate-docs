# UC-C1 — Hỏi đáp grounded kèm trích dẫn (nhiều lượt, song ngữ, streaming)

| Module | Story | AC | Test |
|---|---|---|---|
| RagChat | US-4.1 (main flow), US-4.3, US-4.5, US-4.7 | `03` §US-4.1 sc.1–2,5 · §US-4.3 · §US-4.5 · §US-4.7 | TC-4.1-01,02,05 · TC-4.3-01 · TC-4.5-01 · TC-4.7-01 + bộ vàng N1 |

**Actor:** người dùng trong phiên có ≥ 1 nguồn `READY` đang bật · **Trigger:** gửi câu hỏi · **Mức:** user goal — **UC trung tâm của sản phẩm**

**Preconditions:** đăng nhập; phiên thuộc user.
**Postconditions — thành công:** câu trả lời xây dựng CHỈ từ chunk truy xuất của đúng phiên/đúng user/nguồn đang bật, kèm trích dẫn click được; Message lưu bền (nội dung + citations + token_usage); tổng token phiên cập nhật. **— nhánh từ chối:** chuyển UC-C2. **— nhánh mâu thuẫn:** UC-C3 (vẫn trong cùng lượt trả lời).

## Main flow
1. Người dùng gõ câu hỏi (tiếng Việt hoặc Anh) → gửi `{câu hỏi, sessionChatId}`.
2. Java embed câu hỏi (Voyage — cùng provider/model với chunk của phiên).
3. Java lấy danh sách nguồn `enabled=true AND status=READY` của phiên → Qdrant search top-K, filter payload `userId + sessionChatId + sourceId ∈ danh sách`.
4. Join kết quả về PostgreSQL theo `chunkId` lấy `original_text` + `contextualized_text` + `location_hint` — point không join được (mồ côi) bị loại.
5. **Gate 1:** không chunk nào ≥ ngưỡng similarity → chuyển UC-C2 (từ chối).
6. Java ghép prompt: system prompt grounded-only (cấm kiến thức ngoài ngữ cảnh, chỉ thị từ chối Gate 2, chỉ thị nêu-đủ-các-bên khi mâu thuẫn — UC-C3) + các chunk kèm metadata nguồn + **lịch sử các lượt trước của phiên** (đủ để hiểu tham chiếu như "còn tiến độ thì sao?") + chỉ thị **trả lời theo ngôn ngữ của câu hỏi**.
7. Claude sinh trả lời, **streaming** về client — first-token mục tiêu ≤ 3s; chữ hiện dần.
8. Kết thúc stream: câu trả lời kèm danh sách trích dẫn `[luận điểm → chunkId → sourceId + location_hint]`; chunk không dùng không xuất hiện trong trích dẫn.
9. Người dùng click một citation → panel phải nhảy tới đoạn gốc trong nguồn tương ứng (định vị theo `location_hint`, hiển thị `original_text`).
10. Java lưu Message (user + assistant) với citations + token_usage (UC-P4); panel token cập nhật.

## Alternative / Exception flows
- **5a. Gate 1 kích hoạt** → UC-C2.
- **7a. Gate 2: LLM phát tín hiệu ngữ cảnh không đủ** → UC-C2 flow 2 (thông điệp từ chối hiển thị TRỌN VẸN — cơ chế: không stream thẳng ra UI phần mở đầu khi chưa loại trừ tín hiệu từ chối; chi tiết Open decisions).
- **6a/8a. Chunks chứa các đoạn mâu thuẫn về đúng câu hỏi** → hành vi theo UC-C3 (nêu các bên) — vẫn là một câu trả lời bình thường về mặt luồng.
- **2a. Hỏi tiếng Việt trên nguồn tiếng Anh (hoặc ngược lại):** không nhánh riêng — embedding đa ngữ lo phần truy xuất; bước 6 bảo đảm ngôn ngữ trả lời = ngôn ngữ câu hỏi. Chất lượng hai chiều đo bằng bộ vàng (`13` §4).
- **7b. LLM đứt kết nối giữa stream:** UI báo "trả lời bị gián đoạn — thử lại"; token đã tiêu vẫn ghi sổ (UC-P4 2b).
- **1a. Câu hỏi rỗng/quá dài:** chặn client + server (giới hạn ký tự — Open decisions).
- **3a. Không nguồn nào khả dụng (chưa có / đều tắt / đều FAILED):** UC-C2 flow 4.
- **Cô lập:** user B hỏi nội dung phiên user A, hoặc phiên Y hỏi nội dung phiên X — cấu trúc filter bước 3 khiến không bao giờ xảy ra; kiểm bằng TC-4.1-05 [MAND].

## Business rules
- **Grounded-only** + **luôn dẫn nguồn** + **không bịa** (nguyên tắc bất biến 1–3, PRD §6.3).
- Mỗi luận điểm bám ≥ 1 chunk; citation dùng `original_text` để hiển thị, `contextualized_text` chỉ phục vụ embed + ngữ cảnh (PRD §6.2).
- Tham số (ngưỡng Gate 1, top-K, budget token, model) là config, tinh chỉnh bằng bộ vàng — không sửa theo cảm tính.

## Data touched
`sources` (đọc enabled) · Qdrant search · `chunks` (join) · `messages` (insert ×2) · `chat_sessions.total_tokens`.

## ⚠️ Open decisions
- **Cơ chế "từ chối trọn vẹn khi streaming" (7a):** default — LLM được chỉ thị nếu từ chối thì phát token tín hiệu chuẩn NGAY ĐẦU output; Java buffer phần đầu stream (~vài chục token) đủ nhận diện tín hiệu: có → thay bằng thông điệp từ chối chuẩn gửi một cục; không → xả buffer và stream tiếp bình thường. Đánh đổi: first-token chậm thêm một nhịp buffer.
- Số lượt lịch sử đưa vào prompt: **default 10 lượt gần nhất trong budget token**; tóm tắt hội thoại dài: ngoài phạm vi v1.
- Giới hạn độ dài câu hỏi: **default 4.000 ký tự**.
- Format citation trong stream: **default đánh dấu inline `[n]` + danh sách map ở cuối stream** — chốt cùng UI.
