# Báo cáo cá nhân — Lab Day 10

**Họ và tên:**  Vũ Văn Huân
**Vai trò:** Cleaning & Quality  
**Độ dài:** ~450 từ  

---

## 1. Phụ trách

Trong bài lab này, tôi phụ trách chính phần **Cleaning & Quality**, bao gồm:
- Xây dựng và mở rộng `transform/cleaning_rules.py` (các rule xử lý dữ liệu bẩn)
- Thiết kế và bổ sung expectation trong `quality/expectations.py`
- Thực hiện kiểm thử chất lượng dữ liệu thông qua before/after evaluation
- Hỗ trợ phần grading bằng cách đảm bảo kết quả retrieval đạt yêu cầu

Cụ thể, tôi đã bổ sung các expectation mới như:
- Kiểm tra duplicate chunk (`no_duplicate_doc_chunk`)
- Phát hiện ký tự rác hoặc HTML (`no_html_or_escape_chars`)
- Validate doc_id theo whitelist (`doc_id_in_whitelist`)

Các thành phần này kết nối với pipeline thông qua `cleaned_rows` và được log lại qua các chỉ số như `cleaned_records`, `quarantine_records`.

---

## 2. Quyết định kỹ thuật

Một quyết định quan trọng là phân biệt giữa **halt và warn** trong expectation:

- Với lỗi nghiêm trọng như stale data (ví dụ "14 ngày làm việc"), tôi sử dụng **halt** để chặn pipeline trong điều kiện bình thường.
- Tuy nhiên, trong Sprint 3 (inject test), sử dụng `--skip-validate` để cho phép pipeline tiếp tục, nhằm quan sát ảnh hưởng của dữ liệu lỗi lên retrieval.

Ngoài ra, tôi đánh giá cao cơ chế **idempotent embedding**, đặc biệt là bước `embed_prune_removed`, giúp loại bỏ vector cũ không còn tồn tại trong cleaned data. Điều này rất quan trọng để tránh trường hợp retrieval trả về context lỗi dù dữ liệu đã được fix.

---

## 3. Sự cố / anomaly

Một anomaly quan trọng được phát hiện trong quá trình inject test:

- Khi chạy với flag `--no-refund-fix --skip-validate`, expectation `refund_no_stale_14d_window` FAIL (violations=1), chứng tỏ dữ liệu chứa thông tin lỗi ("14 ngày").
- Tuy nhiên, kết quả retrieval vẫn trả về answer đúng ("7 ngày") do ranking.

Dù vậy, kiểm tra `hits_forbidden = yes` cho thấy trong top-k vẫn tồn tại chunk chứa thông tin sai.

Nguyên nhân:
- Vector DB vẫn chứa dữ liệu stale do không bị chặn ở bước validate.

Cách khắc phục:
- Sau khi chạy pipeline chuẩn, hệ thống thực hiện `embed_prune_removed=1`, loại bỏ vector lỗi.
- Kết quả là `hits_forbidden = no`, đảm bảo context sạch hoàn toàn.

---

## 4. Before/after

**Log:**
- Inject run:
  - `expectation[refund_no_stale_14d_window] FAIL (halt)`
- Clean run:
  - `expectation[...] OK`
  - `embed_prune_removed=1`

**CSV evidence:**

- BEFORE (`before.csv`):
  - `contains_expected = yes`
  - `hits_forbidden = yes`

- AFTER (`after.csv`):
  - `contains_expected = yes`
  - `hits_forbidden = no`

→ Điều này chứng minh pipeline đã cải thiện chất lượng dữ liệu không chỉ ở mức câu trả lời mà còn ở mức context.

---

## 5. Cải tiến thêm 2 giờ

Nếu có thêm thời gian, tôi sẽ:
- Tách rule versioning (ví dụ HR cutoff date) khỏi code và đọc từ `contracts/data_contract.yaml`
- Điều này giúp pipeline linh hoạt hơn, tránh hard-code và dễ dàng thay đổi theo business rule
- Đồng thời, có thể mở rộng thêm monitoring để detect thay đổi version tự động

---