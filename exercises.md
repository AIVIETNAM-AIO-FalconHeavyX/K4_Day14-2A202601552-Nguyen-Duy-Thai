# Day 14 — Bài làm AI Evaluation & Benchmarking

## Domain: OrbitTech Store Customer Support

### Part 1 — Warm-up

#### Exercise 1.1 — Ngưỡng RAGAS

| Metric | Điểm thấp vẫn có thể chấp nhận | Khi nào là critical | Hành động |
|---|---|---|---|
| Faithfulness | Một vài câu hỏi mở hoặc adversarial có câu trả lời giới hạn; vẫn phải tránh claim không có evidence. | Dưới 0.6 trên câu hỏi chính sách, thanh toán, an toàn hoặc privacy. | Kiểm tra claim ngoài context, thêm grounding guardrail và test hallucination. |
| Answer Relevance | Câu hỏi có nhiều ý và câu trả lời còn thiếu một ý phụ nhưng vẫn đúng intent chính. | Dưới 0.6 hoặc trả lời lệch intent, nhất là câu hỏi về đơn hàng và bảo mật. | Cải thiện intent routing, prompt yêu cầu trả lời từng phần. |
| Context Recall | Một số câu easy chỉ cần một đoạn nên recall dao động nhẹ. | Thấp trên câu hỏi nhiều điều kiện, vì generator không thể trả lời đủ nếu retriever bỏ sót evidence. | Mở rộng query/chunking, thêm gold evidence và regression case. |
| Context Precision | Có thể chấp nhận một ít noise ở các câu tổng quát nếu relevant chunk vẫn đứng đầu. | Thấp kéo dài hoặc relevant chunk bị đẩy xuống sau nhiều noise. | Rerank, cải thiện BM25/query và theo dõi AP@K. |
| Completeness | Câu trả lời ngắn nhưng đã bao phủ toàn bộ claim cần thiết. | Dưới 0.6 ở câu có ngày, số tiền, điều kiện hoặc ngoại lệ bắt buộc. | Thêm checklist điều kiện/ngoại lệ và tăng evidence coverage. |

#### Exercise 1.2 — Bias của LLM-as-a-Judge

**Câu 1 — thí nghiệm position bias:** dùng cùng 30 cặp câu trả lời A/B, giữ nguyên question và rubric. Chạy condition 1 với A đứng trước B, condition 2 đảo thành B đứng trước A; randomize thứ tự mỗi cặp và dùng temperature 0. So sánh tỷ lệ chọn A/B và chênh lệch điểm. Nếu câu trả lời thắng thay đổi chủ yếu theo vị trí, đó là position bias.

**Câu 2 — giảm verbosity bias:** rubric chấm coverage của claim bắt buộc, correctness và evidence trước độ dài. Quy định câu trả lời ngắn nhưng đủ điều kiện đạt điểm cao; câu dài chỉ được cộng điểm khi có thông tin đúng và liên quan. Phạt lặp lại, preamble và claim ngoài evidence.

**Câu 3 — cần calibrate với human labels:** human review tạo mốc tham chiếu cho các ca khó như refusal, privacy, ngày hiệu lực và ngoại lệ. Calibration giúp phát hiện judge quá dễ/quá gắt, đo agreement, chỉnh rubric và tránh biến điểm tự động thành sự thật tuyệt đối.

#### Exercise 1.3 — Evaluation trong CI/CD

| Metric | Threshold block deploy | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Claim không có căn cứ có rủi ro cao, đặc biệt với payment, warranty và safety. |
| Answer Relevance | 0.65 | Đảm bảo agent xử lý đúng intent thay vì chỉ lặp lại context. |
| Completeness | 0.65 | Giữ lại ngày, số tiền, điều kiện và ngoại lệ quan trọng. |

Offline evaluation chạy ở mỗi release, prompt change hoặc retriever/chunking change. Online evaluation theo dõi traffic thật, drift, latency và complaint signals sau deploy. Human review bắt buộc cho privacy, safety, fraud, disputed warranty và các thay đổi rubric lớn.

### Part 2 — Core Coding

Đã hoàn thiện các Task 1–5 trong `template.py` và đồng bộ sang `solution/solution.py`:

- `QAPair`, `EvalResult`, `overall_score()`.
- Faithfulness, relevance, completeness.
- Context recall, rank-aware context precision và wiring qua `run_full_eval()`.
- `LLMJudge`, bias detection.
- `BenchmarkRunner`, report, regression và failure filtering.
- `FailureAnalyzer`, suggestions và improvement log.
- Bonus `rerank_by_overlap()`.

Kiểm thử cuối: `python -m unittest tests.test_solution -q` → **42 tests OK**.

### Part 3 — Golden Dataset & Benchmark

#### Exercise 3.1 — Kết quả dataset

| Hạng mục | Kết quả |
|---|---:|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

