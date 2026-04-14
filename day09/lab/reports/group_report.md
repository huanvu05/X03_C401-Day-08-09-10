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

Nhóm xây dựng Day 09 theo pattern Supervisor-Worker với 3 worker chính: `retrieval_worker`, `policy_tool_worker`, và `synthesis_worker`. `graph.py` làm supervisor, nhận task đầu vào và quyết định route dựa trên keyword logic. `retrieval_worker` thu thập evidence từ ChromaDB hoặc fallback local docs khi ChromaDB không khả dụng. `policy_tool_worker` áp dụng chính sách, gọi MCP khi cần, và `synthesis_worker` tổng hợp câu trả lời cuối cùng với thông tin evidence.

**Routing logic cốt lõi:**
> Mô tả logic supervisor dùng để quyết định route (keyword matching, LLM classifier, rule-based, v.v.)

Supervisor dùng rule-based keyword matching trong `graph.py` để quyết định route. Vì vậy:
- task chứa `refund`, `flash sale`, `access` hoặc `emergency` ưu tiên route vào `policy_tool_worker`
- task chứa `P1`, `SLA`, `ticket` ưu tiên route vào `retrieval_worker`
- task multi-hop SLA + access vẫn có thể gọi `retrieval_worker` trước rồi `policy_tool_worker` sau để bảo đảm evidence đủ.

`route_reason` được ghi vào trace để giải thích quyết định. Ví dụ `gq02` trong `artifacts/grading_run.jsonl` có `route_reason: "task contains policy/access keyword"` và `supervisor_route: "policy_tool_worker"`.

**MCP tools đã tích hợp:**
> Liệt kê tools đã implement và 1 ví dụ trace có gọi MCP tool.

- `search_kb`: `policy_tool_worker` dùng khi cần tìm chứng cứ chính sách trong KB. Trace `gq02`, `gq04`, `gq10` đều có `mcp_tools_used: ["search_kb"]`.
- `get_ticket_info`: `gq03` và `gq09` gọi để lấy thông tin ticket P1 và tăng độ chính xác quyết định.
- `check_access_permission`: `gq03` và `gq09` gọi khi nhiệm vụ yêu cầu access level/emergency override.

---

## 2. Quyết định kỹ thuật quan trọng nhất (200–250 từ)

> Chọn **1 quyết định thiết kế** mà nhóm thảo luận và đánh đổi nhiều nhất.
> Phải có: (a) vấn đề gặp phải, (b) các phương án cân nhắc, (c) lý do chọn phương án đã chọn.

**Quyết định:** Chọn route dựa trên rule-based keyword trong supervisor thay vì dùng một LLM classifier toàn diện.

**Bối cảnh vấn đề:**

Nhóm cần chuyển sang Day 09 với multi-agent mà vẫn giữ trace giải thích được. Lựa chọn là: dùng LLM classifier để phân loại intent hoặc duy trì route rule-based và giữ supervisor nhẹ.

**Các phương án đã cân nhắc:**

| Phương án | Ưu điểm | Nhược điểm |
|-----------|---------|-----------|
| LLM classifier | Có thể xử lý intent phức tạp, nhiều ngữ cảnh | Khó debug, route không rõ ràng trong trace |
| Rule-based keyword | Trace giải thích được, dễ sửa lỗi | Cần mở rộng keyword để tránh bỏ sót |

**Phương án đã chọn và lý do:**

Nhóm chọn rule-based keyword để tối ưu tính minh bạch và debugability. Trace của Day 09 cần chứng minh rõ worker nào chịu trách nhiệm, nên `route_reason` và `supervisor_route` phải rõ ràng. Khi chạy actual grading log, những câu như `gq01` và `gq09` cho phép chúng tôi xác định nhanh worker chậm/fast bằng cách xem trace mà không cần inspect output toàn diện.

**Bằng chứng từ trace/code:**

Trong `graph.py`, state được gán:

```
state["route_reason"] = "task contains policy/access keyword"
state["supervisor_route"] = "policy_tool_worker"
```

