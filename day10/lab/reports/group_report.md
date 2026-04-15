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

_________________

**Kết quả định lượng (từ CSV / bảng):**

_________________

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

_________________

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

_________________

---

## 6. Rủi ro còn lại & việc chưa làm

- …
