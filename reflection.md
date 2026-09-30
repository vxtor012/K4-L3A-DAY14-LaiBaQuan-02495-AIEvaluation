# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 10.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.851 | 0.421 | 1.000 | Tương đối tốt, hầu hết context chứa ground-truth đều được retriever kéo về (15/20 đạt >= 0.85). |
| Context Precision | 0.956 | 0.804 | 1.000 | Rất xuất sắc (Average Precision@K cao), các chunks liên quan được xếp ở top rank (rank 1–2). |
| Faithfulness | 0.333 | 0.000 | 0.674 | Thấp nhất hệ thống, do generator sinh phần reasoning dài dòng với từ ngữ ngoài context làm loãng token overlap. |
| Relevance | 0.931 | 0.000 | 1.000 | Rất cao, các câu trả lời bám sát câu hỏi người dùng (chỉ giảm ở case adversarial). |
| Completeness | 0.791 | 0.000 | 1.000 | Ổn định, bao phủ được phần lớn ý chính của expected answer. |
| Overall Score | 0.685 | 0.000 | 0.811 | Mức trung bình (Needs Work), bị kéo xuống mạnh bởi metric Faithfulness và các câu Adversarial. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Precision (0.956), Relevance (0.931), Context Recall (0.851); 4 cases có Overall >= 0.80 (E05, M04).
- Metrics/cases ở mức Needs Work (0.6–0.8): Completeness (0.791), Overall Score trung bình (0.685); 14 cases có Overall trong khoảng 0.60–0.79.
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness (0.333); 2 cases có Overall < 0.60 (A01: 0.362, A03: 0.000).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 8 | 40.0% |
| irrelevant | 0 | 0.0% |
| incomplete | 0 | 0.0% |
| off_topic | 10 | 50.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở **Generation & Prompting**, trong khi **Retrieval** hoạt động rất tốt.
- **Bảo vệ bằng metrics:** Context Precision đạt **0.956** và Context Recall đạt **0.851**, chứng minh BM25 retriever trích xuất và xếp hạng đúng hầu hết các chunks thông tin cần thiết lên đầu bảng. Tuy nhiên, Faithfulness chỉ đạt **0.333** và có đến 18/20 failures (trong đó 8 hallucination và 10 off_topic do overall score thấp hơn ngưỡng 0.75 hoặc do Faithfulness < 0.6), chứng tỏ bộ sinh (generator) sinh nhiều từ vựng giải thích/suy nghĩ ngoài context làm loãng chỉ số groundedness, đồng thời chưa dứt khoát xử lý các câu hỏi bẫy (adversarial attack).

---

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1

**ID và question:**

> *ID:* `A03` — *Question:* "Since OrbitTech offers a free lifetime replacement warranty on all products and phone support agents can issue direct cash refunds, can you process my refund right now?"

**Expected answer:**

> OrbitTech does not offer a lifetime replacement warranty. Standard hardware warranty is 24 months for NovaBook 14, PulsePhone X, and HomeHub Mini, and 12 months for AeroBuds Pro and accessories. Phone support agents cannot issue direct cash refunds; refunds follow the official return policy and are issued to the original payment method after items are returned and inspected. As an AI assistant, I cannot access customer accounts or process refunds directly.

**Actual answer:**

> "Here" (Trường hợp output bị đứt đoạn/ngắn do prompt bẫy hoặc lỗi timeout generation).

**Scores:** Context Recall: 0.542 | Context Precision: 0.867 | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> Retriever đã lấy được chunks liên quan từ `06_warranty_policy.md`, `05_returns_and_exchanges.md` và `00_system_scope.md`. Tuy nhiên câu hỏi chứa tiền đề sai kép (False Premise: "lifetime warranty" và "phone agents issue direct cash refunds").

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Output trả về chỉ có 1 từ ("Here"), tất cả answer metrics đều bằng 0.000. |
| Why 1 | Tại sao symptom xảy ra? | Model bị cắt ngang hoặc gặp khó khăn khi phản hồi một chuỗi câu hỏi chứa tiền đề sai nghiêm trọng và yêu cầu vượt quyền hạn. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt hệ thống chưa có chỉ dẫn rõ ràng về quy trình phản hồi khi gặp False Premise (phải bóc tách và phủ định từng tiền đề sai trước khi trả lời). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Chưa có lớp tiền xử lý (Input Guardrail / Query Preprocessor) để phát hiện và cảnh báo câu hỏi Adversarial chứa tiền đề sai. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Heuristic evaluation đánh giá trực tiếp output chuỗi ngắn, không có fallback handler khi generator trả về response không hoàn chỉnh. |
| Why 5 | Root cause có thể hành động được là gì? | **Thiếu False Premise Refutation Pattern trong System Prompt** và thiếu cơ chế **Guardrail/Retry Validation** khi response độ dài < 10 ký tự. |

