# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dựa trên kết quả thực tế từ `artifacts/benchmark_results.json` và context trace từ `artifacts/actual_answers.json`.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 25.0% (5 / 20 passed)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.792 | 0.415 | 1.000 | Tốt trên single-hop, giảm mạnh ở multi-hop/adversarial |
| Context Precision | 0.897 | 0.500 | 1.000 | Retriever xếp hạng chunk liên quan lên top đầu rất tốt |
| Faithfulness | 0.581 | 0.154 | 1.000 | Thấp do câu trả lời trích xuất lan man hoặc chưa khớp claim |
| Relevance | 0.533 | 0.250 | 0.800 | Nhiều câu trả lời mang tính định nghĩa chung, chưa trúng trọng tâm |
| Completeness | 0.607 | 0.133 | 1.000 | Bị thiếu các điều kiện con, mốc thời gian hoặc ngoại lệ |
| Overall Score | 0.574 | 0.179 | 0.804 | Trung bình toàn hệ thống nằm ở ngưỡng cần can thiệp sâu (<0.6) |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): `Context Precision` (0.897); 5 cases pass (`M04`, `E03`, `M01`, `M07`, `M02`).
- Metrics/cases ở mức Needs Work (0.6–0.8): `Context Recall` (0.792), `Completeness` (0.607); 6 cases (`E05`, `A02`, `E04`, `A03`, `A01`, `H01`).
- Metrics/cases ở mức Significant Issues (<0.6): `Faithfulness` (0.581), `Relevance` (0.533), `Overall Score` (0.574); 9 cases (`H04`, `M06`, `H03`, `H02`, `H05`, `E02`, `M05`, `M03`, `E01`).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 2 | 10.0% |
| irrelevant | 0 | 0.0% |
| incomplete | 4 | 20.0% |
| off_topic | 9 | 45.0% |
| refusal | 0 | 0.0% |
| **Passed (No failure)** | 5 | 25.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:*
> Vấn đề chính hiện tại nằm ở **Generation (Synthesis)** là chủ đạo, kết hợp với một phần **Retrieval ở các câu hỏi phức hợp (multi-hop)**:
> 1. **Retrieval Performance rất khả quan:** `Context Precision` đạt trung bình **0.897** và `Context Recall` đạt **0.792**. Điều này chứng minh retriever đã tìm và đưa hầu hết các thông tin cốt lõi (gần 80% keyword facts) vào top context window.
> 2. **Generation Performance sụt giảm nghiêm trọng:** `Faithfulness` chỉ đạt **0.581** và `Relevance` chỉ đạt **0.533**, kéo theo `Completeness` đạt **0.607**. Mặc dù context chứa đủ thông tin, tầng sinh câu trả lời (generator) lại chọn lọc các câu văn mở đầu chung chung, bỏ sót các chi tiết số liệu cụ thể (như thời hạn 10 ngày, 15 ngày, mức hoàn tiền) hoặc đưa ra nội dung lan man làm loãng độ tương đồng với câu hỏi.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> `H04` (Hard): *"How are refunds calculated and disbursed when an order paid with a combination of gift cards and credit cards is returned?"*

**Expected answer:**

> *"Refunds are processed within 5 to 7 business days after inspection to the original payment methods. Cash cannot be refunded for gift-card funded amounts; that portion is returned as a replacement gift card, while original standard shipping fees are non-refundable for preference returns."*

**Actual answer:**

> *"Customers may pay by supported credit or debit card, OrbitTech gift card, or bank transfer. Up to two gift cards may be combined with one card payment. Gift cards, digital activation codes, personalized items, and completed services are non-returnable."*

