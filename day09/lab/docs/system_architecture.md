# System Architecture — Lab Day 09

**Nhóm:** X03-C401
**Ngày:** 14/04/2026
**Version:** 1.0

---

## 1. Tổng quan kiến trúc

> Mô tả ngắn hệ thống của nhóm: chọn pattern gì, gồm những thành phần nào.

**Pattern đã chọn:** Supervisor-Worker  
**Lý do chọn pattern này (thay vì single agent):**

Supervisor-Worker cho phép tách riêng phần điều phối route, tìm kiếm bằng chứng và tổng hợp câu trả lời. Cách này giúp trace rõ ràng hơn, dễ mở rộng MCP/tool và giảm rủi ro khi cần debug hoặc cập nhật từng phần.

---

## 2. Sơ đồ Pipeline

> Vẽ sơ đồ pipeline dưới dạng text, Mermaid diagram, hoặc ASCII art.
> Yêu cầu tối thiểu: thể hiện rõ luồng từ input → supervisor → workers → output.

**Sơ đồ thực tế của nhóm:**

```
User Request
     │
     ▼
┌──────────────┐
│  Supervisor  │  ← route_reason, risk_high, needs_tool
└──────┬───────┘
       │
   [route_decision]
       │
  ┌────┴────────────────────┐
  │                         │
  ▼                         ▼
Retrieval Worker     Policy Tool Worker
  (evidence)           (policy check + MCP)
  │                         │
  └─────────┬───────────────┘
            │
            ▼
      Synthesis Worker
        (answer + cite)
            │
            ▼
         Output
```

---

## 3. Vai trò từng thành phần

### Supervisor (`graph.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **Nhiệm vụ** | Điều phối và lựa chọn worker dựa trên nội dung câu hỏi và đánh giá rủi ro. |
| **Input** | Task từ user và state ban đầu (history, flags). |
| **Output** | supervisor_route, route_reason, risk_high, needs_tool |
| **Routing logic** | Policy/access keyword → policy_tool_worker; ticket/SLA keyword → retrieval_worker; else fallback retrieval. |
| **HITL condition** | Nếu task chứa lỗi không rõ `err-` và risk_high, trigger human_review trước khi tiếp tục. |

### Retrieval Worker (`workers/retrieval.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **Nhiệm vụ** | Tìm kiếm evidence phù hợp từ ChromaDB hoặc fallback local docs. |
| **Embedding model** | `all-MiniLM-L6-v2` nếu có, hoặc OpenAI embeddings, nếu không thì random fallback. |
| **Top-k** | 3 |
| **Stateless?** | Yes |

### Policy Tool Worker (`workers/policy_tool.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **Nhiệm vụ** | Phân tích policy, nhận diện exceptions và gọi MCP tools khi cần thêm thông tin. |
| **MCP tools gọi** | `search_kb`, `get_ticket_info`, `check_access_permission` |
| **Exception cases xử lý** | Flash Sale refund, digital product/license, activated product, access control emergency, SLA ticket lookup. |

### Synthesis Worker (`workers/synthesis.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **LLM model** | `gpt-4o-mini` if available, otherwise fallback generation logic. |
| **Temperature** | 0.1 |
| **Grounding strategy** | Dùng context từ `retrieved_chunks` và `policy_result`, chỉ trả lời theo tài liệu. |
| **Abstain condition** | Nếu không có evidence hoặc insufficient info, trả về "Không có thông tin trong tài liệu". |

### MCP Server (`mcp_server.py`)

| Tool | Input | Output |
|------|-------|--------|
| search_kb | query, top_k | chunks, sources |
| get_ticket_info | ticket_id | ticket details |
| check_access_permission | access_level, requester_role | can_grant, approvers |
| create_ticket | priority, title, description | ticket_id, url, created_at, note |

---

## 4. Shared State Schema

> Liệt kê các fields trong AgentState và ý nghĩa của từng field.

| Field | Type | Mô tả | Ai đọc/ghi |
|-------|------|-------|-----------|
| task | str | Câu hỏi đầu vào | supervisor đọc |
| supervisor_route | str | Worker được chọn | supervisor ghi |
| route_reason | str | Lý do route | supervisor ghi |
| retrieved_chunks | list | Evidence từ retrieval | retrieval ghi, synthesis đọc |
| policy_result | dict | Kết quả kiểm tra policy | policy_tool ghi, synthesis đọc |
| mcp_tools_used | list | Tool calls đã thực hiện | policy_tool ghi |
| final_answer | str | Câu trả lời cuối | synthesis ghi |
| confidence | float | Mức tin cậy | synthesis ghi |
| hitl_triggered | bool | Flag khi supervisor quyết định cần human review | supervisor ghi |

---

## 5. Lý do chọn Supervisor-Worker so với Single Agent (Day 08)

| Tiêu chí | Single Agent (Day 08) | Supervisor-Worker (Day 09) |
|----------|----------------------|--------------------------|
| Debug khi sai | Khó — không rõ lỗi ở đâu | Dễ hơn — test từng worker độc lập |
| Thêm capability mới | Phải sửa toàn prompt | Thêm worker/MCP tool riêng |
| Routing visibility | Không có | Có route_reason trong trace |
| Trace detail | Chỉ có output cuối | Có workers_called và mcp_tools_used cũng lưu lại |

**Nhóm điền thêm quan sát từ thực tế lab:**

Supervisor-Worker giúp xác định nhanh worker nào xử lý query, nên debug multi-hop SLA + access request dễ hơn. Trace log rõ ràng giúp chứng minh pipeline chạy đúng theo routing pattern, không chỉ dựa vào output cuối.

---

## 6. Giới hạn và điểm cần cải tiến

> Nhóm mô tả những điểm hạn chế của kiến trúc hiện tại.

1. Fallback embedding và retrieval hiện tại có chất lượng không đồng đều khi ChromaDB chưa sẵn sàng. 
2. Policy worker vẫn là rule-based, cần mở rộng bằng LLM hoặc thêm nhiều case policy hơn. 
3. HITL node hiện chỉ auto-approve placeholder, cần thực hiện review thực tế khi triển khai sản phẩm. 
