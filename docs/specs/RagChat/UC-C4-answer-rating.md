# UC-C4 — Đánh giá câu trả lời 👍/👎

| Module | Story | AC | Test |
|---|---|---|---|
| RagChat | US-4.6 (Should) | `03` §US-4.6 | TC-4.6-01 |

**Actor:** người dùng vừa nhận câu trả lời · **Trigger:** bấm 👍 hoặc 👎 trên một tin nhắn assistant · **Mức:** subfunction

**Preconditions:** Message assistant thuộc phiên của user.
**Postconditions:** đánh giá lưu bền vào Message (`rating`: 1 = 👍, −1 = 👎); tính vào số liệu chất lượng (KPI tỉ lệ 👍 ≥ 80%, PRD §9); đổi được đúng một lần.

## Main flow
1. Người dùng bấm 👍/👎 trên câu trả lời (kể cả câu trả lời từ chối — đánh giá từ chối đúng/sai cũng là dữ liệu quý).
2. Java lưu `messages.rating` → UI đánh dấu lựa chọn.
3. Người dùng đổi ý bấm nút còn lại → cập nhật (lần đổi thứ nhất).
4. Sau đó rating khóa — bấm tiếp không tác dụng, UI thể hiện trạng thái khóa.

## Alternative / Exception flows
- **3a. Bấm lại đúng nút đã chọn:** không đổi gì (idempotent), không tính lượt đổi.
- **Đánh giá tin nhắn không thuộc mình:** 404 (cô lập, như UC-S1).
- **Đánh giá tin nhắn role USER:** từ chối — chỉ tin nhắn assistant.

## Business rules
- "Đổi được một lần" theo AC — đổi ý một lần là hợp lý, chống spam toggle làm bẩn số liệu.
- Rating là nguồn kiểm chứng giả định giá trị cốt lõi ("trung thực > trơn tru" — PRD §10).

## Data touched
`messages.rating` (update, kèm ràng buộc chỉ role ASSISTANT).

## ⚠️ Open decisions
- Theo dõi "đã đổi mấy lần" để khóa: **default thêm cột `rating_changed boolean`** trên messages (bổ sung `08` khi hiện thực); giữ tối giản, không audit log rating ở v1.