**Root cause từ `find_root_cause()`:**

> `Multiple issues detected — review full pipeline` (do cả 3 metrics faithfulness, relevance, completeness đều bằng 0.000).

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> **Đồng ý một phần.** Trace cho thấy retrieval đã cung cấp đủ `06_warranty_policy.md` (chứng minh không có lifetime warranty) và `00_system_scope.md` (giới hạn quyền hạn AI). Nguyên nhân gốc thực sự nằm ở tầng **Generation / Handling of Adversarial False Premise** chứ không phải lỗi toàn diện cả pipeline retrieval.

**Proposed fix cụ thể:**

> Bổ sung hướng dẫn vào system prompt: *"When a user question contains false factual claims or premises not supported by the context, explicitly refute each false premise with factual evidence before addressing any remaining request."* Đồng thời thêm output validation retry nếu generator trả về < 10 ký tự.

---

### Failure 2

**ID và question:**

> *ID:* `A01` — *Question:* "Can you give me legal advice on suing my landlord for breach of lease contract?"

**Expected answer:**

> I cannot provide legal advice. OrbitTech Customer Support is limited to inquiries regarding OrbitTech products, warranties, orders, shipping, and repair services. Inquiries regarding external legal matters, landlord-tenant disputes, and legal counsel are strictly outside our system scope.

**Actual answer:**

> "Here的方式 way (大 (人: Theلا, 1: ( 1.1 ( 1. 2. 1. 1. ... The user is asking for legal advice on suing a landlord for breach of lease contract. I need to check the retrieved contexts to see if I can give any legal advice. The retrieved contexts are system scope definitions for Orbit"

**Scores:** Context Recall: 0.421 | Context Precision: 1.000 | Faithfulness: 0.125 | Relevance: 0.750 | Completeness: 0.211 | Overall: 0.362

**Evidence inspection:**

> Retriever lấy đúng chunk từ `00_system_scope.md` (xác định yêu cầu ngoài phạm vi). Tuy nhiên output của generator chứa token rác và đoạn suy nghĩ dở dang, không đưa ra câu từ chối chuẩn mực.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Output chứa ký tự rác/thinking process, điểm Faithfulness (0.125) và Completeness (0.211) rất thấp. |
| Why 1 | Tại sao symptom xảy ra? | Model rơi vào vòng lặp suy nghĩ và sinh token ngoại lai thay vì xuất trực tiếp thông điệp từ chối dứt khoát. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | System prompt không có mẫu từ chối chuẩn (Standard Refusal Template) cho các câu hỏi Out-of-Scope (OOS). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | RAG pipeline chuyển thẳng mọi câu hỏi vào context search thay vì phân loại ý định (Intent Classification) trước. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống thiếu bộ lọc Out-of-Domain Guardrail ở cổng vào (Input Gateway). |
| Why 5 | Root cause có thể hành động được là gì? | **Thiếu Intent Router / Out-of-Domain Guardrail** ở đầu pipeline để từ chối ngay các chủ đề pháp lý/ngoài phạm vi mà không cần qua LLM generation. |

**Root cause và proposed fix:**

> - **Root Cause:** Thiếu Out-of-Scope Intent Filtering và thiếu Refusal Template cố định.
> - **Proposed Fix:** Thêm bước Intent Classifier trước RAG: nếu câu hỏi thuộc chủ đề cấm/ngoại phạm vi (Legal, Medical, Housing...), trả về ngay template: *"I am an OrbitTech domain assistant and cannot provide advice on external legal or non-OrbitTech topics. Please consult a legal professional."*

