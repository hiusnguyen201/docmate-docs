# UC-C2 — Từ chối khi thiếu dữ liệu (không bao giờ bịa)

| Module | Story | AC | Test |
|---|---|---|---|
| RagChat | US-4.2 | `03` §US-4.2 | TC-4.2-01..04 + bộ vàng N2 (**≥ 95% — chỉ tiêu quan trọng nhất**) |

**Actor:** người dùng gửi câu hỏi · **Trigger:** một trong 4 tình huống thiếu căn cứ dưới đây, phát hiện trong luồng UC-C1 · **Mức:** user goal (nhánh của UC-C1, tách riêng vì là lời hứa cốt lõi của sản phẩm)

**Preconditions:** như UC-C1.
**Postconditions:** người dùng nhận thông điệp từ chối **chuẩn, nhất quán** kèm gợi ý hành động; KHÔNG câu trả lời một phần bằng suy diễn; Message + token vẫn ghi sổ (UC-P4 2a).

**Thông điệp chuẩn (một nguồn sự thật, dùng ở mọi nhánh 1–2):**
> *"Tôi không tìm thấy thông tin trong tài liệu của bạn."* + gợi ý: thử diễn đạt lại câu hỏi, hoặc thêm nguồn liên quan vào phiên.

## Các flow

### Flow 1 — Gate 1: retrieval không đạt ngưỡng (từ UC-C1 bước 5)
1. Không chunk nào ≥ ngưỡng similarity → Java KHÔNG gọi LLM trả lời.
2. Trả thông điệp chuẩn + gợi ý; ghi Message; token = embed câu hỏi (UC-P4 2a).

### Flow 2 — Gate 2: LLM thấy ngữ cảnh không đủ (từ UC-C1 bước 7a)
1. Retrieval có kết quả vượt ngưỡng nhưng LLM (được chỉ thị trong system prompt) xác định ngữ cảnh không trả lời được đúng câu hỏi → phát tín hiệu từ chối.
2. Java nhận diện tín hiệu (cơ chế buffer — UC-C1 Open decisions) → gửi **cùng thông điệp chuẩn** với Flow 1, hiển thị trọn vẹn không nửa vời (TC-4.5-02).
3. Người dùng không phân biệt được từ chối do Gate nào — nhất quán là chủ đích (nội bộ log rõ Gate nào để tinh chỉnh ngưỡng).

### Flow 3 — Câu hỏi ngoài phạm vi (small talk / kiến thức chung)
1. Câu như "thủ đô Pháp là gì?", "bạn khỏe không?" → bot **lịch sự giải thích** chỉ trả lời từ nguồn của phiên — dùng thông điệp định vị, KHÔNG dùng nguyên văn thông điệp "không tìm thấy" (câu này dành cho "đã tìm mà không thấy").
2. Về mặt máy: phần lớn rơi tự nhiên vào Gate 1/Gate 2; system prompt chỉ thị khi nhận diện small talk thì trả lời định vị thay vì thông điệp chuẩn.

### Flow 4 — Phiên không có nguồn khả dụng
1. Chưa có nguồn nào `READY` (hoặc mọi nguồn đang tắt — UC-P2 3a) → KHÔNG chạy retrieval.
2. Bot hướng dẫn thêm nguồn ở panel phải (empty-state đồng bộ UC-P1); nếu do tắt hết → thông điệp "các nguồn của phiên đang tắt".

## Exception flows
- **Flow 2 – 2a. LLM vừa từ chối vừa "lỡ" kèm suy diễn:** chấm trượt trong bộ vàng N2 (`13` §3) — chống bằng prompt + đo, không có cách chặn cơ học tuyệt đối; ngưỡng ≥ 95% là hàng rào.
- **Câu hỏi nửa-có-nửa-không (một phần có căn cứ):** trả lời phần có căn cứ kèm trích dẫn + nói rõ phần không tìm thấy — KHÔNG lấp phần thiếu bằng suy diễn. (Ranh giới tinh — thuộc nhóm N3 mơ hồ của bộ vàng, chấm rubric tay.)

## Business rules
- Không bịa là nguyên tắc bất biến #2; ≥ 95% từ chối đúng là KPI quan trọng nhất (PRD §9).
- Hai Gate cùng MỘT thông điệp (PRD §6.3); ngưỡng Gate 1 chỉnh bằng bộ vàng, chú ý lệch ngôn ngữ (`13` §4).

## Data touched
Như UC-C1 (Message vẫn lưu; llmIn/Out = 0 với Flow 1/4).

## ⚠️ Open decisions
- Thông điệp Flow 3 và Flow 4 (nguyên văn): **default** — F3: "Mình chỉ trả lời dựa trên tài liệu trong phiên này. Bạn thử hỏi về nội dung các nguồn đã thêm nhé."; F4: "Phiên này chưa có nguồn nào sẵn sàng — thêm file hoặc dán URL ở panel bên phải để bắt đầu." Chốt cùng UX writing khi làm UI.
- Thông điệp "mọi nguồn đang tắt" (Flow 4): cần vá 1 dòng vào AC US-4.2 — flag chung với UC-P2.
