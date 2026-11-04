# SS Model — Self-Hosted RAG Platform

> This file is the single source of truth for the project. It is used as the Claude Project
> instructions (for both chats) and as `CLAUDE.md` in the repo root (for Claude Code).
> Keep both copies identical. Update the **Status** section at the end of every work session.

---

## 1. What we are building

An open-source platform that answers one question companies are asking:
**"Can we replace a paid LLM API (GPT/Claude) with an open model (Llama) running on our own GPU — and keep the quality?"**

The platform:
1. **Answers questions over a document set with citations** (RAG).
2. **Serves many users at once, fast and cheaply, on a single GPU** (vLLM, quantization, a custom CUDA kernel).
3. **Measures quality, speed and cost** against a paid-API baseline.
4. **Blocks any change that makes quality or speed worse** (CI/CD quality gates).

**Final deliverable:** public GitHub repo, one-command setup (`docker compose up`), before/after benchmark
table, README with architecture diagram, 3-minute demo video, short write-up.

**Timeline:** 20 days. Days 1–12 each person builds their part **independently**. Days 13–20 we integrate,
add CI/CD and write up.

---

## 2. How Claude should work in this project

There are two chats in this project, one per track. **Each chat starts by stating its track.**

- In the **Track A** chat: help only with Track A work (Section 4). Treat Track B as a black box that
  follows the contracts in Section 3.
- In the **Track B** chat: help only with Track B work (Section 5). Treat Track A as a black box that
  follows the contracts in Section 3.
- **Never change a shared contract (Section 3) silently.** If a change is needed, say so clearly and write
  it down under "Contract change requests" in the Status section, so the other chat can pick it up.
- Explain reasoning and trade-offs, not just code — this project is also for learning and interviews.
- Every performance claim needs a saved benchmark result; every quality claim needs an eval run.
- Prefer small, testable steps. Each step should end with something that runs.

---

## 3. Shared contracts (agree once, then work independently)

These are the only points where the two tracks touch. If both sides follow them, integration is mostly
changing one URL.

### 3.1 LLM endpoint (Track B provides, Track A consumes)
- An **OpenAI-compatible** chat completions API: `POST {LLM_BASE_URL}/v1/chat/completions`.
- Track A reads the address from an environment variable: `LLM_BASE_URL`, and the model name from `LLM_MODEL`.
- **During independent work**, Track A points at Ollama's OpenAI-compatible endpoint
  (`http://localhost:11434`). **At integration**, it points at Track B's vLLM server
  (`http://vllm:8000` inside Docker Compose). No code change — only the variable.
- Track B keeps vLLM's standard OpenAI-compatible behavior (streaming optional, `temperature`, `max_tokens` honored).

### 3.2 RAG API (Track A provides; used by the load test and the demo)
- `POST /ask` with body `{"question": "..."}`
- Response:
  ```json
  {
    "answer": "...",
    "citations": [{"doc": "file name", "page": 12, "chunk_id": "..."}],
    "latency_ms": 840,
    "model": "llama-..."
  }
  ```
- `GET /health` returns `{"status": "ok"}`.

### 3.3 Benchmark result format (both tracks write this)
Every benchmark run saves one JSON file to `bench/results/YYYY-MM-DD_<short-name>.json`:
```json
{
  "date": "2026-10-23",
  "track": "A or B",
  "what": "short description, e.g. vllm-awq-4bit vs ollama",
  "hardware": "GPU model, VRAM",
  "model": "model name + quantization",
  "settings": {"concurrency": 50, "max_tokens": 256},
  "metrics": {
    "throughput_tokens_per_s": 0,
    "latency_p50_ms": 0,
    "latency_p95_ms": 0,
    "gpu_memory_gb": 0,
    "accuracy": null,
    "retrieval_hit_at_5": null,
    "cost_per_1k_queries_inr": null
  }
}
```
Unused metrics stay `null`. This lets us build the final before/after table automatically.

### 3.4 Eval set format (Track A owns; Track B reuses to check quantization quality)
`eval/questions.jsonl`, one question per line:
```json
{"id": "q001", "question": "...", "answer": "...", "source_doc": "file name", "source_page": 12}
```
Track B can send these questions **directly to the LLM endpoint** (no retrieval) to compare 4-bit vs 8-bit
answers before integration.

### 3.5 Repo layout and Git rules
```
rag/              Track A — ingestion, retrieval, FastAPI service
eval/             Track A — questions, scoring scripts, API baseline runs
serving/          Track B — vLLM config, quantization scripts, Dockerfile
kernels/          Track B — CUDA kernel, PyTorch binding, tests, benchmarks
bench/            Shared — load-test scripts and results/
.github/workflows Shared — CI/CD (built in the integration phase)
docker-compose.yml Shared — each track adds its own service; edit only your own service block
CLAUDE.md         This file
```
- Branches: `track-a/<feature>` and `track-b/<feature>`; merge into `main` via pull request.
- Each person only edits their own folders. Shared files (`bench/`, `docker-compose.yml`, `CLAUDE.md`)
  — small, clearly described changes only.

---

## 4. Track A — RAG system and evaluation (Soham)

