# Thuyết minh kỹ thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đỗ Tuấn Sơn

**Dữ liệu thực nghiệm:** 5.000 bản ghi nguồn; 2.105 bài sau exact dedup; 1.500 bài/chunk trong lab; 176 triple được trích xuất bằng OpenAI; 340 node, 219 edge và 0 edge thiếu provenance trong Neo4j.

## 1. Coreference sai hoặc gặp khó khăn ở tình huống nào?

Chunk `4f1346392056a403277d::c0000` mô tả giao dịch giữa Aeris và Ericsson. Một đoạn chứa đồng thời bên mua, bên chuyển giao và các tài sản “IoT Accelerator” và “Connected Vehicle Cloud”; vì vậy các cụm như “the company” hoặc “its” có thể bị gán nhầm giữa Aeris và Ericsson. Nếu coreference đảo antecedent, extractor có thể tạo cạnh `ACQUIRED` sai chiều hoặc gán công nghệ cho sai doanh nghiệp.

Pipeline áp dụng chính sách bảo thủ: chỉ resolve khi antecedent rõ trong cùng chunk; trường hợp mơ hồ giữ nguyên và log `unresolved_mentions`. Output OpenAI lần cuối không xuất tổng số unresolved/fallback, nên không đủ bằng chứng để tuyên bố tỷ lệ coreference thành công. Đây là một thiếu sót audit cần bổ sung.

## 2. Entity Resolution threshold bao nhiêu và vì sao?

Ngưỡng cosine similarity là `0.90`. Mức này ưu tiên precision vì false merge nguy hiểm hơn missed merge: gộp sai hai entity sẽ tạo đường suy luận giả trên toàn bộ graph. ANN chỉ tạo candidate; quyết định cuối còn qua type constraint và lexical guard `SequenceMatcher >= 0.72` sau khi bỏ hậu tố doanh nghiệp.

Audit ghi nhận `Fidelity National Information Services` và `Fidelity National Information Services Inc.` có similarity `0.924524` và được `MERGE_VECTOR`. Đây là quyết định hợp lý vì khác biệt chỉ là hậu tố `Inc.`.

## 3. Candidate similarity cao nào không nên merge?

Lần chạy này không phát sinh `REJECT_GUARD`; `entity_resolution_audit.csv` chỉ có một `MERGE_VECTOR`. Vì vậy không có candidate runtime hợp lệ để trích dẫn như một cặp similarity > 0.85 bị từ chối. Không nên bịa ví dụ thay cho log thực nghiệm.

Để audit đầy đủ hơn, hệ thống nên ghi cả top-k candidate dưới threshold với `BELOW_VECTOR_THRESHOLD` và mọi candidate vượt vector threshold nhưng trượt lexical guard với `REJECT_GUARD`. Cách này tăng khả năng giải thích mà không hạ threshold merge.

## 4. Top 3 node theo degree là gì?

| Hạng | Entity | Type | Degree |
|---:|---|---|---:|
| 1 | Amazon Web Services | Company | 5 |
| 2 | Raleon | Company | 4 |
| 3 | L&T Technology Services Limited | Company | 3 |

Degree lớn nhất chỉ là 5, nên không node nào đạt ngưỡng super-node 100. Cả năm truy vấn đều có `graph_supernode_events = 0`.

## 5. Vì sao ưu tiên 50 edge mới nhất có thể đúng hoặc sai?

Policy degree > 100 → tối đa 50 edge mới nhất giúp chặn BFS explosion, kiểm soát context/token và ưu tiên trạng thái hiện hành. Tổng traversal còn bị giới hạn bởi `GLOBAL_EDGE_CAP = 250` và graph context 14.000 ký tự.

Rủi ro là câu hỏi lịch sử có thể cần một edge cũ nhưng quan trọng. Recency cũng không đồng nghĩa relevance. Production ranking nên kết hợp semantic relevance, confidence, recency và relation diversity, đồng thời dành một quota cho edge lịch sử.

## 6. Flat RAG thắng nhóm nào?

Flat RAG không thắng về điểm trung bình chất lượng. Hai phương pháp cùng đạt comprehensiveness 4,8 và multi-hop 4,8; GraphRAG đạt faithfulness 5,0 so với 4,8 của Flat. Flat thắng rõ về latency tổng thể: 1,138 giây so với 1,879 giây, nhanh hơn khoảng 39,4% nếu lấy Graph làm mốc.

Theo nhóm, Flat nhanh hơn ở `factoid` (0,868 so với 1,598 giây) và `cross-doc` (1,474 so với 2,670 giây). Đây là lợi thế kiến trúc vì không phải chạy seed extraction và Neo4j traversal.

## 7. GraphRAG thắng nhóm nào?

Lợi thế chất lượng xuất hiện ở `cross-doc`: faithfulness trung bình 5,0 so với 4,5 của Flat. Ở G5000-07, GraphRAG đưa được cả feature nội bộ của ServiceNow và chương trình AI Lighthouse với NVIDIA/Accenture nên Judge cho faithfulness 5/5; Flat chỉ đạt 4/5.

Ở câu `multi-hop` G5000-05, hai phương pháp cùng đạt 5/5, nhưng GraphRAG nhanh hơn (0,858 so với 1,006 giây) và dùng ít token hơn (533 so với 641). Sample chỉ có một câu multi-hop nên chưa đủ để khái quát.

## 8. Trade-off latency và token như thế nào?

| Metric trung bình | Flat RAG | GraphRAG | Graph − Flat |
|---|---:|---:|---:|
| Comprehensiveness | 4.80 | 4.80 | 0.00 |
| Faithfulness | 4.80 | 5.00 | +0.20 |
| Multi-hop reasoning | 4.80 | 4.80 | 0.00 |
| Latency (s) | 1.138 | 1.879 | +0.741 |
| Total tokens | 609.2 | 598.4 | -10.8 |

GraphRAG tăng faithfulness và giảm khoảng 1,8% token, nhưng chậm hơn khoảng 65,1% do thêm seed extraction, graph query và linearization. Token không tăng vì GraphRAG dùng top-4 vector chunks, còn Flat dùng top-6.

## 9. Đề xuất nào của AI Coding Agent không được dùng?

Không dùng pairwise cosine trên toàn bộ entity/chunk vì độ phức tạp `O(N²)` sẽ gây bùng nổ CPU/RAM ở quy mô lớn. Thay vào đó pipeline dùng FAISS ANN sinh top-k candidate rồi mới áp lexical guard. Cũng không hạ threshold chỉ để tạo đủ 10 audit row, vì cách đó làm tăng false merge và khiến graph kém tin cậy.

## 10. Khi scale lên 350 MB, bottleneck đầu tiên là gì?

Bottleneck đầu tiên là coreference và NER/RE qua LLM: số request, token cost, rate limit và khả năng chạy lại batch lỗi. Lượt Groq trước từng đạt quota 199.981/200.000 token; chuyển sang OpenAI giúp extraction đạt 176 triple và 0 batch lỗi nhưng không loại bỏ bản chất bottleneck.

Kiến trúc scale gồm: bounded worker queue; retry/backoff; checkpoint idempotent theo `chunk_id`; model routing để chỉ gửi chunk có khả năng chứa relation; lưu raw response và validation error; entity blocking + HNSW/FAISS; Neo4j `UNWIND` theo batch; incremental update; community partition và query router theo loại câu hỏi.
