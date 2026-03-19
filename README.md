# LLM-Powered Vulnerability Report Triage

> An LLM pipeline that automatically classifies CVE descriptions by severity, CWE category, and affected component — benchmarked against NVD ground truth.

---

## Goal

Security teams drown in CVE noise. This project builds an LLM-powered triage system that reads raw vulnerability descriptions (the kind that arrive in advisories, NVD entries, or bug trackers) and extracts structured intelligence: severity, weakness category, affected component, and a brief analyst summary. The benchmark against NVD ground truth is what separates this from a demo.

---

## Research Questions

- How accurately can a prompted LLM (zero-shot) classify CVE severity vs a fine-tuned small model?
- Which structured extraction tasks benefit most from few-shot prompting vs fine-tuning?
- What failure modes emerge at scale — hallucinated CWE IDs, wrong severity ratings, ambiguous component names?
- Can RAG over historical CVE data improve classification of novel vulnerability patterns?
- How does classification accuracy vary by vulnerability domain (web, memory corruption, crypto, etc.)?

---

## Project Phases

### Phase 1 — Dataset construction
- Pull CVE records from NVD JSON feeds (free, public)
- Filter to records with: CVSS score, CWE ID, CPE (affected component)
- Clean and split into train/validation/test sets stratified by severity and CWE category
- Build evaluation harness: exact match + partial credit scoring for CWE hierarchy

### Phase 2 — Baseline: zero-shot prompting
- Design a structured output prompt (JSON schema) for: severity bucket, CWE ID, affected component, one-sentence summary
- Test against GPT-4o-mini or Claude Haiku (cost-efficient)
- Measure accuracy on held-out NVD records
- Document prompt failure cases systematically

### Phase 3 — Improved prompting strategies
- Few-shot examples (5–10 per category)
- Chain-of-thought: ask the model to reason before outputting the JSON
- Output validation: reject and retry if schema is malformed or CWE ID doesn't exist

### Phase 4 — Fine-tuning a small model (optional but high signal)
- Fine-tune a small open model (Mistral-7B or similar) on NVD records
- Compare accuracy, speed, and cost against API-based approach
- Evaluate on a held-out set from a different year (tests generalization)

### Phase 5 — RAG layer
- Build a FAISS index over historical CVE summaries
- At inference time, retrieve top-k similar past CVEs and include in context
- Measure whether retrieval improves classification of low-frequency CWE categories

### Phase 6 — Interface and demo
- FastAPI backend: `POST /triage` accepts raw advisory text, returns structured JSON
- Simple React or Streamlit frontend showing side-by-side: raw text → extracted fields → confidence
- Batch mode: process a CSV of CVE descriptions and export results

---

## Repo Structure

```
vuln-triage-llm/
├── data/
│   ├── nvd_feeds/            # Raw NVD JSON (gitignored, download script provided)
│   └── processed/
│       ├── train.jsonl
│       ├── val.jsonl
│       └── test.jsonl
├── notebooks/
│   ├── 01_dataset_construction.ipynb
│   ├── 02_baseline_prompting.ipynb
│   ├── 03_prompt_engineering.ipynb
│   ├── 04_finetuning.ipynb   # Optional
│   └── 05_rag_evaluation.ipynb
├── src/
│   ├── data/
│   │   ├── nvd_downloader.py # Fetches and parses NVD JSON feeds
│   │   └── preprocessor.py
│   ├── prompts/
│   │   ├── templates.py      # Zero-shot and few-shot prompt templates
│   │   └── few_shot_examples.py
│   ├── pipeline/
│   │   ├── classify.py       # Main inference pipeline
│   │   ├── validate.py       # Output schema validation + retry logic
│   │   └── rag.py            # FAISS retrieval layer
│   └── evaluate/
│       └── metrics.py        # Accuracy, F1, CWE hierarchy partial credit
├── api/
│   └── main.py
├── frontend/                 # Optional Streamlit or React UI
├── scripts/
│   └── download_nvd.sh
├── tests/
├── requirements.txt
└── README.md
```

---

## Stack

| Purpose | Library |
|---|---|
| LLM inference | `openai`, `anthropic`, or `ollama` |
| Orchestration | `langchain` or raw API calls |
| Vector search | `faiss-cpu`, `sentence-transformers` |
| Data handling | `pandas`, `jsonlines` |
| Evaluation | `scikit-learn`, custom CWE hierarchy scorer |
| Serving | `fastapi`, `pydantic` |
| Frontend | `streamlit` or `react` |

---

## Datasets

- **NVD CVE JSON feeds** — https://nvd.nist.gov/developers/vulnerabilities (free, no auth required for bulk)
- **MITRE CWE list** — for validating predicted CWE IDs and building the hierarchy scorer
- **OSV (Open Source Vulnerabilities)** — additional records for generalization testing

---

## Output Schema

Each triage result should conform to:

```json
{
  "cve_id": "CVE-2024-XXXXX",
  "severity": "CRITICAL | HIGH | MEDIUM | LOW",
  "cvss_estimate": 8.1,
  "cwe_id": "CWE-89",
  "cwe_name": "SQL Injection",
  "affected_component": "Apache Struts HTTP request parser",
  "summary": "Unauthenticated attacker can execute arbitrary SQL via crafted GET parameter.",
  "confidence": 0.91
}
```

---

## Key Metrics to Track

| Metric | Target |
|---|---|
| Severity bucket accuracy | > 85% (4-class) |
| CWE top-1 accuracy | > 70% |
| CWE top-5 accuracy | > 90% |
| Component extraction F1 | > 0.75 (partial match) |
| Latency per record | < 2s (API), < 500ms (local model) |

---

## Stretch Goals

- [ ] Slack/email integration: auto-post triage results for new CVEs matching a keyword watchlist
- [ ] Confidence calibration: flag low-confidence predictions for human review queue
- [ ] Multi-advisory format support: PDF advisories, GitHub security advisories, vendor bulletins
- [ ] Active learning loop: analyst corrections feed back into few-shot example selection

---

## References

- NVD JSON data feeds: https://nvd.nist.gov/developers/vulnerabilities
- MITRE CWE: https://cwe.mitre.org/data/
- "Large Language Models for Cyber Security" (survey, arXiv 2023)
- LangChain docs: https://docs.langchain.com

---

## Status

- [ ] Phase 1 — Dataset construction
- [ ] Phase 2 — Zero-shot baseline
- [ ] Phase 3 — Prompt engineering
- [ ] Phase 4 — Fine-tuning
- [ ] Phase 5 — RAG layer
- [ ] Phase 6 — Interface
