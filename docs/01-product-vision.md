# DocMate — Product Vision

| | |
|---|---|
| **Sản phẩm** | DocMate |
| **Phiên bản tài liệu** | 1.3 — 2026-07-06 |
| **Tham chiếu** | `PRD-docmate-v1.3.md` |

---

## Vision Statement

> **Dành cho** cá nhân có tập tài liệu riêng cần tra cứu (chuyên viên, freelancer, sinh viên, nhà nghiên cứu), **những người** mất thời gian đọc lại thủ công từng file và không tin được các chatbot AI hay bịa, **DocMate** là một web app SaaS **cho phép** tạo các phiên chat kiểu notebook — vừa hỏi đáp vừa thêm nguồn (file đa định dạng, URL, YouTube) ngay trong lúc chat. **Khác với** ChatGPT/NotebookLM và các công cụ đại trà, DocMate **cam kết** chỉ trả lời từ nguồn người dùng cung cấp, luôn kèm trích dẫn, minh bạch token tiêu thụ theo từng request, và nói thẳng *"Tôi không tìm thấy thông tin"* khi nguồn không có đáp án — không bao giờ bịa.

## Mục tiêu (Goals)

| # | Mục tiêu | Đo bằng |
|---|---|---|
| G1 | Câu trả lời đáng tin tuyệt đối | ≥ 95% từ chối đúng khi không có đáp án; ≥ 90% trả lời đúng + nguồn đúng (bộ test vàng song ngữ) |
| G2 | Nạp được "mọi thứ người dùng có" ngay trong phiên chat | Ingest thành công ≥ 95% với các định dạng hỗ trợ (PDF/Office/ảnh/audio/URL/YouTube/ZIP/EPub) |
| G3 | Trải nghiệm tra cứu nhanh, không rời hội thoại | First-token ≤ 3s, p95 ≤ 10s; nguồn văn bản ≤ 20MB sẵn sàng ≤ 2 phút; thêm nguồn không gián đoạn chat |
| G4 | Minh bạch với người dùng | Token in/out + thời gian phản hồi hiển thị realtime theo từng request; tổng token theo phiên |

## Đối tượng & Giá trị

| Persona | Giá trị nhận được |
|---|---|
| Chuyên viên văn phòng | Gom quy trình/hợp đồng/báo cáo vào một phiên, hỏi thay vì đọc lại từng file |
| Sinh viên / nhà nghiên cứu | Notebook cho từng môn/đề tài, hỏi đáp + trích dẫn chính xác |
| Freelancer / consultant | Mỗi khách hàng một phiên riêng, nạp brief + tài liệu và tra cứu tức thì |

## Phạm vi v1

- Auth: Google, email/password.
- Phiên chat = notebook: nguồn thuộc phiên, thêm nguồn ngay trong lúc chat.
- Panel phải: nguồn + trạng thái, bật/tắt nguồn, tổng quan, token realtime, preview Markdown.
- Upload đa định dạng (Golang → S3) + dán URL/YouTube.
- Chat RAG: contextual embeddings, trích dẫn, từ chối khi thiếu dữ liệu, song ngữ Việt–Anh, streaming, 👍/👎.

## Nguyên tắc sản phẩm (bất biến)

1. **Grounded-only:** mọi câu trả lời phải truy xuất từ nguồn của phiên.
2. **Không bịa:** thiếu căn cứ → *"Tôi không tìm thấy thông tin trong tài liệu của bạn."*
3. **Luôn dẫn nguồn:** mỗi câu trả lời chỉ rõ nguồn/đoạn làm căn cứ.
4. **Dữ liệu của ai người đó thấy:** cô lập theo user và theo phiên; xóa là xóa triệt để.
5. **Minh bạch:** người dùng luôn thấy hệ thống tiêu thụ bao nhiêu token cho họ.

## Tech Stack (đã chốt)

| Tầng | Công nghệ |
|---|---|
| Frontend | React + TypeScript (chat UI + panel nguồn) |
| Upload Service | **Golang** — nhận file, đẩy S3, callback Java |
| Core Backend | Java 21 + Spring Boot — logic nghiệp vụ, RAG, orchestration |
| Extraction Service | Python (FastAPI) + MarkItDown |
| Storage | S3-compatible (AWS S3 / MinIO trên VPS) |
| Database | PostgreSQL (data) + Qdrant (vector, self-host) |
| AI | Claude API (trả lời + sinh context cho chunk) · Voyage `voyage-4-lite` (embedding, dự phòng OpenAI) |
| Hạ tầng | VPS + Docker Compose + Nginx + GitHub Actions |

## Team & Nhịp làm việc

- **Quy mô:** 1 người (solo dev, Scrum rút gọn).
- **Sprint:** 2 tuần.
- **Ước lượng lộ trình v1:** ~6 sprint cho toàn bộ Must + 1 sprint cho nhóm Should (xem Product Backlog).
