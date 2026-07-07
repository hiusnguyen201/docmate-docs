# DocMate — Test Cases

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Tham chiếu** | `03-acceptance-criteria.md` v1.3 (nguồn AC) · `11-test-strategy.md` |
| **Quy ước ID** | `TC-<story>-<số>` — ví dụ TC-4.1-03. Cột **Tầng**: U=Unit, I=Integration, C=Contract, E=E2E, M=Manual-checklist, G=Golden-set |

> Mỗi test case ánh xạ 1-1 hoặc n-1 với scenario trong AC. Case đánh dấu **[MAND]** thuộc mandatory suite (11 §3) — điều kiện DoD.

---

## E0 — Platform & Foundation

### US-0.1 — Docker Compose
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-0.1-01 | E | Máy sạch chỉ có Docker → clone → `docker compose up -d` → `docker compose ps` | 6/6 container healthy |
| TC-0.1-02 | E | Chạy `scripts/smoke-test.sh` | Pass cả 5 kết nối chéo: Go→Java, Java→Py, Java→PG, Java→Qdrant, Go→MinIO |
| TC-0.1-03 | E | Từ host, gọi thẳng port Java/Golang/Python/PG/Qdrant/MinIO (không qua Nginx) | Từ chối kết nối — chỉ Nginx trả lời |

### US-0.2 — `/extract`
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-0.2-01 | I | Put PDF hợp lệ lên MinIO → `POST /extract {s3Key}` | 200; body có `markdown` khác rỗng + `metadata.format=pdf`, `pages`, `processingMs` |
| TC-0.2-02 | I | `/extract` với file `.xyz` không hỗ trợ | 4xx, `errorCode=UNSUPPORTED_FORMAT` |
| TC-0.2-03 | I | `/extract` với PDF cắt cụt (hỏng) | 4xx, `errorCode=CORRUPTED_FILE` — không 500 |
| TC-0.2-04 | I | `/extract {s3Key}` không tồn tại trên S3 | 4xx `errorCode=FETCH_FAILED`, detail rõ |

### US-0.3 — Schema PG + Qdrant
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-0.3-01 **[MAND]** | I | Seed 2 user × 2 phiên × chunks+points → search vector với filter `userId=A, sessionChatId=X` | Kết quả đúng thứ tự similarity; 100% point thuộc đúng A/X |
| TC-0.3-02 **[MAND]** | I | Xóa 1 chunk khỏi PG (point còn trên Qdrant) → search trúng point đó → join PG theo `chunkId` | Point mồ côi bị loại khỏi kết quả cuối |
| TC-0.3-03 **[MAND]** | I | Xóa 1 Source trong PG → kiểm PG + chạy job xóa Qdrant | Chunks/messages liên quan biến mất (cascade); Qdrant không còn point có `payload.sourceId` đó |
| TC-0.3-04 | I | Chạy migration trên DB trắng 2 lần liên tiếp | Lần 2 no-op, không lỗi (idempotent) |

### US-0.4 — CI/CD *(hiện thực ở Sprint 6)*
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-0.4-01 | E | Merge PR có test fail cố ý vào `main` | Workflow dừng ở bước test; VPS không đổi, bản cũ vẫn chạy |
| TC-0.4-02 | E | Merge PR hợp lệ | 3 image build+push; VPS pull tag mới; healthcheck sau deploy pass |
| TC-0.4-03 | M | Rollback: đặt lại tag SHA trước → up -d | Bản cũ chạy lại, healthcheck pass |

### US-0.5 — SPIKE MarkItDown *(đầu ra là báo cáo, không phải test case — xem Sprint 0 plan §2.3)*

### US-0.6 — Bộ test vàng *(test case = chính eval harness — đặc tả tại `13-golden-test-set.md`; nghiệm thu US-0.6 bằng checklist 13 §6)*

### US-0.7 — Upload Service
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-0.7-01 | E | Upload PDF 10MB hợp lệ, JWT đúng + `sessionChatId` đúng | File lên S3 (stream — RSS Golang không tăng ~kích thước file); callback Java thành công; client nhận `sourceId` |
| TC-0.7-02 | I | Upload với JWT sai / hết hạn | 401; **không** object mới nào trên S3 |
| TC-0.7-03 | I | Upload file 51MB / file `.exe` | Từ chối ngay với mã lỗi rõ; không gì lên S3 |
| TC-0.7-04 | I | Giả lập Java chết → upload → Golang retry hết N lần | Client nhận lỗi; object S3 bị dọn hoặc đánh dấu orphan (kiểm bằng job dọn) |
| TC-0.7-05 | C | Callback lặp cùng `s3Key` (giả lập retry sau timeout mà lần đầu đã thành công) | Lần 2 nhận 409; PG chỉ có 1 Source |
| TC-0.7-06 | C | Callback thiếu `X-Internal-Token` / token sai | 401; không tạo Source |

