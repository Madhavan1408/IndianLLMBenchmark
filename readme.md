# IndicRAG Evaluation Framework

LangSmith-powered benchmark that evaluates RAG quality across 12 Indic languages — faithfulness, cross-lingual consistency, and retrieval precision.

Built to amplify BharatBench work from Sarvam AI and AI4Bharat.

---

## Supported languages

| Code | Language   | Script      | Tier |
|------|-----------|-------------|------|
| hi   | Hindi      | Devanagari  | High |
| ta   | Tamil      | Tamil       | High |
| te   | Telugu     | Telugu      | High |
| kn   | Kannada    | Kannada     | High |
| ml   | Malayalam  | Malayalam   | High |
| bn   | Bengali    | Bengali     | High |
| mr   | Marathi    | Devanagari  | High |
| gu   | Gujarati   | Gujarati    | Mid  |
| pa   | Punjabi    | Gurmukhi    | Mid  |
| or   | Odia       | Odia        | Mid  |
| as   | Assamese   | Bengali     | Low  |
| sat  | Santali    | Ol Chiki    | Low  |

---

## Quick start

### 1. Clone and install

```bash
git clone https://github.com/your-org/indicrag.git
cd indicrag

# Using uv (recommended)
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"

# Or pip
pip install -e ".[dev]"
```

### 2. Configure environment

```bash
cp .env.example .env
# Edit .env and fill in your API keys:
#   LANGCHAIN_API_KEY   → from smith.langchain.com
#   OPENAI_API_KEY      → for RAGAS judge (GPT-4o)
#   SARVAM_API_KEY      → for Sarvam-1 generator
#   HF_TOKEN            → for AI4Bharat IndicBERT embeddings
```

### 3. Run the smoke test (Phase 0)

```bash
python scripts/smoke_test.py
# All checks pass? You're ready.

python scripts/smoke_test.py --with-trace
# Also fires a real LangSmith trace — verify it in the dashboard.
```

### 4. Build datasets (Phase 1)

```bash
# Dry run — build locally, skip LangSmith push
python -m src.pipeline.dataset_builder --lang hi --dry-run

# Push a single language to LangSmith
python -m src.pipeline.dataset_builder --lang ta --push

# Push all 12 languages at once
python -m src.pipeline.dataset_builder --all --push

# With a version tag
python -m src.pipeline.dataset_builder --all --push --version v2
```

Datasets are saved locally to `datasets/raw/<lang>.jsonl` and pushed to LangSmith as `indicrag-<lang>-v1`.

---

## Project structure

```
indicrag/
├── configs/
│   └── languages.py          # Language registry — add new languages here
├── src/
│   ├── pipeline/
│   │   ├── dataset_builder.py  # Phase 1: build & push datasets
│   │   ├── retriever.py        # Phase 2: multilingual FAISS retriever
│   │   └── rag_chain.py        # Phase 2: generator + LangSmith tracing
│   ├── evaluators/
│   │   ├── ragas_evaluator.py  # Phase 3: RAGAS metrics as LangSmith evaluators
│   │   └── crosslingual.py     # Phase 3: cross-lingual drift metric
│   └── utils/
│       └── schema.py           # QAPair, RAGOutput Pydantic models
├── datasets/
│   ├── raw/                    # JSONL files per language
│   └── processed/              # Post-processed / augmented
├── evals/
│   └── run_benchmark.py        # Phase 4: full benchmark runner
├── scripts/
│   └── smoke_test.py           # Phase 0: environment & connectivity check
├── tests/
├── .env.example
└── pyproject.toml
```

---

## Metrics

| Metric | Description | Tool |
|--------|-------------|------|
| Faithfulness | Are answer claims grounded in retrieved docs? | RAGAS |
| Answer relevancy | Does the answer address the question? | RAGAS |
| Context precision | Are retrieved chunks useful? | RAGAS |
| Context recall | Is all ground-truth context retrieved? | RAGAS |
| Cross-lingual drift | Score delta vs Hindi anchor | Custom |

---

## Adding more QA pairs

Edit `src/pipeline/dataset_builder.py` → `SEED_DATA` dict, or load from an external file:

```python
# Load from a JSONL file you've prepared
import json
with open("my_ta_pairs.jsonl") as f:
    pairs = [QAPair(**json.loads(line)) for line in f]
```

For production-quality evaluation, aim for **200+ pairs per language** with human verification (`verified=True`).

---

## Next phases

- **Phase 2** — Multilingual RAG pipeline: IndicBERT embeddings → FAISS → Sarvam-1 generator
- **Phase 3** — Evaluators: RAGAS + custom cross-lingual drift, wrapped as LangSmith `RunEvaluator`s
- **Phase 4** — Full benchmark: `langsmith evaluate()` across all 12 languages with experiment tracking