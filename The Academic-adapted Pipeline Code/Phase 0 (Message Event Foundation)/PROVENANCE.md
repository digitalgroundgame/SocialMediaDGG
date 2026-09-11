# Provenance of the Phase 1 canonical message-event table

Every statement here is reproduced mechanically by the script and recorded in the tables
named; the SHA256 of every input, of every output and of the script is in each
`<output>.manifest.yml` sidecar. Paths are relative to the project root.

## 1. Source files used (read-only)

| Record type | File | Rows | Columns | Writer | SHA256 |
|---|---|---:|---:|---|---|
| comment | `data_sample/comments/2026-07/comments_dedicated_politics_r16_2203c337b7b4_0000.parquet` | 119,888 | 11 | parquet-cpp-arrow 17.0.0 | `1d0ae40f0114db76866c579422450c91c06bd0d3c503e56c49aa6267f17343a7` |
| comment | `data_sample/comments/2026-07/comments_dedicated_politics_r16_2203c337b7b4_0001.parquet` | 119,888 | 11 | parquet-cpp-arrow 17.0.0 | `f4a55f61b62ffba6aeea521f1b1c5a3ccda49d5e99d25fcb263c18dcb6231335` |
| comment | `data_sample/comments/2026-07/comments_dedicated_politics_r16_2ca3a0b0d532_0000.parquet` | 119,512 | 11 | parquet-cpp-arrow 17.0.0 | `d6721fa8995ec1c696671aeeab4fe2d7b365795e2f920c71d7a083215d2d9651` |
| comment | `data_sample/comments/2026-07/comments_dedicated_politics_r16_2ca3a0b0d532_0001.parquet` | 119,511 | 11 | parquet-cpp-arrow 17.0.0 | `708d4f4314b705b995965df750740db223677ead59bac03d897be18f45357dbb` |
| comment | `data_sample/comments/2026-07/comments_dedicated_politics_r16_86fe50d736f5_0000.parquet` | 119,645 | 11 | parquet-cpp-arrow 17.0.0 | `b2c6920fd67befc99aab44a9d608a9f69de179d52cbaf9e5ed77c354659056c8` |
| comment | `data_sample/comments/2026-07/comments_dedicated_politics_r16_86fe50d736f5_0001.parquet` | 119,645 | 11 | parquet-cpp-arrow 17.0.0 | `6472c1d68a6cd01f77a739be578cd3386a330b4243f6065edf3de34b29f48353` |
| comment | `data_sample/comments/2026-07/comments_dedicated_politics_r16_c515478cb715_0000.parquet` | 173,308 | 11 | parquet-cpp-arrow 17.0.0 | `181a67539e4398e3855b13590d5a32a5b40b3e3788f66e150eb87c1bd5e1307b` |
| submission | `data_sample/submissions/2026-07/submissions_pooled_1a7_0000.parquet` | 10,747 | 13 | DuckDB v1.5.5 | `a65d68daa926953e876629c40ac18632a7e83ae736d3c4b87c167b3bdd5c9ead` |

Also read: the two ingest sidecars `data_sample/comments/2026-07/manifest.jsonl` and
`data_sample/submissions/2026-07/manifest.jsonl` (row counts, byte sizes, time ranges; compared
with the files, never used as data). Read only for reconciliation, never as data: five tables of
the earlier metadata inventory (`r_analysis_outputs/metadata_inventory/tables/`) and the earlier
delivered Tier 2 document table (`r_analysis_outputs/lda_topics_tier2/tables/document_topic_distribution_K080.csv`,
git-ignored; the cross-check is skipped when it is absent).

The hashes were taken before reading and again after every output had been written: all eight
unchanged (`qc/tables/raw_data_integrity.csv`), and equal to the hashes the metadata inventory
recorded on 2026-09-07.

## 2. Fields used from each source

Every column of both sources is read and carried; none is dropped:

- comments (11): `id`, `parent_id`, `link_id`, `author`, `body`, `created_utc`, `retrieved_on`,
  `score`, `subreddit`, `subreddit_id`, `raw_json`;
- submissions (13): `id`, `title`, `selftext`, `url`, `author`, `created_utc`, `retrieved_on`,
  `score`, `num_comments`, `upvote_ratio`, `subreddit`, `subreddit_id`, `raw_json`.

The seven comment files declare identical schemas (verified); the two writers annotate
strings differently but the physical types are the same (`qc/tables/source_schema.csv`).

## 3. Transformations, in order, with record counts