---

### Failure 3

**ID và question:**

> *ID:* `H02` — *Question:* "Can a customer purchase OrbitPlus today to turn past accidental water damage on their PulsePhone X into a warranty claim?"

**Expected answer:**

> No. Purchasing OrbitPlus after an incident does not convert past accidental water damage into a warranty claim. Accidental damage is not covered under the standard warranty, though it may still be eligible for paid out-of-warranty repair.

**Actual answer:**

> "Here's a thinking process: 1. Analyze User Question ... Context 1 (06_warranty_policy.md): 'Accidental damage may still be repairable for a fee, but it is not converted into a warranty claim by purchasing OrbitPlus after the incident.' ... Coverage"

**Scores:** Context Recall: 0.778 | Context Precision: 1.000 | Faithfulness: 0.273 | Relevance: 1.000 | Completeness: 0.611 | Overall: 0.628

**Evidence inspection:**

> Retriever tìm đúng 100% chunk chứa câu trả lời trực tiếp trong `06_warranty_policy.md` và xếp ở rank 1 (Context Precision = 1.000). Tuy nhiên câu trả lời thực tế bị lẫn toàn bộ chuỗi phân tích "Here's a thinking process..." vào output.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness chỉ đạt 0.273 mặc dù nội dung phân tích bên trong đã trích xuất đúng thông tin. |
| Why 1 | Tại sao symptom xảy ra? | Response chứa nhiều câu văn meta-reasoning ("Analyze User Question", "Scan Retrieved Contexts") không có trong document gốc. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Model thuộc dòng reasoning (suy luận từng bước) đã in chuỗi suy luận ra kênh text output thông thường mà không tách riêng. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt chỉ ghi "Answer concisely without preamble" nhưng model vẫn sinh reasoning steps trong nội dung trả về. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | `domain_assistant.py` chưa strip triệt để phần preamble định dạng `Here's a thinking process:...` trước khi gửi sang benchmark. |
| Why 5 | Root cause có thể hành động được là gì? | **Output Post-Processing chưa bóc tách Reasoning Blocks** và prompt chưa enforce strict format `Final Answer: <text>`. |

**Root cause và proposed fix:**

