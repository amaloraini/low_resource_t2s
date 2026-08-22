# Text-to-SQL for Low-Resource Languages

This repository contains the code and resources for the paper **Text-to-SQL for Low-Resource Languages: A Unified Large Language Model Framework Evaluated on Arabic, Japanese, and Vietnamese**.

## Overview

This project introduces a unified large language model (LLM)-based pipeline designed for low-resource text-to-SQL semantic parsing. While most research focuses on English, this framework provides a language-agnostic pipeline evaluated on Arabic, Japanese, and Vietnamese benchmarks.

The system fine-tunes the `Qwen2.5-Coder-7B-Instruct` model using QLoRA. Separate language adapters are trained using the same architecture, where only the training data and native-language column descriptions vary by language.

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

The framework achieves strong results across the evaluated low-resource languages, matching the previous best Arabic result and surpassing prior results for Japanese (+3.3 pp) and Vietnamese (+9.2 pp).

| Language | Execution Accuracy (EX) | Normalized Exact Set Match (ESM) |
| :--- | :--- | :--- |
| **Arabic** | 75.2% | 66.6% |
| **Japanese** | 75.2% | 66.0% |
| **Vietnamese** | 80.0% | 72.3% |

Raw (un-normalized) ESM is 65.4%, 64.8%, and 71.3% respectively; the normalized figures apply four validated gold-blind rewrite rules (NOT IN ↔ EXCEPT, MIN/MAX ↔ ORDER BY LIMIT 1, IN-lists ↔ OR-chains, BETWEEN ↔ range predicates) to both predicted and gold SQL. An English same-pipeline reference run reaches 85.3% EX, and Llama-3.1-8B-Instruct reaches 74.6% EX on Arabic under the identical pipeline.

## Repository Contents

This repository includes:
* `Arabic_Text2SQL_FT_and_Inference.ipynb` – end-to-end pipeline (training + inference + evaluation) for Arabic (Ar-Spider).
* `Japanese_Text2SQL_FT_and_Inference.ipynb` – end-to-end pipeline for Japanese (MultiSpider-JA).
* `Vietnamese_Text2SQL_FT_and_Inference.ipynb` – end-to-end pipeline for Vietnamese (MultiSpider-VI).
* Evaluation and normalization scripts (EM normalizer ablation and official Spider Exact Set Match evaluation are included as cells at the end of each notebook).
* A 149-entry dataset-audit spreadsheet identifying translation errors in the Ar-Spider dataset (15 in the development set, 134 in the training set).

The official repository for the paper is https://github.com/amaloraini/low_resource_t2s.

## Pre-trained Adapters

