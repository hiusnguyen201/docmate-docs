# UC-P2 — Bật / tắt nguồn khỏi phạm vi truy xuất

| Module | Story | AC | Test |
|---|---|---|---|
| Sources | US-3.3 (Should) | `03` §US-3.3 | TC-3.3-01 [MAND] |

**Actor:** người dùng đang mở phiên · **Trigger:** gạt công tắc bật/tắt trên một nguồn · **Mức:** user goal

**Preconditions:** nguồn thuộc phiên của user.
**Postconditions:** cờ `enabled` đổi và **có hiệu lực ngay câu hỏi kế tiếp** — nguồn tắt không tham gia retrieval; bật lại là tham gia lại. Không dữ liệu nào bị xóa.

## Main flow
1. Người dùng gạt công tắc trên nguồn trong panel.
2. Java cập nhật `sources.enabled` → UI phản ánh ngay (nguồn tắt hiển thị mờ/nhãn "tắt").
3. Câu hỏi kế tiếp trong phiên: bước retrieval lấy danh sách nguồn `enabled=true AND status=READY` **tại thời điểm hỏi** → filter Qdrant `sourceId ∈ danh sách` (UC-C1 bước 3) — nguồn tắt không có chunk nào lọt vào.
4. Bật lại → câu hỏi kế tiếp truy xuất được ngay, kèm trích dẫn từ nguồn đó.

## Alternative / Exception flows
- **3a. Tắt hết mọi nguồn rồi hỏi:** xử lý như phiên không có nguồn khả dụng → UC-C2 flow 4 (hướng dẫn), hoặc thông điệp "mọi nguồn đang tắt — bật lại để hỏi" (rõ hơn; xem Open decisions).
- **2a. Gạt trên nguồn `PROCESSING`/`FAILED`:** cho phép đổi cờ (có tác dụng khi READY); UI vẫn thể hiện trạng thái xử lý là chính.
- **Race:** câu hỏi đang chạy khi người dùng gạt công tắc → câu đó dùng danh sách tại thời điểm nó bắt đầu; hiệu lực "ngay câu kế tiếp" (đúng chữ AC).

## Business rules
- Bật/tắt là filter tầng retrieval, không đụng dữ liệu (PRD §4.3) — khác về bản chất với xóa (UC-P3).
- Đây là công cụ người dùng "tắt bớt nguồn gây nhiễu" khi gặp mâu thuẫn (PRD §6.3 — nối UC-C3).

## Data touched
`sources.enabled` (update) — không đụng chunks/Qdrant.

## ⚠️ Open decisions
- Thông điệp khi mọi nguồn đều tắt: **default dùng thông điệp riêng "các nguồn của phiên đang tắt"** thay vì thông điệp từ chối chuẩn — trung thực hơn với người dùng (họ có nguồn, chỉ đang tắt). Cần thêm 1 dòng vào AC US-4.2 nếu chốt — flag để vá AC.
