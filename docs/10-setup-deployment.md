# DocMate — Setup & Deployment Guide

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Tham chiếu** | `07-architecture.md` · US-0.1 (dev env) · US-0.4 (CI/CD — Sprint 6) |
| **Trạng thái từng phần** | Phần A (Dev): chuẩn theo US-0.1 · **Phần B (Prod): THIẾT KẾ DỰ KIẾN — chưa hiện thực, chốt lại khi làm US-0.4** |

---

# Phần A — Môi trường Dev (local)

## A1. Yêu cầu máy

| | Tối thiểu |
|---|---|
| Docker + Docker Compose | Docker Engine 24+, Compose v2 |
| RAM trống | ~6 GB cho 6 container (Extraction nặng nhất do dependency OCR/audio) |
| Ổ đĩa | ~10 GB (image + volume dữ liệu) |
| Khác | Không cần cài JDK/Go/Python trên host — mọi thứ trong container. Cài thêm nếu muốn chạy service ngoài Docker khi dev (xem A5) |

## A2. Cấu hình `.env`

Copy `.env.example` → `.env` tại root repo code. Nhóm biến:

| Nhóm | Biến | Ghi chú |
|---|---|---|
| PostgreSQL | `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | Dev đặt tùy ý |
| MinIO | `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` / `S3_BUCKET` | Dev dùng MinIO thay AWS S3 |
| Qdrant | (mặc định không cần auth trong mạng nội bộ dev) | Prod bật API key |
| JWT | `JWT_SECRET` (hoặc cặp khóa nếu RS256 — chốt ở US-1.2) | Java phát hành, Golang verify — cùng giá trị |
| S2S | `INTERNAL_TOKEN` | Shared secret Golang→Java (07 §2.3) |
| AI | `ANTHROPIC_API_KEY` / `VOYAGE_API_KEY` | **Chưa cần ở Sprint 0** — chỉ cần từ US-2.2 |

**Quy tắc:** `.env` nằm trong `.gitignore`; `.env.example` chứa tên biến + giá trị giả, không bao giờ chứa giá trị thật.

## A3. Khởi động & kiểm tra

```bash
docker compose up -d
docker compose ps          # kỳ vọng: 6/6 healthy (golang, java, python, postgres, qdrant, minio)
./scripts/smoke-test.sh    # kiểm tra kết nối chéo theo AC US-0.1
```

Smoke test tối thiểu phải xác nhận:

- [ ] Golang → Java: gọi được API nội bộ (S2S token đúng)
- [ ] Java → Python: `POST /extract` trả lời
- [ ] Java → PostgreSQL + Qdrant: kết nối + collection tồn tại
- [ ] Golang → MinIO: put/get object được
- [ ] Chỉ Nginx trả lời từ ngoài mạng Docker; gọi thẳng port service khác từ host → không được

## A4. Chạy test & migration

| Việc | Lệnh (chốt cụ thể khi hiện thực US-0.1/0.3) |
|---|---|
| Migration Postgres | Flyway chạy tự động khi Java khởi động; chạy tay: `docker compose exec java ./gradlew flywayMigrate` |
| Test Java | `docker compose exec java ./gradlew test` |
| Test Golang | `docker compose exec golang go test ./...` |
| Test Python | `docker compose exec python pytest` |

> Trước khi US-0.4 Done, "CI green" trong DoD được thay bằng: cả 3 bộ test trên pass bằng lệnh local (`04-dor-dod.md` §2.1).

## A5. Dev nhanh một service (tùy chọn)

Khi sửa một service liên tục, chạy service đó ngoài Docker và trỏ vào phần còn lại của stack:

```bash
docker compose up -d --scale java=0   # tắt riêng java trong stack
# chạy java trên host với .env trỏ postgres/qdrant/minio về localhost:cổng-mapped
```

Chỉ dùng khi cần vòng lặp nhanh; **DoD vẫn kiểm bằng full stack trong compose** ("chạy được từ sạch").

## A6. Reset dữ liệu dev

```bash
docker compose down -v     # xóa volume: Postgres + Qdrant + MinIO về trắng
docker compose up -d       # migration + tạo collection chạy lại từ đầu
```

---

# Phần B — Prod trên VPS ⚠️ THIẾT KẾ DỰ KIẾN

> **Chưa hiện thực.** Phần này là thiết kế để US-0.4 (Sprint 6) làm theo; mọi chi tiết dưới đây được phép chốt lại khi hiện thực. Đánh dấu ⚠️ ở các điểm chắc chắn cần quyết định thêm.

## B1. Hình dạng hệ thống prod

- **1 VPS**, toàn bộ stack chạy Docker Compose (file compose prod riêng: `docker-compose.prod.yml`).
- **Nginx** trên VPS: TLS (Let's Encrypt/certbot), reverse proxy 2 đường công khai — frontend/API Java và endpoint upload Golang. Mọi container khác không map port ra ngoài.
- Khác biệt so với dev:

| | Dev | Prod |
|---|---|---|
| S3 | MinIO container | ⚠️ Chọn khi làm US-0.4: AWS S3 hoặc tiếp tục MinIO trên VPS (PRD cho phép cả hai) |
| Qdrant | không auth | Bật API key, chỉ mạng nội bộ |
| Secret | `.env` file | GitHub Actions Secrets → truyền vào VPS khi deploy (không lưu bản rõ trong repo); trên VPS: file env chỉ root đọc được |
| Image | build local | Pull từ registry (GHCR) theo tag |

## B2. Luồng deploy (theo AC US-0.4)

```
merge vào main
  → GitHub Actions: build + test cả 3 service
  → build 3 Docker image, push GHCR (tag = commit SHA)
  → SSH vào VPS: cập nhật tag trong compose prod → docker compose pull → up -d
  → healthcheck sau deploy (script gọi endpoint health qua Nginx)
  → fail ở bất kỳ bước nào → DỪNG, bản cũ vẫn chạy (container cũ chưa bị thay)
