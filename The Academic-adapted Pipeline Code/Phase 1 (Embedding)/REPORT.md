# Full-corpus embedding run — Qwen3-Embedding-0.6B on the RTX 3090 (TEI, float16)

Generated 2026-09-10T23:07:19Z. Model 0.6B via `ghcr.io/huggingface/text-embeddings-inference:86-1.9.3`; precision **float16**, last-token pooling, L2-normalized, no truncation. This is the production embedding run for the complete canonical Phase 1 dataset (all 902,144 documents); the canonical table was read only.

## Required evidence

| Item | Value |
|---|---|
| Corpus scope | full canonical corpus: 902,144 documents, 31 UTC dates [2026-07-01 … 2026-07-31]  (2026-07-01T00:00:02+00:00 … 2026-07-31T23:59:59+00:00) |
| Documents embedded (expected = 902,144) | 902,144  (equals 902,144: True) |
| Total Qwen tokens (n_tokens_4B) | 35,388,381 |
| Untimed warm-up batches | 3 |
| Batch-token setting used (max_batch_tokens) | 327,680  (as fixed) |
| Observed total GPU memory (whole-card peak) | 18,874.2 MiB = 18.432 GiB (baseline before start 1,330.5 MiB; TEI's own peak ~17,543.6 MiB; under 19 GiB: True) |
| Elapsed timed processing time | 0:14:59 (899.5 s) |
| Ready-to-complete wall-clock (TEI ready → done) | 0:16:56 (1015.9 s) |
| Documents / tokens done at first ETA | 50,184 docs / 1,965,458 tokens (5.6% docs, 5.6% tokens) |
| First estimated time to completion | ~0:14:57 remaining (33,422,923 tokens, 851,960 docs left); est. completion 2026-09-10T23:06:33Z |
| Measured docs/s and tokens/s at first ETA | 1008.6 docs/s, 37265 tokens/s |
| Final documents embedded | 902,144 (dim 1024) |
| Final tokens processed | 35,388,381 planned; TEI x-compute-tokens 35,388,382 |
| Actual completed wall-clock processing time (timed region) | 0:14:59 (899.5 s) |
| Final sustained docs/s and tokens/s (end-to-end) | 976.7 docs/s, 38250 tokens/s |
| TEI backend-only rate (GPU ceiling, all batches) | 58417 tokens/s over 605.8 s of inference |
| Every expected doc_key has an embedding | True (902,144/902,144, unique keys: True, all finite: True) |
| Embedded doc_key set == canonical corpus | True (missing 0, extra 0; canonical size 902,144) |
| Total token count matches the corpus token artifact | True (planned 35,388,381 = token-table 35,388,381; TEI x-compute 35,388,382 = artifact + 1 from the empty-text placeholder) |
| Empty-text documents embedded from a placeholder | 1 (placeholder ' '; doc_key(s): t1_ovhv2aj) — TEI rejects empty inputs, so one embedding is still produced per doc_key |

## How the estimate compares with the actual

- First ETA (made at 5.6% of tokens): predicted **0:15:23** of total timed processing (0:00:26 elapsed + ~0:14:57 remaining); estimated completion 2026-09-10T23:06:33Z.

- Actual total timed processing: **0:14:59** (899.5 s) over 106 timed batches.


## Method

- TEI `ghcr.io/huggingface/text-embeddings-inference:86-1.9.3`, `--model-id` the local 0.6B snapshot, `--max-batch-tokens 327680`, `--auto-truncate false`, no `--max-batch-requests`; **no vram_cap shim and no allocator cap** (the same launch as the proven three-day run).

- Documents ordered by (created_utc, doc_key); packed into 109 client requests of up to 327,680 tokens each; each request POSTs its texts to `/embed` with `truncate=false, normalize=true`; embeddings return as float32.

- 3 untimed warm-up batches (their embeddings are kept) precede the timed region; TEI also runs its own start-up warm-up at the full budget. Memory was sampled whole-card via NVML every 50 ms.

- 1 canonical document(s) with empty text (t1_ovhv2aj) were embedded from a ' ' placeholder: TEI 1.9.3 rejects an empty input with HTTP 400, so a minimal non-empty string is sent to keep exactly one embedding per doc_key. This adds 1 token(s) to TEI's x-compute total relative to the corpus token artifact (the empty string counts as 1 token, the placeholder as 2); the planned/budgeted token totals are unchanged.


Outputs in `results/`: `embeddings.npy` (float32, git-ignored), `embedding_index.parquet` (row → doc_key), `corpus_order.parquet`, `run.json`, `batches.csv`, `verification.json`, `environment.json`, `container_final.log`, each with a `.manifest.yml` sidecar.
