# Reflection Cá nhân & Action Plan — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đỗ Tuấn Sơn · **Khóa:** AICB-K34 · Track 3: GraphRAG · **Ngày:** 2026-08-19

---

## 1. Mapping Bài giảng → Code

| Khái niệm bài giảng | Module | Hàm/khối code cụ thể | Quan sát thực tế |
|--------------------|--------|----------------------|------------------|
| Conservative Coreference | M1 | `resolve_coref_batch()`, `run_coref()` (cell 1.7) | Chỉ phân giải khi tiền ngữ ở cùng chunk; log `unresolved_mentions`. `‹FILL: số mention chưa giải›` |
| Exact Dedup + Chunking | M1 | `standardize_news()` (SHA-1 dedup), `chunk_text()` (220/40) | Dedup `‹FILL: before→after›`; nạp **deterministic** theo `raw_row` để khớp golden |
| Schema Allowlist Guard | M2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, filter trong `run_extraction()` | Loại bỏ quan hệ ngoài 8 loại → tăng precision |
| Edge Provenance | M2 | `bulk_insert_edges()` set `source_chunk_id/published_date/evidence/confidence` | `invalid_provenance_edges == 0` (cell 2.4) |
| Bulk Cypher Ingestion | M2 | `bulk_insert_nodes/edges()` dùng `UNWIND $rows` batch 1000 | Không insert từng dòng; constraint `Entity.id` unique |
| Entity Resolution + Union-Find | M3 | `build_resolution_map()` (FAISS ANN 0.90 + `merge_guard` 0.72 + `UF`) | Audit: `‹FILL: #MERGE_MANUAL/#MERGE_VECTOR/#REJECT_GUARD›` |
| Seed + BFS Traversal | M4 | `match_seeds()`, `retrieve_graph_context()` | max_hops=2, fuzzy 0.66 |
| Super-node Degree Cap | M4 | `retrieve_graph_context()` (degree>100→≤50), `test_supernode_policy()` | `‹FILL: có kích hoạt cap không?›` |
| Hybrid Context | M4 | `answer_graph_rag()` ghép `=== GRAPH ===` + `=== VECTOR ===` | |
| LLM-as-a-Judge | M5 | `judge_answer()` (Comp/Faith/Multi-hop 1–5) | JUDGE_PROVIDER=`‹FILL›` |

---

## 2. Quá trình Debugging & Bài học

- **Vấn đề khó nhất tôi gặp/nhận diện:** notebook scaffold để **driver lines bị comment** → `Restart & Run All` không thực thi pipeline (không build chunk, không nạp Neo4j, không export CSV). Đã **kích hoạt driver** cho từng cell.
- **Lỗi lệch dữ liệu (data mismatch):** golden dataset tham chiếu `evidence_row_ids_0based` trải rộng **33–4997**, nhưng scale guard mặc định (random-sample 1500 bài, trích 400 chunk đầu) khiến hầu hết bài evidence **không vào graph** → GraphRAG trả lời trượt gần hết.
  - **Cách xử lý:** (1) đổi nạp dữ liệu sang **deterministic `head` theo `raw_row`** để chỉ số hàng khớp golden; (2) **ưu tiên chunk thuộc bài evidence** vào ngân sách trích xuất 400; (3) **lọc eval** chỉ giữ câu có toàn bộ evidence nằm trong corpus (≈13 câu, đủ 3 nhóm). Nhờ đó benchmark trở nên công bằng.
- **Bài học:** provenance (`source_chunk_id`, `published_date`) không chỉ để chấm điểm — nó là công cụ **debug** để truy vết vì sao một cạnh tồn tại/sai.
- `‹FILL: thêm lỗi runtime cụ thể bạn gặp khi chạy Colab — exact error message + cách fix›`

---

## 3. Kế hoạch Áp dụng vào Đồ án (Action Plan)

- **Tên đồ án:** `‹FILL›`
- **Bài toán có cần GraphRAG không?** `‹FILL›`
  - *Khung quyết định:* nếu truy vấn chủ yếu **tra cứu 1 sự thật** → Flat/Hybrid RAG là đủ. Nếu cần **multi-hop / cross-document / temporal reasoning** trên thực thể có quan hệ rõ → **GraphRAG** (hoặc router Flat↔Graph).
- **Cấu trúc Node/Relation dự kiến:**
  - Nodes: `‹FILL: vd Company, Person, Product, Document…›`
  - Relations: `‹FILL: vd ACQUIRED, WORKS_AT, CITES…›` + **provenance bắt buộc** trên mọi cạnh.
- **Chiến lược Entity Resolution:** manual alias map cho các thực thể lớn + Vector ANN (ngưỡng ~0.90) + Lexical Guard chống False Merge + Union-Find; xuất **audit table** để review.
- **Chiến lược Super-node:** degree cap + ưu tiên theo thời gian/loại quan hệ; ở quy mô lớn dùng **Community Detection** cho Global Search.
- **Timeline:** `‹FILL: Tuần 1… / Tuần 2…›`

---

## 4. Tự đánh giá

| Tiêu chí | Điểm (1–5) | Ghi chú |
|----------|-----------|---------|
| Hiểu bài giảng GraphRAG | `‹FILL›` | |
| Kiểm soát AI Coding Agent | `‹FILL›` | Đã từ chối/điều chỉnh đề xuất nào (xem technical_defense §5) |
| Chất lượng Knowledge Graph | `‹FILL›` | provenance 100%, audit minh bạch |
| Phân tích & debug hệ thống | `‹FILL›` | |
