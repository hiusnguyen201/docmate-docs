# DocMate — Product Backlog

| | |
|---|---|
| **Phiên bản** | 1.3 — 2026-07-06 |
| **Thang Story Point** | Fibonacci (1, 2, 3, 5, 8) |
| **Ước lượng velocity** | ~15–18 SP / sprint 2 tuần, solo dev (hiệu chỉnh sau Sprint 1) |

> Acceptance Criteria chi tiết (Given-When-Then): xem `03-acceptance-criteria.md`.

---

## Tổng quan Epic

| Epic | Tên | Mục tiêu | Tổng SP |
|---|---|---|---|
| **E0** | Platform & Foundation | Docker Compose, 3 service skeleton, DB, CI/CD, spike rủi ro, bộ eval | 29 |
| **E1** | Auth & Account | Đăng ký/đăng nhập Google + email/pass, JWT dùng chung Java/Golang · xem hồ sơ | 14 |
| **E2** | Ingestion (session-scoped) | Thêm nguồn vào phiên: Golang → S3 → Java → extract → contextual embed → index | 30 |
| **E3** | Sources Panel | Panel phải: nguồn, bật/tắt, xóa cascade, tổng quan, token realtime | 14 |
| **E4** | RAG Chat | Hỏi đáp grounded, trích dẫn, từ chối, đa phiên, song ngữ | 25 |
| | | **Tổng** | **112** |

**Must = 89 SP** (~5,5 sprint) · Must + Should = 106 SP (~7 sprint). Đề xuất: **v1.0 = toàn bộ Must**, các Should phát hành ở sprint kế tiếp.

---

## E0 — Platform & Foundation

| ID | User Story | MoSCoW | SP |
|---|---|---|---|
| US-0.1 | As a developer, I want môi trường dev Docker Compose (Golang + Spring Boot + FastAPI + PostgreSQL + Qdrant + MinIO), so that toàn bộ stack khởi động bằng một lệnh. | Must | 5 |
| US-0.2 | As a developer, I want Extraction Service (FastAPI) với endpoint `POST /extract` bọc MarkItDown (nhận file từ S3 hoặc URL), so that Core Backend gửi nguồn và nhận về Markdown + metadata. | Must | 5 |
| US-0.3 | As a developer, I want schema PostgreSQL (kèm migration, `ON DELETE CASCADE` theo chuỗi Session → Source → Chunk/Message) + Qdrant collection với payload index (`userId`, `sessionChatId`, `sourceId`), so that pipeline lưu data và truy vấn vector theo phiên được ngay. | Must | 3 |
| US-0.4 | As a developer, I want CI/CD GitHub Actions (build → test → Docker images cho cả 3 service → deploy VPS, Nginx + TLS), so that merge vào `main` là deploy tự động. | Must | 5 |
| US-0.5 | **[SPIKE]** As a developer, I want kiểm chứng MarkItDown với tài liệu tiếng Việt (PDF văn bản/scan, DOCX, XLSX, PPTX, ảnh OCR, audio), so that quyết định Go / Go-with-limits / No-Go trước khi build pipeline — kết luận ghi thành ADR. | Must | 3 |
| US-0.6 | As a developer, I want bộ test vàng song ngữ **4 nhóm** (có đáp án / không có đáp án / mơ hồ / các đoạn mâu thuẫn cùng-và-khác nguồn) kèm case đo lệch ngôn ngữ Việt↔Anh + script eval tự động (chạy được cả chế độ có/không contextual embedding), so that mọi thay đổi đo được bằng số. | Must | 3 |
| US-0.7 | As a developer, I want Upload Service (Golang): nhận multipart file kèm JWT + `sessionChatId` → validate → stream lên S3 → callback API nội bộ sang Java (xác thực service-to-service), so that upload tách khỏi Core và Java chỉ lo logic. | Must | 5 |

## E1 — Auth & Account

| ID | User Story | MoSCoW | SP |
|---|---|---|---|
| US-1.1 | As a new user, I want đăng ký bằng email/mật khẩu và xác minh email, so that tôi có tài khoản an toàn. | Must | 5 |
| US-1.2 | As a user, I want đăng nhập/đăng xuất bằng JWT có thời hạn — token được cả Java lẫn Golang xác thực chung một cơ chế, so that phiên của tôi an toàn trên mọi service. | Must | 3 |
| US-1.3 | As a user, I want đăng nhập bằng Google OAuth, so that tôi vào app không cần tạo mật khẩu. | Must | 3 |
| US-1.4 | As a user, I want luồng quên mật khẩu qua email, so that tôi lấy lại quyền truy cập khi quên. | Should | 3 |

## E2 — Ingestion (session-scoped)

| ID | User Story | MoSCoW | SP |
|---|---|---|---|
| US-2.1 | As a user, I want thêm file vào phiên ngay trong lúc chat qua panel phải (validate định dạng + size ≤ 50MB tại Golang trước khi lên S3), so that tôi bổ sung nguồn không rời hội thoại. | Must | 5 |
| US-2.2 | As a user, I want file được xử lý bất đồng bộ: Java nhận callback → tải từ S3 → extract → chunk → **contextual embedding** (Claude sinh context + prompt caching) → lưu chunk vào PostgreSQL + vector vào Qdrant theo `sessionChatId`, so that nguồn sẵn sàng để hỏi với độ chính xác truy xuất cao. | Must | 8 |
| US-2.3 | As a user, I want trạng thái từng nguồn (PROCESSING/READY/FAILED kèm lý do dễ hiểu) cập nhật realtime trong panel, so that tôi biết khi nào hỏi được và vì sao lỗi. | Must | 3 |
| US-2.4 | As a user, I want dán URL trang web vào panel để nạp nội dung (snapshot, chống SSRF), so that tôi hỏi đáp trên nội dung web không cần file. | Must | 5 |
| US-2.5 | As a user, I want dán YouTube URL để nạp transcript video, so that tôi hỏi đáp trên nội dung video. | Should | 3 |
| US-2.6 | As a user, I want upload ZIP và hệ thống tự tách từng file bên trong thành nguồn riêng của phiên, so that tôi nạp cả bộ tài liệu một lần. | Could | 3 |
| US-2.7 | As a user, I want upload audio và hệ thống transcribe thành văn bản, so that tôi hỏi đáp trên ghi âm cuộc họp/bài giảng. | Could | 3 |

