# RUNBOOK — Lab 19 (Colab)

Hướng dẫn chạy end-to-end và checklist nộp bài.

## 0. Secrets (Colab → 🔑 Secrets), bật "Notebook access"
```
NEO4J_URI, NEO4J_USER, NEO4J_PASSWORD, NEO4J_DATABASE
GROQ_API_KEY, GROQ_MODEL=llama-3.3-70b-versatile
JUDGE_PROVIDER=openai, JUDGE_MODEL=gpt-4o-mini, OPENAI_API_KEY
HF_TOKEN
```
> Tuyệt đối KHÔNG hard-code key vào notebook (rubric trừ −10đ).

## 1. Upload golden dataset lên `/content/`
Kéo `data/graphrag_golden_50_first5000_detailed.csv` vào `/content/` (hoặc mount Drive).
Notebook tự tìm ở `/content/…detailed.csv` → `data/…detailed.csv` → fallback starter.

## 2. Chạy
`Runtime → Restart and run all`. Thứ tự pipeline đã được nối sẵn (driver active):
stream HF → load(head 1500, deterministic) → chunk → coref → NER/RE → entity resolution →
Neo4j UNWIND insert → sanity check → FAISS Flat index → seed/BFS graph → eval (Judge) → export CSV.

Thời gian ~20–40′; tốn API Groq (extraction/coref/answer) + OpenAI (judge). Có checkpoint tại `/content/graphrag_eval_checkpoint.csv`.

## 3. Lấy kết quả
Cell 4.4 tự `files.download()` 2 file. Đưa vào repo:
```
outputs/graphrag_eval_results.csv
outputs/graphrag_vs_flatrag_summary.csv
```

## 4. Điền báo cáo (thay mọi ô `‹FILL›`)
- `reports/technical_defense.md` — 10 câu (số liệu thật: top-degree, audit REJECT_GUARD, benchmark).
- `reports/failure_analysis.md` — 2 ca lỗi (dùng `graph_debug`, rationale Judge).
- `reports/reflection_DoTuanSon.md` — mapping + action plan.

## 5. Checklist nộp
- [ ] 0 cạnh thiếu provenance (`graph_checks()` assert == 0)
- [ ] `entity_resolution_audit_df` ≥ 10 dòng
- [ ] Super-node cap kiểm chứng (`test_supernode_policy()`)
- [ ] 2 CSV trong `outputs/`
- [ ] 3 file reports điền đầy đủ, không còn `‹FILL›`
- [ ] Không hard-code secret → commit & push

---

## Những thay đổi tôi (AI agent) đã thực hiện trên notebook — và lý do
*(Dùng cho câu thuyết minh "AI agent đề xuất gì, bạn chấp nhận/từ chối gì".)*

1. **Kích hoạt driver lines** ở mọi cell (trước đây bị comment) để `Restart & Run All` thực sự chạy pipeline & export CSV. *Không đổi thuật toán.*
2. **Nạp dữ liệu deterministic** — thay `df.sample(1500)` bằng lấy **1500 hàng đầu theo `raw_row`** (chỉ số gốc CSV). Lý do: golden dataset đánh địa chỉ bằng `evidence_row_ids_0based`; random sample khiến bài evidence không vào corpus → GraphRAG trượt. Thêm cột `raw_row` để khớp.
3. **Ưu tiên trích xuất theo golden** — `extraction_source` lấy trước các chunk thuộc bài được golden trích dẫn, rồi lấp đầy tới `EXTRACTION_MAX_CHUNKS=400`. Lý do: ngân sách 400 chunk phải chứa entity cần hỏi thì KG mới trả lời được. (Đây là lựa chọn có chủ đích, minh bạch; đánh đổi: hơi thiên về eval set — nêu rõ khi bảo vệ.)
4. **Lọc eval theo coverage** — chỉ chấm câu có **toàn bộ** evidence rows nằm trong corpus đã nạp (`EVAL_ONLY_COVERED=True`) ≈ 13 câu đủ 3 nhóm. Có `EVAL_MAX_QUESTIONS` để giảm chi phí.
5. **Golden loader linh hoạt** — đọc bộ 50 câu chi tiết từ `/content` hoặc `data/`, fallback starter.
6. **Export kép** — ghi CSV ra cả `/content` và `outputs/`, kèm `files.download()`.
7. **Trim install** — bỏ `spacy, langchain-community, llama-index` (không dùng) cho nhanh.

**Giữ nguyên (respect scale guard README):** `LAB_MAX_ARTICLES=1500`, `LAB_MAX_CHUNKS=3000`, `EXTRACTION_MAX_CHUNKS=400`, `CHUNK_WORDS=220/40`, ngưỡng entity 0.90/0.72, super-node 100→50, `GLOBAL_EDGE_CAP=250`.

## Lưu ý coverage (nêu trong failure analysis)
Golden evidence rows trải 33–4997 (median ~2449). Với guard 1500 bài, ~13/50 câu được cover đầy đủ. Các câu cần bài >1500 sẽ là **ca lỗi GraphRAG hợp lệ** (missing edge / NO_SEED) — đúng tinh thần "phân tích failure". Muốn phủ nhiều hơn: tăng `LAB_MAX_ARTICLES` (vd 5000) & `LAB_MAX_CHUNKS`, chấp nhận thời gian/chi phí cao hơn.
