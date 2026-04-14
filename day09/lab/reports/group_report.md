# Báo Cáo Nhóm — Lab Day 09: Multi-Agent Orchestration

**Tên nhóm:**
X03-C401

**Thành viên:**
| Tên | Vai trò | Email |
|-----|---------|-------|
| Thắng | Supervisor Owner | hqt0810@gmail.com |
| Thắng | Worker Owner | hqt0810@gmail.com |
| Huân | MCP Owner | huanv3596@gmail.com |
| Huân | Trace & Docs Owner | huanv3596@gmail.com |

**Ngày nộp:** 14/04/2026

**Repo:** 
https://github.com/huanvu05/X03_C401-Day-08-09-10

**Độ dài khuyến nghị:** 600–1000 từ

---

> **Hướng dẫn nộp group report:**
> 
> - File này nộp tại: `reports/group_report.md`
> - Deadline: Được phép commit **sau 18:00** (xem SCORING.md)
> - Tập trung vào **quyết định kỹ thuật cấp nhóm** — không trùng lặp với individual reports
> - Phải có **bằng chứng từ code/trace** — không mô tả chung chung
> - Mỗi mục phải có ít nhất 1 ví dụ cụ thể từ code hoặc trace thực tế của nhóm

---

## 1. Kiến trúc nhóm đã xây dựng (150–200 từ)

> Mô tả ngắn gọn hệ thống nhóm: bao nhiêu workers, routing logic hoạt động thế nào,
> MCP tools nào được tích hợp. Dùng kết quả từ `docs/system_architecture.md`.

**Hệ thống tổng quan:**

Nhóm xây dựng hệ thống Day 09 theo pattern Supervisor-Worker với 3 worker chính: `retrieval_worker`, `policy_tool_worker`, và `synthesis_worker`. `graph.py` giữ vai trò supervisor, nhận task đầu vào và quyết định route dựa trên keyword. `retrieval_worker` tìm evidence từ ChromaDB hoặc fallback local docs, `policy_tool_worker` kiểm tra chính sách và gọi MCP khi cần, còn `synthesis_worker` tổng hợp câu trả lời dựa trên context đã thu thập.

**Routing logic cốt lõi:**
> Mô tả logic supervisor dùng để quyết định route (keyword matching, LLM classifier, rule-based, v.v.)

Supervisor dùng rule-based keyword matching, không dùng classifier. Nếu task chứa refund/flash sale/access/emergency thì route sang `policy_tool_worker`; nếu task chứa P1/SLA/ticket thì route sang `retrieval_worker`; với các câu multi-hop pha SLA + access thì hệ thống vẫn gọi retrieval trước rồi policy_tool để đảm bảo có evidence. `route_reason` được ghi rõ trong state để trace.

**MCP tools đã tích hợp:**
> Liệt kê tools đã implement và 1 ví dụ trace có gọi MCP tool.

- `search_kb`: gọi khi `policy_tool_worker` cần thêm evidence từ KB; trace ghi `mcp_tools_used` có `tool=search_kb`.
- `get_ticket_info`: dùng cho câu hỏi liên quan ticket P1; trace lưu `ticket_info` trong `policy_result`.
- `check_access_permission`: kiểm tra access level và emergency override cho trường hợp cấp quyền.

---

## 2. Quyết định kỹ thuật quan trọng nhất (200–250 từ)

> Chọn **1 quyết định thiết kế** mà nhóm thảo luận và đánh đổi nhiều nhất.
> Phải có: (a) vấn đề gặp phải, (b) các phương án cân nhắc, (c) lý do chọn phương án đã chọn.

**Quyết định:** Chọn route dựa trên rule-based keyword trong supervisor thay vì dùng một LLM classifier toàn diện.

**Bối cảnh vấn đề:**

Nhóm cần tách pipeline Day 08 sang Day 09 nhưng vẫn đảm bảo trace rõ ràng và worker có thể test độc lập. Một phương án là dùng LLM phân loại intent, phương án còn lại là dùng rule-based từ khóa đơn giản để giữ route giải thích được.

**Các phương án đã cân nhắc:**

| Phương án | Ưu điểm | Nhược điểm |
|-----------|---------|-----------|
| LLM classifier | Có thể xử lý ý định phức tạp hơn và multi-hop mềm dẻo | Khó debug, dễ thay đổi route mà trace không giải thích trực tiếp |
| Rule-based keyword | Route giải thích rõ, trace dễ đọc và code đơn giản | Có thể bỏ sót biến thể văn bản nếu từ khóa không đủ đủ phủ |

**Phương án đã chọn và lý do:**

Nhóm chọn rule-based keyword để ưu tiên trace và debugability. Với mục tiêu Day 09 ta cần chứng minh khả năng route rõ ràng và gọi worker đúng, nên việc `route_reason` được ghi ra là giá trị lớn hơn so với classifier có thể làm mờ cơ chế. Đây cũng là lý do chúng tôi giữ logic supervisor trong `graph.py` thay vì đẩy tất cả vào LLM.

**Bằng chứng từ trace/code:**
> Dẫn chứng cụ thể (VD: route_reason trong trace, đoạn code, v.v.)

```
state["route_reason"] = "task contains policy/access keyword"
state["supervisor_route"] = "policy_tool_worker"
```

Trace ví dụ `artifacts/traces/run_20260414_120111.json` ghi rõ `supervisor_route` và `workers_called`, giúp xác định ngay worker chịu trách nhiệm.

---

