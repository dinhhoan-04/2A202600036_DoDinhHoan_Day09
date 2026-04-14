# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Trần Quốc Khánh  
**Vai trò trong nhóm:** Worker Owner (Policy Tool) + Contracts Owner  
**Ngày nộp:** 2026-04-14  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào?

**Module/file tôi chịu trách nhiệm:**
- File chính: `workers/policy_tool.py` + `contracts/worker_contracts.yaml`
- Functions tôi implement: `analyze_policy()`, `_call_mcp_tool()`, `run()` (policy_tool.py); toàn bộ `worker_contracts.yaml`

Tôi implement Policy Tool Worker — worker chịu trách nhiệm kiểm tra chính sách hoàn tiền, quyền truy cập, và các exception cases. Worker này là "domain expert" về policy: khi supervisor route task liên quan đến refund/access sang tôi, tôi phải trả lời đúng exception nào áp dụng. Tôi cũng viết `worker_contracts.yaml` — tài liệu định nghĩa interface của tất cả workers và MCP server.

**Cách công việc của tôi kết nối với phần của thành viên khác:**

`policy_result` tôi ghi vào state là input quan trọng cho synthesis_worker của Đỗ Đình Hoàn — đặc biệt field `exceptions_found`. Nếu tôi detect sai exception (VD: Flash Sale nhưng không detect), synthesis sẽ trả lời sai "được hoàn tiền". Contracts YAML tôi viết là interface spec cho cả nhóm — nếu thiếu field hoặc type sai, các thành viên khác implement không khớp.

**Bằng chứng:** Comment `# TQKhanh — Sprint 2` trong `policy_tool.py`, và `worker_contracts.yaml` có chữ ký trong commit message.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì?

**Quyết định:** Implement `analyze_policy()` bằng rule-based detection thay vì LLM call

Policy worker cần phân tích xem task có thuộc exception case nào không (Flash Sale, digital product, activated product, temporal scoping). Hai cách: (a) rule-based keyword check, hoặc (b) gọi LLM với context để phân tích.

**Lý do tôi chọn rule-based:**

1. **Exception cases là finite và well-defined**: Theo `policy_refund_v4.txt`, chỉ có 3 exception categories rõ ràng. Không cần LLM để detect chúng nếu task và context chứa keywords.
2. **Deterministic**: Rule-based cho kết quả nhất quán — quan trọng khi grading yêu cầu reproducible trace.
3. **Dễ test độc lập**: Tôi có thể test `analyze_policy()` với mock chunks mà không cần LLM API key.

**Trade-off đã chấp nhận:** Rule-based không handle được case mới ngoài danh sách. VD: nếu sản phẩm là "NFT" — chưa được define rõ là digital product hay không — rule-based sẽ bỏ qua.

**Bằng chứng từ trace/code:**

```python
# workers/policy_tool.py — analyze_policy()
if "flash sale" in task_lower or "flash sale" in context_text:
    exceptions_found.append({
        "type": "flash_sale_exception",
        "rule": "Đơn hàng Flash Sale không được hoàn tiền (Điều 3, chính sách v4).",
        "source": "policy_refund_v4.txt",
    })
```

Trace q07 (Flash Sale): `policy_result.exceptions_found: [{"type": "flash_sale_exception", ...}]`, `policy_applies: false`. Synthesis nhận đúng exception → answer đúng.

---

## 3. Tôi đã sửa một lỗi gì?

**Lỗi:** Policy worker gọi MCP `search_kb` ngay cả khi đã có `retrieved_chunks` từ trước

**Symptom:** Khi chạy pipeline với câu "hoàn tiền Flash Sale" (route: `policy_tool_worker`), trace cho thấy `mcp_tools_used` có 1 entry `search_kb` dù `retrieved_chunks` đã có data từ trước. MCP call tốn thêm ~50ms không cần thiết.

**Root cause:** Logic check trong `run()` có điều kiện ngược:

```python
# BUG — gọi MCP ngay cả khi đã có chunks
if needs_tool:   # Thiếu check chunks rỗng!
    mcp_result = _call_mcp_tool("search_kb", {"query": task, "top_k": 3})
```

Đúng logic phải là: chỉ gọi MCP `search_kb` khi `retrieved_chunks` rỗng VÀ `needs_tool=True`.

**Cách sửa:**

```python
# FIX — chỉ gọi MCP nếu chưa có chunks
if not chunks and needs_tool:   # Thêm "not chunks" condition
    mcp_result = _call_mcp_tool("search_kb", {"query": task, "top_k": 3})
```

**Bằng chứng trước/sau:**

- Trước: Trace câu Flash Sale có `mcp_tools_used: [{"tool": "search_kb", ...}]` dù `retrieved_chunks` đã có 3 chunks. Latency: 1450ms.
- Sau: `mcp_tools_used: []` cho câu đã có chunks. Latency: 1200ms. MCP chỉ được gọi khi thực sự cần.

---

## 4. Tôi tự đánh giá đóng góp của mình

**Tôi làm tốt nhất ở điểm nào?**

`worker_contracts.yaml` được viết trước khi code — điều này giúp nhóm không bị blocking lẫn nhau. Đặc biệt việc define rõ `mcp_tools_used` format (array of `{tool, input, output, error, timestamp}`) giúp Nguyễn Công Thành viết `analyze_traces()` đúng từ đầu mà không cần refactor.

Thêm `temporal_scoping` detection trong `analyze_policy()` (check ngày đặt trước 01/02/2026 → v3 policy) là edge case quan trọng cho câu gq02 — nhiều nhóm khác bỏ qua.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Policy worker chưa detect được case "đơn hàng trước 01/02/2026 nhưng không mention ngày rõ ràng". Chỉ detect khi task có "31/01" hoặc "trước 01/02" — nếu user hỏi "đơn đặt tháng 1" mà không nêu ngày cụ thể, temporal scoping bị bỏ qua.

**Nhóm phụ thuộc vào tôi ở đâu?**

`policy_result` với đúng `exceptions_found` là critical cho synthesis. Nếu tôi detect sai Flash Sale exception, synthesis sẽ trả lời "được hoàn tiền" — penalty -50% cho grading question đó. Contracts YAML cũng là tài liệu tham chiếu cho cả nhóm — nếu sai, mọi người implement theo spec sai.

**Phần tôi phụ thuộc vào thành viên khác:**

Policy worker gọi `dispatch_tool()` của Đỗ Đình Hoàn (MCP server). Cần interface ổn định — không thay đổi signature `dispatch_tool(tool_name: str, tool_input: dict) -> dict` trong quá trình lab.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì?

Tôi sẽ upgrade `analyze_policy()` để gọi LLM cho các edge cases không nằm trong danh sách rule cứng. Bằng chứng từ trace: câu gq10 (Flash Sale + lỗi nhà sản xuất + 7 ngày) — policy worker detect đúng Flash Sale exception, nhưng không handle câu hỏi "có được hoàn tiền không?" một cách explanatory. Synthesis phải "guess" reasoning từ exception data. Nếu thêm LLM call trong `analyze_policy()` với prompt "Với các exceptions này và điều kiện đơn hàng này, câu trả lời ngắn là gì?", synthesis sẽ có context rõ hơn và answer quality tăng cho gq10 từ partial lên full marks.

---

