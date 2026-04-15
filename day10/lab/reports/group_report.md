# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** X03  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Admin | Ingestion & Quality Owner | admin@example.com |
| X03_Member | Embed & Docs Owner | x03@example.com |

**Ngày nộp:** 2026-04-15  
**Repo:** https://github.com/huanvu05/X03_C401-Day-08-09-10  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**
Pipeline thực hiện đọc dữ liệu thô từ CSV, gán `run_id`, áp dụng các quy tắc làm sạch (Cleaning) và kiểm định (Quality). Dữ liệu sạch được nạp vào ChromaDB với cơ chế Upsert + Pruning để đảm bảo không bị trùng lặp. Cuối cùng, hệ thống kiểm tra tính Freshness để cảnh báo nếu dữ liệu quá cũ.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**
`python etl_pipeline.py run --run-id official_sprint2_fixed`

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-------------------------|-----------------|---------------------------|-------------------------------|
| `chunk_too_short` | 0 | 1 record quarantined | `artifacts/quarantine/..._fixed.csv` |
| `stripped_html` | 0 | 1 record cleaned (tag added) | `artifacts/cleaned/..._fixed.csv` |
| `redacted_pii` | 0 | 1 record redacted (email removed) | `artifacts/cleaned/..._fixed.csv` |

**Rule chính (baseline + mở rộng):**
- **Baseline:** Harmonize `effective_date`, Allowlist `doc_id`, Fix stale refund window (14d -> 7d).
- **Mở rộng:** Loại bỏ HTML tags, Ẩn thông tin PII (Email), Loại bỏ các chunk quá ngắn (<15 ký tự).

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**
Trong lần chạy `official_sprint2`, expectation `chunk_min_length_8` bị FAIL (hệ thống chỉ WARN). Sau khi sửa logic đưa rule lọc chunk ngắn lên trước, ở lần chạy `fixed`, dữ liệu rác này đã bị đưa vào Quarantine, giúp Expectation PASS.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

Sử dụng flag `--no-refund-fix --skip-validate` để chạy pipeline với `run_id = inject-bad`. Kịch bản này giả lập tình huống:
- Bỏ qua bước fix stale refund window → dữ liệu vẫn chứa chuỗi **"14 ngày làm việc"** từ `policy-v3`.
- Bỏ qua toàn bộ validation → expectation `refund_no_stale_14d_window` ghi nhận FAIL nhưng pipeline vẫn tiến hành embed vào ChromaDB.

Kết quả: Vector Store bị nhiễm chunk sai. Manifest ghi nhận `no_refund_fix: true`, `skipped_validate: true` (`artifacts/manifests/manifest_inject-bad.json`).

**Kết quả định lượng (từ CSV / bảng):**

| Câu hỏi | run | top1_preview | contains_expected | hits_forbidden | Nhận xét |
|---------|-----|--------------|-------------------|----------------|-----------|
| `q_refund_window` | **inject-bad** (before) | "…7 ngày làm việc…" | `yes` | **`yes`** | Top-1 đúng nhưng context ô nhiễm |
| `q_refund_window` | **2026-04-15T08-02Z** (after) | "…7 ngày làm việc…" | `yes` | **`no`** | Context sạch hoàn toàn |
| `q_p1_sla` | cả hai run | "…15 phút… 4 giờ…" | `yes` | `no` | Không bị ảnh hưởng |
| `q_lockout` | cả hai run | "…5 lần đăng nhập sai…" | `yes` | `no` | Không bị ảnh hưởng |
| `q_leave_version` | cả hai run | "…12 ngày phép năm…" | `yes` | `no` | Isolation giữa domain hoạt động tốt |

Nguồn bằng chứng: `artifacts/eval/before.csv`, `artifacts/eval/after.csv`, `artifacts/eval/grading_run.jsonl`.

**Kết luận:** Pipeline clean (`run_id = 2026-04-15T08-02Z`) đã thực hiện `embed_prune_removed = 1`, loại bỏ vector stale. Kết quả grading: 3/3 câu đạt `contains_expected = yes`, `hits_forbidden = no`, `top1_doc_matches = true` (với câu HR versioning).

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

**SLA được chọn:** `freshness_sla_hours = 24` giờ.

Hệ thống đọc trường `latest_exported_at` từ manifest để tính `age_hours` (thời gian từ lúc dữ liệu nguồn được export đến lúc pipeline chạy). Kết quả được phân loại:

| Trạng thái | Điều kiện | Hành động |
|------------|-----------|-----------|
| **PASS** | `age_hours ≤ 24h` | Không cần can thiệp |
| **WARN** | `24h < age_hours ≤ 48h` | Log cảnh báo, thông báo Data Owner |
| **FAIL** | `age_hours > 48h` | Cảnh báo khẩn, trigger rerun hoặc escalate |

**Kết quả chạy thực tế (`run_id = 2026-04-15T08-02Z`):**
- `latest_exported_at = 2026-04-10T08:00:00`
- `run_timestamp = 2026-04-15T08:03:00Z`
- `age_hours ≈ 120h` → **Freshness check: FAIL**

Pipeline phát hiện đúng dữ liệu đã ~5 ngày tuổi. Điều này không ảnh hưởng correctness hiện tại (vì dữ liệu đã được làm sạch đúng) nhưng phản ánh nguy cơ mất tính cập nhật trong môi trường production.

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

Pipeline Day 10 đóng vai trò là tầng **Data Ingestion** làm mới corpus cho Agent Day 08/09. Sau khi pipeline kết thúc, collection `day10_kb` trong ChromaDB (`./chroma_db`) chứa 6 vector sạch phục vụ trực tiếp cho retrieval agent. Cơ chế **idempotent upsert + prune** đảm bảo agent không bao giờ đọc phải chunk stale như "hoàn tiền 14 ngày". Nhờ vậy, agent Day 09 trả lời chính xác ngay cả khi corpus được cập nhật liên tục mà không cần restart.

---

## 6. Rủi ro còn lại & việc chưa làm

- **Freshness chưa tự động:** Hệ thống chỉ detect SLA breach sau khi pipeline chạy, chưa có cơ chế tự động trigger refresh hoặc gửi alert (Slack/Email) khi `age_hours > threshold`.
- **Expectation coverage còn hạn hẹp:** Các expectation hiện tại dùng custom logic, chưa tích hợp framework như **Great Expectations**; chưa bao phủ các kịch bản dữ liệu xấu phức tạp hơn (ví dụ: semantic drift, encoding lỗi).
- **Scale chưa được kiểm thử:** Dataset thực tế chỉ có 10 records; chưa đánh giá hiệu năng pipeline khi xử lý hàng nghìn chunk (ChromaDB lock, memory pressure).
- **Rule versioning còn hard-code:** Ngưỡng nghiệp vụ (ví dụ HR cutoff date, cửa sổ hoàn tiền 7 ngày) được hard-code trong `cleaning_rules.py` thay vì đọc từ `contracts/data_contract.yaml` — khó thay đổi khi policy thay đổi.
- **Thiếu monitoring dashboard:** Chưa có dashboard trực quan (Grafana/Metabase) để theo dõi `quarantine_ratio`, `age_hours`, và expectation pass-rate theo thời gian thực.
