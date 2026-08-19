# Báo cáo thực hành và thuyết minh kỹ thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đỗ Tuấn Sơn

**Khóa học:** AICB-K34 · Track 3: GraphRAG

**Ngày thực hiện:** 19/08/2026

## 1. Tóm tắt kết quả

Pipeline xử lý 5.000 bản ghi nguồn, còn 2.105 bài sau exact dedup và chọn 1.500 bài cho lab. FAISS index chứa 1.500 chunk. Với OpenAI, NER/RE sinh 176 triple và không có batch extraction lỗi; lượt ingestion ghi 297 node và 175 edge. Database sau khi chạy có tổng cộng 340 node, 219 edge và không có edge thiếu provenance. Benchmark gồm 5 câu thuộc đủ ba nhóm `factoid`, `multi-hop`, `cross-doc`.

Chênh lệch giữa 297 node/175 edge của lượt cuối và tổng 340 node/219 edge cho thấy Neo4j còn dữ liệu từ các lượt chạy trước. Vì vậy số tổng và bảng degree phản ánh trạng thái database tích lũy, không chỉ riêng 176 triple của lượt extraction OpenAI; đây là một giới hạn về tính cô lập của benchmark.

Kết quả OpenAI khắc phục failure mode extraction của lượt Groq trước, nhưng Entity Resolution audit mới có một quyết định `MERGE_VECTOR`, chưa đạt yêu cầu tối thiểu 10 dòng. Không có node degree > 100 và không truy vấn nào kích hoạt super-node cap. Các giới hạn này được giữ nguyên trong báo cáo thay vì bổ sung dữ liệu giả.

## 2. Thuyết minh kỹ thuật và phân tích ca lỗi

### 2.1. Coreference Resolution

Coreference dùng `resolve_coref_batch()` với nguyên tắc bảo thủ: chỉ thay đại từ khi antecedent rõ trong cùng chunk; nếu không chắc thì giữ nguyên văn bản và ghi `unresolved_mentions`. Ví dụ, chunk `4f1346392056a403277d::c0000` nói về giao dịch Aeris–Ericsson và chứa nhiều chủ thể/doanh nghiệp trong cùng đoạn. Nếu một tham chiếu như “the company” hoặc “its” bị gán nhầm từ Ericsson sang Aeris, extractor có thể tạo sai chiều của cạnh mua lại hoặc gán tài sản IoT cho sai doanh nghiệp.

Output notebook lần OpenAI không xuất bảng tổng hợp `coref_failures`, nên không đủ bằng chứng để khẳng định tỷ lệ thành công/thất bại của 400 chunk. Tuy nhiên, extraction sau đó hoàn thành cả 100 batch, tạo 176 triple và không ghi extraction error, cho thấy pipeline downstream đã hoạt động ổn định hơn lượt Groq. Để audit coreference chặt chẽ, notebook nên xuất tổng số `COREF_BATCH_FAILED`, phân bố `unresolved_mentions` và một mẫu before/after theo `chunk_id`, thay vì chỉ hiển thị progress bar.

### 2.2. Entity Resolution Threshold và Lexical Guard

Ngưỡng vector matching được đặt là cosine similarity `0.90`. Sau ANN candidate search, `merge_guard()` bỏ hậu tố doanh nghiệp và yêu cầu `SequenceMatcher >= 0.72`. Chỉ entity cùng type mới được so sánh; manual alias xử lý các tên như `MSFT`/`Microsoft Corp` → `Microsoft`.

Audit ghi nhận một phép gộp vector: `Fidelity National Information Services` với `Fidelity National Information Services Inc.`, cosine similarity `0.924524`, quyết định `MERGE_VECTOR`. Lexical guard cho phép gộp vì khác biệt chỉ là hậu tố doanh nghiệp `Inc.` và `strip_suffix()` đưa hai tên về cùng dạng chuẩn.

