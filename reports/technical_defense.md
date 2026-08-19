# Thuyết minh Kỹ thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đỗ Tuấn Sơn · **Khóa:** AICB-K34 · Track 3: GraphRAG
**Dataset:** `HackerNoon/tech-company-news-data-dump` (first-5000-rows subset)

> ⚠️ Các ô đánh dấu `‹FILL: ...›` cần điền **số liệu thật** sau khi chạy `Restart & Run All` trên Colab.
> Nguồn số liệu được ghi rõ ngay trong ô (tên biến / cell notebook).

---

## 1. Coreference Resolution — 1 ca phân giải sai và hậu quả

**Cơ chế (conservative):** `resolve_coref_batch()` (cell 1.7) chỉ phân giải đại từ khi tiền ngữ xuất hiện rõ trong **cùng một chunk**; nếu mơ hồ thì giữ nguyên và log vào `unresolved_mentions`. `temperature=0.0`, strict JSON.

**Ca thực tế:** `‹FILL: chunk_id + câu văn từ extraction_source, cột resolved_text›`
- **Hiện tượng:** ví dụ điển hình trong tin M&A — câu "Aeris will acquire the business; **it** will support cellular IoT". Đại từ *it* có thể bị gán nhầm cho **Ericsson** (chủ thể được nhắc gần nhất) thay vì **Aeris/công nghệ được chuyển giao**.
- **Hậu quả với Knowledge Graph:** phân giải sai tạo **False Edge** — ví dụ sinh cạnh `Ericsson -SUPPORTS-> cellular IoT` thay vì gán cho Aeris. Vì mọi cạnh đều mang provenance (`source_chunk_id`, `evidence`), lỗi này lan sang bước traversal và làm GraphRAG trả lời sai chủ thể của quan hệ.
- **Kiểm chứng:** đếm `‹FILL: số dòng có unresolved_mentions không rỗng›` (in ra ở cell 1.7). Conservative rule ưu tiên **precision** — thà bỏ sót còn hơn tạo cạnh sai.

---

## 2. Entity Resolution Threshold & Lexical Guard

**Ngưỡng đang dùng (cell 2.2):**
- Vector match (FAISS `IndexFlatIP`, embedding cosine): **`threshold = 0.90`**, top-k = 5 láng giềng.
- **Lexical Guard** sau vector: `merge_guard()` strip hậu tố công ty (`Inc, Corp, Ltd, LLC…`) rồi yêu cầu `SequenceMatcher.ratio() >= 0.72`; nếu chuỗi rút gọn trùng khớp thì merge ngay.
- Union-Find (`UF`) gộp cụm; canonical = mention có **tần suất cao nhất** (tie-break theo độ dài, alphabet).

**Vì sao 0.90:** ngưỡng cao để tránh False Merge; các biến thể thật (`Microsoft` vs `Microsoft Corp`) đã được `MANUAL_ALIASES` + strip-suffix xử lý nên không cần hạ ngưỡng vector.

**Cặp similarity cao (>0.85) nhưng bị Guard chặn (`REJECT_GUARD`):** `‹FILL: 1 cặp thật từ entity_resolution_audit_df, cột left/right/similarity, lọc decision==REJECT_GUARD›`
- Ví dụ mẫu kỳ vọng: **`Apple`** vs **`Apple Watch`** (hoặc `Sam Altman` vs `Steve Altman`) — vector similarity cao vì chia sẻ token, nhưng sau khi strip suffix chuỗi vẫn khác đủ để `SequenceMatcher < 0.72` → **không gộp**.
- **Lý do ngữ nghĩa:** đây là **sản phẩm ≠ công ty** (Apple Watch là Technology, Apple là Company) / **hai người khác nhau**. Gộp nhầm sẽ kéo mọi cạnh của sản phẩm về công ty (super-node giả) và làm sai multi-hop.

---

## 3. Super-node Analysis

**Chính sách (cell 3.3):** `SUPER_NODE_DEGREE=100`, node bậc >100 → chỉ lấy **≤50 cạnh** `ORDER BY published_date DESC`; `GLOBAL_EDGE_CAP=250`; `MAX_GRAPH_CONTEXT_CHARS=14000`.

**Top-3 super-node (từ `top_degree_df`, cell 2.4):**

| Hạng | Tên thực thể | Type | Degree |
|------|--------------|------|--------|
| 1 | ‹FILL› | ‹FILL› | ‹FILL› |
| 2 | ‹FILL› | ‹FILL› | ‹FILL› |
| 3 | ‹FILL› | ‹FILL› | ‹FILL› |

> Với subset 400 chunk, degree thực tế có thể < 100 (chưa kích hoạt cap). Nếu vậy, ghi rõ: *"Trong sample lab chưa có node vượt ngưỡng; chính sách được kiểm chứng bằng `test_supernode_policy()` và sẽ kích hoạt khi scale."*
> Số lần cap kích hoạt trong eval: cột `graph_supernode_events` của `eval_results_df`.

**Ưu điểm lấy 50 cạnh mới nhất:**
- Chặn bùng nổ token/context (một node như *Google/Microsoft* có thể nối hàng nghìn thực thể) → giữ latency & chi phí ổn định.
- Ưu tiên **thông tin cập nhật** (`published_date DESC`) — phù hợp tin tức, sự kiện M&A/đầu tư mới nhất thường là câu trả lời đúng.

**Rủi ro:**
- Câu hỏi về **sự kiện lịch sử xa** (cross-doc timeline) có thể bị cắt mất cạnh cũ → thiếu bằng chứng thời điểm đầu.
- Recency bias: nếu quan hệ quan trọng nằm ở bài báo cũ, nó rớt khỏi top-50.
- **Giảm thiểu:** khi câu hỏi mang tính lịch sử, nên bổ sung nhánh Vector (đã có trong Hybrid) hoặc nới cap theo loại quan hệ.

---

## 4. Bảng Benchmark & 2 Ca lỗi điển hình

**Bảng so sánh (nguồn: `comparison_df` / `graphrag_vs_flatrag_summary.csv`):**

| Tiêu chí (trung bình) | Flat RAG | GraphRAG | Δ (Graph−Flat) | Nhận xét |
|----------------------|----------|----------|----------------|----------|
| Comprehensiveness (1–5) | ‹FILL› | ‹FILL› | ‹FILL› | |
| Faithfulness (1–5) | ‹FILL› | ‹FILL› | ‹FILL› | |
| Multi-hop reasoning (1–5) | ‹FILL› | ‹FILL› | ‹FILL› | |
| Latency (s) | ‹FILL› | ‹FILL› | ‹FILL› | Graph thêm seed-extraction + Cypher |
| Token usage | ‹FILL› | ‹FILL› | ‹FILL› | Graph context dài hơn (graph+vector) |

*Gợi ý phân tích:* kỳ vọng GraphRAG **thắng multi-hop / cross-doc** (nối quan hệ qua nhiều bài), **hòa hoặc thua factoid** (câu tra cứu 1 sự thật thì Vector đủ), đổi lại **latency & token cao hơn**.

**Ca 1 — Flat RAG thất bại, GraphRAG thành công:** `‹FILL: chọn 1 id nhóm multi-hop, vd G5000-01 Aeris–Ericsson›`
- *Vì sao Flat thất bại:* Vector top-k kéo về các chunk rời rạc, không **nối** được "Ericsson chuyển giao business → Aeris → hỗ trợ >100M IoT devices" vì 3 mẩu nằm ở 3 bài báo/ngày khác nhau (row 33, 1746, 935). Nếu chỉ 1–2 mẩu lọt top-6, câu trả lời thiếu.
- *GraphRAG giải quyết:* seed `Aeris/Ericsson` → BFS 2-hop đi theo cạnh `ACQUIRED/TRANSFERRED`, linearize kèm `date` + `chunk` → cung cấp đủ chuỗi bằng chứng. Trích rationale của Judge: `‹FILL: graph_judge_rationale›`.

**Ca 2 — GraphRAG thất bại / khó khăn:** `‹FILL: id có graph score thấp›`
- *Nguyên nhân khả dĩ:* (a) **thiếu seed** (`NO_SEED` trong `graph_debug.diagnostics`) khi câu hỏi dùng tên không khớp `name_norm/aliases`; (b) **missing edge** do bài evidence không nằm trong 400 chunk trích xuất; (c) super-node cap cắt mất cạnh cần thiết.
- *Đề xuất khắc phục:* mở rộng `EXTRACTION_MAX_CHUNKS`, thêm alias, bật **Self-Correction** (cell Bonus: hop2→hop3→vector fallback), nới cap theo relation-type.

---

## 5. Trade-offs · Agent Control · Scale 350MB

**Đánh đổi Quality vs Cost vs Latency:**
- GraphRAG trả giá bằng **indexing overhead** (coref + NER/RE + entity resolution + bulk insert) và **latency truy vấn** (seed LLM + nhiều round-trip Cypher), đổi lấy khả năng **multi-hop & cross-doc** mà Flat không có.
- Flat RAG rẻ/nhanh, đủ tốt cho factoid; là **baseline** hợp lý cho phần lớn truy vấn tra cứu.
- Chiến lược production: **router** — factoid → Flat; multi-hop/cross-doc → GraphRAG (hoặc Hybrid mặc định như notebook).

**Đề xuất của AI Coding Agent mà tôi TỪ CHỐI:** `‹FILL: điền quyết định thật của bạn›`
- Ví dụ nên nêu: *"Agent gợi ý so khớp entity bằng pairwise cosine O(N²) trên toàn bộ mentions — tôi từ chối vì tràn RAM khi scale; giữ FAISS ANN top-k + Union-Find (gần O(N·k))."* Hoặc *"Agent đề nghị bỏ Lexical Guard để tăng recall merge — tôi từ chối vì rủi ro False Merge (Apple vs Apple Watch)."*

**Scale lên 350MB (~100k bài) — bottleneck đầu tiên & giải pháp:**
- **Bottleneck #1: LLM extraction** (NER+RE cho ~hàng trăm nghìn chunk) — chi phí & rate-limit lớn nhất. → Async batch + hàng đợi worker, cache theo hash chunk, chỉ trích xuất chunk có tín hiệu entity (pre-filter NER nhẹ), tăng batch size.
- **Entity Resolution:** thay FAISS Flat bằng **HNSW** + **blocking** theo type/token đầu để tránh so khớp toàn cục.
- **Neo4j:** giữ `UNWIND` batch 1000, tạo constraint/index trước; cân nhắc `apoc.periodic.iterate`.
- **Super-node:** ở quy mô này cap 50 là bắt buộc; thêm **Community Detection** (Bonus) để Global Search thay vì duyệt toàn đồ thị.