Ba case đại diện:

| ID | Difficulty | Source | Lý do |
|---|---|---|---|
| E01 | Easy | `01_product_catalog.md` | Tra cứu trực tiếp thông số NovaBook trong một đoạn. |
| H01 | Hard | `09_escalation_and_policy_updates.md`, `03_promotions_and_membership.md` | Phải phân biệt policy version theo ngày đặt hàng với lợi ích OrbitPlus. |
| A02 | Adversarial | `00_system_scope.md` | Kiểm tra prompt injection và nguyên tắc không tiết lộ prompt/credential. |

Khó nhất là giữ expected answer đủ ngắn để benchmark ổn định nhưng không bỏ sót ngày, số tiền, điều kiện và ngoại lệ. Evidence được sao chép nguyên văn từ corpus; validator xác nhận mọi đoạn hợp lệ và phủ đủ 10 tài liệu.

- Mọi claim trong expected answer đều có evidence hỗ trợ.
- Không có question trùng ý và không dùng kiến thức ngoài corpus.
- `python validate_golden_dataset.py` báo **PASS**.

#### Exercise 3.2 — Benchmark run

| ID | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure |
|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | 0.958 | 1.000 | 0.575 | 0.875 | 1.000 | 0.817 | Yes | — |
| E02 | 1.000 | 1.000 | 0.786 | 0.750 | 1.000 | 0.845 | Yes | — |
| E03 | 0.857 | 1.000 | 0.909 | 0.600 | 0.714 | 0.741 | Yes | — |
| E04 | 0.833 | 0.950 | 0.800 | 0.875 | 0.667 | 0.781 | Yes | — |
| E05 | 1.000 | 0.887 | 0.706 | 0.625 | 0.632 | 0.654 | Yes | — |
| M01 | 0.967 | 1.000 | 0.941 | 0.750 | 0.833 | 0.842 | Yes | — |
| M02 | 0.906 | 0.806 | 0.829 | 0.250 | 0.906 | 0.662 | No | irrelevant |
| M03 | 0.971 | 1.000 | 0.810 | 0.700 | 0.735 | 0.748 | Yes | — |
| M04 | 0.914 | 0.950 | 0.842 | 0.583 | 0.771 | 0.732 | Yes | — |
| M05 | 0.917 | 1.000 | 0.923 | 0.750 | 0.833 | 0.835 | Yes | — |
| M06 | 0.939 | 0.833 | 0.833 | 0.800 | 0.848 | 0.827 | Yes | — |
| M07 | 0.967 | 1.000 | 0.788 | 0.692 | 0.833 | 0.771 | Yes | — |
| H01 | 0.870 | 1.000 | 0.857 | 0.875 | 0.304 | 0.679 | No | off_topic |
| H02 | 0.857 | 0.867 | 0.500 | 0.923 | 0.571 | 0.665 | Yes | — |
| H03 | 0.852 | 1.000 | 0.600 | 0.857 | 0.630 | 0.696 | Yes | — |
| H04 | 0.976 | 0.806 | 0.558 | 0.643 | 0.561 | 0.587 | Yes | — |
| H05 | 1.000 | 1.000 | 0.700 | 0.500 | 0.650 | 0.617 | Yes | — |
| A01 | 0.857 | 0.804 | 0.294 | 0.700 | 0.286 | 0.427 | No | hallucination |
| A02 | 0.773 | 1.000 | 0.333 | 0.000 | 0.091 | 0.141 | No | irrelevant |
| A03 | 1.000 | 1.000 | 0.476 | 0.625 | 0.400 | 0.500 | No | off_topic |

**Aggregate report:**

- Overall pass rate: **75.0%** (15/20).
- Avg Context Recall: **0.921**.
- Avg Context Precision: **0.945**.
- Avg Faithfulness: **0.703**.
- Avg Relevance: **0.669**.
- Avg Completeness: **0.663**.
- Failure type distribution: `irrelevant=2`, `off_topic=2`, `hallucination=1`.

Ba case thấp nhất là A02 (0.141, irrelevant), A01 (0.427, hallucination), A03 (0.500, off_topic). Retrieval nhìn chung tốt vì recall và precision đều trên 0.92; điểm yếu chủ yếu nằm ở generation/heuristic trên các câu refusal và out-of-scope. M02 cũng cho thấy câu trả lời grounded nhưng overlap với wording của question thấp, nên relevance heuristic phạt mạnh.

#### Exercise 3.3 — Rubric LLM-as-a-Judge domain-specific