**Scores:** Context Recall: 0.833 | Context Precision: 1.000 | Faithfulness: 0.154 |
Relevance: 0.250 | Completeness: 0.133 | Overall: 0.179

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:*
> Retriever đã lấy **đúng và đủ** thông tin cần thiết:
> - Chunk 1: `02_orders_and_payments.md` (`OT-02-P02`) chứa quy tắc hoàn tiền gift card ("OrbitTech cannot refund cash for a gift-card-funded portion; that amount returns to a replacement gift card").
> - Chunk 2: `05_returns_and_exchanges.md` (`OT-05-P05`) chứa quy tắc xử lý hoàn tiền 5–7 ngày làm việc và phí vận chuyển.
> Context Recall đạt 0.833 và Precision đạt 1.000. Tuy nhiên Generator lại lấy nhầm các câu đầu tiên của Chunk 1 (nói về phương thức thanh toán) thay vì các câu về quy tắc hoàn tiền.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời nêu phương thức thanh toán và sản phẩm không được trả lại thay vì nêu quy trình và tỷ lệ hoàn tiền khi kết hợp thẻ + gift card. |
| Why 1 | Tại sao symptom xảy ra? | Generator trích xuất câu dựa trên từ khóa bề mặt ("gift card", "card") từ đầu văn bản mà không tập trung vào vị ngữ "refunds calculated and disbursed". |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt tổng hợp câu trả lời thiếu cơ chế lọc theo ý định câu hỏi (intent filtering) và thiếu hướng dẫn trích xuất đa tài liệu (multi-document synthesis). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Hệ thống chọn fallback generator trích xuất đơn giản từ top chunks mà không có bước rerank câu ở mức sentence-level. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống chưa có bước Guardrail/Evaluator tự kiểm tra tính liên quan (Relevance check) trước khi trả câu lời cuối cùng cho người dùng. |
| Why 5 | Root cause có thể hành động được là gì? | Cần bổ sung **Prompt CoT trích xuất theo tiêu chí câu hỏi** và **Reranker mức câu (Sentence Extractor)** ưu tiên các mệnh đề chứa thực thể "refund / return / disbursement". |

**Root cause từ `find_root_cause()`:**

> `Multiple issues detected — review full pipeline`

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:*
> Đồng ý một phần nhưng cần làm rõ: Mặc dù thuật toán đánh dấu là lỗi toàn pipeline do các metric generation đều thấp (<0.5), nhưng evidence trace chỉ ra **Retrieval hoàn toàn chính xác (Precision 1.000, Recall 0.833)**. Lỗi thực sự 100% nằm ở khâu **Generation / Answer Extraction**.

**Proposed fix cụ thể:**

> Thiết kế prompt cấu trúc dạng: `Trích xuất chính xác: 1. Thời gian xử lý hoàn tiền, 2. Cách hoàn tiền cho phần Gift Card, 3. Cách hoàn tiền cho phần Credit Card`. Đồng thời thêm sentence-level overlap reranking trước khi ghép câu trả lời.

### Failure 2

**ID và question:**

> `M06` (Medium): *"When is an adult signature required for delivery, and what should be done if a package arrives damaged?"*

**Expected answer:**

> *"An adult signature is required for device shipments valued above USD 1,000. If there is visible damage or missing items, it must be reported within 48 hours of delivery along with photos of the label, box, and contents."*

**Actual answer:**

> *"OrbitTech does not authorize a carrier to leave a signature-required package unattended. Orders containing devices valued above USD 1,000 require an adult signature. Express-shipping fees are refunded when an express package arrives after the carrier's committed service date, unless the delay resulted from an incorrect address, unavailable recipient, customs hold, severe weather, or another listed carrier exception."*

**Scores:** Context Recall: 0.444 | Context Precision: 0.700 | Faithfulness: 0.289 |
Relevance: 0.500 | Completeness: 0.296 | Overall: 0.362

**Evidence inspection:**

> *Câu trả lời:*
> Retriever lấy được chunk `OT-04-P02` (về chữ ký người lớn >,000) nhưng **hoàn toàn bỏ sót chunk `OT-04-P04`** (quy định báo cáo hư hỏng trong vòng 48h kèm ảnh chụp hộp và nhãn). Do đó Context Recall bị giảm xuống chỉ còn 0.444.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời chỉ trả lời được vế chữ ký người lớn, hoàn toàn thiếu vế hướng dẫn xử lý khi kiện hàng bị hư hại. |
| Why 1 | Tại sao symptom xảy ra? | Context truyền vào cho generator không có chunk chứa thông tin về "damaged package 48h report". |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Câu hỏi ghép 2 ý khác nhau ("adult signature" VÀ "package arrives damaged"). Tìm kiếm từ khóa đơn lẻ bị chi phối bởi các từ khóa giao hàng và chữ ký. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Hệ thống không có bước phân tích câu hỏi phức hợp (Query Decomposition) thành 2 sub-queries độc lập. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Đơn thuần sử dụng 1 lần truy vấn lexical top-k=5 nên các chunk về hư hại bị đẩy ra ngoài top 5. |
| Why 5 | Root cause có thể hành động được là gì? | Cần triển khai **Query Decomposition (Sub-query generation)** để tách câu hỏi kép thành các truy vấn con trước khi retrieve, sau đó hợp nhất (RRF / Union) context. |