Trace thực tế `artifacts/grading_run.jsonl` ghi `workers_called` và `mcp_tools_used`, ví dụ `gq03` có `workers_called: ["retrieval_worker", "policy_tool_worker", "synthesis_worker"]`, cho thấy supervisor rule-based vẫn hỗ trợ multi-hop rõ ràng.

---

## 3. Kết quả grading questions (150–200 từ)

**Kết quả thực tế:**

Chúng tôi đã chạy grading questions và lưu 10 câu trong `artifacts/grading_run.jsonl`. Mỗi câu có trace thực tế, gồm `supervisor_route`, `workers_called`, `mcp_tools_used`, `confidence`, và `latency_ms`.

Do rubric raw score chưa có trong artifact, chúng tôi chưa thể báo điểm số chính xác. Tuy nhiên, qua trace có thể rút ra rằng pipeline đã xử lý tốt các câu refund/Flash Sale và multi-hop P1/access.

**Câu xử lý tốt nhất:**
- `gq02`, `gq04`, `gq10` là các câu chính sách refund. Tất cả đều route vào `policy_tool_worker` và gọi `search_kb`, với `confidence` 0.95.

**Câu multi-hop tốt:**
- `gq03` và `gq09` cần cả SLA và access. Trace cho thấy hai câu này đều gọi `retrieval_worker` trước rồi `policy_tool_worker`, và sử dụng `get_ticket_info` cùng `check_access_permission`.

**Câu gặp hạn chế:**
- `gq01`, `gq05`, `gq07` chỉ dùng `retrieval_worker` và `synthesis_worker`. Điều này cho thấy phần routing cần mở rộng keyword hoặc logic để phân biệt rõ hơn các tình huống ticket/SLA với access/emergency.

---

## 4. So sánh Day 08 vs Day 09 — Điều nhóm quan sát được (150–200 từ)

**Metric thay đổi rõ nhất:**

`artifacts/eval_report.json` cho thấy Day 09 có `avg_confidence` 0.907 và `avg_latency_ms` 17020. Multi-agent cũng dùng MCP cho 55% câu hỏi (`mcp_usage_rate: 11/20`) và chỉ 5% cần HITL (`hitl_rate: 1/20`).

**Điều bất ngờ:**

Multi-agent giúp chúng tôi thấy rõ ràng câu nào cần policy/MCP và câu nào chỉ cần retrieval. Trace `workers_called` và `mcp_tools_used` tạo ra một layer debug mà Day 08 không có.

**Khi multi-agent không giúp ích:**

Các câu đơn giản vẫn bị chậm hơn do overhead điều phối. `avg_latency_ms` 17020ms phản ánh chi phí của việc thêm supervisor và worker chaining, nhất là với câu chỉ cần một document như `gq01` hoặc `gq07`.

**Top sources:**

`eval_report.json` liệt kê top sources là `access_control_sop.txt`, `sla_p1_2026.txt`, `policy_refund_v4.txt`, `it_helpdesk_faq.txt`, và `hr_leave_policy.txt`, cho thấy Day 09 cần kết hợp policy và quy trình nội bộ.

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

Nhóm chia nhiệm vụ rõ ràng giữa routing/retrieval và MCP/policy. Việc lưu `workers_called` và `mcp_tools_used` giúp cả nhóm review trace dễ dàng và tìm ra nguyên nhân vấn đề nhanh chóng.

**Điểm cần cải thiện:**

Ban đầu chưa thống nhất đầy đủ keyword route cho multi-hop SLA + access. `gq03`/`gq09` cho thấy chúng tôi cần bổ sung các rule cho các task yêu cầu thông tin ticket và access cùng lúc.

---

## 6. Nếu có thêm 1 ngày, nhóm sẽ làm gì? (50–100 từ)

Nhóm sẽ mở rộng supervisor rule để xử lý tốt hơn các tình huống multi-hop SLA + access, và thêm bước đánh giá raw score dựa trên `grading_run.jsonl` để biết chính xác câu nào pass/fail. Đồng thời chúng tôi sẽ tối ưu fallback retrieval để dùng similarity document thay vì keyword khi `retrieved_chunks` rỗng.
