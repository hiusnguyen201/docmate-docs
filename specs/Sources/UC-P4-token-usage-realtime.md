# UC-P4 — Hiển thị token usage & thời gian phản hồi realtime

| Module | Story | AC | Test |
|---|---|---|---|
| Sources | US-3.5 | `03` §US-3.5 | TC-3.5-01..02 |

**Actor:** người dùng đang hỏi đáp trong phiên · **Trigger:** một câu trả lời hoàn tất / mở lại phiên cũ · **Mức:** subfunction (minh bạch — nguyên tắc sản phẩm #5)

**Preconditions:** phiên thuộc user.
**Postconditions:** panel hiển thị token in/out (embedding + LLM) + latency của request gần nhất ngay khi response hoàn tất; tổng token phiên cộng dồn đúng và **bền vững** (mở lại vẫn đúng).

## Main flow
1. Người dùng gửi câu hỏi (UC-C1); trong quá trình xử lý Java đo: token embed câu hỏi, token LLM in/out, latency tổng.
2. Response hoàn tất (kể cả trường hợp từ chối) → Java lưu `token_usage {embedIn, llmIn, llmOut, latencyMs}` vào Message + cộng `total_tokens` của phiên.
3. Số liệu đẩy về client trong cùng response/stream cuối → panel "Token & hiệu năng" hiển thị ngay: request gần nhất + tổng phiên.
4. Mở lại phiên cũ → tổng token phiên đọc từ dữ liệu bền (khớp tuyệt đối tổng các Message — TC-3.5-02).

## Alternative / Exception flows
- **2a. Câu bị từ chối Gate 1 (không gọi LLM trả lời):** vẫn ghi nhận token embed câu hỏi + latency; llmIn/Out = 0 — minh bạch cả request "rẻ".
- **2b. Request lỗi giữa chừng (LLM đứt kết nối):** ghi nhận phần token đã tiêu được provider báo; đánh dấu request lỗi trong Message hoặc không lưu Message trả lời — token KHÔNG được âm thầm bỏ qua (xem Open decisions).
- **Token ingest (contextual):** KHÔNG thuộc panel này theo request — nó log theo IngestJob (UC-I1); tổng phiên trên panel là token hỏi đáp. Tránh nhầm hai dòng chi phí.

## Business rules
- Log token theo request từ ngày đầu (PRD §8 chi phí AI); panel là mặt hiển thị của cùng dữ liệu — không hai hệ đếm.
- `total_tokens` denormalized trên `chat_sessions` (`08` §3) phải luôn khớp sum messages — lệch là bug.

## Data touched
`messages.token_usage` (insert) · `chat_sessions.total_tokens` (update cộng dồn).

## ⚠️ Open decisions
- 2b chi tiết: **default lưu Message assistant trạng thái lỗi kèm token đã tiêu** (giữ sổ sách trung thực) — chốt khi hiện thực US-3.5.
- Panel có hiển thị token contextual của phiên (dòng riêng "chi phí ingest") không: **default chưa ở v1** — PRD chỉ yêu cầu request hỏi đáp; ghi backlog nếu người dùng cần.
