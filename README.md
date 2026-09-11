# Text-to-SQL for Low-Resource Languages

This repository contains the code and resources for the paper **Text-to-SQL for Low-Resource Languages: A Unified Large Language Model Framework Evaluated on Arabic, Japanese, and Vietnamese**.

## Overview

This project introduces a unified large language model (LLM)-based pipeline designed for low-resource text-to-SQL semantic parsing. While most research focuses on English, this framework provides a language-agnostic pipeline evaluated on Arabic, Japanese, and Vietnamese benchmarks.

The system fine-tunes the `Qwen2.5-Coder-7B-Instruct` model using QLoRA. A separate adapter is trained per language on a shared frozen backbone; the architecture, prompt template, pipeline constants, and inference procedure are identical across languages, and only the training data, native-language column descriptions, and translation hints differ. Every reported configuration is measured in a single seeded evaluation harness with the original Spider ESM evaluator, 95% bootstrap confidence intervals, and paired McNemar tests.

## Framework Components

The prompt-enrichment pipeline consists of five key components:
* **M-Schema Representation:** Provides a clear structure of the database schema, including native language column descriptions.
* **Value Injection:** Grounds the model in actual database content using value hints and BGE-M3 matched values.
* **Sample Rows:** Includes up to three sample rows from relevant tables to help the model infer data types and JOIN relationships.
* **Translation Hints:** Appends a Google Translate English version of the target-language question for supplementary context.
* **Column Linking:** Uses BGE-M3 multilingual embeddings to link target-language question tokens to English schema elements.

At the inference stage, the pipeline also employs:
* **Multi-temperature self-consistency voting:** Generates eight SQL candidates and uses execution-based voting to select the final query.
* **Execution-guided repair stage:** Applies five post-processing procedures: identifier fixing, hallucinated-filter removal, value-literal grounding, error-feedback self-correction (up to two retries), and a low-confidence candidate-expansion fallback (a second eight-candidate run with a raw-DDL prompt when the winning vote group holds three or fewer votes).

## Key Results

All numbers are on the 1,034-sample development sets (Ar-Spider, MultiSpider-JA, MultiSpider-VI) under the unified seeded harness. Brackets give 95% bootstrap confidence intervals (B = 10,000).

| Language | EX (full system) | EX (greedy control) | Raw Spider ESM | Normalized ESM (diagnostic) | Strongest published ESM baseline |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Arabic** | 75.2 [72.6, 77.8] | 71.1 | 65.9 [63.0, 68.8] | 67.0 [64.1, 69.9] | 66.63 (LGESQL + XLM-R + CSR) |
| **Japanese** | 74.8 [72.2, 77.4] | 71.4 | 64.1 [61.2, 67.0] | 65.1 [62.2, 68.1] | 61.6 (RAT-SQL + XLM-R + SAVe) |
| **Vietnamese** | 79.9 [77.5, 82.3] | 76.8 | 71.3 [68.6, 74.0] | 72.2 [69.6, 75.0] | 67.8 (RAT-SQL + XLM-R + SAVe) |

* **Raw Spider ESM** is the primary comparison metric: Japanese and Vietnamese surpass the strongest multilingual MultiSpider baseline by +2.5 pp and +3.5 pp; Arabic lies 0.7 pp below the strongest prior Ar-Spider parser.
* **Normalized ESM** is a secondary diagnostic that applies four guarded, gold-blind rewrite rules (literal IN-lists ↔ OR-chains, NOT IN ↔ EXCEPT, MIN/MAX ↔ ORDER BY LIMIT 1, BETWEEN ↔ range predicates) to both predicted and gold SQL. It flips 12, 10, and 10 verdicts (+1.2, +1.0, +1.0 pp), all of which are execution-correct. A fifth candidate rule (integer-threshold rewriting) was rejected because it produced execution-incorrect flips.
* **The EX–ESM gap** (8.6–10.7 pp) persists under the greedy no-voting/no-repair control (8.4–10.8 pp), so it is a property of the generated SQL rather than of the decoder; set-operation (IUEN) generation is the lowest-scoring ESM component in every language.
* **Ablations:** removing all value-grounding components costs −7.3, −4.8, and −5.7 pp EX (the largest single effect); removing all schema-grounding components costs up to −2.5 pp; removing voting and repair jointly costs −4.1, −3.4, and −3.1 pp. The full system exceeds the minimal baseline by 12.3, 8.4, and 11.3 pp.
* An **English same-pipeline reference run** reaches 85.3% EX [83.0, 87.4], giving same-pipeline gaps of −10.1, −10.5, and −5.4 pp for Arabic, Japanese, and Vietnamese; **Llama-3.1-8B-Instruct** reaches 74.6% EX [72.0, 77.2] on Arabic under the identical pipeline.

