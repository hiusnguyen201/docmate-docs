# UC-I4 — Theo dõi trạng thái nguồn realtime

| Module | Story | AC | Test |
|---|---|---|---|
| Ingestion | US-2.3 | `03` §US-2.3 | TC-2.3-01..03 |

**Actor:** người dùng đang mở phiên có nguồn xử lý · **Trigger:** trạng thái nguồn thay đổi phía server · **Mức:** subfunction (phục vụ UC-I1/I2/I3)

**Preconditions:** phiên có ≥ 1 Source.
**Postconditions:** trạng thái trên panel luôn phản ánh server mà không cần F5; nguồn `READY` dùng được ngay câu hỏi kế tiếp; nguồn `FAILED` cho biết lý do bằng ngôn ngữ dễ hiểu.

## Main flow
1. Client mở phiên → thiết lập kênh cập nhật trạng thái cho phiên đó.
2. Server đẩy sự kiện mỗi khi Source của phiên đổi: `PROCESSING`(kèm % chunk đã xử lý nếu đang ở giai đoạn contextual/embed) → `READY` hoặc `FAILED {failure_code}`.
3. Panel cập nhật icon/nhãn trạng thái + progress ngay khi nhận sự kiện.
4. Với `FAILED`: client map `failure_code` → thông điệp người-đọc-hiểu (bảng dưới), hiển thị khi người dùng xem chi tiết.

**Bảng map lý do (mở rộng khi phát sinh code mới — giữ đồng bộ với `07` §2.2):**

| `failure_code` | Thông điệp hiển thị |
|---|---|
| `UNSUPPORTED_FORMAT` | Định dạng file không được hỗ trợ |
| `CORRUPTED_FILE` | File bị hỏng hoặc có mật khẩu bảo vệ |
| `FETCH_FAILED` | Liên kết không truy cập được |
| `AUTH_REQUIRED` | Trang/nội dung yêu cầu đăng nhập |
| `SSRF_BLOCKED` | URL không được hỗ trợ |
| `NO_TRANSCRIPT` | Video không có phụ đề/transcript |
| `AI_PROVIDER_ERROR` | Lỗi xử lý tạm thời — thử lại sau |
| `SIZE_EXCEEDED` | Nội dung vượt giới hạn kích thước |

## Alternative / Exception flows
- **1a. Kênh realtime đứt (mất mạng, server restart):** client tự reconnect + khi nối lại **kéo trạng thái hiện tại một lần** (reconcile) — không tin sự kiện đã lỡ.
- **2a. Người dùng không mở phiên khi trạng thái đổi:** không sao — trạng thái là dữ liệu bền trên `sources`, mở phiên là thấy đúng (kênh realtime chỉ là tối ưu, không phải nguồn sự thật).
- **4a. `failure_code` lạ chưa có trong bảng:** hiển thị thông điệp chung "xử lý thất bại — thử lại hoặc dùng file khác", KHÔNG hiển thị stack trace/mã thô.

## Business rules
- Không bao giờ hiển thị stack trace cho người dùng (AC US-2.3).
- Nguồn vừa `READY` nằm trong phạm vi truy xuất ngay câu hỏi kế tiếp (kiểm ở TC-2.3-03).

## Data touched
`sources` (status, `failure_code`) · `ingest_jobs` (progress) — chỉ đọc/đẩy sự kiện.

## ⚠️ Open decisions
- Cơ chế kênh: AC cho phép SSE/WebSocket/polling — **default SSE** (một chiều server→client là đủ, nhẹ hơn WS, hạ tầng Nginx hỗ trợ thẳng); quyết chính thức khi hiện thực US-2.3, ghi vào `07`.
- Tần suất progress: **default đẩy mỗi 10% hoặc tối thiểu mỗi 5 giây** khi đang contextual/embed.