Không có cặp `REJECT_GUARD`, nên không thể trích một trường hợp similarity > 0.85 bị chặn mà không bịa dữ liệu. Audit chỉ có một dòng, vẫn thấp hơn yêu cầu 10+. Để tăng audit coverage mà không làm sai chính sách merge, phiên bản production nên log cả top-k candidate dưới threshold với quyết định `BELOW_VECTOR_THRESHOLD`, và log `REJECT_GUARD` cho mọi candidate vượt ngưỡng vector nhưng không đạt lexical ratio. Không nên hạ threshold chỉ để tạo thêm dòng audit.

### 2.3. Đặc trưng đồ thị và Super-node Mitigation

Top 3 node theo degree trong database:

| Hạng | Thực thể | Type | Degree |
|---:|---|---|---:|
| 1 | Amazon Web Services | Company | 5 |
| 2 | Raleon | Company | 4 |
| 3 | L&T Technology Services Limited | Company | 3 |

Không node nào đạt `SUPER_NODE_DEGREE = 100`; degree lớn nhất chỉ là 5 và cả 5 truy vấn benchmark đều có `graph_supernode_events = 0`. Vì thế lượt chạy chỉ xác nhận nhánh bình thường, chưa kích hoạt thực nghiệm nhánh cap.

Policy trong code giới hạn node có degree > 100 còn tối đa 50 edge mới nhất, toàn retrieval không quá 250 edge và graph context không quá 14.000 ký tự. Ưu điểm là chặn BFS explosion, giảm latency/token và ưu tiên trạng thái gần hiện tại. Rủi ro là câu hỏi lịch sử có thể mất edge cũ nhưng quan trọng; sắp xếp chỉ theo ngày cũng có thể ưu tiên tin mới nhưng kém liên quan. Cải tiến phù hợp là xếp hạng kết hợp relevance, recency, confidence và relation diversity, đồng thời dành một quota nhỏ cho edge lịch sử.

### 2.4. Benchmark Flat RAG và GraphRAG

Các số dưới đây là trung bình trên 5 Golden queries:

| Tiêu chí | Flat RAG | GraphRAG | Δ Graph − Flat | Nhận xét |
|---|---:|---:|---:|---|
| Comprehensiveness (1–5) | 4.80 | 4.80 | 0.00 | Hai phương pháp ngang nhau |
| Faithfulness (1–5) | 4.80 | 5.00 | +0.20 | GraphRAG tốt hơn ở G5000-07 |
| Multi-hop reasoning (1–5) | 4.80 | 4.80 | 0.00 | Hai phương pháp ngang nhau |
| Latency trung bình (s) | 1.138 | 1.879 | +0.741 | GraphRAG chậm hơn khoảng 65,1% |
| Token usage trung bình | 609.2 | 598.4 | -10.8 | GraphRAG dùng ít hơn khoảng 1,8% |

Không có ca nào mà Flat thất bại hoàn toàn còn Graph thành công hoàn toàn. Lợi thế rõ nhất của GraphRAG là **G5000-07**: cả hai đạt 4/5 về comprehensiveness và multi-hop, nhưng GraphRAG đạt faithfulness 5/5 trong khi Flat chỉ đạt 4/5. Graph answer đưa được cả nhóm feature `Now Assist for Virtual Agent`, case summarization, text-to-code và tách chúng khỏi chương trình AI Lighthouse với NVIDIA/Accenture. Việc kết hợp context theo entity/relationship giúp câu trả lời bám evidence tốt hơn, dù so sánh hai nhánh vẫn chưa đủ sắc để đạt comprehensiveness 5/5.

Ca khó khăn chung cũng là **G5000-07**. Judge nhận xét hai câu trả lời chưa mô tả đủ chức năng cụ thể của ecosystem program và chưa so sánh hai sáng kiến thật trực diện. Đây không phải lỗi super-node vì `graph_supernode_events = 0`; nguyên nhân là evidence của AI Lighthouse chủ yếu mô tả chương trình ở mức khái quát, trong khi schema allowlist không có relation `LAUNCHED`. Cải tiến là mở rộng schema bằng `LAUNCHED`, thêm node `Program/Product`, và yêu cầu generator lập bảng đối chiếu “internal platform feature” với “external ecosystem program”.