## Repository Contents

The code is organised as three notebook types, one per stage of the pipeline, plus one evaluation notebook shared by all languages. Run them in the order **fine-tune → inference/ablation → ESM evaluation**.

### Fine-tuning notebooks (training only)

| Notebook | Benchmark | What it does |
| :--- | :--- | :--- |
| `Arabic_Text2SQL_FineTune_Only.ipynb` | Ar-Spider | Builds the prompt-aligned training set from `train.json` (M-Schema, value hints, column linking, value injection, sample rows, English hint, 20% feature dropout), fine-tunes `Qwen2.5-Coder-7B-Instruct` with QLoRA, and saves the adapter + tokenizer to Google Drive. |
| `Japanese_Text2SQL_FineTune_Only.ipynb` | MultiSpider-JA | Same pipeline; additionally generates the Japanese column descriptions and table glosses from `tables_ja.json`. |
| `Vietnamese_Text2SQL_FineTune_Only.ipynb` | MultiSpider-VI | Same pipeline; additionally generates the Vietnamese column descriptions and table glosses from `tables_vi.json`. |

These notebooks contain no inference or evaluation cells. Skip them entirely if you use the [pre-trained adapters](#pre-trained-adapters).

### Inference and ablation notebooks (no training)

| Notebook | Benchmark |
| :--- | :--- |
| `Arabic_Text2SQL_Inference_Ablation_Suite.ipynb` | Ar-Spider |
| `Japanese_Text2SQL_Inference_Ablation_Suite.ipynb` | MultiSpider-JA |
| `Vietnamese_Text2SQL_Inference_Ablation_Suite.ipynb` | MultiSpider-VI |

Each notebook loads the language's QLoRA adapter from Google Drive and evaluates the full pipeline (multi-temperature self-consistency voting + execution-guided repair) on the 1,034-question dev set, then runs a **seeded, resumable ablation suite** in which components are switched off at inference time from the same adapter. The suite evaluates eleven configurations:

| Configuration | Description |
| :--- | :--- |
| `full_system` | Complete pipeline (headline result) |
| `greedy_control` | Voting OFF, repair OFF (single greedy decode) |
| `abl_no_voting` | − Self-consistency voting |
| `abl_no_repair` | − Repair stage |
| `abl_no_column_linking` | − Column-linking hints |
| `abl_no_value_injection` | − Value injection |
| `abl_no_sample_rows` | − Sample rows |
| `grp_no_schema_grounding` (G1) | − All schema-grounding components |
| `grp_no_value_grounding` (G2) | − All value-grounding components |
| `grp_no_translation` (G3) | − External translation |
| `baseline_minimal` | Raw DDL + question, greedy, no repair |

For every configuration the suite reports EX, official Spider ESM (`taoyds/spider` `evaluation.py`, computed per sample), EM, a per-hardness breakdown, 95% bootstrap confidence intervals, and paired McNemar tests against `full_system`. Per-sample results are written to `<EXP_ROOT>/<lang>/<config>.jsonl` and merged into a single `ablation_suite_<lang>.json`. Completed samples are skipped on re-run, so an interrupted suite can be resumed.

### ESM evaluation notebook

* `ESM_Evaluation_full_system_documented.ipynb` – computes the official Spider Exact Set Match for the `full_system` run (and optionally `greedy_control`) of all three languages in one place: raw ESM with difficulty and per-component breakdowns, ESM after the four gold-blind canonical rewrite rules (IN-list ↔ OR-chain, NOT IN ↔ EXCEPT, MIN/MAX ↔ ORDER BY LIMIT 1, BETWEEN ↔ range predicates), a soundness audit of every flipped verdict against the stored EX flag, per-rule attribution, and a cross-check against the `ESM_official` value stored by the ablation suite. Reads the suite output directly; no manual uploads are needed. Results are saved to `esm_analysis_full_system.json`.

### Correspondence with the manuscript

| Manuscript | Notebook / file |
| :--- | :--- |
| Section 3.6 training procedure, Table 2 dropout, Section 4.4 hyperparameters | `*_FineTune_Only.ipynb` |
| Table 5 (EX / raw ESM, full system, − repair, greedy control), Table 6 (vote-count analysis), Table 7 (ablations, rows 1–6 and G1–G4) | `*_Inference_Ablation_Suite.ipynb` — `full_system`, `abl_no_repair`, `greedy_control`, and the remaining suite configurations |
| Table 4 (ESM vs. baselines), Section 4.2 / Appendix B normalization, Table A1 per-rule counts | `ESM_Evaluation_full_system_documented.ipynb` |
| Table 11 / Figure 6 (Ar-Spider translation-error audit) | `ArSpider_Dataset_four_class_error_statistics.xlsx` |

The column-linking sensitivity sweep (Section 5.4, Table 8), the English reference run (Section 5.5), and the Llama-3.1-8B-Instruct run (Section 5.6) are not part of the released notebooks. The English and Llama runs use the same notebooks with the base model, dataset paths, and translation hint changed as described in Sections 4.1 and 4.4.

### Dataset audit

* `ArSpider_Dataset_four_class_error_statistics.xlsx` – a manual full-dataset audit of Ar-Spider identifying 149 translation errors in which the Arabic question diverges from the intent of the gold SQL: 15 in the development set (1.45%) and 134 in the training set (1.55%), classified as entity mistranslation (104), value typo (12), added/dropped columns and constraints (27), and transliteration mismatch (6). Datasets were used exactly as released; none of these errors were corrected or excluded.

The official repository for the paper is https://github.com/amaloraini/low_resource_t2s.

## Pre-trained Adapters

The trained QLoRA adapters for each language are available for download:
* **Arabic (AR) Adapter:** [Download Here](https://drive.google.com/file/d/1WVcGFI0HicPsdw_d2AdOC0S9Ot3PNdQ3/view?usp=sharing)
* **Japanese (JP) Adapter:** [Download Here](https://drive.google.com/file/d/10NbBwrdR6dvmt8BTb4ebqbfdvOsmZYIk/view?usp=sharing)
* **Vietnamese (VI) Adapter:** [Download Here](https://drive.google.com/file/d/1650Oyk6DzjPfTF81LmIc3-ffi4evLrUn/view?usp=sharing)

To use a pre-trained adapter without retraining, unzip it into your Google Drive under the folder name expected by the inference notebook (`<language>_text2sql_7b_adapter_v5_aligned/`, see [Google Drive layout](#4-google-drive-layout)) and go directly to the corresponding `*_Inference_Ablation_Suite.ipynb` notebook.

## Getting Started

The notebooks are written for **Google Colab** and assume a mounted Google Drive. Before running any cell, you must obtain the datasets yourself and place them in your Google Drive. The datasets are **not** redistributed in this repository.

### 1. Requirements

* Google Colab with a GPU runtime. The reported adapters were trained on a single **NVIDIA A100 80 GB** and evaluated on a single **NVIDIA L4 24 GB**. An A100 40 GB also fits QLoRA training of the 7B model plus the BGE-M3 embedding model (~2.3 GB VRAM). A T4 (16 GB) can run inference but training is tight; set `TRAIN_LOAD_EMBEDDINGS = False` if you run out of memory.
* A Google Drive account with at least ~5 GB free (datasets, caches, adapter checkpoints, result files).
* Internet access from the runtime (Hugging Face Hub for the base model and `BAAI/bge-m3`, and Google Translate through `deep-translator` for the translation hints).

All Python dependencies (`transformers`, `peft`, `bitsandbytes`, `trl`, `datasets`, `sentence-transformers`, `sqlparse`, `nltk`, `gdown`, `deep-translator`) are installed by the first cell of each notebook.

### 2. Download the datasets

**You must download the benchmark datasets before running the code.** They are the property of their respective authors; please download them from the official sources and cite them (see [Citation](#citation)).

| Language | Dataset | Official source |
| :--- | :--- | :--- |
| Arabic | **Ar-Spider** | https://github.com/sasmohaimeed/Ar-Spider |
| Japanese, Vietnamese | **MultiSpider** (`ja`, `vi` splits) | https://github.com/longxudou/multispider (also mirrored on Hugging Face: `dreamerdeo/multispider`) |
| Databases + `tables.json` (all languages) | **Spider** | https://yale-lily.github.io/spider (Hugging Face mirror: `xlangai/spider`) |

Ar-Spider, MultiSpider-JA, and MultiSpider-VI each contain 8,659 training and 1,034 development questions over the same 166 Spider databases (146 training and 20 development databases; the development databases are disjoint from the training databases). The three benchmarks are parallel: they share the same databases, gold SQL, and query structures, and only the natural-language questions differ. Of each training set, 5% (433 samples) is held out for validation, so 8,226 examples per language are used for gradient updates. Both Ar-Spider and MultiSpider reuse the original Spider SQLite databases and `tables.json`. Make sure the `database/` folder (one sub-folder per `db_id`, each containing a `.sqlite` file) is present; execution accuracy (EX) and value hints cannot be computed without it.

### 3. Put the datasets in Google Drive

Each notebook expects the dataset in a specific layout. The simplest approach is to zip the prepared folder, upload it to Google Drive, and point the download cell at your copy.

**Arabic (Ar-Spider)** – the notebook's "Download Ar-Spider Dataset" cell fetches a zip with `gdown` and unzips it to `/content/arspider`. Replace the `gdown --id ...` line with the Drive file ID of your own zip (or simply `!cp` the zip from your mounted Drive). After unzipping, the expected structure is:

```
/content/arspider/
├── train.json        (renamed from Ar_train_spider.json by the notebook)
├── dev.json          (renamed from Ar_dev_spider.json by the notebook)
├── tables.json
└── database/
    ├── <db_id>/<db_id>.sqlite
    └── ...
```

**Japanese / Vietnamese (MultiSpider)** – the "Download MultiSpider Dataset" cell uses `huggingface_hub.snapshot_download` to fetch `dreamerdeo/multispider` (with `xlangai/spider` as fallback for the databases) and builds `/content/multispider` with the same layout as above. If you prefer to work offline or from a local copy, download the repository once, upload it to Google Drive, and change the cell to copy from your Drive path instead of calling `snapshot_download`. The notebook needs:

```
/content/multispider/
├── train.json        (MultiSpider <lang> train split, 'question' holds the native-language text)
├── dev.json
├── tables.json       (English Spider tables.json)
├── database/<db_id>/<db_id>.sqlite
└── tables_ja.json / tables_vi.json   (per-database native-language schema names, used to build glosses)
```

### 4. Google Drive layout

The notebooks read and write the following files under `/content/drive/MyDrive/`. They are created automatically on first run; you only need to place the adapter folder there if you are using a pre-trained adapter.

| File / folder (prefix `ar_`, `ja_`, or `vi_`) | Purpose |
| :--- | :--- |
| `<lang>_column_descriptions.json` | Native-language column descriptions. For JA/VI these are generated by Section C0 of the notebook from `tables_<lang>.json` + Google Translate. For Arabic they are generated automatically with Google Translate; the released Arabic notebooks load them from this cache file, so download them from the adapter release (or generate your own) before running. No column description is manually authored. |
| `<lang>_table_glosses.json` | Native-language table glosses for the bilingual M-Schema. |
| `<lang>_*_train_english.json` | Cached Google Translate translations of the training questions. |
| `<lang>_*_english_grounded.json` | Cached, DB-grounded English translations of the dev questions. |
| `<lang>_translate_cache.json` (JA/VI) | Translation cache used while filling gaps in descriptions. |
| `arabic_text2sql_7b_adapter_v5_aligned/` `japanese_text2sql_7b_adapter_v5_aligned/` `vietnamese_text2sql_7b_adapter_v5_aligned/` | Saved QLoRA adapter + tokenizer. Put the downloaded pre-trained adapter here. |
| `gold_hashes_<LANG>.json` | Execution-result hashes of the gold SQL (used for voting diagnostics). |
| `t2s_ablation_suite/<lang>/<config>.jsonl` | Per-sample results of each ablation-suite configuration (written by the inference notebooks; `EXP_ROOT` is configurable). |
| `t2s_ablation_suite/<lang>/ablation_suite_<lang>.json` | Master results file for the language: all configurations, metrics, CIs, McNemar tests. |
| `t2s_ablation_suite/esm_analysis_full_system.json` | Output of the ESM evaluation notebook. |

All paths are exposed as Colab form fields (`#@param`) at the top of the relevant cells, so you can change them without editing code.

### 5. Run the notebooks

All notebooks run top-to-bottom. All paths and hyperparameters are exposed as Colab form fields (`#@param`).

**A. Fine-tuning** (`<Language>_Text2SQL_FineTune_Only.ipynb`) – skip if you use a pre-trained adapter.

1. **Install dependencies → Download dataset → Core imports, schema loading.**
2. **Configuration** (`Configuration` and `Tier 1 Feature Configuration` cells): LoRA rank/alpha, epochs, learning rate, sequence length, and the on/off toggles for each prompt component (M-Schema, value hints, column linking, value injection, sample rows, English hint). The `Training Configuration Sanity Check` cell asserts that all prompt components are enabled, which is the configuration reported in the paper.
3. **Shared function definitions** (Section B) and **pre-training resource loading** (Section C): loads BGE-M3 and the native-language description caches. For JA/VI, Section C0 first builds the descriptions from `tables_<lang>.json`.
4. **Training data construction** (Section D) with prompt alignment and 20% feature dropout, followed by the preview cells (Section E).
5. **`Load Model + Apply QLoRA` → `Train` → `Save Adapter & Tokenizer to Google Drive`.** Training takes about 6 h 10 min per language on an A100 80 GB (QLoRA rank 64, α = 128, adapter dropout 0.05, NF4 double quantization with bf16 compute, learning rate 5e-5 with cosine schedule and 0.05 warmup, weight decay 0.01, gradient clipping 1.0, 3 epochs, 1,029 optimizer steps, per-device batch 6 × gradient accumulation 4 = effective batch size 24, max sequence length 4,096, 5% validation hold-out with seed 42). The adapter is saved to `<language>_text2sql_7b_adapter_v5_aligned/` on Drive; check `DRIVE_SAVE_DIR` first so you do not overwrite a downloaded adapter.

**B. Inference and ablations** (`<Language>_Text2SQL_Inference_Ablation_Suite.ipynb`) – requires the adapter on Drive.

1. **Install dependencies → Download dataset → Core imports, schema loading, metrics → Configuration.** The `Tier 1 Feature Configuration` cell holds the prompt-component and inference-technique toggles (self-consistency, multi-temperature, self-correction, identifier fixing, filter removal, SQL value grounding); the `Sanity Check` asserts all are on.
2. **`Load dev.json` → `Load Adapter from Google Drive` → BGE-M3 → native descriptions/glosses → translation hints** (cached to Drive).
3. **Inference**: post-processing functions, batched generation with self-consistency voting, self-correction, and the combined pipeline / `evaluate_single_sample` function.
4. **Experiment Suite**: `Configuration` (set `EXP_ROOT`; run a smoke test with `SMOKE_TEST_N = 5` first, then set it back to `0`), `Helpers`, `Run selected configurations`, and `Summary & master JSON`. A full 8-candidate configuration takes about 8 h 10–30 min on an L4 24 GB (Appendix C, Table A2) and a greedy configuration about 1.2 h (roughly 3× faster on an A100). Candidates are decoded with the fixed multi-temperature schedule (1 greedy, 2 × T=0.3, 3 × T=0.7, 2 × T=1.1), nucleus p = 0.95, a 300-token cap, deterministic per-sample seeds (base seed 1234), read-only SQLite connections, and a 60 s per-query execution limit. Results are written locally and mirrored to Drive every `SYNC_EVERY` samples; re-run the `Run` cell after a disconnect to resume. `RUN_CONFIGS` accepts a comma-separated subset of configuration names (e.g. `full_system,greedy_control`).

**C. ESM evaluation** (`ESM_Evaluation_full_system_documented.ipynb`) – after the suite has produced `full_system` results.

Run cells 1–6 in order: environment setup (installs NLTK `punkt` and fetches the official Spider evaluator), optional Drive mount, configuration (`EXP_ROOT`, `CONFIGS_TO_EVAL`), normalizer + in-process evaluator, evaluation, summary and export. Every markdown cell explains what the following code cell computes and how to read its output.

### 6. Common issues

* **`Missing: .../tables.json` or `database directory not found`** – the dataset was not unzipped to the expected path; check step 3.
* **Out-of-memory during training** – lower `PER_DEVICE_BATCH_7B`, raise `GRAD_ACCUM_7B`, reduce `MAX_SEQ_LENGTH_7B`, or set `TRAIN_LOAD_EMBEDDINGS = False`.
* **`The notebook must be loaded in full-system mode before running the suite`** – one of the `ENABLE_*` toggles is off in the inference notebook; the suite switches components off itself, so leave all toggles on.
* **All gold queries fail to parse in the ESM notebook (ESM ≈ 0%)** – NLTK `punkt` is missing; re-run cell 1.
* **`Cross-check vs suite master JSON: MISMATCH`** – the ESM notebook environment differs from the one that ran the suite; do not report those numbers until the two agree.
* **Google Translate rate limits** – translations are cached to Drive after the first run; re-run the cell and it will resume from the cache.
* **Execution accuracy is `None`/`WARN` for some samples** – the corresponding `.sqlite` file is missing from `database/`.
* **Different numbers from the paper** – sampling-based voting introduces small run-to-run variance; the notebooks set deterministic seeds (`SC_DETERMINISTIC_SEEDS`) but results may still vary by a few tenths of a point across GPU types and library versions.


## Citation

If you use this code, the adapters, or the Ar-Spider audit, please cite the paper (submitted to *Electronics*, MDPI):

```bibtex
@article{aloraini2026lowresource,
  title   = {Text-to-SQL for Low-Resource Languages: A Unified Large Language Model Framework Evaluated on Arabic, Japanese, and Vietnamese},
  author  = {Aloraini, Abdulrahman},
  journal = {Electronics},
  year    = {2026},
  note    = {Submitted}
}
```

Please also cite the benchmark datasets: Ar-Spider (Almohaimeed et al., ACM SAC 2024), MultiSpider (Dou et al., AAAI 2023), and Spider (Yu et al., EMNLP 2018).

## License and Data Usage

The code in this repository is released for research purposes. The Ar-Spider, MultiSpider, and Spider datasets are distributed under their own licenses by their respective authors; please consult and comply with those licenses before use.
