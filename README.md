# DocMate — Tài liệu dự án

DocMate là web app SaaS: mỗi phiên chat là một notebook — vừa hỏi đáp AI vừa thêm nguồn (file/URL), câu trả lời chỉ dựa trên nguồn đã nạp, luôn kèm trích dẫn, không bịa.

## Cấu trúc

| File | Nội dung |
|---|---|
| `PRD-docmate-v1.3.md` | Product Requirements Document (bản mới nhất) |
| `docs/01-product-vision.md` | Product Vision — Nhóm 1 |
| `docs/02-product-backlog.md` | Product Backlog: Epic → Story → MoSCoW → SP — Nhóm 1 |
| `docs/03-acceptance-criteria.md` | Acceptance Criteria (Given-When-Then) — Nhóm 1 |

## Tech stack (đã chốt)

Golang (upload → S3) · Java 21 Spring Boot (core, RAG) · Python FastAPI + MarkItDown (extraction) · PostgreSQL (data) + Qdrant (vector) · Claude API + Voyage embeddings · React + TypeScript · VPS + Docker Compose + Nginx + GitHub Actions

## Quy ước

- Tài liệu tiếng Việt, thuật ngữ kỹ thuật giữ tiếng Anh; sơ đồ dùng Mermaid.
- Mỗi thay đổi tài liệu = 1 commit, message theo dạng `docs(scope): nội dung`.
