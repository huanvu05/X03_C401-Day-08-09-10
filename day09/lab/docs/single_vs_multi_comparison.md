# Single Agent vs Multi-Agent Comparison — Lab Day 09

**Nhóm:** X03-C401
**Ngày:** 14/04/2026

> **Hướng dẫn:** So sánh Day 08 (single-agent RAG) với Day 09 (supervisor-worker).
> Phải có **số liệu thực tế** từ trace — không ghi ước đoán.
> Chạy cùng test questions cho cả hai nếu có thể.

---

## 1. Metrics Comparison

> Dữ liệu lấy từ Day 08 `python eval.py` và Day 09 `python eval_trace.py`.

| Metric                | Day 08 (Single Agent) | Day 09 (Multi-Agent) | Delta    | Ghi chú                                            |
| --------------------- | --------------------- | -------------------- | -------- | -------------------------------------------------- |
| Avg confidence        | N/A                   | 0.901                | N/A      | Day 08 không đo confidence rõ ràng                 |
| Avg latency (ms)      | ≈5,000*               | 13,081               | +8,081   | Day 09 chậm hơn do supervisor + worker chaining    |
| Hitl / low-confidence | N/A                   | 5%                   | N/A      | Day 09 có 2/35 trace cases hitl-triggered          |
| Multi-hop support     | limited / opaque      | stronger / traceable | +        | Day 09 có evidence chain và worker sequence        |
| Routing visibility    | ✗                     | ✓                    | +        | Day 09 có `route_reason` trong trace               |
| MCP usage             | N/A                   | 51%                  | +51%     | Day 09 dùng MCP tools cho 18/35 traces            |

*Day 08 latency không được đo trực tiếp bởi `eval.py`.

---

## 2. Phân tích theo loại câu hỏi

### 2.1 Câu hỏi đơn giản (single-document)

| Nhận xét    | Day 08                                                                     | Day 09                                                                    |
| ----------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Accuracy    | Rất tốt về faithfulness/relevance, nhưng thiếu route visibility             | Tốt, nhiều câu có confidence 0.95 và output ổn định                        |
| Latency     | Thấp hơn do pipeline đơn giản hơn                                           | 13,081 ms trung bình                                                        |
| Observation | Câu đơn giản hoạt động tốt, nhưng khi cần debug thì thiếu thông tin route   | Câu đơn giản vẫn đúng, dễ xác định worker chịu trách nhiệm nếu lỗi        |

**Kết luận:** Multi-agent cải thiện khả năng debug và trace cho câu đơn giản, dù chi phí latency cao hơn.

_________________

### 2.2 Câu hỏi multi-hop (cross-document)

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Accuracy | Kém hơn và thiếu trace, nhiều câu variant trả về “Không đủ dữ liệu” | Tốt hơn, nhiều câu multi-hop được route đúng và có evidence rõ ràng |
| Routing visible? | ✗ | ✓ |
| Observation | Day 08 dễ lẫn giữa policy và access, khó localize lỗi | Day 09 cho thấy rõ `retrieval_worker` → `policy_tool_worker` → `synthesis_worker` |

**Kết luận:** Day 09 có lợi thế lớn cho câu multi-hop nhờ trace và worker phân tách rõ ràng.

_________________

### 2.3 Câu hỏi cần abstain

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Abstain / low-confidence | Không rõ chính xác, nhưng variant nhiều lần trả về “Không đủ dữ liệu” | 5% hitl/low-conf cases |
| Hallucination cases | Khó đánh giá, vì không có trace rõ | Giảm nhờ evidence và low-confidence detection |
| Observation | Day 08 thiếu cơ chế review, dễ blind spot | Day 09 có trace, hitl và confidence để kiểm soát |

**Kết luận:** Multi-agent hỗ trợ cơ chế kiểm soát tốt hơn cho câu cần abstain hoặc human review.

---

## 3. Debuggability Analysis

> Khi pipeline trả lời sai, mất bao lâu để tìm ra nguyên nhân?

### Day 08 — Debug workflow
```
Khi answer sai → kiểm tra toàn bộ RAG pipeline và prompt → tìm lỗi ở retrieval/generation
Không có trace route_reason → khó xác định nguyên nhân
Ước tính: 25–35 phút
```

### Day 09 — Debug workflow
```
Khi answer sai → mở trace → xem `supervisor_route` và `route_reason`
  → nếu route sai → sửa `graph.py`
  → nếu retrieval sai → test `retrieval_worker`
  → nếu policy sai → test `policy_tool_worker`
  → nếu synthesis sai → test `synthesis_worker`
Ước tính: 8–12 phút
```

**Ví dụ thực tế:**
Trong `gq03` và `gq09`, trace cho thấy cả `retrieval_worker` và `policy_tool_worker` đều được gọi, nên nhóm chỉ cần điều chỉnh supervisor rule và MCP tool mà không phải debug toàn bộ pipeline.

---

## 4. Extensibility Analysis

> Dễ extend thêm capability không?

| Scenario | Day 08 | Day 09 |
|---------|--------|--------|
| Thêm 1 tool/API mới | Sửa prompt hoặc pipeline core | Thêm MCP tool và route rule |
| Thêm 1 domain mới | Sửa lại prompt/knowledge và pipeline | Thêm worker/domain-specific logic |
| Thay đổi retrieval strategy | Sửa trong `rag_answer` | Chỉ sửa `retrieval_worker` |
| A/B test một phần | Khó, phải thay đổi toàn bộ pipeline | Dễ hơn, swap worker hoặc rule |

**Nhận xét:**
Day 09 dễ mở rộng hơn nhờ worker/module tách biệt và trace flow rõ ràng.

---

## 5. Cost & Latency Trade-off

> Multi-agent thường tốn nhiều LLM calls hơn.

| Scenario | Day 08 calls | Day 09 calls |
|---------|-------------|-------------|
| Simple query | 1 LLM call | 1–2 calls |
| Complex query | 1 LLM call | 2–3 calls |
| MCP tool call | N/A | 51% câu |

**Nhận xét về cost-benefit:**
Day 09 có chi phí tính toán và latency cao hơn, nhưng đổi lại được trace rõ ràng, routing giải thích được và khả năng mở rộng MCP.

---

## 6. Kết luận

> **Multi-agent tốt hơn single agent ở điểm nào?**

1. Trace và routing visibility rõ ràng.
2. Debug nhanh hơn nhờ trace và worker phân tách.
3. Hỗ trợ multi-hop và policy queries tốt hơn.

> **Multi-agent kém hơn hoặc không khác biệt ở điểm nào?**

1. Latency cao hơn: Day 09 trung bình 13,081 ms.
2. Hệ thống phức tạp hơn, cần thêm bước supervisor và worker coordination.

> **Khi nào KHÔNG nên dùng multi-agent?**

Khi câu hỏi đơn giản và cần phản hồi nhanh, hoặc khi tài nguyên LLM bị hạn chế.

> **Nếu tiếp tục phát triển hệ thống này, nhóm sẽ thêm gì?**

Thêm raw scoring từ grading run, tinh chỉnh supervisor rule cho multi-hop SLA+access, và tối ưu worker chaining để giảm latency.
