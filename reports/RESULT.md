# RAG evaluation results

## Run information

| Field                              | Value |
| ---------------------------------- | ----- |
| Evaluation date                    | 2026-09-25 (results file timestamp: 2026-09-25) |
| Framework and version              | Custom eval script (`group_project/evaluation/evaluate.py`), retrieval-only mode |
| Evaluator model                    | Not run — RAGAS/LLM-judge scoring needs `OPENAI_API_KEY`; skipped for this report |
| Generator model                    | Not run — no generation step in retrieval-only mode |
| Embedding model                    | bge-m3 (`src/task4_chunking_indexing.embed_texts`, used by dense/hybrid search) |
| Corpus version/commit              | Working tree at time of run (legal/*.md + news/*.md docs) |
| Golden dataset size                | 20 questions (`group_project/evaluation/golden_dataset.json`) |
| `top_k`                            | 5 |
| Fallback threshold and calibration | threshold = 0.532 (balanced accuracy 1.0); in-domain min 0.6179 / mean 0.7314, out-of-domain max 0.4465 / mean 0.3798, near-domain (Lazada/Tiki/TikTok Shop) scores 0.5499–0.5973 — see `group_project/evaluation/threshold_calibration.json` |

## Configurations

This report only covers the **retrieval-only** eval (`evaluate.py --retrieval-only`), which does not need an API key. It compares three retrieval strategies, not the RAGAS A/B configs (`A_dense` vs `B_hybrid`) from the script's full mode:

- **dense:** semantic search only (`src/task5_semantic_search.semantic_search`)
- **bm25:** lexical search only (`src/task6_lexical_search.lexical_search`)
- **hybrid_rrf:** dense + lexical fused with RRF and reranking (`src/task9_retrieval_pipeline.retrieve`, `use_reranking=True`) — this corresponds to "Config B — hybrid + RRF"

Full-mode faithfulness / answer relevance / context recall / context precision (RAGAS, needs `OPENAI_API_KEY`) were **not run** — `group_project/evaluation/results/ragas_results.json` does not exist. Those rows are marked N/A below rather than filled with invented numbers.

Hai config phải dùng cùng golden dataset, generator, evaluator, prompt và `top_k`; chỉ thay retrieval strategy — điều kiện này chỉ áp dụng cho A/B RAGAS, chưa chạy trong lần đánh giá này.

## Overall scores

Retrieval-only metrics (from `group_project/evaluation/results/retrieval_results.json`), averaged over 20 golden questions, `top_k=5`:

| Metric                    |   dense | bm25   | hybrid_rrf | Delta (hybrid−dense) |
| ------------------------- | ------: | -----: | ---------: | --------------------: |
| Hit@5                     |    0.90 |   0.90 |       0.90 |                  0.00 |
| Mean reciprocal rank      |  0.6958 | 0.8250 |     0.8100 |                +0.1142 |
| Context coverage          |  0.9582 | 0.9700 |     0.9778 |                +0.0196 |
| Mean latency (s)          |  0.0275 | 0.0018 |     0.0285 |                +0.0010 |

RAGAS-only metrics (faithfulness, answer relevance, context recall, context precision) are **N/A** — generation/evaluator step was skipped.

## A/B comparison

- Cấu hình tốt hơn: **hybrid_rrf** trên MRR (0.810 vs 0.696 cho dense) và context coverage (0.978 vs 0.958), tương đương dense và bm25 về Hit@5 (0.90 cả ba). bm25 có MRR cao nhất (0.825) nhưng đơn thuần từ khóa nên dễ vỡ với câu hỏi diễn giải lại (paraphrase).
- Evidence: xem `retrieval_results.json` — dense bỏ sót q07, q19 (context_coverage 0.68 và 0.48); bm25 bỏ sót q02, q07 (0.72 và 0.68); hybrid_rrf bỏ sót q02, q07 nhưng bù lại context_coverage cao hơn và tìm được q19 dù ở rank 5 (RR 0.2).
- Trade-off về latency/cost: bm25 gần như miễn phí về latency (~1.8ms) so với dense (~27.5ms) và hybrid_rrf (~28.5ms, do phải chạy cả dense + bm25 + rerank). Không có dữ liệu cost/latency cho bước generation vì chưa chạy.

## Worst performers

Dựa trên `retrieval_results.json` (không có dữ liệu faithfulness/relevance/recall/precision vì chưa chạy RAGAS):

|   # | Question                                                                                         | Config     | Hit   | Reciprocal rank | Context coverage | Failure stage | Root cause |
| --: | ------------------------------------------------------------------------------------------------- | ---------- | ----- | ---------------: | ----------------: | -------------- | ---------- |
|   1 | q07 — "Nếu tôi tự sắp xếp gửi trả hàng cho đơn không thuộc Shopee Mall, Shopee hỗ trợ phí trả hàng như thế nào?" | all 3      | false | 0.0              |             0.6818 | retrieval      | Câu hỏi đề cập "phí trả hàng" nhưng nguồn (`news/article_02.md`) diễn đạt bằng bảng số Shopee Xu theo tỉnh/thành; không config nào (dense/bm25/hybrid) truy xuất đúng chunk nguồn ở top-5 — mismatch từ vựng + chunk có thể bị tách khỏi bảng số liệu. |
|   2 | q19 — "Tôi có thể bán Bitcoin hoặc tiền ảo trên Shopee không?" | dense      | false | 0.0              |             0.4828 | retrieval      | Dense bỏ sót hoàn toàn chunk nguồn (`shopee-prohibited-items.md`, coverage 0.48); bm25 và hybrid_rrf tìm đúng (hybrid ở rank 5, RR 0.2) nhờ khớp từ khóa "tiền ảo/Bitcoin" theo lexical match, cho thấy dense embedding chưa phân biệt tốt câu hỏi ngắn/mang tính liệt kê. |
|   3 | q02 — "Những ai được sử dụng hình thức Trả hàng COM...?" | bm25, hybrid_rrf | false | 0.0         |        0.72 / 0.875 | retrieval      | Cả bm25 và hybrid_rrf không xếp đúng chunk lên top-5 (dense thì có, RR 0.25); câu hỏi dùng "Trả hàng COM" như alias, nguồn dùng đại từ "i./ii." liệt kê điều kiện — lexical overlap thấp khiến bm25 lệch hướng và kéo hybrid theo. |

## Recommendations

| Priority | Action | Evidence from failure analysis | Expected impact | How to verify |
| -------: | ------ | ------------------------------ | ---------------- | -------------- |
|        1 | Chạy đầy đủ eval RAGAS (`evaluate.py` không kèm `--retrieval-only`, cần `OPENAI_API_KEY`) để có faithfulness/relevance/recall/precision thực tế cho Config A vs B | Report hiện tại thiếu hoàn toàn 4 metric RAGAS vì bước generation bị skip theo yêu cầu | Cho phép so sánh A/B đầy đủ như template yêu cầu | Kiểm tra `group_project/evaluation/results/ragas_results.json` được tạo và có 4 metric cho cả 2 config |
|        2 | Cải thiện retrieval cho câu hỏi có alias/liệt kê số (q02, q19) bằng cách bổ sung few-shot/alias mapping ("Trả hàng COM", "tiền ảo") vào chunk metadata hoặc query expansion | q02 và q19 chỉ được 1/3 config bắt đúng; lexical mismatch giữa câu hỏi và văn bản nguồn | Tăng Hit@5 và MRR cho các câu hỏi dùng thuật ngữ viết tắt/paraphrase | Re-run `evaluate.py --retrieval-only`, kiểm tra hit=true và reciprocal_rank tăng cho q02, q19 ở cả 3 strategy |
|        3 | Xem lại cách chunk `news/article_02.md` quanh bảng phí Shopee Xu (q07) — có thể bảng bị cắt khỏi câu văn giải thích "phí trả hàng" | q07 là câu duy nhất mà cả 3 strategy đều miss (context_coverage cao nhất chỉ 0.68) | Giảm về 0 số câu hỏi bị miss ở cả 3 config | Re-run retrieval-only eval, kiểm tra q07 đạt hit=true ở ít nhất 1 strategy với context_coverage ≥ 0.7 |

## Bonus experiments

| Experiment | Baseline | Metric delta | Latency/cost delta | Conclusion |
| ---------- | -------- | ------------: | -------------------: | ---------- |
| Domain-fallback threshold calibration (in-domain vs out-of-domain vs near-domain dense score) | — | Threshold 0.532 tách hoàn hảo in-domain (min 0.6179) khỏi out-of-domain (max 0.4465), balanced accuracy 1.0 | Không đo latency/cost (chỉ chạy dense score, không sinh câu trả lời) | Ngưỡng 0.532 an toàn cho in/out-of-domain, nhưng 3 câu near-domain (Lazada/Tiki/TikTok Shop, 0.5499–0.5973) đều vượt ngưỡng — cần theo dõi false-positive risk khi câu hỏi về đối thủ cạnh tranh bị nhận nhầm là in-domain |

**Note on scope:** báo cáo này chỉ dựa trên các file kết quả đã có sẵn (`retrieval_results.json`, `threshold_calibration.json`) — không gọi thêm API nào (OpenAI/RAGAS) theo yêu cầu. Muốn có đầy đủ 4 metric RAGAS và so sánh Config A/B như template gốc, cần chạy `python group_project/evaluation/evaluate.py` (không có `--retrieval-only`) với `OPENAI_API_KEY` hợp lệ.