---

## E1 — Auth & Account

### US-1.1 — Đăng ký email/mật khẩu
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-1.1-01 | I | Đăng ký email mới, mật khẩu ≥ 8 ký tự | User tạo `email_verified=false`; email xác minh gửi; hash trong DB là bcrypt/argon2 (không plaintext, không MD5/SHA thuần) |
| TC-1.1-02 | E | Click link xác minh còn hạn → đăng nhập | Kích hoạt, đăng nhập OK; link dùng lại lần 2 → vô hiệu |
| TC-1.1-03 | I | Đăng ký lại email đã tồn tại (kể cả khác hoa thường) | Lỗi rõ; không tạo bản ghi trùng (citext) |
| TC-1.1-04 | U | Mật khẩu 7 ký tự | Validate chặn |

### US-1.2 — JWT dùng chung
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-1.2-01 | I | Đăng nhập đúng → dùng JWT gọi API Java (list phiên) và API Golang (upload) | Cả hai 2xx — cùng token |
| TC-1.2-02 | I | Sai mật khẩu / email không tồn tại | Cùng một thông điệp lỗi chung — không phân biệt được email có tồn tại |
| TC-1.2-03 | I | JWT hết hạn (phát hành token exp quá khứ trong test) gọi Java và Golang | Cả hai 401 |
| TC-1.2-04 | U | JWT chữ ký sai / thuật toán `none` | Verify từ chối ở cả hai service |

### US-1.3 — OAuth Google
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-1.3-01 | E(M) | Đăng nhập Google với email chưa có tài khoản | User mới `email_verified=true`, vào thẳng app |
| TC-1.3-02 | E(M) | Đăng nhập Google với email trùng tài khoản email/pass sẵn có | Linking tự động (`auth_provider=BOTH`) + thông báo; đăng nhập được bằng CẢ hai cách; không user thứ hai trong DB |

### US-1.4 — Quên mật khẩu
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-1.4-01 | I | Yêu cầu reset cho email tồn tại vs không tồn tại | UI/response giống hệt nhau; chỉ email tồn tại nhận mail |
| TC-1.4-02 | I | Dùng link reset: quá 30 phút / đã dùng 1 lần | Cả hai bị từ chối |
| TC-1.4-03 | E | Reset thành công → đăng nhập mật khẩu mới + thử mật khẩu cũ | Mới OK, cũ 401 |

### US-1.5 — Xóa tài khoản
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-1.5-01 **[MAND]** | E | User có 2 phiên, nguồn, hội thoại → xác nhận xóa (gõ lại email) | PG: users + toàn cascade sạch; Qdrant: 0 point `userId`; S3: 0 object của user; đăng nhập lại → 401 |
| TC-1.5-02 | I | Xác nhận gõ sai email | Không xóa gì |

---

## E2 — Ingestion

### US-2.1 — Thêm file vào phiên
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-2.1-01 | E | Đang chat, kéo-thả PDF hợp lệ vào panel | Source hiện ngay `PROCESSING`; khung chat vẫn thao tác được (không block) |
| TC-2.1-02 | M | Chọn file đổi đuôi (`.exe` → `.pdf`) | UI/Golang chặn theo MIME thật, không chỉ extension |
| TC-2.1-03 | U | File 50MB đúng hạn vs 50MB+1byte | Đúng hạn qua; quá hạn chặn kèm thông báo định dạng/giới hạn |

