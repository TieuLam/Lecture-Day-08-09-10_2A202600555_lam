# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

> User / agent thấy gì? (VD: trả lời “14 ngày” thay vì 7 ngày)
Người dùng nội bộ chat với Agent và nhận được câu trả lời: "Bạn có 14 ngày để hoàn tiền" hoặc "Bạn được nghỉ 10 ngày phép năm" — là các chính sách đã hết hiệu lực từ năm 2025.

---

## Detection

> Metric nào báo? (freshness, expectation fail, eval `hits_forbidden`)
- Đánh giá `eval_retrieval.py` báo `hits_forbidden=yes` tại các câu liên quan đến chính sách hoàn tiền.
- Freshness Check báo `FAIL` liên tục do dữ liệu quá cũ so với `SLA_HOURS=24.0`.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Tìm thấy `latest_exported_at` bị cũ, chứng tỏ data load từ DB lỗi. |
| 2 | Mở `artifacts/quarantine/*.csv` | Thấy thiếu record của chính sách mới (chứng tỏ bị chặn sai) hoặc không có record bị chặn ở chính sách cũ. |
| 3 | Chạy `python eval_retrieval.py` | Phát hiện top-1 document được trích xuất là document cũ (sai version). |

---

## Mitigation

> Rerun pipeline, rollback embed, tạm banner “data stale”, …
Lập tức sửa đổi file cleaning rule để thêm luật Quarantine với dòng chữ cũ, sau đó rerun pipeline (chạy lại lệnh `etl_pipeline.py run`).
Trong thời gian đó, có thể báo lên UI của Agent dòng "Hệ thống tri thức đang được bảo trì" để tránh hiểu nhầm cho người dùng.

---

## Prevention

> Thêm expectation, alert, owner — nối sang Day 11 nếu có guardrail.
Thêm Expectation cứng (Halt severity) vào file `expectations.py` để chặn hoàn toàn chuỗi text chứa "14 ngày làm việc" hoặc "10 ngày phép năm". Nếu raw CSV bị lỗi và có các chuỗi này xuất hiện sau khâu Cleaned, pipeline sẽ sập ngay và không upsert vào VectorDB.
