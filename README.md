# researcher — Standalone Autonomous Security Research Agent

`researcher` is a **100% standalone** Python tool. It continuously collects
*publicly available* security research, normalizes it into a durable markdown
knowledge archive, reasons about how new research relates to what it already
knows, and synthesizes evolving testing checklists, gap reports, and
daily/weekly digests. It has **no dependency on Claude Code** or any cloud LLM
service at runtime — all reasoning runs through **Ollama** locally, and vectors
live in **Qdrant**.

It is designed to make you a **better tester**, not to attack for you.

---

## Table of contents

1. [Guardrail](#guardrail)
2. [What it does](#what-it-does)
3. [Architecture overview](#architecture-overview)
4. [Repository layout](#repository-layout)
5. [Requirements](#requirements)
6. [Install](#install)
7. [Bringing up the services](#bringing-up-the-services)
8. [Configuration](#configuration)
9. [Quick start](#quick-start)
10. [CLI reference](#cli-reference)
11. [Core components](#core-components)
12. [Where data lives](#where-data-lives)
13. [The knowledge archive (source of truth)](#the-knowledge-archive-source-of-truth)
14. [Moving to another machine](#moving-to-another-machine)
15. [Upgrading phase by phase](#upgrading-phase-by-phase)
16. [Troubleshooting](#troubleshooting)

---

## Guardrail

`researcher` is a **research and methodology** tool, not an attack tool.

- It ingests only **public, defensively-oriented sources** (research blogs,
  advisories, disclosed write-ups, OWASP, CVE/GHSA, arXiv, conference pages,
  YouTube transcripts).
- It extracts **patterns, techniques, and testing ideas** — not ready-to-fire
  exploit payloads.
- It never targets, scans, fingerprints, or attacks any system.
- It does not store or emit ready-to-fire payloads. Techniques are
  generalized; specifics are reduced to testing ideas and detection/mitigation
  notes.
- It does not replace tester judgment; it reduces the time spent finding,
  organizing, and connecting new public research into practical methodologies.

This guardrail is stated in `docs/GUARDRAIL.md` and enforced by design: there is
no scanning, targeting, or payload-firing code anywhere in the codebase.

---

## What it does

In one sentence: it turns a firehose of public security research into a
searchable, evolving, source-grounded methodology you can browse and query.

Concretely, per cycle it:

1. **Collects** from configured public sources (RSS feeds, advisories APIs,
   arXiv, GitHub advisories, NVD, optional scraped pages / YouTube transcripts).
2. **Cleans** raw HTML/PDF/transcripts into markdown and **archives** it
   durably under `knowledge/raw/YYYY/MM/`.
3. **Extracts** a structured `ResearchItem` (summary, preconditions, testing
   methodology, manual test cases, bypasses, techniques, references, confidence)
   from each raw document — via Ollama when available, a deterministic heuristic
   fallback otherwise.
4. **Reviews** each item for grounding in its source, downgrading confidence
   when claims aren't supported by the source text.
5. **Classifies** each item against a taxonomy (CWE hints + keyword scoring).
6. **Stores** metadata in SQLite and vectors in Qdrant (semantic search).
7. **Deduplicates** by exact content-hash and semantic similarity, merging
   references/examples/techniques into the kept item.
8. **Evolves** the knowledge base (the Knowledge Evolution Engine): for each new
   item, semantic-search the corpus and decide whether the technique is
   genuinely *new*, a *variation*, an *update* to an existing one, or an
   *invalidation* of an older methodology — then write/update the generalized
   technique files and log the decision.
9. **Scores** each item on novelty, exploitability, reproducibility, impact,
   frequency, and ease.
10. **Builds methodology** checklists per domain (additions only — the
    checklist grows, never shrinks).
11. **Analyzes gaps** between the taxonomy and the corpus.
12. **Summarizes** the day (and optionally the week), including evolution deltas.

---

## Architecture overview

```
collect → clean → raw markdown archive → extract (Ollama) → normalized markdown
       → review → taxonomy → store (SQLite metadata + Qdrant vectors)
       → dedup → KNOWLEDGE EVOLUTION ENGINE → score → methodology → gap → summarize
```

Three stores, each with a distinct role:

| Store | Role |
|---|---|
| **Markdown archive** (`knowledge/`) | Durable, human-readable, version-controllable **source of truth**. Everything else is derived from it. |
| **SQLite** (`data/researcher.db`) | Relational metadata: item payloads, raw items, dedup links, checklist, run log, evolution log. |
| **Qdrant** (`data/qdrant/`) | Vectors for semantic search and similarity-based dedup/evolution. Falls back to an in-memory TF-IDF store when Qdrant is unreachable. |

```
                    ┌────────── Ollama (local, all LLM tasks) ──────────┐
                    │  extraction · classification · evolution          │
                    │  summarization · gap · methodology                │
                    └───────────────────────┬──────────────────────────┘
                                            │ (heuristic fallback if offline/down)
  sources ──► collectors ──► raw archive ──► extractor ──► ResearchItem
                                                              │
                         ┌───────────────────────┬────────────┴──────────┐
                         ▼                       ▼                       ▼
                   SQLite metadata      Qdrant vectors          processed markdown
                         │                       │                + technique files
                         └──────── evolution ────┴───► methodology, gap, summaries
```

---

## Repository layout

```
/home/rohan/researcher/
  cli.py                  # Typer CLI entry point (researcher = "cli:app")
  pyproject.toml          # flat packages, deps, extras
  docker-compose.yml      # Qdrant service
  README.md               # this file

  config/                 # yaml configs + loader
    __init__.py            #   load_config(), AppConfig, SourceConfig, paths
    sources.yaml           #   all sources (enabled + disabled)
    taxonomy.yaml          #   domain → sub_category tree
    ollama.yaml            #   Ollama base_url + model-per-task
    qdrant.yaml            #   Qdrant url, collection, vector_dim, embedder kind

  collectors/             # public-source adapters
    base.py  dispatch.py   #   SourceAdapter ABC + hybrid fetch chain
    rss.py  scraper.py  cdp.py  pdf.py  github.py
    youtube.py  papers.py  conference.py  __init__.py (registry)

  extractors/             # RawItem → ResearchItem seam
    base.py  ollama_extractor.py  heuristic.py  __init__.py

  llm/                    # Ollama client + heuristic fallback + prompts
    base.py  ollama.py  heuristic.py  __init__.py

  embedder/               # pluggable embedding seam (model deferred)
    base.py  tfidf_hash.py  __init__.py

  vectorstore/            # Qdrant + in-memory TF-IDF fallback
    base.py  qdrant_store.py  tfidf_store.py  __init__.py

  knowledge/              # durable markdown archive + metadata
    schema.py  frontmatter.py  archive.py  db.py  evolution_log.py
    raw/        YYYY/MM/<slug>.md
    processed/  <domain>/<slug>.md
    techniques/ bypasses/ methodologies/ testcases/

  pipeline/               # 12-stage orchestration
    runner.py             #   run_all() + run_stage(name)
    collector.py cleaner.py extractor.py reviewer.py dedup.py taxonomy.py
    evolution.py           #   the Knowledge Evolution Engine
    scorer.py store.py methodology.py gap.py summarizer.py

  scheduler/              # in-process scheduling loop
    __init__.py  scheduler.py

  ui/                     # read-only FastAPI web app
    __init__.py  web.py  templates/*.html  static/style.css

  tests/                  # pytest suite (offline, no network)
    fixtures.py  test_schema.py  test_frontmatter.py  test_archive.py
    test_taxonomy.py  test_dedup.py  test_evolution.py
    test_embedder.py  test_pipeline_smoke.py

  data/                   # runtime (gitignored)
    researcher.db          #   SQLite metadata
    qdrant/                #   Qdrant storage volume
    output/                #   checklists, gap-report, digests, evolution.md

  docs/
    ARCHITECTURE.md  USAGE.md  SOURCES.md  GUARDRAIL.md
```

---

## Requirements

- **Python ≥ 3.10** (developed on 3.13). Use a virtualenv (Kali and other
  externally-managed environments require it).
- **Docker** — only to run Qdrant. The tool itself has no Docker dependency.
- **Ollama** — for LLM extraction/evolution. **Optional**: without it (or with
  `--offline`) the pipeline runs on the deterministic heuristic extractor.
- **Optional extras**: Chromium/Playwright (JS-rendered sources), pypdf (PDF
  sources), youtube-transcript-api (transcripts), fastapi/uvicorn/jinja2 (web
  UI), schedule (scheduler). All are pulled by `pip install -e ".[all]"`.

---

## Install

```bash
cd /home/rohan/researcher

# 1. virtualenv (required on Kali / externally-managed Pythons)
python3 -m venv .venv
.venv/bin/pip install -U pip

# 2. install the tool + every extra (collectors, UI, scheduler, tests)
.venv/bin/pip install -e ".[all]"

#    minimal instead (core only; no web UI / scheduler / cdp / pdf / yt):
#    .venv/bin/pip install -e .
```

The extras, in case you want a subset:

| Extra | Installs | Enables |
|---|---|---|
| `[cdp]` | playwright | JS-rendered sources (Chrome DevTools collector) |
| `[pdf]` | pypdf | PDF sources |
| `[yt]` | youtube-transcript-api | YouTube transcript sources |
| `[web]` | fastapi, uvicorn, jinja2 | read-only web UI (`researcher ui`) |
| `[scheduler]` | schedule | in-process scheduler (falls back to a sleep loop without it) |
| `[dev]` | pytest | test suite |
| `[all]` | all of the above | everything |

---

## Bringing up the services

### Qdrant (vectors + semantic search)

```bash
docker compose up -d qdrant        # starts on :6333 (REST) and :6334 (gRPC)
curl -s http://localhost:6333/      # health check → {"title":"qdrant ..."}
```

Qdrant is **optional at runtime**: if it isn't reachable, `researcher` falls
back to an in-memory TF-IDF store automatically (and prints a notice). To force
one or the other, use `--store qdrant|tfidf` or `--no-qdrant`.

### Ollama (LLM extraction + evolution)

```bash
ollama serve                       # if not already running
ollama pull qwen3.5:4b             # or any chat model you prefer
```

Point the tool at it via `config/ollama.yaml` or `OLLAMA_BASE_URL`. Ollama is
**optional at runtime**: with `--offline` (or if Ollama is unreachable) the
pipeline uses the heuristic extractor — everything still runs, just with
sparser fields.

### Chromium (JS-rendered sources — only if you enable `dynamic: true`)

```bash
.venv/bin/playwright install chromium
```

Only needed for sources marked `dynamic: true` (none are enabled by default).

---

## Configuration

All config lives in `config/*.yaml` and is loaded by `config.load_config()`.
You usually don't need to touch Python.

### `config/sources.yaml`

The public sources to crawl. Each entry maps to a collector by `type`:

```yaml
- id: projectzero
  type: rss                 # rss|scraper|cdp|pdf|github|youtube|papers|conference|nvd
  name: "Google Project Zero"
  url: "https://googleprojectzero.blogspot.com/feeds/posts/default"
  enabled: true
  page_size: 30             # optional, for paginated JSON APIs (NVD/GHSA)
  dynamic: false            # true => force the Playwright/CDP collector
```

Currently enabled by default (9): `portswigger`, `krebs`, `owasp`, `projectzero`,
`msrc`, `devto-sec` (RSS), `arxiv-cscr` (papers), `nvd`, `ghsa` (advisory APIs).
Disabled by default (need extra setup): `hackerone` (JS-heavy, `dynamic: true`),
`blackhat`, `defcon` (scrapers), `yt-portswigger` (YouTube).

`rate_limit_seconds` (default 2) spaces requests to a given source;
`http_timeout` (default 20) bounds each request.

### `config/taxonomy.yaml`

Domain → sub_category tree used for classification and gap analysis. The 11
domains: `Authentication`, `Authorization`, `Input Validation`, `Business Logic`,
`API`, `Cloud`, `Mobile`, `CI/CD`, `Containers`, `Kubernetes`, `Supply Chain`.

### `config/ollama.yaml`

```yaml
base_url: http://localhost:11434      # override with OLLAMA_BASE_URL
models:
  default: qwen3.5:4b
  extraction: qwen3.5:4b              # different model per task is fine
  classification: qwen3.5:4b
  summarization: qwen3.5:4b
  evolution: qwen3.5:4b
  gap: qwen3.5:4b
  methodology: qwen3.5:4b
```

You can assign a larger/smarter model to `evolution` and `extraction` and a
smaller one to `summarization` if you want.

### `config/qdrant.yaml`

```yaml
url: http://localhost:6333            # override with QDRANT_URL
collection: research_items
vector_dim: 384                       # matches a future MiniLM; tfidf_hash projects to this
distance: Cosine
embedder: tfidf_hash                  # tfidf_hash now; ollama/sentencetransformer later
```

The **embedder is a deliberately deferred decision**. `tfidf_hash` is a
dependency-free deterministic default (TF-IDF weighting + signed hashing trick
into 384 dims) so semantic search works today with no model download. To swap
to a real embedding model later, add the embedder module and change this key
(see [Upgrading](#upgrading-phase-by-phase)).

---

## Quick start

```bash
# --- fully offline: heuristic extraction, TF-IDF store, no Qdrant/Ollama ---
.venv/bin/researcher --offline run-all

# --- live: Ollama extraction + Qdrant ---
.venv/bin/researcher run-all

# --- explore the corpus ---
.venv/bin/researcher search "SSRF cloud metadata IMDS"
.venv/bin/researcher checklist --domain Cloud
.venv/bin/researcher items --domain Authentication
.venv/bin/researcher evolution            # the knowledge-evolution log
.venv/bin/researcher gap                  # coverage vs taxonomy
.venv/bin/researcher summary --weekly
.venv/bin/researcher stats

# --- read-only web UI at http://localhost:8000 ---
.venv/bin/researcher ui

# --- run autonomously ---
.venv/bin/researcher schedule --every 24h
```

---

## CLI reference

Global flags (apply to every subcommand):

| Flag | Meaning |
|---|---|
| `--offline` | Force heuristic extraction; no Ollama calls. |
| `--store auto\|qdrant\|tfidf` | Vector store selection (`auto` = Qdrant if reachable, else TF-IDF). |
| `--no-qdrant` | Force the in-memory TF-IDF store. |

Subcommands:

| Command | What it does |
|---|---|
| `run-all [--stages a,b,c]` | Run the full pipeline (or a comma-separated subset of stages). |
| `collect [--only SRC]` | Stage 1: collect from sources into the raw DB. |
| `clean` | Stage 2: clean + archive raw items to `knowledge/raw/`. |
| `extract` | Stage 3: extract ResearchItems → `knowledge/processed/` + SQLite. |
| `review` | Stage 4: grounding check; downgrade ungrounded confidence. |
| `taxonomy` | Confirm domain/sub_category against `taxonomy.yaml`. |
| `process` | Run dedup + taxonomy + evolution + score + store together. |
| `evolution` | The Knowledge Evolution Engine: reason against the corpus. |
| `score` | Compute novelty/exploitability/reproducibility/impact/frequency/ease. |
| `store` | Upsert items + vectors into Qdrant (or TF-IDF). |
| `methodology` | Build/extend checklists + methodology docs. |
| `gap` | Coverage vs taxonomy → `data/output/gap-report.md`. |
| `summary [--weekly]` | Daily (or weekly) digest → `data/output/daily-*.md`. |
| `search <query>` | Semantic search across the corpus (ranked hits). |
| `checklist [--domain D]` | Print checklist steps. |
| `items [--domain D]` | List items in the corpus. |
| `stats` | Corpus stats + provider info. |
| `evolution-log [--limit N]` | Print recent evolution decisions. |
| `schedule --every 24h [--once] [--cron]` | Run on a schedule, once, or print a crontab line. |
| `ui [--host H --port P]` | Serve the read-only web UI. |

---

## Core components

### 1. Collectors (`collectors/`)

Public-source adapters. Each subclasses `SourceAdapter` and yields `RawItem`s.

| Collector | File | What it fetches |
|---|---|---|
| RSS / Atom | `rss.py` | Generic RSS/Atom feeds (PortSwigger, Krebs, OWASP, Project Zero, MSRC, Dev.to security). |
| Scraper | `scraper.py` | httpx GET → HTML→markdown (article/conference pages). |
| CDP | `cdp.py` | Playwright headless Chromium: render JS, auto-scroll for infinite scroll, capture the rendered DOM. Used when `dynamic: true`. |
| PDF | `pdf.py` | Download PDF → text via `pypdf`. |
| GitHub advisories | `github.py` (`GHSAAdapter`) | GitHub Security Advisories REST API (references handled as str-or-dict). |
| NVD | `github.py` (`NVDAdapter`) | NVD CVE 2.0 JSON API (defaults to the last 14 days). |
| YouTube | `youtube.py` | Transcripts via `youtube-transcript-api` (no video download); timestamped markdown. |
| Papers | `papers.py` | arXiv (cs.CR) Atom API; abstract → markdown, PDF link in meta. |
| Conference | `conference.py` | Thin registry delegating to rss/scraper/cdp per site. |

**Hybrid dispatch** (`collectors/dispatch.py`): for each source, try the
declared `type` adapter first; if it yields nothing, fall back to a plain
HTTP→HTML scrape, then to CDP if `dynamic: true`. The first adapter that
returns usable content wins — resilient to a feed going stale or a page
becoming a JS app. The registry (`collectors/__init__.py`) maps `type` → adapter.

### 2. Extractors (`extractors/`)

The seam that turns a `RawItem` into a `ResearchItem` + `Technique`s.

- `ollama_extractor.py` — structured-output extraction via Ollama's JSON-schema
  mode. The extractor prompt enforces the guardrail: extract **patterns and
  testing ideas, not ready-to-fire payloads**; never invent facts not in the
  source; empty fields rather than guesses.
- `heuristic.py` — deterministic fallback (CWE hints + keyword classification +
  pattern detection). Keeps the whole pipeline runnable offline.
- `get_extractor(cfg)` returns the Ollama extractor when available; the Ollama
  extractor also degrades to heuristic automatically on any model failure, so
  the pipeline never hard-fails on a flaky model.

### 3. LLM (`llm/`)

- `ollama.py` — `OllamaClient` talks to Ollama's `/api/chat` with the `format`
  JSON-schema parameter (Ollama structured outputs). No Anthropic SDK, no cloud
  calls. `ollama_available()` is the reachability check.
- `heuristic.py` — the offline extractor + classifier.
- `__init__.py` — `get_llm(cfg, task)` returns an Ollama client or `None`
  (meaning "use heuristic"), plus the shared extraction system prompt + JSON
  schema.

### 4. Embedder (`embedder/`)

- `tfidf_hash.py` — `TfidfHashEmbedder(dim=384)`: TF-IDF weighting scattered into
  a fixed-width vector via a signed hashing trick, L2-normalized.
  Deterministic, dependency-free, no model download. The "skip embedding for
  now" seam: 384 dims so a future MiniLM embedder is a drop-in swap.
- `__init__.py` — `get_embedder(kind, dim)`. Only `tfidf_hash` is wired; `ollama`
  / `sentencetransformer` raise a clear "deferred" error until you add them.

### 5. Vector store (`vectorstore/`)

- `qdrant_store.py` — `QdrantStore` against `localhost:6333`; auto-creates the
  collection at the configured dim; `upsert`/`query`/`count`. Point ids are
  stable `uuid5` of the item id (Qdrant needs uuid/uint).
- `tfidf_store.py` — in-memory numpy cosine store used as a fallback (or when
  forced). Idempotent upserts overwrite by id.
- `__init__.py` — `get_vector_store(cfg)`: `auto` picks Qdrant if reachable and
  the client is installed, else TF-IDF (with a notice).

### 6. Knowledge layer (`knowledge/`)

- `schema.py` — `RawItem` (pre-extraction), `ResearchItem` (normalized), and
  `Technique` (generalized pattern), with `to_dict`/`from_dict` round-trips.
- `frontmatter.py` — read/write YAML front matter + markdown body (home-rolled
  on `pyyaml`; no `python-frontmatter` dep).
- `archive.py` — `write_raw()`, `write_processed()`, `read_processed()`,
  `list_processed()`, slug generation, year/month partitioning. The durable
  markdown files are the **source of truth**.
- `db.py` — `Database`: SQLite with tables for `raw_items`, `items`,
  `dedup_links`, `checklist`, `runs`, and `evolution_log`. Single persistent
  connection; generators materialize rows to avoid nested-cursor bugs.
- `evolution_log.py` — renders `data/output/evolution.md` from the log table.

### 7. Pipeline (`pipeline/`)

12 stages, each a standalone `run(cfg, db, ...)` returning a small result dict.
`runner.py` orchestrates them idempotently and threads one shared
store/embedder through the stages that need them.

| # | Stage | File | Purpose |
|---|---|---|---|
| 1 | collect | `collector.py` | Run adapters, insert raw into SQLite (hash dedup). |
| 2 | clean | `cleaner.py` | Normalize text, write `knowledge/raw/YYYY/MM/`. |
| 3 | extract | `extractor.py` | RawItem → ResearchItem → `knowledge/processed/<domain>/` + SQLite. |
| 4 | review | `reviewer.py` | Grounding check; downgrade + flag ungrounded claims. |
| 5 | taxonomy | `taxonomy.py` | Confirm domain/sub_category (CWE hints + keyword scoring). |
| 6 | store | `store.py` | Upsert SQLite row + Qdrant vector. |
| 7 | dedup | `dedup.py` | Exact-hash + semantic near-dup merge; record `dedup_links`. |
| 8 | evolution | `evolution.py` | **Knowledge Evolution Engine** (see below). |
| 9 | score | `scorer.py` | Multi-dimensional risk/quality scores. |
| 10 | methodology | `methodology.py` | Grow checklists + `knowledge/methodologies/<domain>.md`. |
| 11 | gap | `gap.py` | Coverage vs taxonomy → `gap-report.md`. |
| 12 | summary | `summarizer.py` | Daily/weekly digest, incl. evolution deltas. |

> The store runs *before* dedup/evolution so those stages can query a populated
> corpus on the first pass. Score runs *after* evolution so `novelty` (set by
> evolution) is included.

#### Knowledge Evolution Engine (`pipeline/evolution.py`)

The component that turns the system from a passive collector into a reasoning
engine. For each new item it:

1. Semantic-searches the corpus (Qdrant) for related techniques.
2. Reasons (Ollama when available; heuristic merge-by-similarity otherwise):
   - Is this technique **genuinely new**, or a **variation** of an existing one?
   - Does it **apply** to other technologies/domains?
   - Which existing **test cases** should be updated?
   - Does it **invalidate** an older methodology?
   - What **generalized pattern** emerges?
3. Writes/updates `knowledge/techniques/<slug>.md` (front matter: name, domain,
   generalized pattern, applies_to, variants, test cases, evidence sources,
   decisions, invalidates).
4. For invalidations, marks the older technique file `status: invalidated`.
5. Logs a `new`/`updated`/`variation`/`invalidated` decision to `evolution_log`
   and sets `item.scores['novelty']`.

On the first run everything is `new`; on subsequent runs the corpus grows
through `variation`/`updated`/`invalidated` decisions — methodology continuously
improves instead of just accumulating documents.

### 8. Scheduler (`scheduler/`)

`run_loop(cfg, interval_seconds, once, stages)` — uses the `schedule` package
when installed, else a `time.sleep` loop. `--once` runs a single cycle;
`--cron` prints a system crontab line for external cron.

### 9. Web UI (`ui/`)

A **read-only** FastAPI app (Jinja2 templates, no write endpoints):

| Route | Shows |
|---|---|
| `/` | Dashboard: item count, recent items, recent evolution. |
| `/items[?domain=]` | Browse all items (filterable). |
| `/item/{id}` | Single item detail. |
| `/search?q=` | Semantic search. |
| `/checklists` | Rendered checklists per domain. |
| `/reports` | Gap report + evolution log + latest daily. |
| `/healthz` | Health check. |

---

## Where data lives

| Path | What |
|---|---|
| `knowledge/raw/YYYY/MM/<slug>-<hash>.md` | Raw archived source documents (durable). |
| `knowledge/processed/<domain>/<slug>-<hash>.md` | Normalized research items with rich front matter. |
| `knowledge/techniques/<slug>.md` | Generalized techniques (written by the evolution engine). |
| `knowledge/bypasses/`, `knowledge/methodologies/`, `knowledge/testcases/` | Evolved knowledge. |
| `data/researcher.db` | SQLite relational metadata. |
| `data/qdrant/` | Qdrant vectors (Docker volume). |
| `data/output/` | Checklists, `gap-report.md`, `daily-*.md`, `weekly-*.md`, `evolution.md`. |

`data/` and `knowledge/raw/`+`knowledge/processed/` are gitignored runtime
data; `config/` and the code are the version-controllable parts.

---

## The knowledge archive (source of truth)

Every normalized item is a markdown file with YAML front matter, e.g.
`knowledge/processed/input-validation/ssrf-to-cloud-metadata-…-<hash>.md`:

```markdown
---
id: itm_...
title: "SSRF to cloud metadata 169.254.169.254"
source: ghsa
url: https://github.com/advisories/GHSA-...
category: Input Validation
sub_category: SSRF
target: aws imds
confidence: 0.8
tags: []
references:
- https://github.com/advisories/GHSA-...
techniques:
- name: Server-side request forgery
  variants:
  - IPv6 metadata access
  - X-Forwarded-For bypass
scores:
  novelty: 0.62
  overall: 0.74
---

# SSRF to cloud metadata 169.254.169.254

## Summary
SSRF reaches the cloud metadata endpoint 169.254.169.254 to steal IAM creds.

## Testing methodology
- Identify user-controlled URL parameters reaching outbound HTTP clients.

## Manual test cases
- Confirm 169.254.169.254/latest/meta-data is reachable from the SSRF sink.

## Common bypasses
- IPv6 form of 169.254.169.254
- X-Forwarded-For header spoofing

## References
- https://github.com/advisories/GHSA-...
```

Because the archive is plain markdown, you can `grep`, diff, review, and
version-control it directly — SQLite and Qdrant are derived indexes you can
rebuild from it at any time.

---

## Moving to another machine

```bash
# 1. copy the project
rsync -av researcher/ newhost:~/researcher/

# 2. on the new host
cd ~/researcher
python3 -m venv .venv
.venv/bin/pip install -U pip
.venv/bin/pip install -e ".[all]"

# 3. bring up Qdrant
docker compose up -d qdrant

# 4. bring up Ollama + a model (optional but recommended)
ollama serve
ollama pull qwen3.5:4b

# 5. optional: Chromium for JS sources
.venv/bin/playwright install chromium

# 6. verify
.venv/bin/researcher stats
.venv/bin/researcher --offline run-all     # smoke test with no services
.venv/bin/researcher run-all              # full live run
```

To **migrate the existing knowledge base** along with it, also copy `knowledge/`
and `data/researcher.db` (the markdown archive + SQLite metadata). Qdrant vectors
don't need to be copied — the `store` stage rebuilds them from the archive:

```bash
.venv/bin/researcher store      # re-index the archive into Qdrant
```

Environment overrides (handy for non-default hosts/ports):

```bash
OLLAMA_BASE_URL=http://ollama-host:11434  QDRANT_URL=http://qdrant-host:6333 \
  .venv/bin/researcher run-all
```

---

## Upgrading phase by phase

The tool is built with explicit seams so you can upgrade one piece at a time
without rewriting the rest. Suggested phases (do them in any order, one at a
time):

**Phase A — Real embeddings.** The embedding model is intentionally deferred.
Add `embedder/ollama.py` (e.g. `nomic-embed-text`) or
`embedder/sentencetransformer.py` (e.g. MiniLM), wire it in
`embedder/__init__.py`, and set `embedder: ollama|sentencetransformer` in
`config/qdrant.yaml`. The 384-dim default already matches MiniLM, so it's a
drop-in. Then re-run `researcher store` to re-index. With a dense embedder,
raise the dedup/evolution similarity thresholds in `pipeline/dedup.py` and
`pipeline/evolution.py` (they're tuned low for `tfidf_hash`).

**Phase B — Smarter per-task models.** Give `evolution` and `extraction` a
larger Ollama model in `config/ollama.yaml`, keep `summarization` small. No code
changes — the per-task `model_for(task)` already supports it.

**Phase C — More sources.** Add entries to `config/sources.yaml`. New collector
types plug in via the `collectors/__init__.py` registry. Enable the JS-heavy
ones after `playwright install chromium`.

**Phase D — Stronger review/grounding.** `pipeline/reviewer.py` is currently a
heuristic grounding check; it can be upgraded to an Ollama-based reviewer using
the same structured-output seam (`llm/__init__.py`).

**Phase E — UI / scheduler polish.** The web UI is intentionally read-only; you
can add export endpoints, per-domain views, or wire the scheduler to systemd.

Each phase is localized to one module/config pair, so you can upgrade single
single phase as required without touching the pipeline contract.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `researcher stats` shows `extraction provider: heuristic` | Ollama is offline/unreachable, or you passed `--offline`. Start `ollama serve` and `ollama pull qwen3.5:4b`. |
| `vector store: tfidf` instead of Qdrant | Qdrant isn't reachable. `docker compose up -d qdrant`; `curl localhost:6333`. Or pass `--store qdrant` to see the error. |
| `[cdp] playwright not installed` / JS sources empty | `pip install -e ".[cdp]"` then `.venv/bin/playwright install chromium`. |
| `[youtube] youtube-transcript-api not installed` | `pip install -e ".[yt]"`. |
| Web UI won't start / `pip install -e ".[web]"` | The `[web]` extra is missing. |
| Ollama extraction falls back to heuristic (limitations: "Heuristic extraction") | The model call failed or timed out. The default timeout is 600s; a 4B model doing constrained JSON can be slow under load. Retry, or use `--offline` for speed. |
| Duplicate items after re-running | Shouldn't happen — content-hash dedup is on by default. If it does, run `researcher dedup`. |
| Want a fully clean slate | `rm -f data/researcher.db && rm -rf knowledge/raw knowledge/processed knowledge/techniques` and `curl -X DELETE localhost:6333/collections/research_items`. |

See `docs/ARCHITECTURE.md`, `docs/USAGE.md`, `docs/SOURCES.md`, and
`docs/GUARDRAIL.md` for more.