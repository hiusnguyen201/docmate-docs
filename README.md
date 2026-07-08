# DocMate — Tài liệu dự án

DocMate là web app SaaS: mỗi phiên chat là một notebook — vừa hỏi đáp AI vừa thêm nguồn (file/URL), câu trả lời chỉ dựa trên nguồn đã nạp, luôn kèm trích dẫn, không bịa.

## Bản đồ tài liệu

| Nhóm | File | Nội dung |
|---|---|---|
| **PRD** | `PRD-docmate-v1.3.md` | Product Requirements Document — nguồn sự thật cấp cao nhất |
| **1 — Sản phẩm** | `docs/01-product-vision.md` | Vision, goals, personas, nguyên tắc bất biến |
| | `docs/02-product-backlog.md` | Epic → Story → MoSCoW → SP (115 SP, Must 89) + phân bổ 7 sprint |
| | `docs/03-acceptance-criteria.md` | AC Given-When-Then cho toàn bộ story |
| **2 — Sprint** | `docs/04-dor-dod.md` | Definition of Ready & Done (hiệu chỉnh solo dev) |
| | `docs/05-sprint-0-plan.md` | Sprint Goal + Sprint Backlog Sprint 0 (16 SP) |
| | `docs/06-meeting-templates.md` | Template biên bản Planning / Review / Retro |
| **3 — Kỹ thuật** | `docs/07-architecture.md` | Kiến trúc, hợp đồng API nội bộ, luồng ingest/query, quy tắc dual-write |
| | `docs/08-erd.md` | ERD PostgreSQL + mapping Qdrant payload + index |
| | `docs/09-adr-001-postgres-qdrant.md` | ADR-001: PostgreSQL + Qdrant thay MongoDB |
| | `docs/10-setup-deployment.md` | Setup dev (chuẩn) + deploy prod (thiết kế dự kiến cho US-0.4) |
| **4 — Kiểm thử** | `docs/11-test-strategy.md` | Chiến lược test, các tầng, mandatory suite |
| | `docs/12-test-cases.md` | Test case chi tiết theo story (case [MAND] gắn DoD) |
| | `docs/13-golden-test-set.md` | Đặc tả bộ test vàng song ngữ 4 nhóm + eval harness |
| **5 — Bàn giao** | `docs/14-release-checklist.md` | Checklist Go/No-Go release v1.0 (6 gate) |
| | `docs/15-traceability-matrix.md` | Story → AC → Test → Docs, kèm truy vết NFR |
| **UC Specs** | `docs/specs/README.md` | Quy ước format UC + bản đồ 5 hộp (Auth, SessionChat, Ingestion, Sources, RagChat) |
| | `docs/specs/<Hộp>/UC-*.md` | 16 use case spec — hợp đồng hành vi để code theo (spec-driven) |

Thư mục sinh thêm khi vận hành: `docs/sprints/` (biên bản theo sprint), `docs/releases/` (checklist release đã điền), ADR mới đánh số tiếp (`ADR-002` = kết luận spike MarkItDown).

## Thứ tự đọc đề xuất

- **Người mới vào dự án:** `01-vision` → PRD → `07-architecture` → `08-erd` → `02-backlog`.
- **Bắt đầu một sprint:** `04-dor-dod` → `05-sprint-0-plan` (mẫu) → `06-meeting-templates`.
- **Trước khi code một story:** hàng tương ứng trong `15-traceability-matrix` → **UC spec trong `docs/specs/`** (tra hộp tại `specs/README`) → AC trong `03` → test case trong `12` → mục docs kỹ thuật đặc thù.
- **Trước release:** `14-release-checklist`.

## Tech stack (đã chốt)

Golang (upload → S3) · Java 21 Spring Boot (core, RAG) · Python FastAPI + MarkItDown (extraction) · PostgreSQL (data) + Qdrant (vector) · Claude API + Voyage embeddings · React + TypeScript · VPS + Docker Compose + Nginx + GitHub Actions

## Quy ước

- Tài liệu tiếng Việt, thuật ngữ kỹ thuật giữ tiếng Anh; sơ đồ dùng Mermaid.
- Mỗi thay đổi tài liệu = 1 commit, message theo dạng `docs(scope): nội dung`.
- Thay đổi quyết định kỹ thuật/hành vi ⇒ cập nhật tài liệu liên quan **trong cùng chuỗi commit** (điều kiện DoD) và cập nhật `15-traceability-matrix` nếu chạm story/test.
