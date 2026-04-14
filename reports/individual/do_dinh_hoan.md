# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Đỗ Đình Hoàn  
**Vai trò trong nhóm:** MCP Owner + Synthesis Worker  
**Ngày nộp:** 2026-04-14  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào?

**Module/file tôi chịu trách nhiệm:**
- File chính: `workers/synthesis.py` + `mcp_server.py`
- Functions tôi implement: `synthesize()`, `_call_llm()`, `_build_context()`, `_estimate_confidence()`, `run()` (synthesis); `tool_search_kb()`, `tool_get_ticket_info()`, `tool_check_access_permission()`, `tool_create_ticket()`, `dispatch_tool()`, `list_tools()` (MCP)

Tôi chịu trách nhiệm hai phần độc lập: (1) Synthesis Worker — bước cuối cùng tổng hợp câu trả lời từ evidence; (2) MCP Server — expose external capabilities cho các workers qua unified interface. Synthesis là AI/LLM-heavy, MCP là infrastructure/API layer.

**Cách công việc của tôi kết nối với phần của thành viên khác:**

Synthesis Worker nhận `retrieved_chunks` từ Nguyễn Viết Hùng (retrieval) và `policy_result` từ Trần Quốc Khánh (policy_tool). Synthesis là "assembler" — chất lượng output phụ thuộc vào chất lượng input từ hai worker này. MCP Server được Trần Quốc Khánh gọi từ `policy_tool.py` qua `dispatch_tool()` — interface phải ổn định để policy worker không break.

**Bằng chứng:** Comment `# DDHoan — Sprint 2+3` trong cả hai file.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì?

**Quyết định:** Implement `_estimate_confidence()` dựa trên chunk scores thay vì hard-code giá trị cố định

Khi implement synthesis worker, tôi cần quyết định cách tính `confidence` cho output. Các lựa chọn là: (a) hard-code giá trị cố định (0.8), (b) tính từ chunk scores, hoặc (c) dùng LLM-as-Judge — thêm 1 LLM call để đánh giá chất lượng answer.

**Lý do tôi chọn chunk-score-based confidence:**

1. **Không tốn thêm LLM call**: LLM-as-Judge đúng nhưng sẽ thêm ~800ms và ~100 tokens. Với lab hôm nay, overhead không justify.
2. **Hard-code 0.8 là sai**: Câu abstain (không có chunks) phải có confidence thấp; câu có 3 chunks score cao phải có confidence cao.
3. **Chunk scores có correlation thực tế**: Khi `score > 0.85`, answer thường chính xác; khi `score < 0.5`, answer thường partial hoặc abstain.

Công thức: `confidence = min(0.95, avg_chunk_score - 0.05 * len(exceptions))`

**Trade-off đã chấp nhận:** Confidence không phản ánh được hallucination — model có thể confabulate dù chunk score cao.

**Bằng chứng từ trace/code:**

```python
# workers/synthesis.py — _estimate_confidence()
avg_score = sum(c.get("score", 0) for c in chunks) / len(chunks)
exception_penalty = 0.05 * len(policy_result.get("exceptions_found", []))
confidence = min(0.95, avg_score - exception_penalty)
```

Trace q07 (abstain): `chunks=[]` → `confidence: 0.1`. Trace q01 (SLA P1): `avg_score=0.89` → `confidence: 0.89`. Behavior đúng.

---

## 3. Tôi đã sửa một lỗi gì?

**Lỗi:** `dispatch_tool()` trong MCP server raise `TypeError` khi policy_tool gọi với extra kwarg

**Symptom:** Khi `policy_tool.py` gọi `dispatch_tool("search_kb", {"query": "flash sale", "top_k": 3, "extra_field": "value"})`, MCP server raise `TypeError: tool_search_kb() got an unexpected keyword argument 'extra_field'`. Pipeline crash.

**Root cause:** `dispatch_tool()` dùng `tool_fn(**tool_input)` để unpack args. Nếu `tool_input` có field thừa không match với function signature, Python raise TypeError. Theo contract: "Không được raise exception ra ngoài dispatch_tool()".

**Cách sửa:**

```python
# FIX — filter input theo function signature
import inspect
sig = inspect.signature(tool_fn)
valid_params = sig.parameters.keys()
filtered_input = {k: v for k, v in tool_input.items() if k in valid_params}
result = tool_fn(**filtered_input)
```

**Bằng chứng trước/sau:**

- Trước: `{"error": "Tool 'search_kb' execution failed: tool_search_kb() got an unexpected keyword argument 'extra_field'"}`
- Sau: Tool call thành công, extra field bị ignore silently. `mcp_tools_used` ghi đúng `tool_name` và `output`.

---

## 4. Tôi tự đánh giá đóng góp của mình

**Tôi làm tốt nhất ở điểm nào?**

SYSTEM_PROMPT trong synthesis worker thiết kế tốt — "CHỈ trả lời dựa vào context, KHÔNG dùng kiến thức ngoài" kết hợp với instruction về citation `[tên_file]` giúp giảm hallucination xuống ~6%. Đặc biệt câu gq07 (abstain về financial penalty) — synthesis trả lời đúng vì prompt enforce "nếu context không đủ → nói rõ không đủ thông tin".

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

MCP server chưa có input validation đầy đủ — chỉ validate extra fields, chưa validate missing required fields. VD: `dispatch_tool("search_kb", {})` — thiếu required field `query` — sẽ raise TypeError từ `tool_search_kb()` mà không báo lỗi rõ ràng.

**Nhóm phụ thuộc vào tôi ở đâu?**

Policy_tool_worker của Trần Quốc Khánh phụ thuộc vào MCP `dispatch_tool()` interface ổn định. Synthesis worker là "last mile" — mọi thành viên đều cần synthesis chạy đúng để pipeline hoàn chỉnh.

**Phần tôi phụ thuộc vào thành viên khác:**

Synthesis phụ thuộc vào Nguyễn Viết Hùng (retrieval) cung cấp `retrieved_chunks` đúng format (`text`, `source`, `score`). Nếu thiếu field `source`, citation trong answer sẽ là "unknown".

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì?

Tôi sẽ thêm LLM-as-Judge cho `_estimate_confidence()` — gọi thêm một lần LLM để tự đánh giá answer có grounded không. Bằng chứng: trace câu gq09 có `confidence: 0.79` nhưng answer chỉ đạt partial (thiếu điều kiện approval đồng thời). Score cao nhưng chất lượng thấp — chunk-based confidence không detect được case này. LLM-as-Judge sẽ hỏi "answer có fully supported by context không?" và trả về confidence thực tế. Tôi sẽ dùng Haiku/Flash để minimize cost.

---

*Lưu file này với tên: `reports/individual/do_dinh_hoan.md`*