### US-2.2 — Pipeline contextual embedding
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-2.2-01 | E | Upload tài liệu mẫu ~10 trang → đợi READY → kiểm DB | Chunks có `original_text` + `contextualized_text` (khác nhau, contextual dài hơn) + `embedding_model`/`dimension`; Qdrant có point tương ứng đủ payload 4 trường; Source READY |
| TC-2.2-02 | I | Kiểm chunking trên fixture Markdown có heading | Chunk theo heading, ~500–1000 token, overlap 10–15%, `position` tăng dần, `location_hint` đúng heading |
| TC-2.2-03 | I | Giả lập Claude trả 429 hai lần đầu (mock) | Retry backoff, lần 3 thành công → READY; token contextual log vào IngestJob |
| TC-2.2-04 | I | Giả lập Voyage lỗi vĩnh viễn quá N lần | Source FAILED + `failure_code` phân loại; token đã tiêu vẫn được log |
| TC-2.2-05 **[MAND]** | I | Kiểm mọi chunk của Source thuộc phiên X | 100% `session_chat_id = X` (PG lẫn payload Qdrant) |
| TC-2.2-06 | E | Nguồn văn bản ~15–20MB, điều kiện thường | READY ≤ 2 phút; trong lúc chạy UI hiển thị % chunk đã xử lý |
| TC-2.2-07 | I | Giả lập crash giữa bước ghi PG xong, Qdrant chưa upsert | Chunk không xuất hiện trong search (thiếu vector = an toàn); job đối soát re-embed → sau đó search được |

### US-2.3 — Trạng thái realtime
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-2.3-01 | E | Upload → nhìn panel không F5 | `PROCESSING` → `READY` tự cập nhật |
| TC-2.3-02 | M | Upload PDF có mật khẩu / URL cần đăng nhập | Lý do FAILED bằng ngôn ngữ dễ hiểu, không stack trace |
| TC-2.3-03 | E | Nguồn vừa READY → hỏi câu có đáp án trong nguồn đó | Trả lời được kèm trích dẫn từ nguồn mới |

### US-2.4 — URL + chống SSRF
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-2.4-01 | E | Dán URL công khai (trang tài liệu HTML) | Snapshot qua pipeline → READY; Source ghi `origin_url` |
| TC-2.4-02 **[MAND]** | I | Lần lượt: `http://127.0.0.1`, `http://169.254.169.254/…`, `http://10.x.x.x`, hostname resolve về private IP, URL public redirect 302 về private | Tất cả bị chặn `SSRF_BLOCKED`; có log bảo mật |
| TC-2.4-03 | I | DNS rebinding (resolver test: lần 1 public, lần 2 private) | Kiểm sau resolve → chặn |
| TC-2.4-04 | I | URL trả content-type ngoài whitelist / body vượt giới hạn | Dừng, `errorCode` cụ thể |
| TC-2.4-05 | I | Link 404 vs link 401/403 (cần đăng nhập) | FAILED với 2 lý do phân biệt được |

### US-2.5 / 2.6 / 2.7 — YouTube · ZIP · Audio
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-2.5-01 | I | YouTube URL có transcript công khai | Source READY; metadata tiêu đề/kênh/link |
| TC-2.5-02 | I | Video không transcript / bị chặn | FAILED nêu đúng nguyên nhân |
| TC-2.6-01 | I | ZIP: 2 file hợp lệ + 1 `.exe` + 1 file hỏng | 2 Source riêng READY; danh sách bị bỏ qua hiển thị đủ 2 mục kèm lý do |
| TC-2.7-01 | I | Audio định dạng hỗ trợ (giọng Việt — theo kết luận spike) | Transcript thành Source; giới hạn (nếu có) hiển thị trong UI |

---

## E3 — Sources Panel

### US-3.1 — Danh sách + xóa cascade
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-3.1-01 | E | User có phiên X (2 nguồn) và Y (1 nguồn) → mở panel X | Chỉ thấy 2 nguồn của X, đủ tên/loại/trạng thái/thời điểm |
| TC-3.1-02 **[MAND]** | E | Hỏi câu trả lời được từ nguồn S → xóa S (xác nhận) → chạy job → hỏi lại đúng câu cũ | PG/Qdrant/S3 sạch dữ liệu S; câu hỏi lại → "không tìm thấy" (test đinh "xóa rồi hỏi lại") |
| TC-3.1-03 | I | Bấm xóa nhưng hủy ở bước xác nhận | Không gì bị xóa |

### US-3.2 — Preview Markdown
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-3.2-01 | E | Mở preview nguồn READY / PROCESSING / FAILED | READY render Markdown; hai trạng thái kia hiển thị trạng thái thay nội dung |

### US-3.3 — Bật/tắt nguồn
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-3.3-01 **[MAND]** | E | Phiên có A (bật) + B (tắt); hỏi nội dung chỉ có trong B → bật B → hỏi lại | Lần 1 "không tìm thấy"; lần 2 trả lời kèm trích dẫn B — hiệu lực ngay câu kế tiếp |