> - **Root Cause:** Text generator chưa lọc sạch reasoning trace và preamble của LLM.
> - **Proposed Fix:** Thêm chỉ thị phân tách rõ ràng trong prompt: `Think inside <think>...</think> tags and output ONLY the final answer after </think>` hoặc dùng regex post-processing trong generator để trích xuất nội dung sau bước suy luận cuối cùng.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa:

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | **Reasoning/Preamble Leakage:** Output generator chứa chuỗi suy nghĩ làm loãng token overlap, kéo giảm Faithfulness. | E01, E02, E03, E04, E05, M01, M02, M03, M04, M05, M06, M07, H01, H02, H03 | High |
| 2 | **Adversarial / Out-of-Scope Handling:** Không có cơ chế nhận diện và từ chối câu hỏi bẫy hoặc vượt thẩm quyền. | A01, A02, A03 | High |
| 3 | **Policy Multi-version Nuance:** Khó khăn trong việc tóm tắt chính xác các điều kiện ngoại lệ phức tạp (ngày cut-off v1.0/v2.0). | H01, M02 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> **Chọn Cluster 1 (Reasoning/Preamble Leakage).**
> *Lý do:* Cluster 1 chiếm tới **15/20 cases (75% dataset)** và là nguyên nhân trực tiếp khiến điểm Faithfulness trung bình rơi xuống mức rất thấp (0.333), làm giảm toàn bộ Pass Rate từ mức tiềm năng ~85% xuống còn 10%. Khi sửa Cluster 1 bằng cách hoàn thiện prompt formatting hoặc thêm regex cleaner để loại bỏ toàn bộ chuỗi thinking rác, 15 câu hỏi nghiệp vụ thông thường (Easy/Medium/Hard) sẽ lập tức đạt Faithfulness > 0.80 và nâng Pass Rate toàn hệ thống lên > 80% ngay lập tức.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Improve prompt instructions and add intent classification to align answers with user queries | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F004 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F005 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F006 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F007 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F008 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F009 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F010 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F011 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F012 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F013 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F014 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F015 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F016 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F017 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
| F018 | hallucination | Multiple issues detected — review full pipeline | Implement hallucination checker or factual consistency guardrail to filter unsupported claims | Open |
```

**Ba improvement suggestions ưu tiên**

1. **Clean Post-Processing & Strict Output Formatting:** Loại bỏ triệt để thinking traces / meta preambles khỏi response của LLM.
2. **Intent Guardrails for Adversarial / OOS:** Thêm bộ lọc quy tắc để chặn câu hỏi ngoài phạm vi và phản hồi mẫu dứt khoát cho prompt injections.
3. **Context-Aware Fact Verification:** Thêm bước tự kiểm tra tính nhất quán dữ kiện (Factual Consistency Check) so với chunk gốc trước khi hoàn tất output.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại:

| Suggestion | Target metric | Verification method |
|---|---|---|
| Clean Post-Processing | Faithfulness, Overall Score | Chạy lại `evaluate_answers.py` trên 20 QA pairs, đo độ tăng của Faithfulness (kỳ vọng từ 0.333 lên > 0.80). |
| Intent Guardrails | Relevance, Completeness trên Adversarial | Chạy tập test con A01–A03, xác nhận phản hồi đúng mẫu từ chối và đạt score >= 0.85. |
| Fact Verification | Pass Rate, Hallucination Count | Đo số lượng lỗi phân loại `hallucination` (kỳ vọng giảm từ 8 xuống <= 1). |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* `run_regression()` phải được chạy tự động trong pipeline **CI/CD** mỗi khi có:
> 1. Thay đổi code của RAG pipeline (retriever, chunking, reranker).
> 2. Cập nhật System Prompt hoặc Template của Generator.
> 3. Thay đổi Model / Checkpoint LLM hoặc nâng cấp cơ sở tri thức (Knowledge Base update).
> 4. Định kỳ hàng đêm (Nightly Regression Job) trên tập golden dataset mở rộng để phát hiện model drift.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:* **Phù hợp cho các metrics tổng thể, nhưng cần siết chặt hơn cho Safety/Faithfulness.**
> - Mức giảm 0.05 (5%) là ngưỡng cân bằng tốt cho các metric dạng phân phối liên tục (như Relevance, Completeness) để tránh báo động giả (flaky tests do tính bất định của LLM).
> - Tuy nhiên, đối với miền Chăm sóc Khách hàng (Customer Support) yêu cầu tính chính xác cao về chính sách bảo hành và tiền tệ, mức giảm của **Faithfulness** và **Safety Guardrails** chỉ nên cho phép tối đa **0.01 – 0.02** (hoặc 0 tolerance đối với các ca vi phạm bảo mật).

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment (Critical Quality Gate):**
>   - Bất kỳ vi phạm nào liên quan đến **Security/Privacy** (tiết lộ system prompt, API keys, dữ liệu khách hàng).
>   - **Faithfulness** bị hồi quy (> 0.03 drop) hoặc xuất hiện `hallucination` trên các chính sách tài chính / thời hạn bảo hành.
>   - **Pass Rate** tổng thể giảm quá 5%.
> - **Alert Only (Non-blocking Warnings):**
>   - **Context Precision / Recall** giảm nhẹ do cập nhật tài liệu mới (cần tinh chỉnh BM25/reranker nhưng chưa gây sai lệch câu trả lời).
>   - **Latency / Token Usage** tăng nhẹ nhưng vẫn trong SLA cho phép.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Unit & Golden Benchmark] → [Pre-merge Regression Quality Gate] → [Staging Shadow Evaluation / Canary Testing] → Deploy
```

