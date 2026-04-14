# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Công Thành  
**Vai trò trong nhóm:** Trace & Docs Owner  
**Ngày nộp:** 2026-04-14  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào?

**Module/file tôi chịu trách nhiệm:**
- File chính: `eval_trace.py`
- Functions tôi implement: `run_test_questions()`, `run_grading_questions()`, `analyze_traces()`, `compare_single_vs_multi()`, `save_eval_report()`
- Docs: `docs/system_architecture.md`, `docs/routing_decisions.md`, `docs/single_vs_multi_comparison.md`

Tôi là Trace & Docs Owner — chịu trách nhiệm chạy pipeline với 15 test questions, phân tích trace, và viết 3 tài liệu kỹ thuật của nhóm. Phần của tôi là "observability layer" của hệ thống: không tạo ra answer, nhưng đo lường và ghi lại tất cả.

**Cách công việc của tôi kết nối với phần của thành viên khác:**

`eval_trace.py` gọi `run_graph()` của Nguyễn Xuân Tùng, đọc trace output, và tổng hợp metrics. Tôi phụ thuộc vào tất cả thành viên implement đúng: nếu `route_reason` rỗng, metric của tôi bị sai; nếu `workers_called` thiếu, routing distribution không chính xác. Ngược lại, docs tôi viết (`routing_decisions.md`) là evidence cho cả nhóm về routing accuracy thực tế.

**Bằng chứng:** Comment `# NCThanh — Sprint 4` trong `eval_trace.py`, và toàn bộ `docs/` folder là contribution của tôi.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì?

**Quyết định:** Lưu trace theo format 1-file-per-run thay vì 1 file JSONL tổng hợp cho `artifacts/traces/`

Khi implement `save_trace()` trong `graph.py` (tôi review và suggest format này với Nguyễn Xuân Tùng), có hai cách: (a) mỗi query append vào 1 file JSONL duy nhất `traces.jsonl`, hoặc (b) mỗi run tạo 1 file JSON riêng với `run_id` làm tên file.

**Lý do tôi chọn 1-file-per-run:**

1. **Debug từng câu độc lập**: Khi cần xem lại trace của 1 câu cụ thể (VD: gq09), mở đúng file đó mà không cần parse toàn bộ JSONL.
2. **Không corrupt khi crash**: Nếu pipeline crash giữa chừng khi đang append JSONL, file có thể bị malformed JSON. Với 1 file per run, các file trước không bị ảnh hưởng.
3. **Parallel execution friendly**: Nếu sau này chạy pipeline parallel, nhiều process ghi cùng 1 JSONL sẽ có race condition. 1 file per run tránh vấn đề này.

**Trade-off đã chấp nhận:** `analyze_traces()` phải đọc nhiều file thay vì 1 file. Nhưng với Python `os.listdir()` + loop, overhead không đáng kể.

**Bằng chứng từ code:**

```python
# eval_trace.py — analyze_traces()
trace_files = [f for f in os.listdir(traces_dir) if f.endswith(".json")]
for fname in trace_files:
    with open(os.path.join(traces_dir, fname)) as f:
        traces.append(json.load(f))
```

Metrics từ 15 trace files: `routing_distribution: {retrieval_worker: 9/15 (60%), policy_tool_worker: 5/15 (33%), human_review: 1/15 (7%)}`.

---

## 3. Tôi đã sửa một lỗi gì?

**Lỗi:** `analyze_traces()` tính `avg_confidence: 0.0` dù tất cả traces đều có confidence > 0

**Symptom:** Sau khi chạy 15 test questions và gọi `analyze_traces()`, output là `avg_confidence: 0.0`. Tất cả trace files đều có `confidence` field nhưng metric bị sai.

**Root cause:** Code trong `analyze_traces()` có bug logic:

```python
# BUG — kiểm tra conf bằng if conf (falsy check)
conf = t.get("confidence", 0)
if conf:          # confidence=0.0 sẽ bị skip (0.0 là falsy trong Python!)
    confidences.append(conf)
```

Câu abstain có `confidence=0.1` được append. Nhưng câu có `confidence=0.0` (lỗi pipeline) bị skip thay vì được include.

**Cách sửa:**

```python
# FIX — kiểm tra explicit None thay vì falsy
conf = t.get("confidence", None)
if conf is not None:   # Include 0.0 một cách chính xác
    confidences.append(conf)
```

**Bằng chứng trước/sau:**

- Trước: `avg_confidence: 0.0` (sai vì bug)
- Sau: `avg_confidence: 0.81` — tính đúng từ 15 traces, bao gồm cả abstain cases với confidence 0.1-0.3

---

## 4. Tôi tự đánh giá đóng góp của mình

**Tôi làm tốt nhất ở điểm nào?**

Viết `analyze_traces()` produce metrics đủ để fill vào `docs/single_vs_multi_comparison.md` với số liệu thực tế. Đặc biệt `routing_distribution` và `top_sources` giúp nhóm thấy ngay: 60% câu đi vào `retrieval_worker`, `sla_p1_2026.txt` là source được dùng nhiều nhất (8/15 queries) — insight hữu ích để cải thiện routing.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Chưa implement so sánh Day 08 vs Day 09 với data thực tế từ Day 08 — dùng ước tính thay vì chạy lại Day 08 pipeline. Nên dành 30 phút chạy lại Day 08 eval để có baseline thực sự thay vì dùng số ước đoán.

**Nhóm phụ thuộc vào tôi ở đâu?**

`artifacts/grading_run.jsonl` là deliverable bắt buộc phải nộp trước 18:00. Nếu tôi không chạy pipeline với grading questions và tạo file này, nhóm mất toàn bộ 30 điểm phần grading. Đây là phần có deadline cứng nhất.

**Phần tôi phụ thuộc vào thành viên khác:**

`run_grading_questions()` gọi `run_graph()` của Nguyễn Xuân Tùng. Nếu `graph.py` có bug crash, file grading sẽ thiếu entries. Tôi cần `graph.py` ổn định trước 16:30 để có đủ thời gian chạy grading questions trước deadline 18:00.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì?

Tôi sẽ implement Day 08 baseline chạy được để `compare_single_vs_multi()` có số liệu thực tế thay vì ước đoán. Bằng chứng từ trace: `docs/single_vs_multi_comparison.md` hiện tại ghi Day 08 avg_confidence là "~0.72 (ước tính)" — không acceptable cho báo cáo học thuật. Với Day 08 code đã có trong `../../day08/lab/`, tôi chỉ cần chạy lại `python eval.py` với cùng 15 test questions và so sánh kết quả thực tế. Điều này sẽ làm comparison section của nhóm credible hơn nhiều và có thể tăng điểm phần documentation.

---