## E3 — Sources Panel

| ID | User Story | MoSCoW | SP |
|---|---|---|---|
| US-3.1 | As a user, I want panel phải hiển thị danh sách nguồn của phiên (tên, loại, trạng thái) và xóa được nguồn (cascade: S3 + Markdown + chunk/embedding, có xác nhận), so that tôi quản lý notebook của mình. | Must | 5 |
| US-3.2 | As a user, I want xem preview Markdown đã trích xuất của từng nguồn, so that tôi kiểm tra hệ thống "đọc" đúng chưa. | Should | 2 |
| US-3.3 | As a user, I want bật/tắt từng nguồn khỏi phạm vi truy xuất (hiệu lực ngay câu hỏi kế tiếp), so that câu trả lời tập trung đúng nguồn tôi muốn. | Should | 2 |
| US-3.4 | As a user, I want khu tổng quan: số nguồn, tổng dung lượng, tổng chunk đã index, ngôn ngữ phát hiện, so that tôi nắm nhanh notebook có gì. | Should | 2 |
| US-3.5 | As a user, I want thấy token in/out + thời gian phản hồi của request gần nhất hiển thị realtime, cùng tổng token cả phiên, so that tôi hiểu mức tiêu thụ của mình. | Must | 3 |

## E4 — RAG Chat

| ID | User Story | MoSCoW | SP |
|---|---|---|---|
| US-4.1 | As a user, I want hỏi bằng ngôn ngữ tự nhiên và nhận câu trả lời xây dựng CHỈ từ nguồn đang bật của phiên, kèm trích dẫn click được (nhảy tới đoạn gốc trong panel), so that tôi tin và kiểm chứng được từng câu. | Must | 8 |
| US-4.2 | As a user, I want hệ thống trả lời chuẩn "Tôi không tìm thấy thông tin trong tài liệu của bạn" (kèm gợi ý) khi nguồn không có đáp án hoặc tôi hỏi kiến thức chung, so that tôi không bao giờ bị lừa bởi câu trả lời bịa. | Must | 3 |
| US-4.3 | As a user, I want hỏi tiếp trong cùng phiên và bot hiểu ngữ cảnh các lượt trước, so that hội thoại tự nhiên. | Must | 3 |
| US-4.4 | As a user, I want tạo/đổi tên/xóa phiên (xóa phiên = xóa toàn bộ nguồn, có cảnh báo rõ) và xem lại lịch sử, so that tôi tách các chủ đề và kiểm soát dữ liệu. | Must | 3 |
| US-4.5 | As a user, I want câu trả lời streaming (first-token ≤ 3s), so that tôi không nhìn màn hình trống. | Should | 3 |
| US-4.6 | As a user, I want đánh giá câu trả lời 👍/👎, so that chất lượng được đo bằng phản hồi thật. | Should | 2 |
| US-4.7 | As a user, I want hỏi tiếng Việt trên nguồn tiếng Anh (và ngược lại) và nhận câu trả lời theo ngôn ngữ câu hỏi, so that rào cản ngôn ngữ không cản việc tra cứu. | Must | 3 |

---

## Phân bổ sprint đề xuất (để duyệt)

| Sprint | Trọng tâm | Story dự kiến | SP |
|---|---|---|---|
| **Sprint 0** | Nền tảng + khử rủi ro lớn nhất | US-0.1, US-0.2, US-0.5, US-0.3 | 16 |
| **Sprint 1** | Upload Service + Auth lõi | US-0.7, US-1.1, US-1.2 | 13 |
| **Sprint 2** | OAuth + luồng thêm nguồn E2E | US-1.3, US-2.1, US-2.2 (bắt đầu) | 16 |
| **Sprint 3** | Pipeline contextual hoàn chỉnh + panel | US-2.2 (hoàn tất), US-2.3, US-3.1 | 16 |
| **Sprint 4** | RAG core — trái tim sản phẩm | US-4.1, US-4.2, US-0.6, US-4.7 | 17 |
| **Sprint 5** | Chat hoàn thiện + URL + token panel | US-4.3, US-4.4, US-2.4, US-3.5 | 14 |
| **Sprint 6** | CI/CD + Should → **release v1** | US-0.4, US-4.5, US-3.3, US-3.4, US-3.2, US-4.6 | 16 |
| Sau v1 | Should/Could còn lại | US-1.4, US-2.5, US-2.6, US-2.7 | 12 |

> **Nguyên tắc xếp:** (1) rủi ro lớn nhất — MarkItDown tiếng Việt — khử ngay Sprint 0; (2) US-2.2 (contextual pipeline, 8 SP) là story phức tạp nhất, được trải qua 2 sprint có chủ đích; (3) RAG core có ngay khi pipeline chạy; (4) CI/CD hoàn chỉnh trước release. Velocity thực tế sau Sprint 1–2 sẽ quyết định có dồn/giãn sprint hay không.
