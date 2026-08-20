# Text-to-SQL for Low-Resource Languages

This repository contains the code and resources for the paper **Text-to-SQL for Low-Resource Languages: A Unified Large Language Model Framework Evaluated on Arabic, Japanese, and Vietnamese**. The framework was developed by Abdulrahman Aloraini[cite: 1] (العُريني).

## Overview

This project introduces a unified large language model (LLM)-based pipeline designed for low-resource text-to-SQL semantic parsing[cite: 1]. While most research focuses on English, this framework provides a language-agnostic pipeline evaluated on Arabic, Japanese, and Vietnamese benchmarks[cite: 1]. 

The system fine-tunes the `Qwen2.5-Coder-7B-Instruct` model using QLoRA[cite: 1]. Separate language adapters are trained using the same architecture, where only the training data and native-language column descriptions vary by language[cite: 1].

## Framework Components

The prompt-enrichment pipeline consists of five key components:
* **M-Schema Representation:** Provides a clear structure of the database schema, including native language column descriptions[cite: 1].
* **Value Injection:** Grounds the model in actual database content using value hints and BGE-M3 matched values[cite: 1].
* **Sample Rows:** Includes up to three sample rows from relevant tables to help the model infer data types and JOIN relationships[cite: 1].
* **Translation Hints:** Appends a Google Translate English version of the target-language question for supplementary context[cite: 1].
* **Column Linking:** Uses BGE-M3 multilingual embeddings to link target-language question tokens to English schema elements[cite: 1].

At the inference stage, the pipeline also employs:
* **Multi-temperature self-consistency voting:** Generates eight SQL candidates and uses execution-based voting to select the final query[cite: 1].
* **Execution-guided repair stage:** Applies post-processing procedures like identifier fixing, hallucinated-filter removal, and value-literal grounding[cite: 1].

## Key Results

The framework achieves strong results across the evaluated low-resource languages, matching the previous best Arabic result and surpassing prior results for Japanese (+3.3 pp) and Vietnamese (+9.2 pp)[cite: 1].

| Language | Execution Accuracy (EX) | Normalized Exact Set Match (ESM) |
| :--- | :--- | :--- |
| **Arabic** | 75.2% | 66.6% |
| **Japanese** | 75.2% | 66.0% |
| **Vietnamese** | 80.0% | 72.3% |

*(Note: The persistent 9-10 percentage-point EX-ESM gap reflects limitations in LLM-generated SQL, demonstrating that single-instance EX should be reported alongside ESM for a complete evaluation[cite: 1].)*

## Repository Contents

This repository includes:
* The code for the unified LLM-based text-to-SQL pipeline[cite: 1].
* Evaluation and normalization scripts[cite: 1].
* A 149-entry dataset-audit spreadsheet identifying translation errors in the Ar-Spider dataset[cite: 1].

## Pre-trained Adapters

The trained QLoRA adapters for each language are available for download[cite: 1]:
* **Arabic (AR) Adapter:** [Download Here](https://drive.google.com/file/d/1WVcGFI0HicPsdw_d2AdOC0S9Ot3PNdQ3/view?usp=sharing)
* **Japanese (JP) Adapter:** [Download Here](https://drive.google.com/file/d/10NbBwrdR6dvmt8BTb4ebqbfdvOsmZYIk/view?usp=sharing)
* **Vietnamese (VI) Adapter:** [Download Here](https://drive.google.com/file/d/1650Oyk6DzjPfTF81LmIc3-ffi4evLrUn/view?usp=sharing)

[cite: 1]    `