## 3. Kết quả grading questions (150–200 từ)

> Sau khi chạy pipeline với grading_questions.json (public lúc 17:00):
> - Nhóm đạt bao nhiêu điểm raw?
> - Câu nào pipeline xử lý tốt nhất?
> - Câu nào pipeline fail hoặc gặp khó khăn?

**Tổng điểm raw ước tính:** Chưa có grading log chính thức do file `grading_questions.json` chưa được public hoặc chưa chạy trong thời điểm báo cáo này.

**Câu pipeline xử lý tốt nhất:**
- ID: chưa có cụ thể — nhưng qua test nội bộ, hệ thống ghi nhận `policy_tool_worker` xử lý tốt các câu refund/Flash Sale với `exceptions_found` rõ ràng.

**Câu pipeline fail hoặc partial:**
- ID: chưa rõ do chưa có grading log — quan sát trace cho thấy các câu multi-hop cần cả SLA và access đôi khi route sang `policy_tool_worker` trước nhưng vẫn cần retrieval bổ trợ.
  Root cause: thiếu evidence hoặc rule-based route thiếu từ khóa phủ hết.

**Câu gq07 (abstain):** Nhóm xử lý thế nào?

Ngay cả khi không có grading question chính thức, hệ thống được thiết kế để abstain rõ ràng khi `retrieved_chunks` rỗng. `synthesis_worker` trả về "Không có thông tin trong tài liệu" với confidence thấp 0.1 thay vì cố gắng suy đoán.

**Câu gq09 (multi-hop khó nhất):** Trace ghi được 2 workers không? Kết quả thế nào?

Thiết kế hiện tại đảm bảo multi-hop SLA + access sẽ gọi `retrieval_worker` và `policy_tool_worker`. Trong ví dụ test, `workers_called` có đủ `retrieval_worker` và `policy_tool_worker` trước khi vào `synthesis_worker`, nên trace hỗ trợ xác nhận multi-hop đã được xét qua hai luồng.

---

## 4. So sánh Day 08 vs Day 09 — Điều nhóm quan sát được (150–200 từ)

> Dựa vào `docs/single_vs_multi_comparison.md` — trích kết quả thực tế.

**Metric thay đổi rõ nhất (có số liệu):**

Day 09 có `avg_confidence` 0.907 và `mcp_usage_rate` 55% theo `artifacts/eval_report.json`. So với Day 08 không có log confidence, Day 09 rõ ràng hơn về chất lượng và trace. Tuy nhiên latency tăng lên `17020ms` do multi-step.

**Điều nhóm bất ngờ nhất khi chuyển từ single sang multi-agent:**

Multi-agent không chỉ giúp debug nhanh hơn, mà còn cho thấy ngay câu nào cần MCP và câu nào chỉ cần retrieval. Việc trace lưu `workers_called` và `mcp_tools_used` làm rõ trách nhiệm của từng phần.

**Trường hợp multi-agent KHÔNG giúp ích hoặc làm chậm hệ thống:**

Đối với các câu đơn giản chỉ cần một document, multi-agent vẫn chậm hơn do thêm bước điều phối và kiểm tra policy. `avg_latency_ms` tăng một lượng đáng kể, điều này là chi phí đổi lấy khả năng trace và extensibility.

---

## 5. Phân công và đánh giá nhóm (100–150 từ)

> Đánh giá trung thực về quá trình làm việc nhóm.

**Phân công thực tế:**

| Thành viên | Phần đã làm | Sprint |
|------------|-------------|--------|
| Thắng | Supervisor logic, `graph.py`, route decision | Sprint 1 |
| Thắng | Retrieval integration, fallback docs search | Sprint 2 |
| Huân | MCP server, `policy_tool.py`, policy exception handling | Sprint 3 |
| Huân | Trace evaluation, `eval_trace.py` support, docs và comparison | Sprint 4 |

**Điều nhóm làm tốt:**

Chúng tôi phối hợp rõ ràng theo sprint: một người chịu trách nhiệm phần core routing/retrieval, người kia chịu trách nhiệm phần MCP, policy và trace. Việc có `route_reason` và `mcp_tools_used` trong trace giúp cả nhóm đồng thuận về design.

**Điều nhóm làm chưa tốt hoặc gặp vấn đề về phối hợp:**

Ban đầu chưa thống nhất đủ các keyword routing, dẫn đến một số query multi-hop bị route sai. Cần thêm check kỹ hơn cho trường hợp SLA + access để tránh mất trace.

**Nếu làm lại, nhóm sẽ thay đổi gì trong cách tổ chức?**

Nhóm sẽ dành thêm thời gian review sớm hơn cho `supervisor_node` và `policy_tool` để thống nhất routing keywords, đồng thời phân chia test cases rõ ràng cho từng worker ngay từ sprint 1.

---

## 6. Nếu có thêm 1 ngày, nhóm sẽ làm gì? (50–100 từ)

> 1–2 cải tiến cụ thể với lý do có bằng chứng từ trace/scorecard.

Nhóm sẽ thêm hai cải tiến: hoàn thiện grading run với `grading_questions.json` để có điểm raw cụ thể, và nâng cấp `policy_tool.py` thành LLM-assisted để xử lý exception phức tạp hơn. Cùng lúc đó sẽ cải thiện fallback retrieval bằng chỉ số similarity cao hơn thay vì random embeddings.

---

*File này lưu tại: `reports/group_report.md`*  
*Commit sau 18:00 được phép theo SCORING.md*
