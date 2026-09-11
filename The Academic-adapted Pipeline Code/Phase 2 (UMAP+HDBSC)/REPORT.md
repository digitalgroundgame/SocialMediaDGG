# REPORT — Full-corpus GPU UMAP + HDBSCAN baseline (902,144 documents)

Run: 2026-09-11T03:30:59–03:35:02Z · RTX 3090 · cuML 26.08.00 (CUDA runtime 12.9,
driver 596.49 / CUDA 13.2) · WSL2 Ubuntu-24.04. All **902,144** records processed
exactly as provided; fixed config; whole-card GPU ceiling **21 GiB (21,504 MiB)**.
Baseline structure-discovery run and **measurement only** — **no cluster is named,
described, ranked, characterised, or interpreted**.

> UMAP is unseeded by design (see README): `random_state=None`, and NN-Descent
> (auto-selected for N > 50k) is non-deterministic, so re-running gives statistically
> similar but not identical output. Figures in `figures/` are labels-only diagnostics.

## Headline measurements (measured this run)

| Operation | Success | Wall clock | Peak whole-card GPU | Peak host RSS | Ceiling breach |
|---|---|---|---|---|---|
| GPU cuML UMAP 5-D (1024 → 5) | **yes** | **44.02 s** | **4.875 GiB** (4,992.4 MiB) | 6.169 GiB | **no** |
| GPU cuML HDBSCAN (5-D) | **yes** | **31.42 s** | **3.620 GiB** (3,706.5 MiB) | 4.886 GiB | **no** |
| GPU cuML UMAP 2-D (viz only) | **yes** | **45.52 s** | **4.937 GiB** (5,055.9 MiB) | 6.734 GiB | **no** |
| **Total compute** | — | **120.96 s (≈2.02 min)** | (max **4.937 GiB**) | — | **no** |

Whole-card GPU baselines before each op: 1,981 / 2,357 / 2,377 MiB (desktop + resident
pools). Per-op increments: UMAP-5D ≈ 3,011 MiB, HDBSCAN ≈ 1,350 MiB, UMAP-2D ≈ 2,679 MiB.
The maximum whole-card peak (4.937 GiB) is **~16.1 GiB below** the 21 GiB ceiling. GPU
total memory 24,576 MiB. End-to-end wall (incl. cuML import, input hashing ×2, 3.7 GB
load, 39 s exact duplicate census, figures) ≈ 4 min 3 s.

## Answers to the 15 report-back items

1. **UMAP 5-D** — runtime **44.02 s**; peak whole-card GPU **4.875 GiB** (4,992.4 MiB); peak host RAM **6.169 GiB** (6,317.2 MiB). Output `[902144, 5]`, all rows processed, all finite. `metric='cosine'` accepted; `build_algo='auto'` → **NN-Descent** (N > 50k).
2. **HDBSCAN** — runtime **31.42 s**; peak whole-card GPU **3.620 GiB** (3,706.5 MiB); peak host RAM **4.886 GiB** (5,003.4 MiB). Every one of the 902,144 rows received a label (incl. -1).
3. **2-D visualisation UMAP** — runtime **45.52 s**; peak whole-card GPU **4.937 GiB** (5,055.9 MiB). Output `[902144, 2]`, all finite. **Visualisation only** — not fed to HDBSCAN, does not change cluster assignments.
4. **Total computation time** — **120.96 s** (≈ 2.02 min), the sum of the three GPU operations. Max whole-card peak across them: **4.937 GiB**.
5. **Number of clusters** — **5,167** (labels 0…5166; noise = -1).
6. **Assigned count and percentage** — **457,880 / 902,144 = 50.7546 %**.
7. **Label -1 (noise) count and percentage** — **444,264 / 902,144 = 49.2454 %**.
8. **Cluster-size summary** — min **10**, Q1 **16**, median **26**, mean **88.62**, Q3 **54**, max **24,862**.
9. **Membership / probability output** — **available and preserved for all 902,144 rows** in `hdbscan_labels.parquet` (`probability`). Range 0–1, mean **0.4238**; **444,264** are exactly 0 (= the noise points), **220,534** are exactly 1.0.
10. **Cluster persistence** — preserved for all **5,167** clusters in `hdbscan_cluster_persistence.csv` and `summary.json`. **Directly observed: every persistence value is exactly 1.0** (min = mean = max = 1.0), i.e. it **again contains only 1.0 values** — the same behaviour as the three-day feasibility run. Reported verbatim as cuML returns it; **not repaired, reinterpreted, replaced, or debugged**, per task.
11. **Hierarchy / tree outputs preserved** —
    - **Condensed tree**: `hdbscan_condensed_tree.parquet` — 915,692 rows (`parent, child, lambda_val, child_size`).
    - **Single-linkage tree**: `hdbscan_single_linkage_tree.parquet` — 902,143 rows (`parent, left_child, right_child, distance, size`).
    - **Minimum spanning tree**: `hdbscan_minimum_spanning_tree.parquet` — 902,143 rows (`from, to, distance`); populated via `gen_min_span_tree=True`.
    - **Outlier scores**: cuML's HDBSCAN does **not** expose `outlier_scores_`; nothing to preserve (same as the three-day run).
