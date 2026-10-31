# Repository instructions

## Build, run, test, and lint

Use Python 3.11 and install the pinned runtime and development dependencies with:

```bash
python -m pip install -r requirements.txt
```

Run the ingestion pipeline and API with:

```bash
python -m app.ingestion.run
python -m uvicorn app.api.main:app --reload --port 8000
```

The ingestion CLI also supports `--no-upsert`, `--no-graph`, `--reset-graph`, and
`--allow-skips`; see `app/ingestion/run.py` before changing sync behavior.

Pytest and Ruff are listed in `requirements.txt`. Run the suite with
`python -m pytest`; run a single test with
`python -m pytest tests/test_file.py::test_name -q`. There is currently no test
tree in this checkout, so verify test files exist before relying on either
command. Lint the application with `python -m ruff check app`. No separate
build command is configured.

## Architecture

This is a Python RAG service with two complementary knowledge stores. Ingestion
loads PDFs, Markdown, and CSVs from `data/raw`, normalizes document metadata and
canonical system names, then chunks and embeds the documents into Qdrant. It
also builds a deterministic Neo4j graph from corpus facts. The end-to-end entry
point is `app/ingestion/run.py`; the component and data-flow details are in
`docs/ARCHITECTURE.md`.

At request time, FastAPI (`app/api/main.py`) handles chat, health, sessions,
and metrics. Meta questions can be answered by `app/api/system_card.py`;
otherwise NeMo Guardrails wraps the LangGraph agent. The agent's
`knowledge_search` tool fuses Qdrant vector results with parameterized Neo4j
queries, while separate graph and web-search tools handle structured or
out-of-corpus questions. The guardrails layer handles input/output policy,
PII, cache access, and grounding. Retrieved evidence flows through a per-run
ledger to produce citations; Langfuse records request and model telemetry.

## Codebase conventions

- Read configuration through `app/config.py`'s cached `get_settings()`. Do not
  read environment variables directly in application modules.
- Keep expensive clients and models lazy or injectable. Core components accept
  settings and dependencies for isolated construction; close resources they
  own, not injected resources.
- Graph access is through the fixed query templates in
  `app/retrieval/graph_queries.py` and `app/agent/tools.py`; Cypher values are
  bound parameters, not model-generated query text.
- Build citations only from retrieved records, never from model prose. Preserve
  structured retrieval/abstention evidence because grounding decisions depend
  on it, not just relevance scores.
- Keep the API/guardrails path async and call `Guardrails.arun` on its owned
  event loop. Per-request guardrail state uses a `ContextVar`; blocking agent
  work is dispatched with `asyncio.to_thread`.
- Ingestion is designed to be idempotent. By default, unreadable source files
  abort the Qdrant sync to avoid purging valid indexed documents; do not bypass
  this safety behavior without an explicit requirement.
- `docs/ARCHITECTURE.md` and `docs/course/` explain design decisions, but some
  course run/test/deployment instructions refer to files or surfaces absent
  from this checkout. Confirm commands and paths against the current source
  tree before using or adding them.