**Root cause và proposed fix:**

> - **Root cause:** Thiếu cơ chế phân tách câu hỏi đa ý (Multi-aspect Query Decomposition), dẫn đến retrieval bị thiên lệch và bỏ sót chunk chứa ý thứ hai.
> - **Proposed fix:** Tích hợp bộ tiền xử lý Query Router / Query Decomposition: nếu câu hỏi chứa liên từ ("and", "đồng thời", "như thế nào và khi nào"), sinh ra 2 sub-queries riêng biệt `q1="adult signature delivery threshold"`, `q2="damaged package reporting procedure"` rồi hợp nhất kết quả tìm kiếm.

### Failure 3

**ID và question:**

> `H03` (Hard): *"What steps and turnaround times are defined for warranty repair diagnosis, and when can a repair case be escalated for alternative remedies?"*

**Expected answer:**

> *"Initial diagnosis takes up to 3 business days after receipt, and covered repairs take up to 10 additional business days when parts are available. If a required part remains unavailable for more than 15 business days, support must offer an escalation review for an alternative remedy."*

**Actual answer:**

> *"Warranty service may result in repair, replacement with an equivalent new or refurbished unit, or refund when the first two remedies are not reasonable. Warranty duration and exclusions are defined in `06_warranty_policy.md`; return eligibility is separate and is defined in `05_returns_and_exchanges.md`. Initial diagnosis normally takes up to three business days after the service centre receives the product."*

**Scores:** Context Recall: 0.806 | Context Precision: 0.917 | Faithfulness: 0.421 |
Relevance: 0.429 | Completeness: 0.258 | Overall: 0.369

**Evidence inspection:**

> *Câu trả lời:*
> Retriever lấy được chunk xuất sắc: `07_repair_and_technical_support.md` (`OT-07-P03`) đứng ngay vị trí Top 1 chứa đầy đủ cả 3 thông số: chẩn đoán 3 ngày, sửa chữa 10 ngày, và leo thang khi thiếu linh kiện quá 15 ngày. Context Precision đạt 0.917 và Context Recall đạt 0.806. Tuy nhiên Generator lại trích các định nghĩa chung từ chunk `OT-06-P04` và dừng lại ngay sau khi nói về 3 ngày chẩn đoán, bỏ sót hoàn toàn mốc 10 ngày và 15 ngày.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời thiếu 2 mốc thời gian cốt lõi: 10 ngày làm việc sửa chữa và 15 ngày thiếu linh kiện để leo thang giải pháp thay thế. |
| Why 1 | Tại sao symptom xảy ra? | Generator chỉ lấy 1 câu đầu của chunk chẩn đoán và bị chiếm dung lượng bởi các định nghĩa chung về bảo hành từ chunk khác. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Generator không có cơ chế trích xuất toàn diện (Exhaustive extraction) đối với các câu hỏi hỏi về chuỗi bước / mốc thời gian ("turnaround times"). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt không yêu cầu liệt kê đầy đủ các con số / điều kiện có trong ngữ cảnh. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Thiếu module kiểm tra độ đầy đủ (Completeness Checker) đối chiếu với các thực thể số trong context. |
| Why 5 | Root cause có thể hành động được là gì? | Cải tiến Prompting với yêu cầu trích xuất số liệu/thời hạn bắt buộc (Constraint & Timeline Extraction Prompting) và tăng max generation length. |

**Root cause và proposed fix:**

