# UC-C3 — Xử lý mâu thuẫn giữa các đoạn truy xuất (yêu cầu cứng)

| Module | Story | AC | Test |
|---|---|---|---|
| RagChat | US-4.1 (phần mâu thuẫn) | `03` §US-4.1 sc.3–4 | TC-4.1-03, 04 [MAND] + bộ vàng N4 |

**Actor:** người dùng hỏi câu mà các chunk truy xuất chứa thông tin trái ngược · **Trigger:** phát sinh trong bước sinh trả lời của UC-C1 · **Mức:** hành vi bắt buộc bên trong UC-C1 (tách spec vì là yêu cầu cứng PRD §6.3)

**Preconditions:** UC-C1 đã qua Gate 1, chunks đưa vào prompt chứa ≥ 2 đoạn mâu thuẫn về đúng điều được hỏi.
**Postconditions:** câu trả lời **nêu rõ sự không thống nhất**, trích dẫn TỪNG đoạn/nguồn; không im lặng chọn một bên; không trộn thành một giá trị; người dùng đủ thông tin tự phán đoán.

## Main flow (delta trên UC-C1 bước 6–8)
1. System prompt (UC-C1 bước 6) chứa chỉ thị cứng: khi các đoạn ngữ cảnh mâu thuẫn về điều được hỏi — nêu tất cả các bên, trích dẫn từng đoạn, tuyệt đối không tự phân xử/không tính trung bình/không chọn "bản mới hơn" trừ khi chính văn bản nói rõ.
2. LLM sinh câu trả lời dạng: nêu sự không thống nhất + từng phía kèm citation. Ví dụ chuẩn (PRD §6.3): *"Mục 2 của [hop-dong.pdf] ghi 500 triệu, nhưng phụ lục điều chỉnh ghi 450 triệu."*
3. Nếu `contextualized_text` của các chunk cho biết ngữ cảnh (đoạn thuộc phụ lục/phiên bản/phần nào của tài liệu) → trình bày kèm ngữ cảnh đó, giúp người dùng phán đoán — vẫn không kết luận thay.
4. Citation đủ cho MỌI bên (bên nào thiếu citation = trượt); click từng citation nhảy đúng đoạn (UC-C1 bước 9).
5. (Tùy chọn UI) Gợi ý người dùng có thể tắt bớt nguồn gây nhiễu bằng UC-P2.

## Hai hình thái phải xử lý như nhau
| Hình thái | Ví dụ | Test |
|---|---|---|
| **Khác nguồn** | Hai bản hợp đồng trong phiên ghi hai mức giá | TC-4.1-03 · bộ vàng D1×D2 |
| **Cùng nguồn** | Điều khoản chính vs phụ lục điều chỉnh trong cùng file | TC-4.1-04 · bộ vàng D3 |

## Exception flows
- **2a. LLM chọn một bên / trộn giá trị:** lỗi hành vi — bắt bằng bộ vàng N4 (chấm "nêu-đủ-các-bên", trượt ngay nếu xuất hiện giá trị trộn — `13` §3); chỉnh prompt và đo lại, không vá bằng hậu xử lý mong manh.
- **2b. Mâu thuẫn "giả" (hai đoạn nói về hai đối tượng khác nhau — giá gói A vs gói B):** không phải mâu thuẫn — trả lời phân biệt rõ hai đối tượng kèm citation; thuộc chất lượng contextual embedding (context giúp LLM phân biệt). Case dạng này nên có trong N3/N4 để theo dõi.
- **Nhiều hơn 2 bên mâu thuẫn:** nêu đủ tất cả các bên xuất hiện trong chunks truy xuất — không giới hạn 2.

## Business rules
- "Không tự phân xử, không trộn, không im lặng chọn một bên" — nguyên văn PRD §6.3, yêu cầu cứng.
- Vai trò của contextual embeddings ở đây là lý do đánh đổi chi phí từ v1 (ADR ngữ cảnh — PRD §6.2).

## Data touched
Như UC-C1 (không khác biệt lưu trữ).

## ⚠️ Open decisions
- Bộ vàng N4 v1 chưa ép ngưỡng % cứng (baseline theo dõi — `14` Gate 3.4): giữ nguyên; đặt ngưỡng sau khi có baseline thật.
