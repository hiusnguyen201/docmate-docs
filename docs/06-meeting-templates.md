# DocMate — Template Biên bản Sprint (Planning · Review · Retrospective)

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Tham chiếu** | `04-dor-dod.md` · `05-sprint-0-plan.md` |
| **Cách dùng** | Mỗi sprint copy 3 template dưới thành file `sprints/sprint-N-notes.md` trong repo docs, điền và commit `docs(sprint): sprint N planning/review/retro notes` |

> Solo dev vẫn viết biên bản vì: (1) quyết định không ghi lại = quyết định sẽ bị cãi lại bởi chính mình 3 tuần sau; (2) dữ liệu velocity/carry-over qua các sprint là căn cứ duy nhất để hiệu chỉnh kế hoạch. Mỗi biên bản mục tiêu ≤ 15 phút điền.

---

## Template 1 — Sprint Planning

```markdown
# Sprint N — Planning

**Ngày:** YYYY-MM-DD · **Sprint:** N (YYYY-MM-DD → YYYY-MM-DD)

## 1. Sprint Goal
> (1–2 câu. Kiểm tra: nếu chỉ hoàn thành 70% story, goal còn đạt được không?
> Goal phải nói VÌ SAO sprint này tồn tại, không phải liệt kê story.)

## 2. Capacity
| | |
|---|---|
| Ngày làm việc thực tế (trừ nghỉ/việc riêng đã biết) | … / 10 |
| Velocity 3 sprint gần nhất | … / … / … SP |
| SP nhận vào sprint này | … SP |

## 3. Sprint Backlog
| Story | SP | Đạt DoR? (§1 `04-dor-dod.md`) | Ghi chú thứ tự / phụ thuộc |
|---|---|---|---|
| US-… | … | ☐ | |

## 4. Story bị từ chối kéo vào (và lý do DoR nào fail)
| Story | Lý do |
|---|---|
| | |

## 5. Rủi ro của sprint + kế hoạch cắt
- Rủi ro: …
- Nếu trượt tiến độ, cắt trước: … (ghi rõ NGAY BÂY GIỜ, lúc còn tỉnh táo —
  không quyết lúc đang trễ)

## 6. Quyết định trong buổi Planning
| Quyết định | Lý do | Ảnh hưởng tài liệu nào (PRD/AC/ADR)? |
|---|---|---|
| | | |
```

---

## Template 2 — Sprint Review

```markdown
# Sprint N — Review

**Ngày:** YYYY-MM-DD

## 1. Sprint Goal: ĐẠT / KHÔNG ĐẠT / ĐẠT MỘT PHẦN
> (1 câu giải thích. Đánh giá theo goal, không theo số SP.)

## 2. Kết quả story
| Story | SP | Trạng thái | Bằng chứng Done (demo gì / test nào / lệnh nào) |
|---|---|---|---|
| US-… | … | Done / Carry-over | |

**Velocity thực tế:** … SP (Done thật theo DoD — carry-over KHÔNG tính, kể cả "xong 90%")

## 3. Demo checklist (tự demo, chạy thật, không mở code)
- [ ] Chạy từ trạng thái sạch, đúng như người dùng/dev mới sẽ chạy
- [ ] Đi qua từng AC của story Done — tick trực tiếp vào `03-acceptance-criteria.md` nếu cần
- [ ] Ghi lại điều bất ngờ nhận ra khi demo (thường lòi ra UX/edge case):
  - …

## 4. Carry-over
| Story | Phần còn thiếu | Ước lượng lại | Nguyên nhân dở dang |
|---|---|---|---|
| | | | |

## 5. Ảnh hưởng tới backlog/lộ trình
- Backlog thay đổi: … (story mới phát sinh / ước lượng lại / đổi thứ tự)
- Lộ trình 7 sprint còn giữ được không? Nếu lệch: lệch bao nhiêu, bù bằng gì?
```

---

## Template 3 — Sprint Retrospective

```markdown
# Sprint N — Retrospective

**Ngày:** YYYY-MM-DD

## 1. Số liệu nhìn lại
| | Sprint này | Sprint trước |
|---|---|---|
| SP nhận / SP Done | … / … | … / … |
| Story carry-over | … | … |
| Số lần vi phạm DoD "cho kịp" (tự thú) | … | … |

## 2. Keep — điều gì hiệu quả, giữ nguyên
- …

## 3. Drop — điều gì gây hại/lãng phí, bỏ
- …

## 4. Try — thử gì mới ở sprint sau (TỐI ĐA 2 mục, có cách đo)
| Thử | Đo bằng gì ở Retro sau |
|---|---|
| | |

## 5. Review lại Try của Retro trước
| Đã thử | Kết quả | Giữ / bỏ |
|---|---|---|
| | | |

## 6. DoR/DoD có cần sửa không?
- ☐ Không
- ☐ Có → sửa `04-dor-dod.md`, bump version, áp dụng từ sprint sau
  (KHÔNG sửa hồi tố cho sprint vừa rồi)

## 7. Một câu thật lòng về sprint này
> (Chỗ duy nhất được viết cảm tính. Solo dev không có ai để nói —
> nói với biên bản. Đọc lại chuỗi câu này sau 5 sprint rất có giá trị.)
```

---

## Quy ước chung cho cả 3 biên bản

| Quy ước | Nội dung |
|---|---|
| **Vị trí file** | `docs/sprints/sprint-N-notes.md` — 3 biên bản của một sprint nằm chung 1 file, nối tiếp nhau theo thời gian |
| **Commit** | Mỗi lần điền xong một biên bản = 1 commit + push ngay: `docs(sprint): sprint N planning notes` / `… review notes` / `… retro notes` |
| **Thời điểm** | Planning: ngày đầu sprint · Review + Retro: ngày cuối sprint (Review trước, Retro sau, tách 2 mạch tư duy: sản phẩm vs quy trình) |
| **Kỷ luật số liệu** | Velocity chỉ tính story Done trọn theo DoD. Mọi con số "gần xong" đều = 0 |

## Changelog

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 2026-07-06 | Bản đầu — 3 template hiệu chỉnh cho solo dev |