| Step | What happens | Records in | Records out |
|---|---|---:|---:|
| Read | `nanoparquet::read_parquet()` on every file, all columns, all rows; each row tagged with `source_file` and `source_row_in_file` | 891,397 + 10,747 | 891,397 comments, 10,747 submissions |
| Key | `doc_key = 't1_' + id` (comments), `'t3_' + id` (submissions); `record_type`; `reddit_id = id` | 902,144 | 902,144 |
| Carry | every source field copied unchanged into a column of the same name; fields native to one type set to NA for the other; `created_utc`, `retrieved_on`, `score`, `num_comments` cast from the reader's double to double (no value change) | 902,144 | 902,144 |
| Thread | `thread_id`: `link_id` (comments) / `'t3_' + id` (submissions); `is_thread_root` | 902,144 | 902,144 |
| Stack | comments then submissions, source order kept | 891,397 + 10,747 | 902,144 |
| Presence | `thread_root_in_dataset = thread_id %in% submission doc_keys`; `parent_kind` from the `parent_id` prefix; `parent_in_dataset = parent_id %in% doc_key` | 902,144 | 902,144 |
| Time | `created_datetime_utc = as.POSIXct(created_utc, tz = "UTC")`; `created_date_utc = as.Date(., tz = "UTC")` | 902,144 | 902,144 |
| Text | `text = body` (comments), `paste(title, selftext, sep = " ")` (submissions); `text_n_chars = stri_length(text)` | 902,144 | 902,144 |
| Write | `nanoparquet::write_parquet()`, zstd level 3, one row group, INT64 for the epoch and count fields | 902,144 | 902,144 rows, 29 columns |

No step filters, deduplicates, reorders, trims, recodes, normalises or interprets anything.
No random operation runs (no seed). The written file is deterministic: two runs on the same
inputs produce the same SHA256 (checked in this session).

## 4. Joins performed and their keys

| Join | Left | Right | Key | Purpose | Result |
|---|---|---|---|---|---|
| thread presence | comment `thread_id` | submission `doc_key` | `thread_id = doc_key` (`'t3_' + id`) | `thread_root_in_dataset` | 883,733 comments matched, 7,664 not |
| parent presence | comment `parent_id` | any `doc_key` | `parent_id = doc_key` | `parent_in_dataset`, `parent_kind` consistency | 886,046 matched (538,485 comment parents, 347,561 submission parents), 5,351 not (3,446 comment parents, 1,905 submission parents) |
| reconciliation | canonical measures | metadata-inventory tables | (`record_type`, `field`, `measure`) text keys; (`record_type`, date) for daily counts; file name for hashes | prove equality with the verified counts | 127 of 127 equal |
| Tier 2 cross-check | canonical `doc_key` | delivered document table `doc_key` | `doc_key` | prove the same 902,144 records, fields and text lengths | 13 of 13 items equal |

Neither join adds rows to the canonical table or removes any; both only set flags. No
relationship is inferred beyond what the stored identifiers say: a missing parent or submission
stays missing and is recorded as such.

## 5. Derived fields

`doc_key`, `record_type`, `thread_id`, `is_thread_root`, `thread_root_in_dataset`, `parent_kind`,
`parent_in_dataset`, `created_datetime_utc`, `created_date_utc`, `text`, `text_n_chars`,
`source_file`, `source_row_in_file`; `reddit_id` is `id` renamed. Rules in `DATA_DICTIONARY.md`
and in `qc/tables/parameters.csv`; the manifest of the Parquet file lists the schema difference
(13 added, 1 renamed, 0 removed, 0 retyped).

## 6. Validation results

`qc/tables/reconciliation_checks.csv`: 23 checks, all pass, none not applicable, none failed;
`qc/tables/discrepancies.csv`: none. In brief:

- rows read equal every file's footer count and its ingest-manifest count, bytes and time range;
- 902,144 events for 902,144 source records; each (`source_file`, row) pair once; `doc_key`
  unique; `reddit_id` unique within type; no cross-type id overlap; nothing missing;
- thread and parent flags equal an independent join for every comment; every `t3_` parent equals
  the comment's own `link_id` (349,466 of 349,466);
- `body`, `title`, `selftext`, `author`, `created_utc`, `retrieved_on`, `score`, `url`,
  `num_comments`, `upvote_ratio`, `subreddit`, `subreddit_id`, `link_id`, `parent_id` identical to
  the source, record by record;
- the written Parquet file reads back identical in all 29 columns (names, order, classes,
  missing pattern, values);
- 127 of 127 values recomputed from the metadata-inventory tables equal (`inventory_reconciliation.csv`);
  50 of 50 values written in the earlier reports reproduced (`documented_value_checks.csv`);
  13 of 13 Tier 2 cross-check items equal, including the text length of all 902,144 records
  (`tier2_crosscheck.csv`);
- the eight source files unchanged during the run and equal to the inventory's hashes.

## 7. Software required to reproduce

R 4.5.3 (x86_64-w64-mingw32, Windows 10 x64) with nanoparquet 0.5.1, dplyr 1.2.0, tidyr 1.3.2,
stringi 1.8.7, readr 2.2.0, ggplot2 4.0.2, scales 1.4.0, digest 0.6.39, yaml 2.3.12,
jsonlite 2.0.0, ps 1.9.1 (`environment/r_environment.csv`, `environment/session_info.txt`).
`arrow` is not used: version 25.0.1 crashes on these comment files on this machine. Git is
optional (only the commit id in the sidecars needs it). The unit test additionally needs
tibble (installed with dplyr).

Execution: see `README.md` section "How to run". The script takes its parameters from
environment variables with defaults; nothing is hard-coded to this machine except the default
project root, which is derived from the script's own location.

## 8. Runtime and resources on this dataset

See `environment/run_info.csv`, `stage_timings.csv`, `resource_use.csv`; the figures of the
delivered run are quoted in `README.md` section "Runtime and resource use". Single R process,
no parallel workers.
