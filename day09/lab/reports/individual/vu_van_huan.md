# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Vũ Văn Huân  
**Vai trò trong nhóm:** MCP Owner / Trace & Docs Owner  
**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- File chính: `mcp_server.py`, `workers/policy_tool.py`, `eval_trace.py`, `docs/system_architecture.md`, `docs/single_vs_multi_comparison.md`, `reports/group_report.md`
- Functions tôi implement: `tool_get_ticket_info`, `tool_check_access_permission`, `dispatch_tool` trong `mcp_server.py`; `_call_mcp_tool`, `run` trong `workers/policy_tool.py`; `run_grading_questions`, `analyze_traces`, `compare_ab` trong `eval_trace.py`

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Tôi làm phần MCP API và trace analytics, nên phần của tôi nhận input từ supervisor (`graph.py`) và retrieval (`retrieval_worker`) rồi trả kết quả thành `policy_result`/`mcp_tools_used` cho `synthesis_worker`. Tôi cũng chịu trách nhiệm viết docs, nên các trace thực tế tôi tạo ra được dùng trực tiếp trong `reports/group_report.md` và `docs/single_vs_multi_comparison.md`.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**
- File `mcp_server.py` có tool definitions và dispatch interface
- File `workers/policy_tool.py` có comment `# MCP Client — Sprint 3` và `state['mcp_tools_used']`
- `eval_trace.py` ghi rõ `mcp_tools_used` và `route_reason` vào `artifacts/grading_run.jsonl`

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi quyết định giữ phần logic MCP ở `policy_tool_worker` và dùng trace để ghi rõ `mcp_tools_used`, thay vì điều khiển toàn bộ việc gọi MCP từ supervisor hoặc giấu tool call trong một prompt lớn.

**Lý do:**
- Cách này giúp giữ supervisor nhẹ và chỉ chuyên trách routing.
- `policy_tool_worker` là nơi duy nhất biết rõ policy/ access và nên chịu trách nhiệm gọi `search_kb`, `get_ticket_info`, `check_access_permission`.
- Trace cần minh bạch để thuyết phục giám khảo Day 09: `SCORING.md` yêu cầu `route_reason` và tool usage rõ ràng.

**Trade-off đã chấp nhận:**
- Tôi đánh đổi độ đơn giản của flow bằng việc thêm nhiều tool call và state field, nhưng đổi lại hệ thống có thể giải thích được vì mỗi câu hỏi để lại `mcp_tools_used` và `policy_result` rõ ràng.
- Điều này làm latency cao hơn, nhưng phù hợp với yêu cầu trace & docs.

**Bằng chứng từ trace/code:**
```
state["mcp_tools_used"].append(mcp_result)
state["history"].append(f"[{WORKER_NAME}] called MCP get_ticket_info")
```

Trace thực tế trong `artifacts/grading_run.jsonl` cho thấy:
- `gq02`, `gq04`, `gq10` gọi `search_kb`
- `gq03`, `gq09` gọi `get_ticket_info` và `check_access_permission`
Điều này khẳng định quyết định thiết kế MCP tool ở worker là đúng.

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Grading log ban đầu không ghi `mcp_tools_used` ở dạng list tool names và `route_reason` chưa được bảo toàn trong output JSONL.

**Symptom (pipeline làm gì sai?):**
- `artifacts/grading_run.jsonl` không phù hợp với định dạng trace yêu cầu của `SCORING.md`.
- Điều này khiến file grading không thể dùng để kiểm tra `mcp_tools_used` và `route_reason` cho từng câu.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**
- Lỗi nằm ở `eval_trace.py` trong phần ghi record: output của `mcp_tools_used` có thể là dict phức tạp và `route_reason` chưa được đảm bảo.

**Cách sửa:**
- Tôi sửa `eval_trace.py` để serialize `mcp_tools_used` thành danh sách tool names:
  `"mcp_tools_used": [t.get("tool") for t in result.get("mcp_tools_used", [])]`
- Tôi đảm bảo `route_reason` luôn ghi vào record và xuất vào `artifacts/grading_run.jsonl`.

**Bằng chứng trước/sau:**
> Trước sửa: `mcp_tools_used` có thể không thống nhất; `route_reason` thiếu.
> Sau sửa: `gq03` trong grading log có `mcp_tools_used: ["get_ticket_info", "check_access_permission"]` và `route_reason: "task contains policy/access keyword"`.

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tôi làm tốt nhất ở việc thiết kế trace và MCP interface sao cho dễ kiểm tra. `eval_trace.py` giờ tạo kết quả rõ ràng, `mcp_server.py` cung cấp tool mock chất lượng, và `policy_tool.py` có workflow kiểm tra chính sách + gọi tool hợp lý.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Tôi còn yếu ở phần xử lý ticket ID thật sự trong `policy_tool.py`: hiện tại `get_ticket_info` vẫn dùng ticket mock cố định `P1-LATEST`. Nếu có thêm thời gian, tôi sẽ parse ticket ID từ câu hỏi để tool trả về dữ liệu cụ thể hơn.

**Nhóm phụ thuộc vào tôi ở đâu?**
Nhóm phụ thuộc tôi ở phần trace và docs. Nếu tôi chưa xong, thì `artifacts/grading_run.jsonl`, `eval_report.json`, và các báo cáo so sánh Day 08/09 sẽ thiếu dữ liệu hoặc format sai.

**Phần tôi phụ thuộc vào thành viên khác:**
Tôi cần `graph.py` và `retrieval_worker` chạy đúng để trace có dữ liệu hợp lệ. Nếu supervisor route sai hoặc retrieval không trả chunks, thì `policy_tool_worker` và `eval_trace.py` không thể tạo trace đúng.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ cải tiến `policy_tool.py` để parse ticket ID trực tiếp từ câu hỏi và dùng `tool_get_ticket_info` với ID chính xác. Trace `gq03`/`gq09` cho thấy công cụ access cần thông tin ticket rõ ràng hơn; hiện tại nó chỉ trả mock chung nên kết quả policy vẫn chưa hoàn toàn chính xác.

---

