# DocMate — Traceability Matrix

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Mục đích** | Khép vòng story → AC → test → tài liệu kỹ thuật: nhìn một hàng biết story được đặc tả ở đâu, kiểm bằng gì, thiết kế nằm đâu. Cột Test trống với story Must = chặn release (Gate 1.5 của `14-release-checklist.md`) |
| **Quy ước cập nhật** | Thêm/sửa story, AC, hay test case ⇒ cập nhật matrix trong cùng chuỗi commit |

> AC của mọi story: `03-acceptance-criteria.md` (mục cùng tên story). Test case: `12-test-cases.md`. Cột **Docs kỹ thuật** chỉ liệt kê mục *đặc thù* của story — mọi story mặc định chịu `04-dor-dod.md` và kiến trúc chung `07`.

## E0 — Platform & Foundation

| Story | MoSCoW | SP | Sprint | Test | Docs kỹ thuật đặc thù |
|---|---|---|---|---|---|
| US-0.1 Docker Compose | Must | 5 | 0 | TC-0.1-01..03 | `10` Phần A · `05` §2.1 |
| US-0.2 `/extract` | Must | 5 | 0 | TC-0.2-01..04 | `07` §2.2 · `05` §2.2 |
| US-0.3 Schema PG + Qdrant | Must | 3 | 0 | TC-0.3-01..04 [MAND×3] | `08` toàn bộ · `09` ADR-001 · `05` §2.4 |
| US-0.4 CI/CD | Must | 5 | 6 | TC-0.4-01..03 | `10` Phần B (+ checklist B4) |
| US-0.5 [SPIKE] MarkItDown VI | Must | 3 | 0 | Đầu ra = báo cáo + **ADR-002** | `05` §2.3 (kịch bản theo kết quả) |
| US-0.6 Bộ test vàng + harness | Must | 3 | 4 | Nghiệm thu = checklist `13` §6 | `13` toàn bộ · `11` §2 (tầng AI eval) |
| US-0.7 Upload Service | Must | 5 | 1 | TC-0.7-01..06 | `07` §2.1 (callback + idempotency 409), §3 |

## E1 — Auth & Account

| Story | MoSCoW | SP | Sprint | Test | Docs kỹ thuật đặc thù |
|---|---|---|---|---|---|
| US-1.1 Đăng ký email/pass | Must | 5 | 1 | TC-1.1-01..04 | `08` bảng USERS |
| US-1.2 JWT chung Java/Golang | Must | 3 | 1 | TC-1.2-01..04 | `07` §2.3 (⚠️ HS256/RS256 chốt khi làm) |
| US-1.3 OAuth Google | Must | 3 | 2 | TC-1.3-01..02 | `08` (`auth_provider`) |
| US-1.4 Quên mật khẩu | Should | 3 | sau v1 | TC-1.4-01..03 | — |
| US-1.5 Xóa tài khoản | Should | 3 | sau v1 | TC-1.5-01 [MAND]..02 | `07` §5.1 (xóa 3 nơi theo `userId`) |

## E2 — Ingestion

| Story | MoSCoW | SP | Sprint | Test | Docs kỹ thuật đặc thù |
|---|---|---|---|---|---|
| US-2.1 Thêm file vào phiên | Must | 5 | 2 | TC-2.1-01..03 | `07` §3 |
| US-2.2 Pipeline contextual | Must | 8 | 2–3 | TC-2.2-01..07 [MAND: 05] | `07` §3, §5.1 · `08` bảng CHUNKS/INGEST_JOBS · PRD §6.2 |
| US-2.3 Trạng thái realtime | Must | 3 | 3 | TC-2.3-01..03 | `07` §2.2 (map errorCode → lý do) |
| US-2.4 URL + chống SSRF | Must | 5 | 5 | TC-2.4-01..05 [MAND: 02] | `07` §1 (fetcher tại Python) · `11` §5 |
| US-2.5 YouTube | Should | 3 | sau v1 | TC-2.5-01..02 | — |
| US-2.6 ZIP | Could | 3 | sau v1 | TC-2.6-01 | — |
| US-2.7 Audio | Could | 3 | sau v1 | TC-2.7-01 | Phụ thuộc kết luận ADR-002 |

## E3 — Sources Panel

| Story | MoSCoW | SP | Sprint | Test | Docs kỹ thuật đặc thù |
|---|---|---|---|---|---|
| US-3.1 Danh sách + xóa cascade | Must | 5 | 3 | TC-3.1-01..03 [MAND: 02 — "xóa rồi hỏi lại"] | `07` §5.1 · `08` §2, §5 |
| US-3.2 Preview Markdown | Should | 2 | 6 | TC-3.2-01 | `08` (`markdown_s3_key`) |
| US-3.3 Bật/tắt nguồn | Should | 2 | 6 | TC-3.3-01 [MAND] | `07` §4 (filter enabled mỗi câu hỏi) |
| US-3.4 Tổng quan phiên | Should | 2 | 6 | TC-3.4-01 | — |
| US-3.5 Token realtime | Must | 3 | 5 | TC-3.5-01..02 | `08` (`token_usage` jsonb, `total_tokens`) |

## E4 — RAG Chat

| Story | MoSCoW | SP | Sprint | Test | Docs kỹ thuật đặc thù |
|---|---|---|---|---|---|
| US-4.1 Grounded + trích dẫn | Must | 8 | 4 | TC-4.1-01..05 [MAND: 03,04,05] + bộ vàng N1/N4 | `07` §4 · PRD §6.1, §6.3 · `08` (`location_hint`, `citations`) |
| US-4.2 Từ chối thiếu dữ liệu | Must | 3 | 4 | TC-4.2-01..04 + bộ vàng N2 (≥95%) | PRD §6.3 (2 Gate) · `13` §2-N2 |
| US-4.3 Nhiều lượt | Must | 3 | 5 | TC-4.3-01 | `07` §4 (lịch sử phiên trong prompt) |
| US-4.4 Quản lý phiên | Must | 3 | 5 | TC-4.4-01..02 [MAND: 02] | `07` §5.1 (xóa theo `sessionChatId`) |
| US-4.5 Streaming | Should | 3 | 6 | TC-4.5-01..02 | `07` §4 (từ chối Gate 2 không stream nửa vời) |
| US-4.6 Đánh giá 👍/👎 | Should | 2 | 6 | TC-4.6-01 | `08` (`rating`) |
| US-4.7 Song ngữ VI↔EN | Must | 3 | 4 | TC-4.7-01 + bộ vàng case hai chiều | `13` §4 (đo lệch similarity) |

## Truy vết ngược — yêu cầu cứng NFR → nơi kiểm

| NFR (PRD §8) | Test/cơ chế | Docs |
|---|---|---|
| Cô lập dữ liệu user/phiên | TC-0.3-01 · TC-2.2-05 · TC-4.1-05 + checklist self-review query | `04` §2.2, §2.3 |
| Xóa triệt để cascade 3 nơi | TC-3.1-02 · TC-4.4-02 · TC-1.5-01 | `07` §5.1 · `08` §2, §5 |
| Nhất quán dual-write | TC-0.3-02 · TC-2.2-07 + job đối soát | `09` §4 · `07` §5.2 |
| Không bịa (≥95% từ chối đúng) | Bộ vàng N2 — Gate 3.2 release | `13` · `14` Gate 3 |
| Chống SSRF | TC-2.4-02..05 | `11` §5 |
| Hiệu năng (first-token ≤3s, p95 ≤10s) | Metric khi chạy E2E + eval, đối chiếu NFR | `11` §2 (ghi chú) |
| Minh bạch token | TC-3.5-01..02 | `08` (`token_usage`) |

## Changelog

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 2026-07-06 | Bản đầu — 31 story, cột Sprint theo phân bổ đề xuất backlog v1.3 |