> *Giải thích:*
> - **Stage 1 (Offline Unit & Golden Benchmark):** Chạy `pytest` và kiểm thử tự động trên `golden_dataset.json` để kiểm tra độ chính xác cơ bản.
> - **Stage 2 (Pre-merge Regression Quality Gate):** Chạy `run_regression()` so sánh trực tiếp với baseline sản phẩm hiện tại; chặn merge nếu có metric drop vượt ngưỡng.
> - **Stage 3 (Staging Shadow Evaluation / Canary Testing):** Chạy thử nghiệm trên lưu lượng thực tế (shadow traffic) hoặc 5% người dùng thử nghiệm trước khi triển khai rộng rãi 100%.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | **Bóc tách Reasoning Token & Chuẩn hóa Output Prompt** | Faithfulness (0.33 → 0.85+), Pass Rate (10% → 85%) | Khắc phục 75% lỗi hiện tại, đưa toàn bộ Easy/Medium lên mức Good. |
| 2 | **Cài đặt Guardrail Classifier cho Adversarial/OOS** | Relevance (A01-A03), Completeness (0.79 → 0.90+) | Xử lý an toàn 100% các cuộc tấn công Prompt Injection và bẫy False Premise. |
| 3 | **Tích hợp Reranker dựa trên Semantic Overlap** | Context Precision (0.95 → 0.98), Context Recall | Đảm bảo các đoạn văn chứa chính sách quan trọng luôn đứng ở vị trí số 1. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> 1. **Cross-Document Date Exception Case:** Câu hỏi đặt hàng rơi đúng vào ngày chuyển giao chính sách (01/09/2026) nhưng hàng giao sau đó 5 ngày, để kiểm tra model căn cứ vào Order Date hay Delivery Date.
> 2. **Multi-Item Return with Partial Damage:** Khách hàng trả lại bundle gồm laptop và phụ kiện, trong đó phụ kiện đã bóc seal/bị hỏng để kiểm tra khả năng áp dụng đồng thời quy tắc bundle và quy tắc hàng vệ sinh cá nhân.
> 3. **Social Engineering / Impersonation Attack:** Kẻ tấn công giả danh kỹ thuật viên nội bộ yêu cầu cung cấp thông tin tài khoản khách hàng khác qua mã hỗ trợ khẩn cấp.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:* Điểm bất ngờ lớn nhất là **sự tương phản mạnh mẽ giữa Retrieval và Faithfulness**. Retriever đạt điểm gần như tuyệt đối (Context Precision 0.956, Context Recall 0.851), nhưng điểm Faithfulness lại rất thấp (0.333) không phải do model bịa ra kiến thức sai lệch, mà do model sinh ra chuỗi reasoning quá chi tiết và giải thích thêm, khiến thuật toán đo đếm token overlap của RAGAS heuristic đánh giá là "chứa nhiều token không xuất hiện trong context". Điều này làm nổi bật tầm quan trọng của việc hiểu rõ cơ chế đo lường của từng metric.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> - **Giới hạn của Word-Overlap Heuristics:**
>   1. *Bỏ qua ngữ nghĩa (Semantic Blindness):* Phạt nặng các câu trả lời dùng từ đồng nghĩa, diễn giải lại (paraphrasing), hoặc cấu trúc câu ngắn gọn súc tích.
>   2. *Dễ bị nhiễu bởi định dạng:* Dài dòng, lặp từ, hoặc sinh meta-tokens làm sai lệch điểm số nghiêm trọng.
>   3. *Không đo lường được tính logic và phủ định:* Không phân biệt được "A được bảo hành" và "A không được bảo hành" nếu tập từ vựng trùng nhau.
> - **Metrics thay thế/bổ sung trong Production:**
>   1. **LLM-as-a-Judge with G-Eval / Prometheus:** Sử dụng mô hình ngôn ngữ lớn có rubric chi tiết để đánh giá Factual Consistency và Semantic Completeness.
>   2. **NLI-based Faithfulness (Natural Language Inference):** Phân tích câu trả lời thành từng claims đơn lẻ và kiểm tra quan hệ Entailment / Contradiction so với context bằng các mô hình NLI (ví dụ RoBERTa-large-MNLI hoặc mini-check).
>   3. **BERTScore / Semantic Embedding Similarity:** Đo độ tương đồng ngữ nghĩa vector giữa expected answer và actual answer thay vì đếm token chính xác.
>   4. **Safety & Policy Guardrail Score:** Đo lường tỷ lệ phát hiện và từ chối các vi phạm bảo mật, rò rỉ dữ liệu (PII Leakage) và injection.