The trained QLoRA adapters for each language are available for download:
* **Arabic (AR) Adapter:** [Download Here](https://drive.google.com/file/d/1WVcGFI0HicPsdw_d2AdOC0S9Ot3PNdQ3/view?usp=sharing)
* **Japanese (JP) Adapter:** [Download Here](https://drive.google.com/file/d/10NbBwrdR6dvmt8BTb4ebqbfdvOsmZYIk/view?usp=sharing)
* **Vietnamese (VI) Adapter:** [Download Here](https://drive.google.com/file/d/1650Oyk6DzjPfTF81LmIc3-ffi4evLrUn/view?usp=sharing)

To use a pre-trained adapter without retraining, unzip it into your Google Drive under the folder name the notebook and skip the training cells.

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

Ar-Spider contains 8,657 training and 1,034 development questions; MultiSpider-JA and MultiSpider-VI each contain 8,659 training and 1,034 development questions, all over the same 166 Spider databases. Both Ar-Spider and MultiSpider reuse the original Spider SQLite databases and `tables.json`. Make sure the `database/` folder (one sub-folder per `db_id`, each containing a `.sqlite` file) is present; execution accuracy (EX) and value hints cannot be computed without it.

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
| `<lang>_column_descriptions.json` | Native-language column descriptions. For JA/VI these are generated by Section C0 of the notebook from `tables_<lang>.json` + Google Translate. For Arabic they were generated with the fine-tuned model; the released Arabic notebook loads them from this cache file, so download them from the adapter release (or generate your own) before running. |
| `<lang>_table_glosses.json` | Native-language table glosses for the bilingual M-Schema. |
| `<lang>_*_train_english.json` | Cached Google Translate translations of the training questions. |
| `<lang>_*_english_grounded.json` | Cached, DB-grounded English translations of the dev questions. |
| `<lang>_translate_cache.json` (JA/VI) | Translation cache used while filling gaps in descriptions. |
| `arabic_text2sql_7b_adapter_v5_aligned/` `japanese_text2sql_7b_adapter_v5_aligned/` `vietnamese_text2sql_7b_adapter_v5_aligned/` | Saved QLoRA adapter + tokenizer. Put the downloaded pre-trained adapter here. |
| `eval_results_7b_dev_<LANG>_documented_full.json` | Final per-sample predictions and metrics. |
| `gold_hashes_<LANG>.json` | Execution-result hashes of the gold SQL (used for voting diagnostics). |

All paths are exposed as Colab form fields (`#@param`) at the top of the relevant cells, so you can change them without editing code.

### 5. Run the notebooks

Run the cells top-to-bottom. Each notebook is organised as:

1. **Install dependencies → Download dataset → Core imports, schema loading, metrics.**
2. **Configuration** (`Configuration` and `Tier 1 Feature Configuration` cells): LoRA rank/alpha, epochs, learning rate, sequence length, and the on/off toggles for each prompt component (M-Schema, value hints, column linking, value injection, sample rows, English hint) and inference technique (self-consistency, multi-temperature, self-correction, identifier fixing, filter removal, SQL value grounding). The `Sanity Check` cell asserts that all components are enabled, which is the configuration reported in the paper; disable it if you want to run ablations.
3. **Shared function definitions** (Section B) and **pre-training resource loading** (Section C): loads BGE-M3 and the native-language description caches. For JA/VI, Section C0 first builds the descriptions from `tables_<lang>.json`.
4. **Training data construction** (Section D) with prompt alignment and 20% feature dropout, followed by the preview cells (Section E).
5. **Training** (`Load Model + Apply QLoRA` → `Train`) and **saving the adapter to Drive**. The training and save cells are wrapped in `'''...'''` in the released notebooks so that the inference path can be run with a pre-trained adapter; remove the quotes to train from scratch. Training takes about 6 h 10 min per language on an A100 80 GB (3 epochs, 1,029 optimizer steps, effective batch size 24, learning rate 5e-5, max sequence length 4,096).
6. **Free Training Memory → Load Adapter from Google Drive.** After training, it is safest to restart the runtime, re-run steps 1–3, and then continue from `Load Adapter from Google Drive`.
7. **Inference**: translation hints, post-processing functions, batched generation with self-consistency voting, self-correction, and `Run Evaluation (Dev Set)`. Evaluation over the full 1,034-sample dev set with 8 candidates per question takes about 8 h 10–30 min on an L4 24 GB (faster on an A100). `RESUME_FROM` lets you continue an interrupted run.
8. **Results**: summary tables, saving to JSON/Drive, the **EM Normalizer Impact Analysis** (progressive normalization ablation), and the **Spider Exact Set Match** cell, which downloads the official Spider evaluation script and computes ESM for direct comparison with the Ar-Spider and MultiSpider papers.

### 6. Common issues

* **`Missing: .../tables.json` or `database directory not found`** – the dataset was not unzipped to the expected path; check step 3.
* **Out-of-memory during training** – lower `PER_DEVICE_BATCH_7B`, raise `GRAD_ACCUM_7B`, reduce `MAX_SEQ_LENGTH_7B`, or set `TRAIN_LOAD_EMBEDDINGS = False`.
* **Google Translate rate limits** – translations are cached to Drive after the first run; re-run the cell and it will resume from the cache.
* **Execution accuracy is `None`/`WARN` for some samples** – the corresponding `.sqlite` file is missing from `database/`.
* **Different numbers from the paper** – sampling-based voting introduces small run-to-run variance; the notebooks set deterministic seeds (`SC_DETERMINISTIC_SEEDS`) but results may still vary by a few tenths of a point across GPU types and library versions.


## License and Data Usage

The code in this repository is released for research purposes. The Ar-Spider, MultiSpider, and Spider datasets are distributed under their own licenses by their respective authors; please consult and comply with those licenses before use.
