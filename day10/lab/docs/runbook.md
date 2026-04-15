# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

## Symptom

- Agent trả lời sai thông tin chính sách (ví dụ: vẫn báo hoàn tiền 14 ngày).
- Dashboard giám sát báo đỏ hoặc Slack channel `#x03-data-ops` nhận được cảnh báo SLA Freshness.
- Kết quả tìm kiếm (retrieval) chứa các đoạn text có tag `[cleaned: stale_...]`.

---

## Detection

## Detection

- **Freshness Alert:** Lệnh `python etl_pipeline.py freshness` trả về `FAIL`.
- **Expectation Fail:** Log pipeline báo `PIPELINE_HALT` do không vượt qua các kiểm định chất lượng (ví dụ: `no_stale_refund_window`).
- **Data Observability:** File `artifacts/quarantine/` tăng đột biến về số lượng record.

---

## Diagnosis

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Xác định `age_hours` vượt quá `sla_hours` (24h). |
| 2 | Mở `artifacts/quarantine/*.csv` | Xem lý do bị cách ly (ví dụ: `stale_hr_policy`). |
| 3 | Kiểm tra log file trong `artifacts/logs/` | Tìm điểm dừng của pipeline (`PIPELINE_HALT` ở bước nào). |

---

## Mitigation

## Mitigation

1. **Rerun pipeline:** Nếu lỗi do dữ liệu nguồn tạm thời chưa đồng bộ, thử chạy lại.
2. **Rollback:** Nếu bản publish mới bị lỗi chất lượng, sử dụng `run_id` cũ để khôi phục Vector Store.
3. **Manual Fix:** Sửa dữ liệu trong `data/raw/` và chạy lại với flag `--run-id hotfix`.
4. **Broadcast:** Thông báo cho người dùng về việc dữ liệu đang được bảo trì qua channel Slack.

---

## Prevention

> Thêm expectation, alert, owner — nối sang Day 11 nếu có guardrail.
