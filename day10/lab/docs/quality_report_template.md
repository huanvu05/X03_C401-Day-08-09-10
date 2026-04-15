# Quality report — Lab Day 10 (nhóm)

**run_id:** inject-bad / 2026-04-15T08-02Z  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước | Sau | Ghi chú |
|--------|-------|-----|---------|
| raw_records | 10 | 10 | Không đổi |
| cleaned_records | 6 | 6 | Sau cleaning |
| quarantine_records | 4 | 4 | Dữ liệu lỗi bị loại |
| Expectation halt? | FAIL | PASS | Trước inject có stale data |

---

## 2. Before / after retrieval (bắt buộc)

> Dữ liệu lấy từ:
- `artifacts/eval/before.csv`
- `artifacts/eval/after.csv`

---

### 🔥 Câu hỏi then chốt: refund window (`q_refund_window`)

#### Trước (inject run)
- Top-1 answer:
  > "Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng."
- `contains_expected = yes`
- `hits_forbidden = yes`

👉 Nhận xét:
- Câu trả lời đúng (7 ngày)
- Nhưng trong top-k context vẫn chứa thông tin sai "14 ngày"
→ tồn tại **stale data trong vector DB**

---

#### Sau (clean pipeline)
- Top-1 answer:
  > "Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng."
- `contains_expected = yes`
- `hits_forbidden = no`

👉 Nhận xét:
- Câu trả lời đúng
- Context đã sạch hoàn toàn
→ dữ liệu stale đã được loại bỏ

---

### 🏆 Merit: HR versioning (`q_leave_version`)

#### Trước
- Top-1 answer:
  > "Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026."
- `contains_expected = yes`
- `hits_forbidden = no`
- `top1_doc_expected = yes`

#### Sau
- Không thay đổi so với trước

👉 Nhận xét:
- HR policy không bị ảnh hưởng bởi inject scenario
- Pipeline đảm bảo **isolation giữa các domain dữ liệu**

---

## 3. Freshness & monitor

- Kết quả:
freshness_check = FAIL
age_hours ≈ 120h > SLA (24h)

👉 Giải thích:
- SLA được đặt là 24 giờ
- Dữ liệu nguồn đã cũ (~5 ngày)

👉 Nhận xét:
- Pipeline phát hiện được dữ liệu stale theo thời gian
- Không ảnh hưởng correctness hiện tại nhưng ảnh hưởng tính cập nhật

---

## 4. Corruption inject (Sprint 3)

### Cách inject

Sử dụng flag:
--no-refund-fix --skip-validate

**Các lỗi được tạo ra:**
- Stale policy:
"14 ngày làm việc" (refund sai)
- Bỏ qua validation:
Expectation FAIL nhưng vẫn embed vào vector DB

**Cách phát hiện:**

Expectation: 
refund_no_stale_14d_window = FAIL
Retrieval:
hits_forbidden = yes
- Nhận xét:
Pipeline có khả năng phát hiện lỗi ở cả:
tầng data validation
tầng retrieval (observability)

## 5. Hạn chế & việc chưa làm

- Chưa xử lý freshness tự động (chỉ detect, chưa auto refresh data)
- Chưa có alert/notification khi SLA bị vi phạm
- Chưa áp dụng framework như Great Expectations (chỉ dùng custom expectation)
- Dataset nhỏ, chưa test với dữ liệu lớn (scalability)

---🚀