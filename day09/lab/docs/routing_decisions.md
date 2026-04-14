# Routing Decisions Log — Lab Day 09

**Nhóm:** X03-C401
**Ngày:** 14/04/2026

> **Hướng dẫn:** Ghi lại ít nhất **3 quyết định routing** thực tế từ trace của nhóm.
> Không ghi giả định — phải từ trace thật (`artifacts/traces/`).
>
> Mỗi entry phải có: task đầu vào → worker được chọn → route_reason → kết quả thực tế.

---

## Routing Decision #1


**Task đầu vào:**
> Ticket P1 được tạo lúc 22:47. Đúng theo SLA, ai nhận thông báo đầu tiên và qua kênh nào? Deadline escalation là mấy giờ?

**Worker được chọn:** `retrieval_worker`
**Route reason (từ trace):** `default route`
**MCP tools được gọi:** None
**Workers called sequence:** `retrieval_worker` → `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): Câu trả lời tổng hợp từ tài liệu SLA (nguồn: sla_p1_2026.txt)
- confidence: 0.75
- Correct routing? Not evaluated (grading log only)

**Nhận xét:** Routing theo mặc định (retrieval) hợp lý vì câu hỏi yêu cầu trích xuất điều khoản SLA.

_________________

---

## Routing Decision #2

**Task đầu vào:**
> Khách hàng đặt đơn ngày 31/01/2026 và gửi yêu cầu hoàn tiền ngày 07/02/2026 vì lỗi nhà sản xuất. Sản phẩm chưa kích hoạt, không phải Flash Sale, không phải kỹ thuật số. Chính sách nào áp dụng và có được hoàn tiền không?

**Worker được chọn:** `policy_tool_worker`
**Route reason (từ trace):** `task contains policy/access keyword`
**MCP tools được gọi:** None
**Workers called sequence:** `policy_tool_worker` → `retrieval_worker` → `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): Câu trả lời tổng hợp từ policy (1 chunk)
- confidence: 0.75
- Correct routing? Not evaluated (grading log only)

**Nhận xét:** Route chọn `policy_tool_worker` phù hợp vì câu hỏi liên quan tới chính sách.

_________________

---

## Routing Decision #3

**Task đầu vào:**
> Engineer cần Level 3 access để khắc phục P1 đang active. Bao nhiêu người phải phê duyệt? Ai là người phê duyệt cuối cùng (người phê duyệt có thẩm quyền cao nhất)?

**Worker được chọn:** `policy_tool_worker`
**Route reason (từ trace):** `task contains policy/access keyword`
**MCP tools được gọi:** None
**Workers called sequence:** `policy_tool_worker` → `retrieval_worker` → `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): Câu trả lời tổng hợp từ policy (1 chunk)
- confidence: 0.75
- Correct routing? Not evaluated (grading log only)

**Nhận xét:** Policy worker là lựa chọn đúng để trả lời câu hỏi về quy trình phê duyệt.

_________________

---



## Tổng kết

### Routing Distribution

| Worker | Số câu được route | % tổng |
|--------|------------------|--------|
| retrieval_worker | 5 | 50% |
| policy_tool_worker | 5 | 50% |
| human_review | 0 | 0% |

### Routing Accuracy

> Trong số X câu nhóm đã chạy, bao nhiêu câu supervisor route đúng?

- Câu route đúng: 8 / 10 
- Câu route sai (đã sửa bằng cách nào?): 2 câu (gq06, gq08). Hai câu này hỏi về "quy định đổi mật khẩu" và "điều kiện làm remote"  (bản chất là policy), nhưng bị đẩy vào retrieval_worker với lý do default route.
  - Cách sửa: Cần mở rộng bộ từ khóa (thêm "quy định", "điều kiện", "mật khẩu") trong rule-based classifier, hoặc nâng cấp lên dùng LLM Intent Classifier thay vì chỉ dùng keyword matching.
- Câu trigger HITL: 0 (Thuộc tính hitl_triggered luôn là false do confidence đang fix cứng ở 0.75).

### Lesson Learned về Routing

> Quyết định kỹ thuật quan trọng nhất nhóm đưa ra về routing logic là gì?  
> (VD: dùng keyword matching vs LLM classifier, threshold confidence cho HITL, v.v.)

1. Điểm hạn chế của Keyword Matching: Logic hiện tại đang phụ thuộc vào việc tìm keyword (như "policy", "access"). Quyết định này giúp hệ thống phân luồng nhanh nhưng kém linh hoạt. Các câu hỏi mang tính chất chính sách nhưng không chứa từ khóa trực tiếp sẽ bị rớt thẳng xuống default route (như câu hỏi về probation hoặc mật khẩu)
2. Thiếu tính động trong Confidence Score: Hệ thống đang chưa kích hoạt được luồng Human-in-the-loop (HITL). Quyết định kỹ thuật tiếp theo phải là xây dựng một hàm tính confidence động dựa trên độ tương đồng (similarity score) của vector retrieval, để tự động trigger human_review khi điểm số rơi xuống dưới ngưỡng an toàn (ví dụ: < 0.6).

### Route Reason Quality

> Nhìn lại các `route_reason` trong trace — chúng có đủ thông tin để debug không?  
> Nếu chưa, nhóm sẽ cải tiến format route_reason thế nào?


- Các route_reason hiện tại (như default route hay task contains policy/access keyword ) mới chỉ dừng ở mức báo cáo rule nào được thỏa mãn, chưa đủ chi tiết để debug.

- Cách cải tiến format:
  - Sửa từ: task contains policy/access keyword thành: Rule match: policy_tool_worker | Trigger keyword matched: "hoàn tiền" hoặc Trigger keyword matched: "access".
  - Điều này sẽ giúp nhóm biết chính xác từ nào trong câu hỏi đã bẻ hướng routing, từ đó dễ tuning bộ keyword hơn.
