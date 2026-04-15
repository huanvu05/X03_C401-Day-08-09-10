# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

- Agent trả lời sai thông tin chính sách (ví dụ: vẫn báo hoàn tiền 14 ngày).
- Dashboard giám sát báo đỏ hoặc Slack channel `#x03-data-ops` nhận được cảnh báo SLA Freshness.
- Kết quả tìm kiếm (retrieval) chứa các đoạn text có tag `[cleaned: stale_...]`.

---

## Detection

- **Freshness Alert:** Lệnh `python etl_pipeline.py freshness` trả về `FAIL`.
- **Expectation Fail:** Log pipeline báo `PIPELINE_HALT` do không vượt qua các kiểm định chất lượng (ví dụ: `no_stale_refund_window`).
- **Data Observability:** File `artifacts/quarantine/` tăng đột biến về số lượng record.

---

## Diagnosis


| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Xác định `age_hours` vượt quá `sla_hours` (24h). |
| 2 | Mở `artifacts/quarantine/*.csv` | Xem lý do bị cách ly (ví dụ: `stale_hr_policy`). |
| 3 | Kiểm tra log file trong `artifacts/logs/` | Tìm điểm dừng của pipeline (`PIPELINE_HALT` ở bước nào). |
| 4 | So sánh `before.csv` và `after.csv` (eval) | Xác định có còn `hits_forbidden = yes` hay không |

---

## Mitigation

1. **Rerun pipeline:** Nếu lỗi do dữ liệu nguồn tạm thời chưa đồng bộ, thử chạy lại.
2. **Rollback:** Nếu bản publish mới bị lỗi chất lượng, sử dụng `run_id` cũ để khôi phục Vector Store.
3. **Manual Fix:** Sửa dữ liệu trong `data/raw/` và chạy lại với flag `--run-id hotfix`.
4. **Broadcast:** Thông báo cho người dùng về việc dữ liệu đang được bảo trì qua channel Slack.
5. **Disable unsafe publish:** Tạm thời chặn bước embed nếu `expectation severity = halt`.

---

## Prevention

- **Expectation Guardrail**
  - Thêm rule bắt buộc:
    - `no_stale_refund_window`
    - `effective_date_iso_yyyy_mm_dd`
  - Phân biệt rõ:
    - `halt` → chặn pipeline
    - `warn` → cho phép publish nhưng log

- **Data Freshness SLA**
  - Đặt SLA rõ ràng:
    - `freshness_sla_hours = 24`
  - Log bắt buộc:
    - `latest_exported_at`
    - `age_hours`

- **Idempotent Embedding**
  - Luôn prune vector cũ:
    - tránh `hits_forbidden = yes` sau khi clean
  - đảm bảo mỗi `run_id` tạo snapshot riêng

- **Contract Enforcement**
  - Schema phải được validate trước khi clean:
    - `doc_id` không rỗng
    - `effective_date` ISO format
  - Fail early trước bước embed

- **Observability Upgrade (khuyến nghị)**
  - Log thêm:
    - `clean_to_embed_delta`
    - `quarantine_ratio`
  - Alert khi:
    - quarantine > 30%
    - freshness FAIL liên tục 2 runs

- **Ownership**
  - Data Owner chịu trách nhiệm freshness
  - Quality Owner chịu trách nhiệm expectation suite
  - Embed Owner chịu trách nhiệm prune & idempotency

---