```

**Rollback thủ công:** deploy = đổi tag image, nên rollback = đặt lại tag SHA cũ → `docker compose up -d`. Giữ tối thiểu 3 tag gần nhất trên GHCR.

## B3. Migration & dữ liệu trên prod

- Flyway chạy khi container Java mới khởi động — migration phải **backward-compatible trong một bản deploy** (bản cũ vẫn chạy được trên schema mới) để rollback tag không kẹt schema. ⚠️ Quy tắc viết migration chi tiết chốt khi làm US-0.4.
- **Backup:** chỉ cần nghiêm túc với **PostgreSQL** (pg_dump định kỳ + giữ off-VPS) và **S3/MinIO data**; Qdrant tái tạo được bằng re-index từ Postgres (ADR-001 §4). ⚠️ Tần suất + nơi lưu backup chốt khi làm US-0.4.

## B4. Việc phải làm khi hiện thực US-0.4 (checklist chuyển giao)

- [ ] Chọn S3 prod: AWS hay MinIO-trên-VPS (B1)
- [ ] Viết `docker-compose.prod.yml` + file env prod trên VPS
- [ ] Workflow GitHub Actions theo B2, kèm healthcheck sau deploy
- [ ] Nginx + certbot TLS, rate limiting kỹ thuật (NFR §8)
- [ ] Quy tắc migration backward-compatible + kịch bản rollback thử thật một lần
- [ ] Backup Postgres + S3 chạy định kỳ, verify restore được một lần
- [ ] Cập nhật tài liệu này: gỡ nhãn "THIẾT KẾ DỰ KIẾN", bump version, commit `docs(setup): ...`

## Changelog

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 2026-07-06 | Bản đầu — Phần A chuẩn theo US-0.1; Phần B là thiết kế dự kiến cho US-0.4 |
