# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi tư vấn mở khiến answer suy luận hợp lý thêm ngoài câu chữ context nhưng không sai fact — score rơi 0.6–0.8 vẫn chấp nhận. | Score < 0.6 trên câu hỏi về giá/bảo hành/chính sách — model bịa thông tin (hallucination) không có trong context, rủi ro cao vì khách hàng dựa vào đó ra quyết định. | Flag review thủ công; chặn deploy nếu dưới 0.6 trên nhóm câu hỏi transactional (giá, đổi trả, bảo hành). |
| Answer Relevance | Câu hỏi adversarial/out-of-scope mà answer đúng đắn từ chối trả lời — có thể bị chấm relevance thấp dù hành vi đúng. | Score thấp trên câu hỏi in-scope rõ ràng — answer lạc đề, không giải quyết đúng intent, gây trải nghiệm tệ và tăng escalation. | Review lại prompt/retrieval cho case in-scope; dùng rubric riêng cho adversarial case thay vì relevance thuần. |
| Context Recall | Câu hard/multi-hop cần tổng hợp nhiều document — retriever miss một phần evidence phụ nhưng vẫn đủ evidence chính. | Score thấp trên câu easy/single-doc — retriever miss cả evidence chính, lỗi retrieval nghiêm trọng ở tầng chunking/embedding cơ bản. | Kiểm tra lại chunk size, embedding model, top-k; sửa retriever trước khi tối ưu generation. |
| Context Precision | Tập retrieved rộng lẫn 1-2 chunk liên quan nhưng rank thấp — chấp nhận nếu evidence chính vẫn đứng đầu và answer đúng. | Score rất thấp — phần lớn chunk không liên quan, gây nhiễu (distraction) làm faithfulness giảm theo, tốn cost/latency. | Tune reranking hoặc giảm top-k; tăng ngưỡng similarity khi retrieve. |
| Completeness | Câu hỏi có sub-claim phụ không bắt buộc — answer đáp ứng phần chính, thiếu chi tiết bổ sung không critical, vẫn chấp nhận. | Score thấp trên câu có claim bắt buộc (điều kiện bảo hành, số ngày đổi trả) — thiếu thông tin cốt lõi gây hiểu lầm cho khách hàng. | Bổ sung expected-answer decomposition rõ hơn trong golden dataset; kiểm tra prompt có yêu cầu liệt kê đủ claims. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Lấy một cặp answer (A, B) cho cùng một câu hỏi, chất lượng đã biết trước qua human label (vd A tốt hơn B). Chạy LLM judge hai lần:
> - Condition 1: đưa A trước, B sau ("Answer 1: A, Answer 2: B").
> - Condition 2: đảo vị trí, đưa B trước, A sau ("Answer 1: B, Answer 2: A").
>
> Nếu judge đổi lựa chọn theo vị trí (luôn chọn "Answer 1" bất kể nội dung) thay vì theo chất lượng thật, đó là dấu hiệu position bias. Lặp trên nhiều cặp và tính % lần judge đảo kết quả chỉ vì đổi vị trí.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Rubric quy định điểm dựa trên số claim đúng/cần thiết, không dựa trên độ dài; đưa ví dụ minh hoạ answer ngắn nhưng đầy đủ vẫn chấm 5 điểm, answer dài nhưng lan man/redundant chấm thấp hơn. Thêm tiêu chí trừ điểm tường minh cho verbosity không cần thiết, và yêu cầu judge liệt kê claim tìm thấy trước khi cho điểm (thay vì đánh giá cảm tính theo độ dài).

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Vì judge có thể hệ thống hoá sai lệch (position, verbosity, self-preference) mà không tự nhận ra. Cần một tập nhỏ ground-truth do người chấm để đo agreement (vd Cohen's kappa) giữa judge và human; nếu agreement thấp thì phải chỉnh rubric/prompt trước khi tin dùng judge ở quy mô lớn — nếu không mọi quyết định (block deploy, so sánh model) dựa trên judge sẽ kế thừa toàn bộ bias mà không phát hiện được.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | ≥ 0.8 | Hallucination trên chatbot support (giá, bảo hành, chính sách) gây rủi ro trực tiếp cho khách hàng/pháp lý — không thể chấp nhận model bịa thông tin trước khi ra production. |
| Answer Relevance | ≥ 0.7 | Answer đúng nhưng lạc đề vẫn khiến khách hàng phải hỏi lại/escalate; 0.7 chừa khoảng dung sai cho câu hỏi mơ hồ nhưng vẫn chặn regression rõ rệt về đúng intent. |
| Completeness | ≥ 0.7 | Thiếu thông tin bắt buộc (điều kiện đổi trả, số ngày bảo hành) dẫn tới hướng dẫn sai cho khách hàng; ngưỡng 0.7 cho phép thiếu chi tiết phụ nhưng không cho thiếu claim cốt lõi. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation** (golden dataset, RAGAS/LLM-judge chạy trong CI/CD): dùng trước mỗi lần merge/deploy để chặn regression sớm, chi phí thấp, lặp lại được, nhưng chỉ phủ được các case đã biết trước trong dataset.
> - **Online evaluation** (theo dõi metric trên traffic thật, A/B test, sampling judge trên response thực tế): dùng sau khi deploy để phát hiện drift, case ngoài golden dataset, và đo tác động thực tế lên user (vd CSAT, escalation rate) mà offline không mô phỏng được.
> - **Human review**: dùng cho case adversarial/nhạy cảm (an toàn, pháp lý, PII), để calibrate LLM judge định kỳ, và khi offline/online score nằm trong vùng nghi ngờ (borderline) cần xác nhận cuối cùng trước khi thay đổi threshold hoặc ra quyết định lớn (rollback, đổi model).

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