GraphRAG dùng ít hơn 10,8 token trung bình nhưng chậm hơn khoảng 0,741 giây. Token gần như ngang nhau vì GraphRAG lấy bốn vector chunks rồi cộng graph context, còn Flat lấy sáu vector chunks. Latency cao hơn phản ánh thêm chi phí seed extraction, Neo4j traversal và linearization; đặc biệt G5000-07 mất 3,521 giây so với 1,953 giây của Flat.

### 2.5. Trade-offs, kiểm soát AI Coding Agent và scale 350 MB

Flat RAG có pipeline index đơn giản, recall tốt cho fact xuất hiện trực tiếp và chi phí vận hành thấp. GraphRAG phải trả thêm chi phí coreference, NER/RE, entity resolution, Neo4j ingestion và traversal, nhưng có tiềm năng tốt hơn khi câu hỏi cần đường quan hệ rõ, provenance hoặc temporal reasoning. Kết quả lần này cho thấy graph chất lượng thấp có thể kém Flat RAG dù kiến trúc phức tạp hơn.

Một đề xuất không áp dụng là so sánh cosine mọi cặp entity/chunk theo `O(N²)`. Ở quy mô 100.000 bài, cách này gây bùng nổ CPU/RAM và khó audit. Pipeline dùng FAISS ANN để sinh top-k candidate, sau đó mới áp lexical guard. Tương tự, không hạ threshold chỉ để tạo đủ audit row vì sẽ tăng false merge.

Khi scale toàn bộ 350 MB, bottleneck đầu tiên là số lời gọi LLM cho coreference và extraction, thể hiện trực tiếp qua lỗi quota/rate limit trong lab. Kiến trúc đề xuất:

1. Queue bất đồng bộ với bounded concurrency, retry/backoff và checkpoint idempotent theo `chunk_id`.
2. Lọc/routing trước bằng heuristic hoặc model nhỏ; chỉ gửi chunk có khả năng chứa relation sang model lớn.
3. Lưu raw response, schema-validation error và token usage để chạy lại riêng batch lỗi.
4. Entity blocking theo type/ticker/domain, ANN HNSW/FAISS và lexical guard thay cho all-pairs.
5. Neo4j `UNWIND` theo batch, partition theo thời gian/community và incremental update.
6. Retrieval kết hợp relation relevance, confidence, thời gian và community summaries.

## 3. Reflection và Action Plan

### 3.1. Mapping bài giảng vào code

| Khái niệm | Module | Hàm/khối code | Quan sát thực tế |
|---|---|---|---|
| Conservative Coreference | Module 1 | `resolve_coref_batch()`, `run_coref()` | Fallback giữ nguyên text tránh hallucination; cần xuất thêm thống kê audit runtime |
| Schema và Allowlist Guard | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `run_extraction()` | OpenAI xử lý 100 batch không lỗi và tạo 176 triple hợp lệ |
| Bulk Cypher Ingestion | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Dùng `UNWIND $rows`; provenance check bằng 0 |
| Entity Resolution và Union-Find | Module 3 | `build_resolution_map()`, `UF`, `merge_guard()` | Gộp đúng một cặp hậu tố `Inc.` ở similarity 0.9245; audit coverage vẫn thấp |
| Flat Vector Retrieval | Module 4 | `build_flat_index()`, `retrieve_flat_context()` | IndexFlatIP chứa 1.500 vector; hoạt động tốt trên bộ Golden đã chọn |
| BFS và Super-node Cap | Module 4 | `retrieve_graph_context()`, `recent_edges()` | Cap 100→50 được cài đặt nhưng không kích hoạt vì degree lớn nhất là 5 |
| Hybrid Generation | Module 4 | `answer_graph_rag()` | Ghép graph context và top-4 vector chunks; faithfulness tốt hơn Flat ở G5000-07 |
| LLM-as-a-Judge | Module 5 | `judge_answer()`, `run_evaluation()` | Chấm 5 câu, có checkpoint; cả hai còn thiếu độ đầy đủ ở G5000-07 |