**Goal:** a RAG service that answers questions over the document set with correct citations, plus the
evaluation that proves how good it is — first against Ollama, later against Track B's server.

**Works independently by:** using local Ollama as the LLM through `LLM_BASE_URL`.

### Day-by-day (Days 1–12)
| Days | Work | Done when |
|---|---|---|
| 1–2 | Repo skeleton (`rag/`, `eval/`), Python env, pick the document set (e.g. annual reports of ~15 Indian listed companies, or one open-source project's docs), download them | Documents in `data/raw/`, a script re-downloads them |
| 3–4 | Ingestion: parse PDFs → chunk (≈500–800 tokens, with overlap) → embed with BGE → store in Qdrant (Docker). Keep doc name + page on every chunk | Qdrant holds all chunks; a script rebuilds the index from scratch |
| 5–6 | FastAPI `/ask` and `/health` following contract 3.2. Retrieve top-k chunks, build the prompt, call `LLM_BASE_URL`, return answer + citations | `curl` to `/ask` returns a cited answer using Ollama |
| 7–8 | Write the eval set: 100 questions in `eval/questions.jsonl` (contract 3.4). Mix easy lookups, numbers, multi-part and "not in the documents" questions | 100 reviewed questions with source pages |
| 9–10 | Scoring script: retrieval hit@5 (was the right page retrieved?), answer correctness (LLM-as-judge or exact/numeric match), faithfulness (answer supported by citations?). Run the paid API (GPT or Claude) on the same questions as a baseline; compute cost per 1,000 queries | One command prints a score table; results saved in contract 3.3 format |
| 11–12 | Improve retrieval: hybrid search (BM25 + vectors), optional reranker. Re-run eval and record the gain. Dockerfile for the RAG service | Before/after retrieval scores saved; RAG service runs in Docker |

### Track A deliverables at Day 12
- [ ] `rag/` service running in Docker, following contract 3.2
- [ ] `eval/questions.jsonl` with 100 questions
- [ ] One-command eval script producing scores + a saved result file
- [ ] Paid-API baseline scores and cost per 1,000 queries
- [ ] Short notes: chunking choices, retrieval choices, what improved scores and why

### Track A interview story
"I built the retrieval and evaluation layer: how I chunked and indexed documents, how I measured
retrieval and answer quality, and how hybrid search improved accuracy by X%."

---

## 5. Track B — model serving, GPU and CUDA (Partner)

**Goal:** serve Llama far more efficiently than the Ollama baseline, prove it with load tests, and write
and benchmark a custom CUDA kernel.

**Works independently by:** load-testing the LLM endpoint directly with sample prompts (and later the
questions from `eval/questions.jsonl`), with no need for the RAG service.

### Day-by-day (Days 1–12)
| Days | Work | Done when |
|---|---|---|
| 1–2 | Check GPU (`nvidia-smi`), CUDA version, VRAM. Write the load-test script in `bench/` (async requests at concurrency 1, 10, 50; measure p50/p95 latency, tokens/s, GPU memory). Run it on current Ollama = **baseline** | Baseline saved in contract 3.3 format |
| 3–4 | vLLM in Docker, OpenAI-compatible server (contract 3.1). Same model family as Ollama | `curl` to vLLM's `/v1/chat/completions` works; Dockerfile/compose block in `serving/` |
| 5–6 | Quantized model (AWQ 4-bit; 8-bit if VRAM allows). Load-test each and compare with baseline | Results saved; first big before/after number |
| 7–8 | CUDA kernel: **RMSNorm** (used in every Llama layer). Write in CUDA C++; optionally prototype in Triton first. Correctness test against PyTorch's reference (max abs error within tolerance) | Tests pass on several tensor shapes |
| 9–10 | Expose the kernel as a PyTorch C++/CUDA extension. Benchmark vs PyTorch's version across sizes; explain results (memory bandwidth, block size, fusion) | Benchmark table + short explanation in `kernels/README.md` |
| 11–12 | Tune vLLM (max batch size, `gpu_memory_utilization`, max model length). Quality check: send `eval/questions.jsonl` directly to 4-bit vs 8-bit and compare answers (with Track A's scorer if ready, else manual spot-check) | Tuned config committed; speed-vs-quality comparison saved |

### Track B deliverables at Day 12
- [ ] Baseline vs vLLM vs quantized load-test results (contract 3.3)
- [ ] `serving/` with Dockerfile / compose block for vLLM, following contract 3.1
- [ ] `kernels/` with RMSNorm CUDA kernel, PyTorch binding, correctness tests, benchmark
- [ ] Tuned vLLM settings + 4-bit vs 8-bit quality comparison
- [ ] Short notes: why each change was faster, what the GPU bottleneck was

### Track B interview story
"I took serving from X to Y users at the same latency on one GPU using batching and quantization, wrote
an RMSNorm CUDA kernel, verified it against PyTorch, and can explain where its speed comes from."

---

## 6. Integration phase (Days 13–20, together)

Start only when both tracks meet their Day-12 deliverables (or agree to integrate with what's ready).

| Days | Work | Done when |
|---|---|---|
| 13 | Merge both tracks into `main`. One `docker-compose.yml` with `qdrant`, `rag`, `vllm`. Set `LLM_BASE_URL=http://vllm:8000` | `docker compose up` → `/ask` answers using vLLM |
| 14 | Full-system runs: eval through the whole pipeline (RAG + vLLM); load test against `/ask` at 1, 10, 50 users | End-to-end quality and speed results saved |
| 15–16 | CI/CD in GitHub Actions: (1) on every PR — lint, unit tests, kernel correctness tests, build images (GitHub-hosted runner); (2) eval + load test on the GPU machine as a **self-hosted runner**; (3) **quality gates** — fail if accuracy drops more than an agreed margin or p95 latency rises past a limit; (4) on release — publish images to GitHub Container Registry | A deliberately bad PR is blocked by the gate |
| 17–18 | Results: generate the before/after table from `bench/results/` (Ollama vs vLLM vs quantized vs paid API — accuracy, p95 latency at 50 users, throughput, cost per 1,000 queries). README + architecture diagram | README tells the whole story with numbers |
| 19–20 | Buffer, demo video (3 min), short blog/LinkedIn write-up | Published |

**If behind schedule, cut in this order:** reranker → kernel inside the full model (keep the standalone
benchmark) → paid-API baseline down to 50 questions. **Never cut** the eval set or the CI quality gates.

---

## 7. Status (update at the end of every session)

### Track A
- Current day: 4 (2026-11-04)
- Done: Days 1–2 (skeleton, uv env, LLM client, 11 annual reports in `data/raw/`; notes in
  `docs/track-a/day-01-02.md`). Days 3–4: `rag/parse.py` (spreads split, running headers removed),
  `rag/chunk.py` (≤ 450 BGE tokens, 64 overlap, one page per chunk, company header in embedded
  text), `rag/embed.py` (bge-small via onnxruntime), `rag/index.py` (`uv run python -m rag.index`
  rebuilds from scratch), `rag/search.py` (dense search CLI). 14 tests pass. Notes in
  `docs/track-a/day-03-04.md`.
- Next: Days 5–6, FastAPI `/ask` + `/health` (contract 3.2) on top of `rag.search` + `rag.llm`.
- Blocked on: nothing. Docker still not installed, so Qdrant runs in local mode (`QDRANT_URL` empty);
  set `QDRANT_URL` to switch to a server. No NVIDIA GPU here, so Ollama runs on the CPU (dev only).
- Latest numbers: index = 10,972 chunks from 5,545 printed pages; full rebuild ~20 min on CPU
  (9.3 chunks/s). Ollama llama3.1 smoke test on CPU: ~40 s for 59 completion tokens. Neither is a
  project benchmark.
- Notes for this Windows machine: run `uv` from PowerShell (Git Bash breaks `uv python install`).
  Smart App Control blocks the `pytest.exe`/`ruff.exe` launchers, so use `uv run python -m pytest`
  and `uv run python -m ruff check .`.

### Track B
- Current day:
- Done:
- In progress:
- Blocked on:
- Latest numbers:

### Contract change requests
- (Track A, 2026-10-26, proposed, not yet agreed) Define `page` in 3.2 and `source_page` in 3.4 as
  the **1-based PDF page index** in the file named by `doc` / `source_doc`. Reason: in these
  annual reports, the printed page numbers and PDF page labels don't match the PDF index (e.g.
  two-page spreads, labels like `A, B, 1`). Needs Track B's OK before the eval set is written
  (Days 7–8).

### Decisions log
- Document set (2026-10-26): FY2025-26 annual reports of 11 Indian listed companies, listed in
  `data/sources.csv` (4,679 PDF pages, ~208 MB). Reliance, HDFC Bank, SBI, Kotak Mahindra Bank,
  Wipro, HCLTech, Asian Paints, Bharti Airtel, Sun Pharma, Tata Steel, Bajaj Auto. Rules: one official
  full-report PDF per company (so PDF page numbers match printed pages for citations), hosted on
  the company's own site, and downloadable by a plain script with an honest User-Agent. Left out
  for failing a rule: TCS, Infosys, HUL, ITC, Titan (block non-browser clients); ICICI Bank
  (report split into chapter PDFs); Axis Bank (linked PDF is a 34-page chapter); NTPC (CAPTCHA).
  NSE archives also block plain clients. Many reports use two-page spreads, so ingestion must split
  double-width pages; details in `docs/track-a/day-01-02.md` §5a.
- Model (name, size, quantization): Track A dev uses Ollama `llama3.1` (8B, Q4_K_M). Confirm with
  Track B that vLLM serves Llama 3.1 8B too, so comparisons are like-for-like.
- Embeddings and index (2026-11-04): `BAAI/bge-small-en-v1.5` (384-d) run with onnxruntime on CPU,
  chunks ≤ 450 tokens (BGE reads at most 512), Qdrant collection `annual_reports` with cosine
  distance. bge-small over bge-base for rebuild speed on CPU (~2.5× faster); try bge-base in
  Days 11–12 by changing `EMBED_MODEL`.
- GPU:
