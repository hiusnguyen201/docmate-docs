# DocMate — Release Checklist v1.0

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Phạm vi release** | v1.0 = toàn bộ story **Must** (89 SP) theo `02-product-backlog.md`; nhóm Should phát hành sprint kế tiếp |
| **Tham chiếu** | `04-dor-dod.md` · `11-test-strategy.md` · `13-golden-test-set.md` · `10-setup-deployment.md` |
| **Cách dùng** | Copy thành `docs/releases/release-v1.0-checklist.md` khi vào sprint release (dự kiến Sprint 6), tick + điền bằng chứng, commit `docs(release): ...`. Mọi mục PHẢI có cột bằng chứng — không tick chay |

---

## Gate 1 — Phạm vi & tài liệu

| # | Mục | Bằng chứng |
|---|---|---|
| 1.1 | ☐ 100% story Must (89 SP) Done theo DoD — không có Must nào carry-over | Biên bản Review các sprint |
| 1.2 | ☐ Story Should/Could chưa làm vẫn nằm nguyên trong backlog (không lặng lẽ biến mất) | Đối chiếu `02-product-backlog.md` |
| 1.3 | ☐ PRD/AC/tài liệu kỹ thuật khớp hệ thống thật — rà các quyết định đổi giữa chừng đã cập nhật tài liệu | Lịch sử commit `docs(...)` |
| 1.4 | ☐ ADR-002 (MarkItDown) đã chốt; nếu Go-with-limits thì giới hạn định dạng đã phản ánh vào PRD §3.3 + UI | `docs/` + màn hình upload |
| 1.5 | ☐ Traceability matrix (`15-traceability-matrix.md`) không còn ô trống ở cột test với story Must | Bản matrix đã cập nhật |

## Gate 2 — Chất lượng: test kỹ thuật

| # | Mục | Bằng chứng |
|---|---|---|
| 2.1 | ☐ Toàn bộ unit + integration + contract test pass trên CI | Link run GitHub Actions |
| 2.2 | ☐ **Mandatory suite** pass đủ 14 case [MAND] (`12-test-cases.md`) trên stack sạch | Log run |
| 2.3 | ☐ Test đinh "xóa rồi hỏi lại" pass ở cả 3 mức: Source, Session, Account (TC-3.1-02, TC-4.4-02, TC-1.5-01*) | Log run |
| 2.4 | ☐ Suite SSRF (TC-2.4-02..05) pass | Log run |
| 2.5 | ☐ E2E luồng chính pass: đăng ký → tạo phiên → upload → READY → hỏi → trích dẫn click được → bật/tắt nguồn → xóa | Log/quay màn hình |

> \* TC-1.5-01 thuộc US-1.5 (Should) — nếu v1.0 chỉ gồm Must, mức Account kiểm bằng script thủ công tương đương và ghi chú lại.

## Gate 3 — Chất lượng: AI (bộ test vàng)

| # | Mục | Bằng chứng |
|---|---|---|
| 3.1 | ☐ Eval full run trên đúng config sẽ release (model, ngưỡng, top-K đóng băng) | Báo cáo `docs/eval-runs/` |
| 3.2 | ☐ N2 từ chối đúng **≥ 95%** (chỉ tiêu quan trọng nhất) | Báo cáo |
| 3.3 | ☐ N1 trả lời đúng + nguồn đúng **≥ 90%**, đo riêng hai chiều VI↔EN không chiều nào dưới ngưỡng bất thường | Báo cáo |
| 3.4 | ☐ N4 mâu thuẫn: kết quả nêu-đủ-các-bên được ghi nhận làm baseline (chưa ép ngưỡng cứng ở v1) | Báo cáo |
| 3.5 | ☐ Config release được snapshot (`eval run` lưu kèm) — tái lập được kết quả | File config trong eval run |

## Gate 4 — Bảo mật & dữ liệu

| # | Mục | Bằng chứng |
|---|---|---|
| 4.1 | ☐ Quét secret toàn repo + git history sạch (không API key/mật khẩu từng bị commit) | Output tool quét (vd gitleaks) |
| 4.2 | ☐ Test cô lập dữ liệu pass (TC-0.3-01, TC-2.2-05, TC-4.1-05) | Log run |
| 4.3 | ☐ Rà tay: mọi endpoint public đều sau JWT; endpoint nội bộ đều sau S2S token; Qdrant/PG/MinIO không expose ra ngoài | Checklist rà + `docker compose ps` prod |
| 4.4 | ☐ Rate limiting kỹ thuật bật trên Nginx | Config + test 429 |
| 4.5 | ☐ TLS hoạt động, HTTP redirect HTTPS | curl kiểm |

## Gate 5 — Vận hành

| # | Mục | Bằng chứng |
|---|---|---|
| 5.1 | ☐ Deploy qua pipeline US-0.4 thành công lên VPS prod | Run Actions + healthcheck |
| 5.2 | ☐ **Rollback đã diễn tập thật một lần** trên prod (đặt lại tag cũ) | Ghi chú giờ diễn tập + kết quả |
| 5.3 | ☐ Backup PostgreSQL + S3 chạy định kỳ và **đã verify restore một lần** | Ghi chú diễn tập restore |
| 5.4 | ☐ Job đối soát Qdrant (dọn vector mồ côi) được lên lịch và đã chạy thật ít nhất một lần | Log job |
| 5.5 | ☐ Logging + metric + cảnh báo cơ bản hoạt động (tỉ lệ lỗi, latency, token usage) | Ảnh dashboard/log |
| 5.6 | ☐ `10-setup-deployment.md` Phần B đã gỡ nhãn "THIẾT KẾ DỰ KIẾN", cập nhật theo thực tế | Commit `docs(setup)` |

## Gate 6 — Trải nghiệm & định vị

| # | Mục | Bằng chứng |
|---|---|---|
| 6.1 | ☐ Onboarding/empty-state nói rõ định vị "chỉ trả lời từ nguồn của bạn" (giảm sốc "không tìm thấy" — rủi ro PRD §10) | Ảnh màn hình |
| 6.2 | ☐ Thông điệp từ chối chuẩn kèm gợi ý hành động hiển thị đúng mọi ngữ cảnh (Gate 1, Gate 2, phiên trống) | Ảnh màn hình |
| 6.3 | ☐ Token panel hiển thị realtime đúng (TC-3.5-01/02) | Log run |
| 6.4 | ☐ Đi thử toàn bộ luồng bằng con mắt người dùng mới trên prod, ghi lại điều gây khó chịu → triage: fix trước release / ghi backlog | Danh sách triage |

## Quyết định release

| | |
|---|---|
| **Go / No-Go** | ☐ Go ☐ No-Go — ngày: … |
| Mục chưa đạt được chấp nhận (nếu Go có điều kiện) | (liệt kê + lý do + kế hoạch vá) |
| Tag release | `v1.0.0` = commit SHA … |
| Baseline eval đính kèm | link báo cáo |

## Changelog

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 2026-07-06 | Bản đầu — template cho sprint release |
