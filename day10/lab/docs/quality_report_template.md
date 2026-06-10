# Quality report — Lab Day 10 (nhóm)

**run_id:** `2026-06-10T08-24Z`
**Ngày:** `2026-06-10`

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (Inject Bad) | Sau (Fix) | Ghi chú |
|--------|-------|-----|---------|
| raw_records | 247 | 247 | Dữ liệu gốc giống nhau |
| cleaned_records | 36 | 36 | (Thực tế data vào db lỗi vì không filter hoàn tiền) |
| quarantine_records | 211 | 211 | |
| Expectation halt? | Fail | Passed | Expectation chặn data cũ phát hiện vi phạm và pass sau khi fix |

---

## 2. Before / after retrieval (bắt buộc)

> Đính kèm hoặc dẫn link tới `artifacts/eval/after_fix_eval.csv` (hoặc 2 file before/after).

**Câu hỏi then chốt:** refund window (`q_refund_window`)  
**Trước (Inject Bad):** `Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc kể từ xác nhận đơn.` (hits_forbidden=yes)  
**Sau (Fix):** `Yêu cầu được gửi trong vòng 7 ngày làm việc làm việc kể từ thời điểm xác nhận đơn hàng.` (hits_forbidden=no)

**Merit (khuyến nghị):** versioning HR — `q_hr_annual_leave_under3` (`contains_expected`, `hits_forbidden`, cột `top1_doc_expected`)

**Trước:** Truy xuất nhầm chính sách 2025 hoặc dữ liệu rác.  
**Sau:** Lấy chuẩn xác HR policy 2026: `Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026.`

---

## 3. Freshness & monitor

> Kết quả `freshness_check` (PASS/WARN/FAIL) và giải thích SLA bạn chọn.
Kết quả `FAIL` (`freshness_sla_exceeded`) do raw data có `exported_at` là `2026-04-11`, trong khi ngày chạy script là tháng 06/2026 (vượt quá `SLA_HOURS=24`). SLA này phù hợp cho data cập nhật hàng ngày (daily dump).

---

## 4. Corruption inject (Sprint 3)

> Mô tả cố ý làm hỏng dữ liệu kiểu gì (duplicate / stale / sai format) và cách phát hiện.
Chạy với cờ `--skip-validate` và `--no-refund-fix` để cố tình đưa dữ liệu refund window (14 ngày cũ) vào ChromaDB. Pipeline phát hiện thông qua expectation `refund_no_stale_14d_window` nhưng bỏ qua halt do có cờ `--skip-validate`.

---

## 5. Hạn chế & việc chưa làm

- Cần tích hợp Great Expectations / Pydantic nếu muốn scale lớn hơn.
