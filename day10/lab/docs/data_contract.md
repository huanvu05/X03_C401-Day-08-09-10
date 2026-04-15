# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn (`doc_id`) | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| **policy_refund_v4** | Sync / Migration | **1. Duplicate data:** Trùng lặp hoàn toàn nội dung chunk (chunk_id 1 và 2).<br>**2. Migration Error:** Đồng bộ nhầm dữ liệu cũ (chunk_id 3 chứa text *"bản sync cũ policy-v3 — lỗi migration"*).<br>**3. Missing Data:** `chunk_text` bị rỗng/Null (chunk_id 5). | - Alert khi tỉ lệ Null/Missing `chunk_text` > 0%.<br>- Metric: Count Duplicate Rows (Cảnh báo nếu > 0).<br>- Alert khi phát hiện các keyword lỗi hệ thống trong text (VD: "lỗi", "sync cũ"). |
| **hr_leave_policy** | Cập nhật chính sách định kỳ | **Version Conflict:** Tồn tại song song hai phiên bản chính sách trái ngược nhau (chunk 7: *"10 ngày phép... bản HR 2025"* và chunk 8: *"12 ngày phép... chính sách 2026"*). | - Metric: Đo lường số lượng phiên bản/năm của cùng một chính sách.<br>- Alert: Cảnh báo ngữ nghĩa khi nội dung có ngày tháng mâu thuẫn với `effective_date`. |
| **legacy_catalog_xyz_zzz** | Legacy Data Load / Test Load | **Dữ liệu rác / Test data:** Chứa các nội dung không có giá trị nghiệp vụ, chỉ dùng để test (chunk 9: *"Chunk nội dung đủ dài để vượt ngưỡng expectation..."*). | - Metric: Tỉ lệ từ khóa test/spam trong dataset.<br>- Alert khi `doc_id` chứa prefix/suffix lạ (như `legacy_`, `_zzz`). |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | Khóa chính (Primary key). Định dạng mới bao gồm `{doc_id}_{index}_{hash}` để đảm bảo tính duy nhất (VD: `policy_refund_v4_1_3b11f30cc4b49d25`). |
| doc_id | string | Có | Mã định danh của tài liệu gốc (VD: `policy_refund_v4`, `sla_p1_2026`). Dùng để ánh xạ về nguồn. |
| chunk_text | string | Có | Nội dung văn bản đã được làm sạch (cleaned text). Các dữ liệu lỗi/cũ được gỡ bỏ hoặc gắn tag (VD: chứa tag `[cleaned: stale_refund_window]`). |
| effective_date | date | Có | Ngày hiệu lực của chính sách/tài liệu, định dạng `YYYY-MM-DD` (VD: `2026-02-01`). Dùng để query đúng phiên bản. |
| exported_at | timestamp| Có | Thời điểm xuất/đồng bộ dữ liệu, định dạng ISO 8601 (VD: `2026-04-10T08:00:00`). Dùng để tracking version ingested. |

---

## 3. Quy tắc quarantine vs drop

> Record bị flag đi đâu? Ai approve merge lại?

Dựa vào logic xử lý từ raw sang cleaned data, quy tắc được áp dụng như sau: 

- **Quy tắc Drop (Xóa bỏ vĩnh viễn khỏi dataset sạch):** 
    - **Duplicate hoàn toàn:** Các dòng trùng lặp y hệt nội dung (VD: `chunk_id` 1 và 2 của `policy_refund_v4`) bị drop chỉ giữ lại 1 bản. 
    - **Missing text:** Các dòng có `chunk_text` là Null (VD: `chunk_id` 5) bị drop. * **Dữ liệu Test/Legacy:** Toàn bộ document có prefix/suffix rác (VD: `legacy_catalog_xyz_zzz`) bị drop khỏi bảng cleaned. * **Phiên bản cũ (Version conflict):** Nếu có 2 bản policy xung đột, bản cũ/hết hạn (VD: chính sách HR 2025) sẽ bị drop thẳng. 
- **Quy tắc Quarantine (Cách ly & Sửa chữa nội tuyến):** 
    - **Dữ liệu sai logic nghiệp vụ nhưng vẫn có giá trị (Stale Data):** Thay vì drop, hệ thống sẽ thực hiện update giá trị trực tiếp trong `chunk_text` (VD: Sửa "14 ngày" thành "7 ngày" trong policy v3 cũ) và gắn thêm tag `[cleaned: stale_refund_window]` ở cuối chuỗi để đánh dấu. 
    - **Quy trình Approve:** Các record có chứa tag `[cleaned: *]` sẽ được gom vào một view riêng (Quarantine View) trên Data Warehouse để Data Steward hoặc Domain Expert (VD: Team CSKH, HR) review định kỳ.

---

## 4. Phiên bản & canonical

> Source of truth cho policy refund: file nào / version nào?

Dựa vào dữ liệu sau làm sạch, **Source of Truth** (Phiên bản chuẩn) cho chính sách hoàn tiền (`policy_refund_v4`) được xác định như sau:

- **Version chuẩn:** Là version có số ngày hoàn tiền là **7 ngày làm việc**, hiệu lực từ ngày `2026-02-01` (dựa trên cột `effective_date`). 
- Các chính sách quy định "14 ngày" từ `policy-v3` được coi là dữ liệu cũ (lỗi migration) và đã bị hệ thống ghi đè/gắn tag thành 7 ngày để ép tuân thủ theo Source of Truth mới.