> - **Root cause:** Generator tóm tắt quá mức và bị phân tán bởi các thông tin định nghĩa ngoài lề thay vì trích xuất toàn bộ chuỗi số liệu thời gian đã có trong top-1 context chunk.
> - **Proposed fix:** Áp dụng Few-shot Prompting với format dạng bảng hoặc gạch đầu dòng rõ ràng cho các câu hỏi về quy trình/thời gian: `- Chẩn đoán: ...`, `- Sửa chữa: ...`, `- Điều kiện leo thang: ...`.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | **Multi-Hop & Compound Query Splitting Gap:** Retriever dùng single-pass keyword search nên bị sót chunk thứ 2 trong các câu hỏi ghép nhiều điều kiện. | `M06`, `H02`, `A01`, `A03` | High |
| 2 | **Generative Extraction & Specificity Dilution:** Retriever tìm đúng chunk nhưng Generator trích xuất câu chung chung, bỏ qua chi tiết số liệu, thời hạn, ngoại lệ. | `H04`, `H03`, `H05`, `E02`, `M03`, `M05` | High |
| 3 | **Token Overlap Heuristic Discrepancy on Negations/Adversarial:** Cơ chế đo word-overlap phạt nặng các câu trả lời ngắn gọn/từ chối an toàn hoặc lệch cấu trúc từ. | `E01`, `E04`, `E05`, `H01`, `A02` | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> Chọn **Cluster 2 (Generative Extraction & Specificity Dilution)** vì:
> 1. **Tỷ trọng ảnh hưởng lớn nhất:** Chiếm 6/15 ca lỗi (40% tổng số lỗi) và bao gồm các ca điểm thấp nhất hệ thống (`H04`, `H03`).
> 2. **ROI (Return on Investment) kỹ thuật cao nhất:** Ở Cluster 2, tầng Retrieval đã hoàn thành xuất sắc nhiệm vụ (Context Recall > 0.8, Precision > 0.9). Do đó không cần tái lập chỉ mục hay thay đổi DB, chỉ cần cải tiến System Prompt (yêu cầu trích xuất có cấu trúc, CoT, strict grounding) là có thể cải thiện ngay điểm `Faithfulness`, `Relevance`, `Completeness` và nâng Pass Rate lên đáng kể.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Multiple issues detected — review full pipeline | Implement hallucination checker to filter unsupported claims | Open |
| F002 | hallucination | Multiple issues detected — review full pipeline | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F003 | incomplete | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F004 | incomplete | Answer is missing key information — increase context window or improve generation | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F005 | incomplete | Answer is missing key information — increase context window or improve generation | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F006 | incomplete | Answer is missing key information — increase context window or improve generation | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F007 | off_topic | Answer is missing key information — increase context window or improve generation | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F008 | off_topic | Context is missing or irrelevant — improve retrieval | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F009 | off_topic | Context is missing or irrelevant — improve retrieval | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F010 | off_topic | Answer does not address the question — improve prompt clarity | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F011 | off_topic | Answer does not address the question — improve prompt clarity | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F012 | off_topic | Answer does not address the question — improve prompt clarity | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F013 | off_topic | Context is missing or irrelevant — improve retrieval | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F014 | off_topic | Answer does not address the question — improve prompt clarity | Improve retrieval query expansion and re-ranking for better context alignment | Open |
| F015 | off_topic | Context is missing or irrelevant — improve retrieval | Improve retrieval query expansion and re-ranking for better context alignment | Open |
```

**Ba improvement suggestions ưu tiên**

1. **Structured Few-Shot Extraction Prompting:** Cung cấp mẫu câu trả lời trực diện, yêu cầu bóc tách đầy đủ tất cả các con số, mốc thời gian và điều kiện ràng buộc.
2. **Multi-Query Decomposition & Hybrid Search:** Tách câu hỏi ghép thành các truy vấn đơn và kết hợp BM25 + Dense Embeddings + RRF để khắc phục tình trạng thiếu chunk ở câu hỏi đa khía cạnh.
3. **Sentence-level Cross-Encoder Reranker:** Rerank các đoạn ngữ cảnh trích xuất và lọc bỏ các câu giới thiệu/định nghĩa chung chung trước khi nạp vào context của LLM.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Structured Few-Shot Extraction Prompting | `Completeness` (dự kiến tăng từ 0.607 lên ≥0.80), `Relevance` (tăng từ 0.533 lên ≥0.75) | Chạy lại `evaluate_answers.py` trên 20 Golden QA pairs và so sánh delta Completeness. |
| Multi-Query Decomposition & Hybrid Search | `Context Recall` (dự kiến tăng từ 0.792 lên ≥0.92, đặc biệt ở tập Hard/Medium) | Đo Context Recall trên `M06`, `H02`, `A01`, `A03` bằng `evaluate_context_recall()`. |
| Sentence-level Cross-Encoder Reranker | `Faithfulness` (tăng từ 0.581 lên ≥0.85) và `Context Precision` (tăng lên ≥0.95) | Đo tỷ lệ câu trả lời bám sát context bằng `evaluate_faithfulness()` và Reranking benchmark. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
> `run_regression()` phải được tích hợp tự động vào:
> 1. **CI/CD Pre-merge Pull Request Pipeline:** Chạy mỗi khi có commit thay đổi code retrieval, prompt template, model weights, chunking strategy hoặc update knowledge docs.
> 2. **Nightly Automated Regression Job:** Chạy định kỳ hàng đêm trên môi trường Staging với tập dữ liệu benchmark mở rộng (gồm cả các ca production logs thực tế được ẩn danh).
> 3. **Post-Incident / Post-Fix Gate:** Chạy bắt buộc trước khi promote bản vá lỗi hotfix lên môi trường Production.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:*
> **Không hoàn toàn phù hợp, mức 0.05 là quá lỏng lẻo đối với một hệ thống Customer Support bán lẻ thiết bị công nghệ cao**:
> - Đối với các chính sách nhạy cảm như hoàn tiền (`05_returns_and_exchanges.md`), bảo hành (`06_warranty_policy.md`), hay cam kết pháp lý, mức sụt giảm 5% (0.05) có thể khiến hàng chục nghìn khách hàng nhận sai quy định về phí hoàn trả hoặc thời hạn bảo hành.
> - **Đề xuất:** Cần phân tầng threshold:
>   - `Faithfulness` & `Safety/Hallucination`: Cho phép **drop = 0.00** (Zero-tolerance).
>   - `Context Recall` & `Completeness`: Cho phép **drop ≤ 0.02** (2%).
>   - `Latency` & `Token Cost`: Cho phép dao động **≤ 0.05** (5%).

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **BLOCK DEPLOYMENT (Hard Quality Gate - Build Fails):**
>   1. Tỷ lệ lỗi `hallucination` > 0 (bất kỳ ca bịa đặt thông tin chính sách bảo hành/hoàn tiền nào đều phải chặn deploy).
>   2. Điểm trung bình `Faithfulness` giảm > 0.02 so với baseline.
>   3. Điểm trung bình `Context Recall` giảm > 0.02.
>   4. Bất kỳ ca nào trong bộ Golden Test Set chuyển từ `Passed` sang `Failed`.
> - **ALERT ONLY (Soft Quality Gate - Gửi Slack/PagerDuty cảnh báo):**
>   1. `Latency_ms` tăng từ 5% – 10% (chưa ảnh hưởng nghiêm trọng đến UX).
>   2. `Cost_usd` / `tokens_used` tăng nhẹ dưới 10%.
>   3. `Context Precision` giảm nhẹ ≤ 0.03 nhưng `Context Recall` và `Faithfulness` không đổi.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit & Syntactic Eval] → [Golden Benchmark Regression Gate] → [LLM-as-Judge & Canary Sandbox] → Deploy
```

