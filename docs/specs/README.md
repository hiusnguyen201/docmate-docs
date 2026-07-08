# DocMate — Use Case Specifications

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Vai trò tầng** | UC spec = *hành vi hệ thống theo luồng, quan sát được* — hợp đồng để code theo (spec-driven). AC (`03`) = tiêu chí nghiệm thu. `07/08` = thiết kế bên trong. Spec **tham chiếu chéo, không chép lại** |

## Bản đồ hộp (module)

| Hộp | UC | Story phủ |
|---|---|---|
| `Auth/` | UC-A1 đăng ký · UC-A2 đăng nhập/JWT · UC-A3 OAuth Google · UC-A4 quên mật khẩu · UC-A5 xóa tài khoản | US-1.1→1.5 |
| `SessionChat/` | UC-S1 tạo/đổi tên/lịch sử · UC-S2 xóa phiên | US-4.4 |
| `Ingestion/` | UC-I1 upload file · UC-I2 nạp URL · UC-I3 YouTube/ZIP/audio · UC-I4 theo dõi trạng thái | US-0.7, 2.1→2.7 |
| `Sources/` | UC-P1 xem/preview/tổng quan · UC-P2 bật/tắt · UC-P3 xóa nguồn · UC-P4 token realtime | US-3.1→3.5 |
| `RagChat/` | UC-C1 hỏi đáp grounded · UC-C2 từ chối thiếu dữ liệu · UC-C3 xử lý mâu thuẫn · UC-C4 đánh giá | US-4.1→4.3, 4.5→4.7 |

E0 không có UC — spec nền tảng là `07-architecture` / `08-erd` / `10-setup-deployment`.

## Format bắt buộc mỗi UC

1. **Header bảng:** module, story, AC, test case liên quan.
2. **Actor / Trigger / Mức** (user goal | subfunction).
3. **Preconditions / Postconditions** — hậu điều kiện ghi cả nhánh thành công lẫn thất bại.
4. **Main flow** — bước đánh số; mỗi bước: ai làm gì, hệ thống phản hồi gì (mức quan sát được; ghi service xử lý trong ngoặc khi cần).
5. **Alternative / Exception flows** — đánh số bám bước chính (`3a`, `3b`…). Mọi nhánh lỗi trong AC phải có mặt tại đây với số riêng.
6. **Business rules** — chỉ trỏ về PRD/07/08, kèm 1 dòng tóm tắt.
7. **Data touched** — bảng/payload chạm tới (trỏ `08`).
8. **⚠️ Open decisions** — điều spec chưa chốt, phải quyết khi code (mỗi mục ghi default đề xuất).

## Quy ước

- Một UC pass khi: code đi đúng main flow + đủ mọi exception flow, và các TC liên kết pass.
- Đổi hành vi ⇒ sửa UC spec **trước** khi sửa code (spec-driven), cùng chuỗi commit `docs(specs): ...`.
- Mapping story → UC bổ sung cho `15-traceability-matrix.md` (matrix giữ nguyên, tra UC tại bảng trên).
