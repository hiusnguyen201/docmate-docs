# DocMate — Golden Test Set (Bộ test vàng) — Đặc tả

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Hiện thực hóa** | US-0.6 (Sprint 4) |
| **Tham chiếu** | PRD §6.6 · AC US-0.6 · `11-test-strategy.md` §3, §5 |
| **Mục đích** | Biến "chất lượng AI" thành con số so sánh được giữa hai cấu hình bất kỳ — mọi thay đổi model/ngưỡng/chunking/contextual phải đi qua đây |

---

## 1. Tập tài liệu mẫu (corpus)

**Nguyên tắc:** cố định, version hóa, tự soạn hoặc dùng tài liệu có quyền dùng — không phụ thuộc nội dung web có thể đổi. Đủ nhỏ để ingest nhanh (~10–15 tài liệu), đủ đa dạng để câu hỏi có đất.

| # | Tài liệu (soạn khi làm US-0.6) | Ngôn ngữ | Vai trò trong bộ test |
|---|---|---|---|
| D1 | Hợp đồng dịch vụ mẫu (có điều khoản giá, thời hạn) | Việt | Nguồn chính nhóm 1; cặp mâu thuẫn với D2 |
| D2 | Phụ lục điều chỉnh hợp đồng D1 (đổi giá + thời hạn) | Việt | Mâu thuẫn **khác nguồn** với D1 |
| D3 | Quy trình nội bộ có phiên bản cũ/mới trong CÙNG file (điều khoản chính + phụ lục sửa đổi) | Việt | Mâu thuẫn **cùng nguồn** |
| D4 | Báo cáo kỹ thuật tiếng Anh (số liệu, bảng) | Anh | Case song ngữ Việt→Anh |
| D5 | Tài liệu hướng dẫn tiếng Việt (thuật ngữ kỹ thuật Anh xen kẽ) | Việt | Case song ngữ Anh→Việt + thuật ngữ trộn |
| D6 | Giáo trình/paper ngắn có cấu trúc heading sâu | Anh | Test chunking theo heading + citation vị trí |
| D7 | Bảng dữ liệu (từ XLSX → Markdown) | Việt | Câu hỏi tra số liệu bảng |
| D8+ | 3–5 tài liệu "nhiễu" không liên quan câu hỏi | Việt + Anh | Làm nền cho nhóm 2 (không có đáp án) và đo nhiễu retrieval |

Corpus nằm trong repo code `/test-fixtures/golden/corpus-v1/`, kèm file `manifest.json` (danh sách + checksum). **Đổi corpus = bump version bộ vàng** — kết quả giữa 2 version corpus không so sánh với nhau.

## 2. Bộ câu hỏi — 4 nhóm, ≥ 40 câu

| Nhóm | Số câu tối thiểu | Đo cái gì | Đáp án kỳ vọng ghi kèm |
|---|---|---|---|
| **N1 — Có đáp án** | 16 | % trả lời đúng + **nguồn đúng** | Đáp án chuẩn + `sourceId`/vị trí chunk kỳ vọng |
| **N2 — Không có đáp án** | 12 | % **từ chối đúng** (chỉ tiêu quan trọng nhất — mục tiêu ≥ 95%) | Kỳ vọng = thông điệp từ chối chuẩn, KHÔNG kèm suy diễn |
| **N3 — Mơ hồ** | 6 | Hành vi với câu thiếu ngữ cảnh ("giá bao nhiêu?" khi có nhiều loại giá) | Kỳ vọng = hỏi lại làm rõ HOẶC nêu các cách hiểu kèm trích dẫn — không đoán bừa một nghĩa |
| **N4 — Mâu thuẫn** | 8 (≥ 4 khác nguồn D1×D2, ≥ 4 cùng nguồn D3) | % **nêu-đủ-các-bên**: có mặt cả hai phía + trích dẫn từng vị trí, không phân xử, không trộn | Danh sách các "bên" phải xuất hiện + vị trí từng bên |

**Định dạng câu hỏi** (`questions-v1.jsonl`):

```json
{"id":"N1-05","group":1,"lang":"vi","question":"Thời hạn thanh toán trong hợp đồng là bao lâu?",
 "expected":{"type":"answer","keyFacts":["30 ngày"],"sources":["D1"],"locationHint":"Điều 4"}}
{"id":"N2-03","group":2,"lang":"vi","question":"Ai là giám đốc công ty bên B?",
 "expected":{"type":"refusal"}}
{"id":"N4-02","group":4,"lang":"vi","question":"Giá trị hợp đồng là bao nhiêu?",
 "expected":{"type":"conflict","sides":[{"fact":"500 triệu","source":"D1"},{"fact":"450 triệu","source":"D2"}]}}
```

## 3. Chấm điểm

