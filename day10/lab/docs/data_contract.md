# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `policy_refund_v4` | Export CSV định kỳ | Có chứa dữ liệu cũ (14 ngày), lặp dữ liệu quá dài | Cảnh báo expectation "must not contain 14 ngày làm việc" |
| `hr_leave_policy` | Sync từ HR system | Xung đột phiên bản chính sách cũ 2025 | Quarantine record đếm số lượng > threshold |
| `access_control_sop` | Export tĩnh | Có thể thiếu allowlist dẫn đến bị quarantine toàn bộ | Tỉ lệ drop > 10% cảnh báo |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | Hash của doc_id, text, seq để đảm bảo idempotent |
| doc_id | string | Có | Phải thuộc `ALLOWED_DOC_IDS` |
| chunk_text | string | Có | Không được chứa "!!!", không quá 300 ký tự, không có "Nội dung không rõ ràng" |
| effective_date | date | Có | Format chuẩn YYYY-MM-DD |
| exported_at | datetime | Có | Giữ nguyên format ISO từ raw |

---

## 3. Quy tắc quarantine vs drop

> Record bị flag đi đâu? Ai approve merge lại?
Tất cả record không thỏa mãn rules sẽ bị đánh dấu "reason" và chuyển vào file CSV nằm trong thư mục `artifacts/quarantine/`.
Các data engineer cần review định kỳ, sửa code cleaning_rules.py để khắc phục dữ liệu sai.
Dữ liệu sai không bị xóa hoàn toàn khỏi raw (drop) nhưng không được đưa vào thư mục `cleaned/` và collection.

---

## 4. Phiên bản & canonical

> Source of truth cho policy refund: file nào / version nào?
`policy_refund_v4` với ngày hiệu lực mới nhất được xem là canonical. Những dòng có chính sách hoàn tiền 14 ngày sẽ được chuẩn hoá hoặc loại bỏ.
Chính sách HR lấy mốc `effective_date >= 2026-01-01` làm canonical, dữ liệu trước 2026 sẽ bị quarantine.
