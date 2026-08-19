# Phân tích ca lỗi — Flat RAG vs GraphRAG

**Học viên:** Đỗ Tuấn Sơn

## 1. Phạm vi và bằng chứng

Phân tích dựa trên 5 Golden queries trong `outputs/graphrag_eval_results.csv`. Cả hai phương pháp đạt điểm cao; không có trường hợp một hệ thống thất bại hoàn toàn. Vì vậy báo cáo tập trung vào chênh lệch có thể quan sát và failure mode còn tồn tại, không tạo ca lỗi giả.

## 2. Ca Flat RAG yếu hơn GraphRAG — G5000-07

**Câu hỏi:** ServiceNow mở rộng generative AI trong tháng 7 khác nhau thế nào giữa feature của platform và chương trình hệ sinh thái với NVIDIA/Accenture?

**Kết quả:**

| Metric | Flat | Graph |
|---|---:|---:|
| Comprehensiveness | 4 | 4 |
| Faithfulness | 4 | 5 |
| Multi-hop reasoning | 4 | 4 |
| Latency (s) | 1.953 | 3.521 |
| Tokens | 706 | 725 |

### Triệu chứng

Flat answer nhận diện đúng case summarization, text-to-code và AI Lighthouse, nhưng phần mô tả chương trình ecosystem còn chung chung. Judge đánh giá nội dung chưa khai thác đầy đủ evidence và cho faithfulness 4/5.

### Root cause

Vector retrieval xếp hạng từng chunk độc lập. Câu hỏi yêu cầu tách hai nhánh thông tin ở các tài liệu khác nhau: feature nội bộ của ServiceNow và chương trình hợp tác bên ngoài. Flat context có đủ tài liệu nhưng không biểu diễn rõ vai trò/quan hệ giữa các entity.

### Vì sao GraphRAG tốt hơn

Hybrid context đưa thêm entity/relationship quanh ServiceNow, NVIDIA và Accenture, giúp generator tách feature platform khỏi ecosystem program. Graph answer đạt faithfulness 5/5. Tuy nhiên comprehensiveness vẫn chỉ 4/5, nên GraphRAG cải thiện nhưng chưa giải quyết hoàn toàn câu hỏi.

### Khắc phục

Thêm relation `LAUNCHED`, node `Program`/`Product`, relation role rõ ràng và prompt yêu cầu bảng đối chiếu hai nhánh. Với Flat, dùng query decomposition thành hai truy vấn con trước khi tổng hợp.

## 3. Ca GraphRAG khó khăn — latency trên cross-doc

**Quan sát:** Nhóm cross-doc có latency GraphRAG 2,670 giây, cao hơn Flat 1,474 giây; token cũng cao hơn 757 so với 651. Riêng G5000-07, GraphRAG mất 3,521 giây.

### Root cause

GraphRAG phải thực hiện thêm các bước không có ở Flat: LLM seed extraction, exact/fuzzy entity matching, nhiều Cypher query cho degree và recent edges, BFS hai hop, textualization, rồi mới ghép vector context. Các bước đang thực hiện tuần tự nên round-trip latency cộng dồn.

### Tác động

Faithfulness tăng 0,5 điểm trong nhóm cross-doc nhưng latency tăng khoảng 81,1%. Với truy vấn tương tác thời gian thực, trade-off này có thể không được chấp nhận nếu câu hỏi đơn giản.

### Khắc phục

- Cache seed extraction và node degree.
- Batch Cypher thay cho một query trên từng node.
- Router bỏ graph traversal cho factoid đơn giản.
- Chạy vector retrieval song song với seed/graph retrieval.
- Dùng sufficiency check để chỉ mở hop tiếp theo khi cần.

## 4. Failure mode dữ liệu và evaluation

Entity audit chỉ có một `MERGE_VECTOR`, chưa đạt yêu cầu 10 dòng và không có `REJECT_GUARD`. Degree lớn nhất là 5 nên super-node branch chưa được kích hoạt. Neo4j có 340 node/219 edge, trong khi lượt OpenAI cuối insert 297 node/175 edge, cho thấy database còn dữ liệu từ lượt trước. Vì vậy benchmark chưa hoàn toàn cô lập theo một lần ingestion.

Lần chạy tiếp theo nên dùng database hoặc `run_id` riêng, xuất coreference statistics, mở rộng audit logging và bổ sung unit test tạo graph fixture có degree > 100 để kiểm chứng cap 50 mà không làm sai dữ liệu thật.

## 5. Kết luận

Flat RAG là baseline mạnh cho bộ dữ liệu nhỏ khi evidence đã được ưu tiên vào index. GraphRAG đem lại lợi thế faithfulness ở cross-document reasoning nhưng trả giá bằng latency. Chất lượng GraphRAG phụ thuộc trực tiếp vào schema, extraction coverage, entity audit và retrieval policy; graph lớn hơn không tự động đồng nghĩa câu trả lời tốt hơn.