### 3.2. Debugging và bài học

Lỗi đầu tiên là `AttributeError: DataFrame has no attribute source_raw`. Nguyên nhân thực là `run_extraction()` trả DataFrame rỗng không có schema sau khi tất cả batch thất bại, không phải `canonicalize_triples()` dùng sai tên cột. Fix là luôn tạo DataFrame với danh sách cột cố định và fail-fast nếu không có triple.

Lỗi thứ hai là HTTP 429 khi Groq model đạt 199.981/200.000 token/ngày trong lúc seed extraction. Sau khi thử thay key, pipeline được chuyển toàn bộ sang provider-neutral wrapper và dùng OpenAI cho coreference, extraction, seed, generation và Judge. Lần chạy OpenAI tạo 176 triple với 0 extraction error, so với chỉ 3 triple và 96 batch lỗi ở lượt Groq. Evaluation checkpoint giúp tiếp tục mà không phải lặp lại phần đã hoàn tất.

Bài học chính là không coi pipeline “chạy hết cell” đồng nghĩa dữ liệu đạt chất lượng. Các tỷ lệ fallback, extraction error, audit coverage và graph freshness phải trở thành quality gate trước benchmark.

### 3.3. Kế hoạch áp dụng

Đề xuất áp dụng cho hệ thống theo dõi tin tức công nghệ và quan hệ doanh nghiệp. Flat RAG đủ cho tra cứu một bài hoặc một fact; Hybrid GraphRAG phù hợp khi cần nối sự kiện đầu tư, mua bán, nhân sự, sản phẩm và đối tác qua nhiều nguồn/thời điểm.

- Nodes: `Company`, `Person`, `Technology`, mở rộng thêm `Product`, `Event`, `Article`.
- Relations: `ACQUIRED`, `INVESTED_IN`, `FOUNDED`, `WORKED_AT`, `DEVELOPED`, `USES`, `PARTNERED_WITH`, `LAUNCHED`, `SUPPORTS`, `MENTIONED_IN`.
- Mọi edge lưu `source_chunk_id`, URL, `published_date`, evidence, confidence và event state.
- Entity resolution dùng manual aliases cho ticker, blocking theo type, ANN candidate search và lexical/domain guard; merge có version và khả năng rollback.
- Super-node dùng per-relation quota, relevance + temporal ranking và community partition thay vì chỉ lấy 50 edge mới nhất.
- Query router chọn Flat cho factoid, Hybrid cho entity/path query và community reports cho câu hỏi tổng hợp toàn cục.

## 4. Tự đánh giá

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Hiểu kiến trúc GraphRAG | 4 | Phân biệt được extraction, resolution, traversal và hybrid retrieval |
| Kiểm soát AI Coding Agent | 4 | Không chấp nhận số liệu/candidate giả và tránh giải pháp `O(N²)` |
| Chất lượng Knowledge Graph | 3 | 176 triple, provenance đầy đủ; audit và super-node test thực tế chưa đạt |
| Khả năng phân tích/debug | 4 | Truy được lỗi DataFrame rỗng, quota API và giới hạn benchmark |

## 5. Artifact đối chứng

- `outputs/graphrag_eval_results.csv`: kết quả từng Golden query và Judge rationale.
- `outputs/graphrag_vs_flatrag_summary.csv`: tổng hợp theo nhóm câu hỏi.
- `outputs/graph_integrity_summary.csv`: 340 node, 219 edge, 0 edge thiếu provenance.
- `outputs/top_degree_entities.csv`: bảng degree dùng cho phân tích super-node.
- `outputs/entity_resolution_audit.csv`: một `MERGE_VECTOR` hợp lệ ở similarity 0.924524; chưa có `REJECT_GUARD`.
