# Rights Regression Radar â€” download bundle

Two archives. Take the small one if you only want the code; take the full one if
you want to run the model without retraining it.

| Archive | Size | Contents |
|---|---|---|
| `rrr-code.zip` | 3.2 MB | All source, tests, docs, data, reports. No model weights. |
| `rrr-full-with-model.zip` | 498.7 MB | Everything above plus the trained model. |

Checksums are in `SHA256SUMS`.

```
729e7272eb42dce71ac6c3d68865cac63ade68e03092b40146e647f9ba540453  rrr-code.zip
3be333778ffd1185ff3e08efc72861f17e1469db17027494d18bbdddac95d7fa  rrr-full-with-model.zip
```

Both extract to a top-level `rrr/` directory.

## Run it

```bash
unzip rrr-code.zip          # or rrr-full-with-model.zip
cd rrr
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

PYTHONPATH=src python -m pytest tests/ -q        # 26 tests
PYTHONPATH=src python -m rrr.demo --compare      # model vs baselines
PYTHONPATH=src python -m rrr.demo --demo         # worked examples
PYTHONPATH=src python -m uvicorn rrr.api.service:app --port 8000
# demo page at http://localhost:8000/demo
```

`rrr-code.zip` has no weights, so the model-dependent tests skip (23 pass, 3
skip) and `rrr.demo --compare` needs a trained model. `rrr-full-with-model.zip`
runs all 26 tests and reproduces macro-F1 0.965 from the extracted copy.

## What is in each archive

### Source â€” `src/rrr/`

| File | Purpose |
|---|---|
| `ingest/pdf.py` | PDF text extraction (pypdf), per-document metadata |
| `ingest/clauses.py` | split a statute into numbered clauses |
| `ingest/amendments.py` | parse amendment instructions; `_parse_ref` picks the reference nearest the verb (direct object) |
| `ingest/build_corpus.py` | drive ingestion over a corpus directory |
| `ml/dataset.py` | build the four-class pairs; splits by document, not by pair |
| `ml/baselines.py` | TF-IDF + logistic regression, and zero-shot NLI |
| `ml/train.py` | DeBERTa-v3 cross-encoder fine-tuning, early stopping, `SaveTokenizer` |
| `ml/metrics.py` | the single source of truth for accuracy, macro-F1, per-class, confusion |
| `ml/calibrate.py` | pick the review threshold on validation only |
| `ml/report.py` | render `reports/REPORT.md` and `reports/metrics.json` |
| `ml/finalize.py` | evaluate a checkpoint and export it for serving; guards the tokenizer |
| `ml/subgroups.py` | accuracy split by pair difficulty (template vs substantive) |
| `retrieval/index.py` | BM25 index |
| `retrieval/dense.py` | dense retriever (`all-MiniLM-L6-v2`) |
| `retrieval/evaluate.py` | recall@k for self- and cross-provision probes |
| `infer.py` | shared inference path used by both the API and the CLI |
| `api/service.py` | FastAPI service: `/classify`, `/analyze`, `/retrieve`, `/metrics`, `/health` |
| `demo.py` | CLI: `--compare` and `--demo` |

### Integration â€” `integration/backend_adapter.py`
Drop-in adapter for your existing SQLAlchemy models. It is the only file that
knows about your schema; nothing under `src/rrr/` imports it.

### Docs
- `docs/LABELS.md` â€” what each of the four classes means and how pairs are built
- `docs/ARCHITECTURE.md` â€” how the pieces fit together
- `docs/EVALUATION.md` â€” protocol, results, and the limitations
- `AGENTS.md` â€” invariants and the traps already hit (read this before changing the ML code)
- `README.md`

### Tests â€” `tests/`
- `test_pipeline.py` â€” ingestion, amendment parsing, dataset splits
- `test_api.py` â€” service endpoints
- `test_subgroups.py` â€” difficulty split and tokenizer guard

### Data â€” `data/`
- `documents.jsonl` â€” 202 documents with filename, title, year, sha256
- `clauses.jsonl` â€” 8,796 clauses
- `pairs.jsonl` â€” 1,820 labelled pairs (train 1,380 / val 180 / test 260)

### Reports â€” `reports/`
- `REPORT.md` â€” generated comparison of all four models
- `metrics.json` â€” the same numbers, machine-readable
- `retrieval_metrics.json` â€” BM25 and dense recall@k
- `subgroups.json` â€” accuracy by pair difficulty

### Model â€” `artifacts/deberta-v3-small-rrr/` (full archive only)
`config.json`, `model.safetensors` (567 MB), `tokenizer.json`,
`tokenizer_config.json`, `trainer_state.json`.

This is the best checkpoint (epoch 1, validation macro-F1 0.9611), exported.
`trainer_state.json` carries the training log. The 1.1 GB `optimizer.pt` files
from training are not included; they are only needed to resume training.

`artifacts/model_predictions.json` holds the test and validation predictions
every reported number is computed from, so you can recompute them yourself.
`artifacts/clause_embeddings.npy` (13 MB, dense retrieval cache) is not included;
it is rebuilt on first run.

### Not included
- The 204 source PDFs (134 MB) â€” they are your corpus, not mine to redistribute.
  Ingestion reads them from a directory you point `build_corpus` at.
- `.venv/`, `.git/`, caches.