### US-3.4 — Tổng quan phiên
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-3.4-01 | E | Thêm rồi xóa nguồn, đối chiếu khu tổng quan | Số nguồn/dung lượng/chunk/ngôn ngữ đúng và cập nhật theo |

### US-3.5 — Token realtime
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-3.5-01 | E | Hỏi 1 câu → response xong | Panel hiện ngay token in/out (embed + LLM) + latency của request; tổng phiên tăng đúng |
| TC-3.5-02 | I | Đóng app, mở lại phiên → so tổng token với sum(`token_usage`) các Message | Khớp tuyệt đối |

---

## E4 — RAG Chat

### US-4.1 — Grounded + trích dẫn
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-4.1-01 | E+G | Hỏi câu có đáp án trong nguồn đang bật | Trả lời đúng; trích dẫn [nguồn+vị trí]; click citation → panel nhảy đúng đoạn (theo `location_hint`) |
| TC-4.1-02 | G | Câu trả lời nhiều luận điểm — đối chiếu thủ công trên bộ vàng | Mỗi luận điểm bám ≥ 1 chunk trích dẫn; không citation thừa |
| TC-4.1-03 **[MAND]** | G | 2 nguồn trong phiên mâu thuẫn (2 hợp đồng 2 mức giá) → hỏi giá | Nêu rõ không thống nhất + trích dẫn từng nguồn; không chọn một bên, không trộn số |
| TC-4.1-04 **[MAND]** | G | 2 đoạn cùng 1 nguồn mâu thuẫn (điều khoản chính vs phụ lục) | Nêu cả hai kèm vị trí; nếu contextualizedText cho biết ngữ cảnh → trình bày kèm |
| TC-4.1-05 **[MAND]** | I | User B hỏi nội dung chỉ có trong phiên user A; và phiên Y hỏi nội dung phiên X (cùng user) | Cả hai: "không tìm thấy" — không rò rỉ chéo |

### US-4.2 — Từ chối khi thiếu dữ liệu
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-4.2-01 | G | Hỏi câu không có đáp án trong nguồn (nhóm 2 bộ vàng) | Đúng thông điệp chuẩn + gợi ý; không suy diễn kèm |
| TC-4.2-02 | I | Ép Gate 2 (mock retrieval trả chunk lạc đề vượt ngưỡng) | Cùng thông điệp từ chối chuẩn |
| TC-4.2-03 | E | Hỏi small talk / kiến thức chung ("thủ đô Pháp?") | Lịch sự giải thích chỉ trả lời từ nguồn của phiên |
| TC-4.2-04 | E | Phiên 0 nguồn READY → hỏi | Hướng dẫn thêm nguồn ở panel phải |

### US-4.3 — Nhiều lượt
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-4.3-01 | E+G | Hỏi "chi phí dự án X" → hỏi tiếp "còn tiến độ thì sao?" | Câu 2 hiểu là tiến độ dự án X, truy xuất + trả lời đúng ngữ cảnh |

### US-4.4 — Quản lý phiên
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-4.4-01 | E | Tạo/đổi tên phiên; mở lại phiên cũ | Phản ánh ngay; lịch sử đủ tin nhắn + citation + token |
| TC-4.4-02 **[MAND]** | E | Xóa phiên có N nguồn | Cảnh báo nêu đúng N + "xóa vĩnh viễn"; xác nhận → PG cascade + Qdrant theo `sessionChatId` + S3 sạch |

### US-4.5 — Streaming
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-4.5-01 | E | Câu hỏi hợp lệ, đo first-token | Chữ hiện dần; first-token ≤ 3s (điều kiện thường) |
| TC-4.5-02 | E | Câu bị Gate 2 từ chối | Thông điệp từ chối hiển thị trọn vẹn, không stream nửa câu trả lời rồi rút |

### US-4.6 — 👍/👎
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-4.6-01 | I | Bấm 👎 rồi đổi thành 👍, rồi thử đổi lần nữa | Lưu vào Message; đổi được đúng một lần theo AC |

### US-4.7 — Song ngữ
| ID | Tầng | Bước | Kỳ vọng |
|---|---|---|---|
| TC-4.7-01 | G | Hỏi tiếng Việt trên nguồn tiếng Anh và ngược lại (case hai chiều bộ vàng) | Truy xuất đúng chunk; trả lời theo ngôn ngữ câu hỏi; số liệu hai chiều nằm trong báo cáo eval |

## Changelog

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 2026-07-06 | Bản đầu — phủ toàn bộ 26 story theo AC v1.3 |
