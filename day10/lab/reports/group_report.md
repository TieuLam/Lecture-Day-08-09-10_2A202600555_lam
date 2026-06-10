# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** AI Action Team  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Nguyễn Văn A | Ingestion / Raw Owner | nva@example.com |
| Trần Văn B | Cleaning & Quality Owner | tvb@example.com |
| Lê Thị C | Embed & Idempotency Owner | ltc@example.com |
| Phạm Văn D | Monitoring / Docs Owner | pvd@example.com |

**Ngày nộp:** 2026-06-10  
**Repo:** `vinuni-ai-action-lab10`  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**
Nguồn raw là file CSV `policy_export_dirty.csv` mô phỏng export từ 5 hệ thống nguồn khác nhau. Pipeline sẽ đọc toàn bộ record, phân loại (dựa vào ALLOWED_DOC_IDS), làm sạch dữ liệu (chuẩn hóa ngày tháng, xử lý duplicate, xử lý dữ liệu hết hạn như HR 2025 và chính sách hoàn tiền cũ 14 ngày), sau đó chạy một bộ expectations để validate data. Những bản ghi lỗi được chuyển vào file CSV riêng (quarantine), còn bản ghi sạch được băm ID (idempotency) và nhúng vào vector DB.
Mã run_id được tạo tự động dựa trên thời gian chạy hoặc theo tham số truyền vào `--run-id`.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**
`python etl_pipeline.py run`

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| `no_exclamation_marks` | Không bị chặn | Xoá bỏ các chữ `!!!` làm sạch văn bản | File log có dòng ghi violations=0 |
| `no_abnormally_long_chunks` | Không quarantine | Quarantine chunk > 300 ký tự | `quarantine_records` tăng lên 211 |
| Rule xóa 'nội dung không rõ ràng:' | Không quarantine | Quarantine chunk chứa câu này | Số chunk cleaned giảm rõ rệt |

**Rule chính (baseline + mở rộng):**
- Thêm `access_control_sop` vào allowlist.
- Loại bỏ chunk có nội dung quá dài > 300 ký tự (chống lặp sync).
- Loại bỏ các chuỗi nhiễu như `!!!` hoặc câu cảnh báo `Nội dung không rõ ràng:`.
- Quarantine toàn bộ text của HR policy cũ (`10 ngày phép năm`).

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**
Khi chạy Sprint 3 với inject corruption, expectation `refund_no_stale_14d_window` đã FAIL với 2 violations. Pipeline đã cảnh báo nhưng vì truyền cờ `--skip-validate` nên cố ý bỏ qua halt để phục vụ demo Before/After.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**
Chúng tôi đã chạy `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate` để tắt bộ lọc chính sách hoàn tiền (giữ nguyên "14 ngày làm việc" sai) và bypass lỗi HALT.

**Kết quả định lượng (từ CSV / bảng):**
- **Trước (Inject Bad - `after_inject_bad.csv`):** Khi hỏi "Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền", top-1 retrieval trả về văn bản cũ với `hits_forbidden=yes` (14 ngày).
- **Sau (Fix - `after_fix_eval.csv`):** Khi đã chạy pipeline hoàn chỉnh, top-1 retrieval trả về văn bản đúng: `Yêu cầu được gửi trong vòng 7 ngày làm việc làm việc kể từ thời điểm xác nhận đơn hàng.` với `hits_forbidden=no`.

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.
Pipeline sử dụng `SLA_HOURS=24.0`. Khi kiểm tra độ tươi, kết quả trả về FAIL do file raw CSV có `exported_at` là tháng 04/2026, nhưng ngày chạy pipeline là tháng 06/2026 (Age: ~1448 hours). Điều này là hoàn toàn hợp lý với thiết kế cảnh báo của hệ thống.

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.
Dữ liệu nhúng tại Day 10 sẽ được agent CS / IT Helpdesk tại Day 09 đọc để trả lời các câu hỏi RAG. Pipeline này đảm bảo Data Quality, giúp LLM tại tầng Agent không bị trích xuất nhầm các chính sách cũ hay văn bản rác, tạo độ tin cậy tuyệt đối cho luồng Multi-agent.

---

## 6. Rủi ro còn lại & việc chưa làm
- Cần mở rộng bộ Validate mạnh mẽ hơn (sử dụng pydantic hoặc GE) cho các file PDF hoặc API ngoài CSV.
