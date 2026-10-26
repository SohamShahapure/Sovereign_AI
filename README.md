# Track A: Days 1–2, repo skeleton and environment

**Goal (from CLAUDE.md §4):** repo skeleton (`rag/`, `eval/`), Python env, pick the document set, download it.
**Done when:** documents are in `data/raw/` and a script re-downloads them.

| Status | Item |
|---|---|
| ✅ | Git repo on `main`, folder layout per contract 3.5 |
| ✅ | Python 3.12 env managed by uv, dependencies locked in `uv.lock` |
| ✅ | LLM config via `LLM_BASE_URL` / `LLM_MODEL` (contract 3.1), verified against Ollama |
| ✅ | Re-download script `scripts/download_docs.py` |
| ✅ | Tests (pytest) and lint (ruff) passing |
| ⬜ | Document set chosen and listed in `data/sources.csv` |
| ⬜ | Documents downloaded into `data/raw/` |

---

## 1. Prerequisites

| Tool | Version used | Needed for |
|---|---|---|
| [uv](https://docs.astral.sh/uv/) | 0.11.19 | Python version and dependencies |
| git | 2.54 | Version control |
| [Ollama](https://ollama.com) | 0.40.9 | Local LLM during independent work |
| Docker Desktop | **not installed yet** | Qdrant (Days 3–4), RAG image (Days 11–12) |

You do **not** need to install Python 3.12 yourself: uv downloads it.

> **Windows note:** run `uv` from **PowerShell**. From Git Bash, `uv python install` fails with
> `Missing expected target directory for Python minor version link`, even though the install
> itself succeeded.

## 2. Setup from a fresh clone

```bash
uv sync
```

```bash
cp .env.example .env
```

```bash
ollama pull llama3.1
```

`uv sync` reads `.python-version` (3.12), downloads that Python if needed, creates `.venv/` and installs
the exact versions in `uv.lock`. In PowerShell, use `Copy-Item .env.example .env` instead of `cp`.

## 3. Check that it works

**Tests and lint:**

```bash
uv run pytest -q
```

```bash
uv run ruff check .
```

**LLM endpoint (contract 3.1):** sends one question to `{LLM_BASE_URL}/v1/chat/completions`:

```bash
uv run python -m rag.llm "In one sentence, what is retrieval-augmented generation?"
```

Expected output: the target URL and model, the answer, then a line like
`[llama3.1] 40101 ms, 22 prompt / 59 completion tokens`.

**Documents:**

```bash
uv run python scripts/download_docs.py
```

Until `data/sources.csv` has rows, this prints `No documents listed in data\sources.csv yet.`

## 4. What is in the repo

```
.env.example          LLM_BASE_URL, LLM_MODEL, LLM_API_KEY, LLM_TIMEOUT_S (copy to .env)
.python-version       3.12, read by uv
pyproject.toml        dependencies + ruff/pytest config
uv.lock               exact dependency versions (commit this)
rag/
  config.py           loads settings from env vars / .env
  llm.py              OpenAI-compatible chat client + smoke test (python -m rag.llm)
eval/                 empty for now (questions.jsonl arrives Days 7–8)
scripts/
  download_docs.py    downloads every row of data/sources.csv into data/raw/
data/
  sources.csv         document manifest: filename, url, company, year, notes
  raw/                downloaded PDFs (git-ignored)
bench/results/        benchmark JSON files (contract 3.3)
tests/test_config.py  URL normalization + env override tests
docs/track-a/         these notes
```

### `rag/config.py`
Reads `LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY` and `LLM_TIMEOUT_S`. Real environment variables take
priority over `.env`, so Docker Compose can override them at integration. The base URL is normalized:
`http://vllm:8000`, `http://vllm:8000/` and `http://vllm:8000/v1` all become `http://vllm:8000`. The
`/v1/chat/completions` path is added in one place.

### `rag/llm.py`
`chat(messages, temperature=0.0, max_tokens=512)` posts to the endpoint and returns a `ChatResult` with
the text, model name, `latency_ms` and token counts. Temperature defaults to 0 so eval runs can be
repeated. The `/ask` endpoint (Days 5–6) will call this function.

### `scripts/download_docs.py`
- Skips files already in `data/raw/`; `--force` re-downloads them.
- Writes to a `.part` file first and renames it only when the download completes, so an interrupted
  download never leaves a half-written PDF behind.
- Rejects `.pdf` responses that don't start with `%PDF-`. Company sites sometimes return an HTML
  error or login page with status 200.
- Prints a short SHA-256 for each file, so you can tell when a company replaces a report.
- Exits non-zero if any download fails.
- Sends a browser-like User-Agent, because some sites block the default client.

## 5. Design decisions and trade-offs

| Decision | Why | Trade-off |
|---|---|---|
| **uv + Python 3.12** (not the system's 3.14) | ML libraries (torch, sentence-transformers, PDF parsers) often lack wheels for the newest Python. 3.12 is widely supported. `uv.lock` makes installs reproducible. | One more tool to install |
| **Plain `httpx` client instead of the `openai` SDK** | The whole request fits in one readable function. Latency is measured around exactly the HTTP call. Works with any OpenAI-compatible server. | No built-in retries or streaming; easy to add if needed |
| **Config only from env vars** | Contract 3.1: moving from Ollama to vLLM must change a variable, not code | None really |
| **Raw PDFs git-ignored, manifest committed** | Keeps the repo small. Anyone can rebuild the exact set with one command. | Depends on source URLs staying online. Hashes show when a file changes. |
| **Minimal dependencies for now** | Each day adds only what it needs, so it's clear why each package is there | — |

## 6. Measured so far

| What | Result | Notes |
|---|---|---|
| Ollama `llama3.1` (8B, Q4_K_M) smoke test | ~40 s for 59 completion tokens | One request on CPU (no NVIDIA GPU on this machine). Not a benchmark; nothing saved to `bench/results/`. |

On CPU, a full 100-question eval through Ollama will be slow. Use a small question subset while
developing and run the full set on Track B's GPU at integration.

## 7. Known issues and open items

- **Docker isn't installed.** Install Docker Desktop (WSL2 backend) before Day 3. If that isn't
  possible, Qdrant can run in-process (`QdrantClient(path="qdrant_storage")`) during development.
- **No GPU on the Track A machine.** BGE embeddings will run on the CPU; `bge-small` or `bge-base`
  keeps indexing time reasonable.
- **Document set not chosen yet.** Leaning toward ~15 annual reports of Indian listed companies: lots
  of numbers make good eval questions, but their tables are hard to parse. Record the decision in the
  CLAUDE.md Decisions log.
- **Model alignment:** confirm with Track B that vLLM will serve Llama 3.1 8B, so comparisons are
  like-for-like.

## 8. Next: Days 3–4 (ingestion)

Parse PDFs → chunk (~500–800 tokens, with overlap) → embed with BGE → store in Qdrant, keeping the doc
name and page on every chunk. Done when one script rebuilds the whole index from `data/raw/`.
