# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Viết Hùng  
**Vai trò trong nhóm:** Worker Owner (Retrieval)  
**Ngày nộp:** 2026-04-14  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào?

**Module/file tôi chịu trách nhiệm:**
- File chính: `workers/retrieval.py`
- Functions tôi implement: `retrieve_dense()`, `_get_embedding_fn()`, `_get_collection()`, `run()`

Tôi implement Retrieval Worker — worker đầu tiên trong pipeline chịu trách nhiệm tìm kiếm evidence từ Knowledge Base (ChromaDB). Worker của tôi là stateless: nhận `task` từ state, query ChromaDB, trả về `retrieved_chunks` và `retrieved_sources`. Không gọi LLM, không có side effects.

**Cách công việc của tôi kết nối với phần của thành viên khác:**

`retrieved_chunks` mà tôi ghi vào state là input chính cho synthesis_worker của Đỗ Đình Hoàn và policy_tool_worker của Trần Quốc Khánh. Nếu chunks rỗng, synthesis sẽ abstain. Nếu chunks sai (sai source, score thấp), câu trả lời sẽ không chính xác dù synthesis có tốt. Nguyễn Xuân Tùng (Supervisor) cần biết signature `run(state) -> state` của tôi để gọi từ `graph.py`.

**Bằng chứng:** Comment `# NVHung — Sprint 2` ở đầu `retrieval.py`, và `worker_io_logs` trong mỗi trace đều có `"worker": "retrieval_worker"`.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì?

**Quyết định:** Dùng `sentence-transformers/all-MiniLM-L6-v2` thay vì OpenAI `text-embedding-3-small` làm embedding model

Khi implement `_get_embedding_fn()`, tôi có hai lựa chọn: embedding local (sentence-transformers) hoặc embedding qua API (OpenAI). Lựa chọn thứ ba là hash/TF-IDF (rất đơn giản nhưng accuracy kém).

**Lý do tôi chọn sentence-transformers:**

1. **Offline — không cần API key**: Khi ChromaDB index đã được build, retrieval worker không cần bất kỳ API call nào. Latency giảm từ ~200ms (API round-trip) xuống ~15ms (local inference).
2. **Deterministic**: Cùng query luôn cho cùng embedding → cùng chunks → reproducible trace. Với OpenAI API, embedding có thể thay đổi khi model update.
3. **Đủ tốt cho domain nhỏ**: 5 tài liệu nội bộ, `all-MiniLM-L6-v2` (384 dimensions) đủ tốt. Accuracy không chênh nhiều so với text-embedding-3-small cho domain này.

**Trade-off đã chấp nhận:** `all-MiniLM-L6-v2` không được train nhiều trên tiếng Việt. Một số câu hỏi thuần Việt có thể có retrieval score thấp hơn. Trong production nên thử `bge-m3` hoặc multilingual model.

**Bằng chứng từ trace/code:**

```python
# workers/retrieval.py — _get_embedding_fn()
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("all-MiniLM-L6-v2")
def embed(text: str) -> list:
    return model.encode([text])[0].tolist()
```

Trace q01: `retrieved_sources: ["sla_p1_2026.txt"]`, `score: 0.89` — chunk chính xác được retrieve với confidence cao.

---

## 3. Tôi đã sửa một lỗi gì?

**Lỗi:** ChromaDB query trả về `distances` là L2 distance thay vì cosine similarity, khiến `score` bị sai

**Symptom:** Sau khi implement `retrieve_dense()`, tôi kiểm tra trace và thấy `score: -0.23` cho một chunk rõ ràng là relevant. Score âm không hợp lý.

**Root cause:** ChromaDB khi dùng collection mặc định (không có metadata `hnsw:space`) dùng L2 distance. Công thức `score = 1 - dist` tôi dùng chỉ đúng khi `dist` là cosine distance (range 0-2). Với L2 distance, `dist` có thể lớn hơn 1, khiến score âm.

**Cách sửa:**

Bước 1: Khi tạo collection, explicitly set `hnsw:space: cosine`:
```python
collection = client.get_or_create_collection(
    "day09_docs",
    metadata={"hnsw:space": "cosine"}  # Thêm dòng này
)
```

Bước 2: Rebuild index với collection mới để đảm bảo embeddings được index theo cosine space.

**Bằng chứng trước/sau:**

- Trước: `score: -0.23` cho chunk "SLA P1 phản hồi 15 phút" khi query "SLA P1 là bao lâu"
- Sau: `score: 0.89` — đúng với mức độ relevance thực tế. Tổng cộng 15/15 test questions có score trong range [0, 1] sau fix.

---

## 4. Tôi tự đánh giá đóng góp của mình

**Tôi làm tốt nhất ở điểm nào?**

Implement `run()` theo đúng worker contract trong `contracts/worker_contracts.yaml` — input/output match spec chính xác. Đặc biệt `worker_io_logs` append đúng format giúp Nguyễn Công Thành viết `analyze_traces()` mà không cần thay đổi code. Worker của tôi hoàn toàn stateless — test độc lập được mà không cần graph.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Chưa implement hybrid retrieval (dense + sparse/BM25). Một số câu hỏi như q09 (ERR-403-AUTH) — nếu có BM25 sẽ match được "ERR" keyword trong FAQ dù ChromaDB không có. Điểm này ảnh hưởng đến abstain rate.

**Nhóm phụ thuộc vào tôi ở đâu?**

Cả `policy_tool_worker` (Trần Quốc Khánh) và `synthesis_worker` (Đỗ Đình Hoàn) đều đọc `state["retrieved_chunks"]` mà tôi ghi. Nếu retrieval sai source hoặc trả về empty khi không nên, toàn bộ downstream answer sẽ sai.

**Phần tôi phụ thuộc vào thành viên khác:**

Phụ thuộc vào Nguyễn Xuân Tùng (Supervisor) set đúng `state["task"]` trước khi gọi `retrieval_run(state)`. Nếu task rỗng hoặc bị transform sai, embedding sẽ query sai.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì?

Tôi sẽ thêm hybrid retrieval (dense + BM25 sparse) trong `retrieve_dense()`. Bằng chứng từ trace: câu q09 (ERR-403-AUTH) và q08 (password policy) — `retrieved_chunks` trả về score thấp (0.41-0.52) vì câu hỏi dùng thuật ngữ kỹ thuật không match tốt với embedding. BM25 sẽ match chính xác "mật khẩu", "90 ngày", "7 ngày" trong `it_helpdesk_faq.txt` mà dense embedding bỏ sót. Kết hợp bằng Reciprocal Rank Fusion để có kết quả tốt nhất từ cả hai phương pháp.

---

*Lưu file này với tên: `reports/individual/nguyen_viet_hung.md`*