12. **Directly observed effects of repeated / tied embeddings** — measured only; no meaning inferred.
    - **Input**: of 902,144 rows, **843,451 are unique**; **7,050 groups** of exactly-identical embeddings contain **65,743 rows**; the largest single identical group has **26,681** members.
    - **5-D UMAP output**: after UMAP the 5-D coordinates are **all distinct** — 0 exact duplicates, largest tie group = 1 (cuML UMAP maps identical inputs to distinct outputs).
    - **2-D UMAP output**: **902,118 unique**; **25 groups** of coincident 2-D points span **51 rows** (largest group = 3) — a small residual, recorded as observed.
    - **HDBSCAN / UMAP fits**: **no warnings or errors** emitted during any of the three fits; **no ceiling breach** and **no failure** associated with the repeated embeddings. The largest single cluster has **24,862** members.
13. **Diagnostic figures created** (all in `figures/`; titles + axis/tick labels only; clusters not named or interpreted):
    - `fig_umap2d_global_density.png` — global 2-D UMAP density of all documents (hexbin, log count)
    - `fig_umap2d_assigned_vs_noise.png` — global 2-D UMAP distinguishing assigned vs label -1 (all points, rasterized)
    - `fig_cluster_size_distribution.png`
    - `fig_ranked_cluster_sizes.png`
    - `fig_assigned_vs_noise.png`
    - `fig_membership_probability_distribution.png`
    - `fig_cluster_persistence_distribution.png`
    - `fig_runtime_by_operation.png`
    - `fig_peak_gpu_memory_by_operation.png`
14. **Verification results** — every check passes (`results/verification.json`; independently re-checked by reading the saved parquet/npy files):
    - input embedding rows = **902,144** ✓
    - 5-D UMAP rows = **902,144** ✓
    - 2-D visualisation UMAP rows = **902,144** ✓
    - HDBSCAN output rows = **902,144** ✓
    - row/doc_key alignment preserved — index `row` = 0…902143; doc_key order **identical** across the 5-D coords, 2-D coords, and labels, and **matches the source index** ✓
    - no duplicate or missing doc_keys introduced — 902,144 unique doc_keys ✓
    - all saved UMAP coordinates finite (5-D and 2-D) ✓
    - every document has exactly one raw HDBSCAN label (labels shape = (902144,); assigned 457,880 + noise 444,264 = 902,144) ✓
    - source embeddings and index **unchanged** — sha256 before == after (`embeddings.npy` `b5755f23…`, `embedding_index.parquet` `2c89f296…`) ✓
15. **Exact output paths** — see "Output paths" below.

## Confirmation that the GPU (cuML) performed the computation

- The estimators are `cuml.manifold.UMAP` and `cuml.cluster.hdbscan.HDBSCAN` (RAPIDS
  cuML 26.08.00); cuML has no CPU fallback for these.
- NVML reported this process (PID) as an active **compute process** on the RTX 3090
  during all three fits (`pid_seen_on_gpu=true`), and whole-card device memory rose
  ~3,011 MiB (UMAP-5D), ~1,350 MiB (HDBSCAN), and ~2,679 MiB (UMAP-2D) over baseline
  while each ran.
- CUDA runtime 12.9 / driver 13.2 via `cupy-cuda12x 14.2.0`; device confirmed as
  "NVIDIA GeForce RTX 3090", compute capability 8.6.

## Scaling vs the three-day feasibility run (directly observed; not an interpretation)

For context only. Three-day measured point (101,722 rows): UMAP-5D 5.31 s / 2.603 GiB;
HDBSCAN 1.32 s / 2.484 GiB. Full corpus is **8.87×** more rows.

| Operation | 101,722 rows | 902,144 rows | Time ratio | Peak-GPU change |
|---|---|---|---|---|
| UMAP 5-D | 5.31 s / 2.603 GiB | 44.02 s / 4.875 GiB | 8.3× | +2.27 GiB |
| HDBSCAN | 1.32 s / 2.484 GiB | 31.42 s / 3.620 GiB | 23.8× | +1.14 GiB |

HDBSCAN time grew super-linearly (23.8× for an 8.87× row increase), consistent with
the default `build_algo='brute_force'` KNN being ~O(N²) in time — but far below the
~78× (8.87²) worst case, and its **memory** stayed low (+1.14 GiB), confirming the
brute-force KNN is tiled rather than materialising an N² distance matrix. Both peaks
remained well within the 21 GiB ceiling.

## Output paths

Base: `ACADEMIC-ADAPTED PIPELINE/phase3_blind_semantic_structure_discovery/full_corpus_gpu_umap_hdbscan_baseline/`

- Code: `scripts/run_full_corpus_umap_hdbscan.py`, `scripts/setup_env.sh`, `scripts/write_extra_manifests.py`, `tests/test_run_full_corpus_functions.py`
- 5-D UMAP coordinates (analytical): `results/umap_5d_coordinates.parquet` (keyed) and `results/umap_5d_coordinates.npy`
- 2-D UMAP coordinates (visualisation only): `results/umap_2d_visualization_coordinates.parquet` (keyed) and `results/umap_2d_visualization_coordinates.npy`
- HDBSCAN labels + membership: `results/hdbscan_labels.parquet`
- Cluster persistence: `results/hdbscan_cluster_persistence.csv`
- Trees: `results/hdbscan_condensed_tree.parquet`, `results/hdbscan_single_linkage_tree.parquet`, `results/hdbscan_minimum_spanning_tree.parquet`
- Measurements: `results/summary.json`, `results/measurements.csv`, `results/gpu_memory_samples_{umap_5d,hdbscan,umap_2d_viz}.csv`
- Verification: `results/verification.json`
- Figures: `figures/fig_*.png` (9)
- Environment: `environment/packages.txt`, `environment/cuml_env.json`, `environment/nvidia_smi.txt`, `environment/env_setup.log`
- Provenance: `<file>.manifest.yml` beside every non-code output (24 sidecars; all validate against the D18 schema)
