# Handoff

State of the project and how to resume. Commands are for Windows PowerShell or any shell with the venv active.

## What exists and was verified

* **Quality gate:** `scripts/check.ps1` (ruff, ruff format, mypy on `src` and `api`, pytest with an 80 percent coverage gate). Unit tests use hand-computed answers and in-memory fakes; integration tests run on the real corpus, indexes and Neo4j graph with a deterministic stand-in LLM; API and MCP contract tests need no data.
* **Data:** EPO OPS corpus of the citation-closed full scope, cached under `data/raw/ops` (gitignored; do not delete: the weekly OPS quota is limited). `data/corpus/patents_full.jsonl` is the corpus; `stores/` holds the indexes.
* **Results** (`results/`): produced on a Google Colab T4 VM by `scripts/colab_setup.sh` and `scripts/pipeline.sh` from the same corpus file, copied back with `scripts/export_results.py` and `scripts/import_results.py` (length and CRC checked). See `docs/DECISIONS.md` for the history of the design rounds.
* **Serving:** the API was checked end to end on the Colab VM after the evaluation (`scripts/e2e_smoke.py` against `patentrag serve-api`: readiness, a search with every method, one agent run as a background job, and its trace; output in `results/e2e_smoke_full.json`). On the laptop the Docker stack (`docker compose --profile app up -d --build`: Neo4j, API, UI; Ollama on the host) serves search well; an agent run on the 8 GB laptop CPU takes minutes and, next to the containers, once used all memory (`docs/KNOWN_ISSUES.md`, K1).

## Rebuild or rerun

1. Environment: `scripts/setup.ps1` (or `make setup`), then `copy .env.example .env` and fill in OPS credentials and a Neo4j password.
2. Credentials: `python -m patentrag.cli check-credentials` (never prints values).
3. Data: `python -m patentrag.cli fetch --scope full` resumes from the cache; after the weekly quota resets (Monday 00:00 GMT) it continues. Then `build-corpus`.
4. Compute (GPU strongly recommended): upload the bundle from `python scripts/make_colab_bundle.py` with `colab/patentrag_colab.ipynb`, or run `bash scripts/pipeline.sh` on any machine with Neo4j, Ollama and a GPU. If the VM stops part way, run `bash scripts/pipeline.sh agent`: saved agent runs are skipped. `bash scripts/pipeline.sh reports` reuses the stored tuning and runs only the tests, the test comparison, the claim-chart reports, the API check and the audit (how the reported results were finished).
5. Results back: copy the export cell's output into a text file, then `python scripts/import_results.py <file>`.
6. Documents: `python scripts/render_tables.py full`, then `python scripts/build_readme.py full`.
7. Neo4j: `docker compose --env-file .env -f docker/docker-compose.yml up -d neo4j`. Its data volume `patentrag_neo4j_data` holds the loaded graph; do not remove it unless you intend to reload.

## Known problems

See `docs/KNOWN_ISSUES.md`.