Các dimension được chọn: correctness, completeness, evidence, safety/privacy và actionability.

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Đúng hoàn toàn theo corpus, đủ mọi điều kiện/ngoại lệ, không claim ngoài evidence, xử lý safety/privacy đúng và đưa next step phù hợp. | Nêu đúng version, ngày, phí, điều kiện; refusal đúng khi cần. |
| 4 | Đúng phần chính, chỉ thiếu một chi tiết nhỏ không làm đổi quyết định; evidence và safety vẫn đúng. | Thiếu một câu giải thích phụ nhưng vẫn nêu đúng deadline và ngoại lệ. |
| 3 | Đúng một phần nhưng bỏ sót điều kiện quan trọng hoặc next step; không có claim nguy hiểm nghiêm trọng. | Nêu 30 ngày nhưng quên OrbitPlus chỉ mở rộng unopened device. |
| 2 | Có sai policy hoặc thiếu nhiều claim bắt buộc; evidence yếu, hướng dẫn chưa an toàn. | Gộp nhầm return window và warranty hoặc hứa refund khi chưa đủ dữ kiện. |
| 1 | Sai/không liên quan, bịa claim, làm theo prompt injection, xin credential hoặc đưa hướng dẫn safety nguy hiểm. | Tiết lộ prompt, yêu cầu OTP hoặc khẳng định chắc chắn một exception không có trong corpus. |

Edge cases:

| Edge case | Vì sao khó | Cách rubric xử lý |
|---|---|---|
| Refusal ở A01/A02 | Câu trả lời ngắn có ít token overlap nhưng lại là hành vi đúng. | Chấm correctness/safety theo policy trước, không thưởng độ dài. |
| H01 policy version | Có nhiều mốc ngày và membership exception. | Thiếu ngày đặt hàng hoặc áp sai version bị trừ mạnh completeness/correctness. |
| A03 thiếu order details | Không thể xác minh live order hay hứa exception. | Câu trả lời phải nêu limitation và hướng support; không phạt vì không đưa refund. |

Bias controls: randomize vị trí câu trả lời khi so sánh; chấm checklist claim bắt buộc thay vì số chữ; dùng nhiều judge hoặc human calibration cho mẫu nhỏ; tách reasoning khỏi score và không cho judge biết model sinh answer.

#### Exercise 3.4 — Framework comparison (bonus)

| Tiêu chí | RAGAS | DeepEval |
|---|---|---|
| Setup complexity | Cần dataset mẫu và metric phù hợp với RAG; cấu hình evaluator rõ. | Gần pytest, thuận tiện tạo test case và assertion trong CI. |
| Metrics | Faithfulness, answer relevancy, context recall/precision và metric RAG khác. | Faithfulness, answer relevancy, contextual metrics và custom/LLM metrics. |
| CI/CD | Có thể chạy batch nhưng cần quản lý dataset/cost. | Tích hợp test gate tự nhiên hơn với pipeline pytest. |
| Kết quả trên cùng input | Có thể nghiêm hơn ở grounding/context vì chấm RAG trực tiếp. | Có thể dễ tùy chỉnh rubric và failure assertion. |
| Insight | Phù hợp chẩn đoán retrieval + generation. | Phù hợp biến tiêu chí thành regression test theo release. |

Với OrbitTech, nên dùng RAGAS-style retrieval metrics để chẩn đoán corpus và DeepEval/pytest để chặn regression. Hai framework có thể không cho cùng score vì prompt judge, calibration và cách định nghĩa relevance khác nhau; vì vậy cần giữ cùng dataset, seed và rubric khi so sánh.

#### Exercise 3.5 — Reranking (bonus)

| ID | Recall trước | Recall sau | Precision trước | Precision sau | Delta |
|---|---:|---:|---:|---:|---:|
| E01 | 0.958 | 0.958 | 1.000 | 1.000 | +0.000 |
| E02 | 1.000 | 1.000 | 1.000 | 1.000 | +0.000 |
| E03 | 0.857 | 0.857 | 1.000 | 1.000 | +0.000 |
| E04 | 0.833 | 0.833 | 0.950 | 1.000 | +0.050 |
| E05 | 1.000 | 1.000 | 0.887 | 1.000 | +0.113 |
| **Avg** | **0.930** | **0.930** | **0.967** | **1.000** | **+0.033** |

Recall không đổi vì reranking chỉ đổi thứ tự, không thêm hoặc xóa chunk; union evidence vẫn giữ nguyên. Reranking không đủ khi relevant evidence không được retrieve, query có synonym/ambiguity lớn, hoặc chunk bị cắt làm mất điều kiện. Khi đó phải sửa retriever, query expansion hoặc chunking.

### Completion checklist

- [x] Required tests pass: 42/42.
- [x] Golden dataset validator PASS.
- [x] Golden dataset đủ 5 Easy + 7 Medium + 5 Hard + 3 Adversarial.
- [x] Benchmark có đủ năm metrics và ba case thấp nhất.
- [x] Rubric 1–5 và bias controls đã hoàn thành.
- [x] Reflection và regression strategy đã hoàn thành trong `reflection.md`.
- [x] `solution/solution.py` đã đồng bộ từ `template.py`.