> *Giải thích:*
> 1. **Unit & Syntactic Eval:** Kiểm tra cú pháp, format JSON output, latency trần và các hàm helper.
> 2. **Golden Benchmark Regression Gate:** Chạy `run_regression()` trên 20 Golden QA pairs và 100+ synthetic QA pairs để chặn các hồi quy về độ chính xác chính sách.
> 3. **LLM-as-Judge & Canary Sandbox:** Sử dụng mô hình giám khảo độc lập chấm điểm chất lượng ngữ nghĩa và định tuyến 5% traffic thực tế (Canary deployment) trước khi mở 100% traffic toàn hệ thống.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Cải tiến Prompt sinh câu trả lời với Chain-of-Thought và Few-shot Schema | `Faithfulness` (+0.25), `Completeness` (+0.20) | Loại bỏ các câu mở đầu lan man, trích xuất chính xác 100% các mốc ngày và điều kiện hoàn tiền. |
| 2 | Tích hợp Query Decomposition và Hybrid Search (Dense + BM25) | `Context Recall` (+0.15) | Bắt trọn vẹn cả 2 khía cạnh trong các câu hỏi ghép nhiều vế như `M06`, `H02`. |
| 3 | Tích hợp Reranker mô hình Cross-Encoder (FlashRank / Cohere / BGE) | `Context Precision` (+0.10) | Đẩy chunk chứa câu trả lời trực tiếp lên vị trí Top 1 trước khi đưa vào LLM. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Multi-condition Policy Interaction:** *"Khách hàng thành viên OrbitPlus mua NovaBook 14 bằng kết hợp 1 Gift Card và Thẻ tín dụng, sau đó hủy đơn sau 35 ngày thì được hoàn tiền như thế nào và thời hạn nhận là bao lâu?"* (Kiểm tra tương tác giữa membership, thời hạn 30 ngày vs 60 ngày của OrbitPlus, và phân bổ phương thức hoàn tiền).
> 2. **Conflict Resolution & Exception Precedence:** *"Nếu bưu tá giao hàng trễ gói chuyển phát nhanh do thời tiết xấu nhưng gói hàng lại bị móp méo rơi vỡ, khách hàng được hoàn phí vận chuyển hay được đổi mới thiết bị?"* (Kiểm tra khả năng phân định ngoại lệ giao hàng trễ do thời tiết vs quy trình đổi trả hàng hỏng trong 48h).
> 3. **Adversarial / Cross-corpus Trap:** *"Tôi có thể mang PulsePhone X đến trung tâm OrbitTech để thay pin miễn phí sau 30 tháng sử dụng không?"* (Kiểm tra khả năng bắt lỗi thời hạn bảo hành 24 tháng và phân biệt bảo hành lỗi kỹ thuật với bảo dưỡng pin hao mòn).

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> Điểm bất ngờ lớn nhất là **Sự phân kỳ rõ rệt giữa Retrieval Quality và Generation Quality**:
> Ban đầu, ta thường giả định rằng nếu Retriever đạt `Context Precision` rất cao (~0.897) và `Context Recall` tốt (~0.792) thì hệ thống RAG sẽ trả lời chính xác và đạt pass rate cao. Tuy nhiên, kết quả thực tế cho thấy Pass Rate chỉ đạt **25%** do khâu Generation bị "Information Distraction" (nhiễu loạn thông tin): khi nhận được 5 chunk context, mô hình dễ bị cuốn vào các đoạn giới thiệu chung chung hoặc định nghĩa tổng quan ở đầu chunk mà bỏ qua các con số quy định cụ thể nằm ở cuối chunk.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> - **Giới hạn của Word-Overlap Heuristics:**
>   1. **Không hiểu ngữ nghĩa (Semantic Blindness):** Phạt nặng các câu trả lời đúng bản chất nhưng dùng từ đồng nghĩa hoặc hành văn ngắn gọn súc tích (ví dụ: "7 business days" vs "một tuần làm việc").
>   2. **Dễ bị đánh lừa bởi Keyword Stuffing:** Một câu trả lời bịa đặt nhưng chứa nhiều từ khóa ngẫu nhiên của context vẫn nhận điểm `Faithfulness` cao do trùng lặp token.
>   3. **Mù với Phủ định và Điều kiện Logic (Negation Blindness):** Không phân biệt được "được phép hoàn tiền" và "không được phép hoàn tiền" nếu chỉ đếm tập hợp từ vựng giao nhau.
>
> - **Thay thế và Bổ sung trong Production:**
>   1. **LLM-as-a-Judge (G-Eval / DeepEval / RAGAS LLM-based):** Sử dụng LLM mạnh (như GPT-4o hoặc Gemini 1.5 Pro) với Rubric thang điểm 1–5 có chain-of-thought để đánh giá Faithfulness, Answer Relevance và Semantic Completeness.
>   2. **Embedding-based Semantic Similarity (BERTScore / Cross-Encoder Sim):** Đo độ tương đồng ngữ nghĩa phân bố không gian thay vì so sánh chuỗi token rời rạc.
>   3. **Factual NLI (Natural Language Inference):** Áp dụng mô hình phân loại Premise-Hypothesis (Entailment / Contradiction / Neutral) để phát hiện chính xác 100% các câu có yếu tố Hallucination.
>   4. **Deterministic Entity / Constraint Extractor:** Dùng regex/rule để kiểm tra tự động xem các số liệu quan trọng (ví dụ `USD 1,000`, `48 hours`, `15 business days`) có xuất hiện chính xác trong câu trả lời hay không.
