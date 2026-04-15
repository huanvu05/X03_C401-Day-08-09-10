# Báo cáo cá nhân — Lab Day 10 (Thực tế thực hiện)

**Họ và tên:** Hoàng Quang Thắng
**Vai trò:** Ingestion & Quality Owner (Nhóm X03)
**Ngày thực hiện:** 15/04/2025

---

## 1. Phụ trách

Tôi chịu trách nhiệm thiết lập luồng dữ liệu chính trong `etl_pipeline.py`, cấu hình các quy tắc làm sạch nâng cao trong `transform/cleaning_rules.py` và bộ kiểm định chất lượng trong `quality/expectations.py`. Ngoài ra, tôi đã hoàn thiện tài liệu kiến trúc hệ thống và quy trình xử lý sự cố (Runbook).

**Đóng góp chính:**
- Thiết kế sơ đồ luồng dữ liệu (Mermaid) trong `pipeline_architecture.md`.
- Triển khai 3 quy tắc làm sạch mới: Loại bỏ HTML tags, Ẩn thông tin PII (Email), và lọc các chunk quá ngắn.
- Cấu hình hạ tầng giám sát trong `data_contract.yaml`.

---

## 2. Quyết định kỹ thuật

**Logic Ordering (Rủi ro quan trọng):** Trong quá trình triển khai, tôi đã đưa ra quyết định quan trọng là di chuyển khối logic các quy tắc làm sạch lên **trước** bước `cleaned.append`. Nếu đặt sau, các record lỗi vẫn bị ghi vào danh sách "sạch" trước khi kịp kiểm tra, làm vô hiệu hóa các bước lọc.

**Halt vs Warn:** Tôi thiết lập mức `halt` cho lỗi sai định dạng ngày (`effective_date`) và lỗi sót chính sách hoàn tiền 14 ngày, vì đây là những lỗi gây sai lệch nghiêm trọng cho kết quả tìm kiếm của Agent. Các lỗi về độ dài chunk (`chunk_min_length_8`) được đặt ở mức `warn` để linh hoạt hơn với nội dung đa dạng.

---

## 3. Sự cố / Anomaly (Lỗi logic)

Trong lần chạy thử nghiệm đầu tiên (`official_sprint2`), tôi phát hiện số lượng `cleaned_records` vẫn là 9 mặc dù đã thêm rule lọc. Sau khi kiểm tra file CSV, tôi phát hiện các chunk ngắn ("Lỗi.") và chunk chứa HTML vẫn lọt vào bộ dữ liệu sạch. 

**Khắc phục:** Tôi đã sửa lại thứ tự thực thi trong `cleaning_rules.py`, đảm bảo biến `fixed_text` được gán chính xác và bước `continue` (đối với chunk ngắn) được thực hiện trước khi ghi dữ liệu. Kết quả lần chạy sau (`official_sprint2_fixed`) đã giảm xuống còn 8 record sạch, đúng theo kỳ vọng.

---

## 4. Before/After Evidence

- **Trước khi sửa:** Pipeline báo PASS nhưng dữ liệu `artifacts/cleaned/` vẫn chứa tag HTML và email thô.
- **Sau khi sửa:** 
    - Log: `cleaned_records=8`, `quarantine_records=5`.
    - CSV `cleaned`: Các dòng đã được gắn tag `[cleaned: stripped_html]` và `[cleaned: redacted_pii]`.
    - CSV `quarantine`: Xuất hiện record bị loại bỏ với lý do `chunk_too_short`.

---

## 5. Cải tiến hướng Distinction

Tôi đã thực hiện **Data Injection** có chủ đích vào file `policy_export_dirty.csv` để kiểm chứng sức mạnh của bộ lọc. Thay vì chỉ sử dụng dữ liệu mẫu có sẵn, việc nhúng thêm các lỗi thực tế (HTML lồng nhau, email nội bộ) đã giúp chứng minh khả năng bảo mật thông tin (PII) và làm sạch dữ liệu của pipeline trước khi đẩy vào hệ thống RAG.
