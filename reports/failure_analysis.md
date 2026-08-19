# Phân tích Ca lỗi (Failure Analysis) — Lab 19

**Học viên:** Đỗ Tuấn Sơn · Track 3: GraphRAG
**Nguồn dữ liệu:** `outputs/graphrag_eval_results.csv` (từ `run_evaluation`)

> Quy trình: mỗi ca gồm **(1) Câu hỏi & nhóm → (2) Điểm Judge Flat vs Graph → (3) Root-cause → (4) Bằng chứng từ retrieval/graph_debug → (5) Đề xuất khắc phục.**
> `‹FILL›` = điền từ file kết quả sau khi chạy.

---

## Ca lỗi #1 — Flat RAG thất bại ở câu MULTI-HOP (GraphRAG thành công)

| Trường | Giá trị |
|--------|---------|
| Question ID / nhóm | `‹FILL› / multi-hop` (gợi ý: G5000-01 Aeris–Ericsson) |
| Câu hỏi | `‹FILL›` |
| Flat scores (Comp/Faith/Multi-hop) | `‹FILL / FILL / FILL›` |
| Graph scores (Comp/Faith/Multi-hop) | `‹FILL / FILL / FILL›` |

**Root-cause của Flat RAG:** Vector search trả về top-6 chunk **tương đồng bề mặt** nhưng không đảm bảo chứa **đủ mắt xích** của chuỗi suy luận. Bằng chứng nằm rải ở nhiều bài/ngày khác nhau (ví dụ row 33 / 1746 / 935) → một vài mắt xích rớt khỏi top-k → câu trả lời **thiếu thực thể hoặc thiếu số liệu**.

**Vì sao GraphRAG vượt qua:** seed entity (`‹FILL: graph_debug.diagnostics.matched_seeds›`) → BFS `max_hops=2` đi theo cạnh `ACQUIRED/DEVELOPED/INVESTED_IN`, gom subgraph **có provenance** (`date`, `chunk`, `evidence`) → linearize thành context tường minh → Judge chấm cao hơn ở Comprehensiveness & Multi-hop.
- Trích rationale Judge (graph): `‹FILL: graph_judge_rationale›`

**Khắc phục cho Flat:** tăng k, thêm reranker, hoặc query-decomposition; nhưng bản chất multi-hop vẫn cần cấu trúc đồ thị.

---

## Ca lỗi #2 — GraphRAG thất bại / khó khăn

| Trường | Giá trị |
|--------|---------|
| Question ID / nhóm | `‹FILL›` |
| Câu hỏi | `‹FILL›` |
| Graph scores | `‹FILL›` |
| `graph_supernode_events` | `‹FILL›` |
| Diagnostics | `‹FILL: NO_SEED? / expanded_nodes / collected_edges›` |

**Cây chẩn đoán nguyên nhân gốc (chọn nhánh đúng với ca của bạn):**
1. **NO_SEED** — `match_seeds()` không tìm được node: tên trong câu hỏi không khớp `name_norm`/`aliases_norm` và vector fuzzy < 0.66. → *Fix:* bổ sung alias, hạ nhẹ `fuzzy_threshold`, cải thiện `extract_seeds`.
2. **Missing edge (extraction gap)** — bài evidence **không nằm trong 400 chunk** trích xuất (ngoài `EXTRACTION_MAX_CHUNKS`) hoặc bị lọc precision. → *Fix:* tăng ngân sách trích xuất / ưu tiên chunk theo golden (đã áp dụng một phần).
3. **Super-node cap** — cạnh cần thiết bị cắt do `ORDER BY published_date DESC LIMIT 50`. → *Fix:* nới cap theo relation-type, hoặc Self-Correction hop3.
4. **False edge từ coref sai** — quan hệ gán nhầm chủ thể (xem `technical_defense.md` §1). → *Fix:* siết conservative prompt, tăng ngưỡng confidence khi insert.

**Nguyên nhân xác định cho ca này:** `‹FILL: chọn 1–2 nhánh trên kèm bằng chứng cụ thể›`

**Đề xuất khắc phục:** `‹FILL›`

---

## Tổng hợp mẫu lỗi theo nhóm câu hỏi

| Nhóm | Ai thắng (kỳ vọng) | Mẫu lỗi thường gặp |
|------|--------------------|--------------------|
| factoid | Flat ≈ Graph | Graph có thể NO_SEED cho câu quá ngắn; Flat đủ dùng |
| multi-hop | **Graph** | Flat mất mắt xích; Graph phụ thuộc extraction coverage |
| cross-doc | **Graph** | recency-cap cắt bằng chứng cũ; temporal state (planned vs completed) dễ nhầm |

**Nhận xét chung:** `‹FILL: 3–5 câu tổng kết dựa trên bảng comparison_df thật›`
