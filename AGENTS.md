# AGENTS.md — graphify

This file provides guidance to AI coding agents working with the graphify codebase.

## Commands

- **Run tests**: `uv run pytest tests/ -q` (full suite), `uv run pytest tests/test_extract.py -q` (single module), `uv run pytest tests/ -q -k "python"` (filter by name)
- **Run lint**: `uv run ruff check graphify/` (configured in pyproject.toml — selects E9/F63/F7/F82 only)
- **Type check**: `uv run pyright` (basic mode, `pyproject.toml` config at `[tool.pyright]`)
- **Security audit**: `uv run bandit -r graphify/` (skips B404 per pyproject.toml)
- **Test coverage**: `uv run pytest tests/ --cov=graphify --cov-report=term`
- **Build**: `uv build` (package: `graphifyy`)
- **Install dev environment**: `uv sync --all-extras` (from project root, installs graphify + all extras + dev deps)
- **Run CLI**: `uv run graphify .` (build graph for current folder)
- **Update graph (AST-only, free)**: `uv run graphify update .`

## Project summary

**graphify** turns any folder of code, docs, papers, images, or videos into a queryable knowledge graph. It ships as a `graphifyy` PyPI package with a `graphify` CLI. The primary user path is via AI coding assistants (Claude Code, Codex, Cursor, Gemini CLI, Cursor, Aider, etc.) where the `/graphify` skill orchestrates the pipeline; the CLI is also usable standalone.

## High-level architecture

### Pipeline (six stages, no shared state)

```
detect()  →  extract()  →  build_graph()  →  cluster()  →  analyze()  →  report()  →  export()
```

Each stage is a single function in its own module under `graphify/`. They communicate through plain Python dicts and NetworkX graphs — no shared state, no side effects outside `graphify-out/`.

### Module map

| Module | Responsibility |
|---|---|
| `__main__.py` | CLI entry point — install, uninstall, extract, serve, hook, prs, provider, global, clone, merge-graphs, label, and all per-platform subcommands |
| `detect.py` | File discovery, classification (CODE/DOCUMENT/PAPER/IMAGE/VIDEO), `.graphifyignore`/`.graphifyinclude` parsing, corpus health checks, incremental change detection via `manifest.json` |
| `extract.py` | Tree-sitter based AST extraction for 33+ languages. Each `extract_<lang>()` returns `{nodes, edges}` dict. Also handles office documents, PDFs, and URLs |
| `build.py` | Assembles extraction dicts into an `nx.Graph`/`nx.DiGraph`. Three-layer dedup: within-file → across-files → semantic merge. `build_merge()` loads existing graph and merges incrementally |
| `cluster.py` | Community detection via Leiden (graspologic, preferred) or Louvain (networkx fallback). Splits oversized communities. Returns `{community_id: [node_ids]}` with stable IDs across runs |
| `analyze.py` | God nodes (most connected), surprising connections (cross-community/language), suggested questions, import cycle detection |
| `report.py` | Renders `GRAPH_REPORT.md` — corpus stats, confidence breakdown, community summaries, god nodes, surprises, import cycles |
| `export.py` | Writes `graph.json`, `graph.html` (interactive D3), Obsidian vault, GraphML, SVG, Neo4j Cypher. Handles backup of semantic artifacts |
| `llm.py` | LLM backends for semantic extraction (Claude, Gemini, OpenAI, Ollama, Kimi, DeepSeek, Bedrock, custom providers via `provider registry`). `detect_backend()` auto-picks from env vars |
| `security.py` | URL validation (SSRF protection via IP range blocking + DNS rebinding), safe fetch with streaming size caps, graph path confinement, label sanitization |
| `validate.py` | Schema enforcement for extraction results — verifies required fields (`id`, `label`, `file_type`, `source_file` for nodes; `source`, `target`, `relation`, `confidence` for edges) |
| `dedup.py` | Entity deduplication pipeline: normalization → entropy gate → MinHash/LSH blocking → Jaro-Winkler verification → union-find merge |
| `cache.py` | Per-file extraction cache with stat-based index (size + mtime + MD5). Configurable output dir via `GRAPHIFY_OUT` env var |
| `symbol_resolution.py` | Deterministic cross-file symbol resolution (Python imports, function calls). Builds import index for conservative call-graph edge generation |
| `serve.py` | MCP stdio server — exposes `query_graph`, `get_node`, `get_neighbors`, `shortest_path`, `list_prs`, `get_pr_impact`, `triage_prs` tools |
| `watch.py` | Filesystem watcher — flags directories for auto-rebuild on change |
| `prs.py` | PR dashboard — CI state, review status, graph-impact analysis, Opus-powered triage, merge-conflict detection |
| `ingest.py` | URL ingestion (arxiv papers, web pages, YouTube videos) — fetches, converts to annotated markdown |
| `global_graph.py` | Cross-project global graph — register/remove/list project graphs, merge into `~/.graphify/global-graph.json` |
| `wiki.py` | Generates an agent-crawlable Markdown wiki from the graph (index + per-community articles) |
| `hooks.py` | Git post-commit and post-checkout hook management |
| `callflow_html.py` | Generates Mermaid architecture/call-flow HTML pages from the graph |
| `always_on/` | Platform-specific instruction blocks injected during `graphify install` (CLAUDE.md, AGENTS.md, GEMINI.md, VS Code, Antigravity rules, Kiro steering) |
| `skills/` | Generated per-platform `SKILL.md` files for all supported AI coding assistants |

### Extraction output schema

Every extractor returns:

```
{
  "nodes": [{"id": str, "label": str, "source_file": str, "file_type": str, "source_location": str|None}],
  "edges": [{"source": str, "target": str, "relation": str, "confidence": "EXTRACTED|INFERRED|AMBIGUOUS", "source_file": str}]
}
```

### Three extraction passes

1. **AST pass** (free, local) — tree-sitter parses code files for classes, functions, imports, call graphs. No API calls.
2. **Video/audio pass** (local) — faster-whisper transcribes media files.
3. **Semantic pass** (costs tokens) — LLM subagents process docs, PDFs, images, and transcripts in parallel.

### Key design decisions

- **Deterministic output**: `detect()` sorts files lexicographically, community IDs use total-order tiebreaking, edge iteration is sorted — `graph.json` is reproducible across runs
- **Confidence labels**: every edge has `EXTRACTED`/`INFERRED`/`AMBIGUOUS` — the consumer always knows what was found vs guessed
- **Security-first**: all external input (URLs, file paths, labels) goes through `security.py` — SSRF protection, zip-bomb detection for office files, sensitive file exclusion, graph path confinement
- **Incremental builds**: `manifest.json` tracks per-file mtime + content hashes (AST and semantic separately). `detect_incremental()` returns only changed files. `build_merge()` grows the graph without replacement
- **Lazy imports**: `__init__.py` uses `__getattr__` with deferred imports so `graphify install` works before heavy deps (tree-sitter, networkx) are installed

## Graph

The project has its own knowledge graph at `graphify-out/` (if built). Always read `graphify-out/GRAPH_REPORT.md` for god nodes and community structure before answering architecture questions. If `graphify-out/wiki/index.md` exists, prefer navigating it over reading raw files. After modifying code files, run `graphify update .` to keep the graph current (AST-only, no API cost).

## Test layout

One test file per module under `tests/`. Fixture files live in `tests/fixtures/`. All tests are pure unit tests — no network calls, no filesystem side effects outside `tmp_path`.