| Nhóm | Cách chấm (tự động trước, người rà sau) |
|---|---|
| N1 | **Đúng đáp án:** `keyFacts` xuất hiện trong câu trả lời (match chuỗi chuẩn hóa; số thì so giá trị). **Đúng nguồn:** citation chứa đúng `sources` kỳ vọng. Tính đúng khi đạt CẢ HAI |
| N2 | Câu trả lời chứa thông điệp từ chối chuẩn VÀ không chứa `keyFacts` của bất kỳ tài liệu nào (chống "từ chối xong vẫn đoán") |
| N3 | Chấm tay theo rubric 3 mức: đạt (hỏi lại/nêu các cách hiểu) / nửa (trả lời một nghĩa nhưng nói rõ giả định) / trượt (đoán bừa) |
| N4 | Tự động: đủ mặt mọi `sides[].fact` + citation đúng `sides[].source`; **trượt ngay** nếu chỉ nêu một bên hoặc xuất hiện con số trộn/trung bình |

**LLM-as-judge:** cho phép dùng để chấm sơ bộ N1/N4 (đối chiếu ngữ nghĩa thay match chuỗi), nhưng kết quả judge phải audit tay được — script lưu câu trả lời thô để người rà lại các case judge phân vân.

## 4. Case đo lệch ngôn ngữ Việt↔Anh (AC US-0.6, US-4.7)

- Trong N1, tối thiểu **4 cặp câu hỏi đối xứng**: cùng một fact, hỏi tiếng Việt trên nguồn Anh (D4) và hỏi tiếng Anh trên nguồn Việt (D1/D5).
- Với mỗi câu, harness log **similarity score của chunk đúng** (không chỉ pass/fail).
- Báo cáo tách riêng: độ chính xác VI→EN vs EN→VI + phân bố similarity hai chiều → phát hiện lệch điểm theo ngôn ngữ (đầu vào để chỉnh ngưỡng Gate 1 — nếu hai chiều lệch nhiều, một ngưỡng chung sẽ từ chối oan một chiều).

## 5. Eval harness — yêu cầu

| Yêu cầu | Chi tiết |
|---|---|
| Chạy một lệnh | `eval run --config <profile>` — ingest corpus vào phiên test sạch → bắn toàn bộ câu hỏi → chấm → báo cáo |
| So sánh 2 cấu hình | `eval compare <runA> <runB>` — bảng chênh lệch theo nhóm; dùng được cho có/không contextual embedding (AC US-0.6), ngưỡng Gate 1 khác nhau, model khác nhau |
| Kết quả tái lập | Cố định: corpus version, câu hỏi version, top-K, ngưỡng, model + version, temperature thấp nhất có thể; mỗi run lưu snapshot config |
| Báo cáo xuất ra | Markdown + JSON: % theo nhóm, hai chiều ngôn ngữ, danh sách case fail kèm câu trả lời thô, tổng token + thời gian chạy |
| Lưu vết | Mỗi run lưu vào `/eval-runs/<date>-<config>/` trong repo code (JSON kết quả; báo cáo Markdown của các mốc quan trọng commit vào repo docs) |

**Mẫu đầu báo cáo:**

```
GOLDEN EVAL — corpus v1 · questions v1 · run 2026-xx-xx
Config: contextual=ON, gate1=0.xx, topK=xx, answer-model=..., ctx-model=...
| Nhóm | Số câu | Pass | % | Mục tiêu |
| N1 có đáp án      | 16 | .. | ..% | ≥ 90% |
| N2 không đáp án   | 12 | .. | ..% | ≥ 95% |
| N3 mơ hồ          |  6 | .. | ..% | (theo dõi) |
| N4 mâu thuẫn      |  8 | .. | ..% | (theo dõi, mục tiêu nêu-đủ-các-bên) |
VI→EN: ..% (sim p50 ..) · EN→VI: ..% (sim p50 ..)
```

## 6. Checklist nghiệm thu US-0.6 (Definition of Done cụ thể)

- [ ] Corpus v1 ≥ 10 tài liệu theo §1, có manifest + checksum
- [ ] ≥ 40 câu hỏi đúng phân bố 4 nhóm §2, mỗi câu có `expected` đầy đủ
- [ ] ≥ 4 cặp câu đối xứng ngôn ngữ, harness log similarity chunk đúng
- [ ] `eval run` chạy một lệnh trên stack compose sạch, xuất báo cáo đúng mẫu §5
- [ ] `eval compare` chạy được giữa 2 run có/không contextual embedding
- [ ] Run baseline đầu tiên được lưu + báo cáo commit vào repo docs (`docs(eval): baseline ...`)

## 7. Quy trình vận hành sau khi có bộ vàng

| Tình huống | Bắt buộc |
|---|---|
| Đổi model trả lời / model context / provider embedding | Chạy full eval, so baseline, đính kết quả vào PR/commit |
| Chỉnh ngưỡng Gate 1 / top-K / chunking | Như trên |
| Thêm câu hỏi mới vào bộ | Được (bump questions version); **sửa/xóa câu cũ** chỉ khi câu sai — ghi lý do trong commit |
| Kết quả tụt so baseline | Không merge; hoặc merge kèm quyết định ghi rõ đánh đổi (ví dụ giảm N1 để tăng N2) |

## Changelog

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 2026-07-06 | Bản đầu — đặc tả để US-0.6 hiện thực ở Sprint 4 |
