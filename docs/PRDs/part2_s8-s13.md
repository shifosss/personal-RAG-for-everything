# Section 8 — Functional Requirements

## 8.1 Framing

Each subsystem below describes *what* the system does, not *how* it does it. Implementation choices (libraries, algorithms, specific models) are in Section 13 (Tech Stack). Acceptance tests corresponding to these requirements are in Section 17. Requirements reference the MoSCoW tiering from Section 6 in brackets: [M] must-have, [S] should-have, [C] could-have.

**Note on performance numbers:** Any specific latency or throughput numbers elsewhere in this section are indicative. Section 9 (Non-Functional Requirements) is the authoritative source for all performance targets.

The subsystems form a layered architecture:

```
┌───────────────────────────────────────────────────────┐
│  Surfaces:  MCP server · CLI · Web UI                 │  §8.5
├───────────────────────────────────────────────────────┤
│  Retrieval pipeline                                   │  §8.3
├───────────────────────────────────────────────────────┤
│  Ingestion adapters                                   │  §8.2
├───────────────────────────────────────────────────────┤
│  Storage & indexing                                   │  §8.4
└───────────────────────────────────────────────────────┘
```

## 8.2 Ingestion subsystem

### 8.2.1 Responsibilities

The ingestion subsystem reads content from a user-specified source, parses it into chunks with appropriate granularity, extracts metadata, computes embeddings for each chunk, and writes everything to storage. Ingestion is explicit — the user points the tool at a source — and is idempotent — re-ingesting the same source does not duplicate chunks.

### 8.2.2 Source types

Each source type is implemented as a distinct adapter with its own parsing, chunking, and metadata extraction logic. [M] The system must support PDF documents, Claude.ai conversation exports, and local git repositories in v0.1. [S] The system should support Obsidian / markdown directories. [C] The system may support Notion exports, Gmail (via user-configured MCP), and arXiv/bioRxiv bookmark lists.

### 8.2.3 Source-specific chunking

[M] Each source type must use a chunking strategy appropriate to its structure. PDFs are chunked by detected section (abstract, introduction, methods, results, discussion, other) where detection succeeds, and by fixed token windows with heading-aware boundaries as a fallback. Claude conversations are chunked by turn (one chunk per user or assistant message), with conversation title and timestamp preserved on each chunk. Code repositories are chunked function-by-function or class-by-class where the language is supported by tree-sitter, and whole-file for unsupported languages. Markdown files are chunked by heading hierarchy, with wikilink targets preserved as edges in metadata.

### 8.2.4 Metadata extraction

[M] Every chunk must carry at minimum: source path, source type, chunk offset within source, chunk length in tokens, ingestion timestamp, and original source hash. [M] PDF chunks must additionally carry: detected title, first-page author string (best-effort), and section label. [M] Conversation chunks must additionally carry: role (user or assistant), conversation ID, conversation title, and message timestamp. [M] Code chunks must additionally carry: file path relative to repo root, commit hash at ingest time, language, and scope (function name / class name / top-level). [S] Markdown chunks must additionally carry: heading path (e.g., `#/Notes/Clinical/March`), frontmatter fields, and wikilink outbound edges.

### 8.2.5 Idempotency and incremental updates

[M] Re-ingesting an unchanged source must not produce duplicate chunks. [M] Re-ingesting a changed source must replace the prior chunks for that source with new ones and preserve consistency of embeddings, indices, and metadata. [M] Detection of "changed" is content-hash based, not timestamp based, to avoid spurious re-ingestion on filesystems with imprecise mtimes. [S] A watch mode detects new, modified, and deleted files within a configured root and applies updates within 30 seconds of the filesystem event.

### 8.2.6 Embedding computation

[M] Every chunk must be embedded with the configured embedding model at ingest time. [M] Embeddings are computed in batches for throughput and on the user's local hardware by default. [C] The user may configure an alternative embedding model via a single configuration entry; changing the model triggers a full re-embed, with user confirmation.

### 8.2.7 Error handling

[M] Ingestion failures on individual files must not abort the run. The adapter records the failure against the source path, continues with remaining files, and surfaces a summary at the end. [M] A file that fails to parse (corrupt PDF, unreadable encoding) is recorded with the error and skipped; subsequent re-ingestion can retry. [S] Ingestion is interruptible and resumable — partial progress on a multi-hundred-file run is not lost on Ctrl-C.

### 8.2.8 Non-mutation guarantee

[M] No ingestion step modifies the source file. The PDF is read, not rewritten. The git repo's working tree is not touched. The note file is not reformatted. This is a hard contract enforced by the adapter design and validated by a test that hashes every source before and after ingestion.

### 8.2.9 Privacy gates

[M] Ingestion is opt-in per source. The tool never scans the user's home directory, email, or browser history without explicit configuration. [M] Network access during ingestion is limited to the embedding model download (one-time, on first run) and any user-explicitly-configured cloud embedding service. All other ingestion is offline.

## 8.3 Retrieval subsystem

### 8.3.1 Responsibilities

The retrieval subsystem takes a natural-language or structured query and returns a ranked list of chunks with provenance, as fast as possible, with quality high enough that a calling agent can reason confidently over the results.

### 8.3.2 Hybrid retrieval

[M] Retrieval combines dense semantic search (vector similarity over embeddings) and sparse lexical search (BM25 over tokenized chunks). Results from both methods are merged via reciprocal rank fusion. [M] Hybrid retrieval must demonstrably outperform either method alone on the eval set (Section 17), under the assumption that neither method alone is sufficient across the range of query types the ICP generates.

### 8.3.3 Reranking

[M] The top candidates from hybrid retrieval are reranked by an LLM-as-reranker (Haiku-class model by default) or a cross-encoder (deferred to v0.2). The reranker receives the query and each candidate chunk and produces a relevance score. [M] The user may disable reranking via configuration for latency-sensitive or fully offline use cases.

### 8.3.4 Query rewriting

[S] The retrieval pipeline may expand a user query into 3–5 sub-queries before hybrid retrieval, via an LLM call. Sub-queries target synonymy, related concepts, and complementary framings. Results from all sub-queries are merged. [S] Query rewriting is architecturally supported from v0.1; the default-on behavior for MCP calls activates in v0.1.1 (see §16.2).

### 8.3.5 Filtering

[M] Retrieval supports filtering by source type, named corpus, and date range as query-time parameters. [S] Retrieval supports filtering by source path prefix (useful for scoping to a specific project). [C] Retrieval supports filtering by metadata key/value for any indexed field.

### 8.3.6 Result structure

[M] Every retrieval returns: ranked chunks with scores, provenance (source path, offset, hash, ingestion timestamp), and source-type-specific metadata as described in §8.2.4. [M] The default result count is 10, configurable up to 100 per query. [M] Returning zero results is a valid response; the system does not fabricate results.

### 8.3.7 Latency

[M] End-to-end retrieval latency must meet the targets in §9.2. See Section 9 for authoritative numbers; in brief, the v0.1 baselines are p95 ≤ 800 ms for hybrid-without-rerank and p95 ≤ 3 s with Haiku-class reranking, on a ≤50k-chunk corpus on the author's workstation. Any case exceeding those targets is a performance regression.

### 8.3.8 No hallucination at retrieval

[M] The retrieval subsystem returns chunks from the index. It does not synthesize, paraphrase, or summarize retrieved content. Synthesis is the calling agent's responsibility. This separation is a hard architectural contract.

## 8.4 Storage & indexing subsystem

### 8.4.1 Responsibilities

The storage subsystem persists chunks, embeddings, metadata, and indices in user-visible files on disk. It provides efficient read and write primitives to ingestion and retrieval, and supports cascade deletion.

### 8.4.2 Data layout

[M] All data lives under a user-configurable root directory (default `~/.contextd`). The layout is transparent and inspectable:

```
~/.contextd/
├── config.toml            # user configuration
├── corpora/
│   └── <corpus-name>/
│       ├── chunks.db      # SQLite: chunk rows, metadata, BM25 index
│       ├── vectors.lance  # LanceDB: embeddings with chunk IDs
│       ├── sources.db     # SQLite: ingested source registry
│       └── logs/
│           └── audit.log  # append-only ingestion/deletion log
└── cache/
    └── <model-name>/      # embedding model weights
```

### 8.4.3 Chunk storage

[M] Chunks are stored as rows in SQLite with columns for all metadata described in §8.2.4. The chunk text is stored inline. A BM25 index is maintained via SQLite's FTS5 extension for sparse retrieval. [M] Row IDs are stable across re-ingestion of the same chunk — chunks are keyed on (source path, chunk offset, content hash) so idempotency is trivial.

### 8.4.4 Embedding storage

[M] Embeddings are stored in LanceDB keyed by chunk ID. The embedding dimension and model name are recorded alongside, so a model change triggers a detectable mismatch rather than silently corrupt retrieval.

### 8.4.5 Source registry

[M] Every ingested source is registered with its path, ingest timestamp, file hash, and chunk count. The registry is the source of truth for "what is in my corpus" and drives the `contextd list` command.

### 8.4.6 Deletion semantics

[M] `contextd forget <path>` removes: all chunks for the source, all embeddings for those chunks, all BM25 index entries for those chunks, and the source registry entry. The deletion is logged to the audit log but the data is physically removed, not soft-deleted. [M] A post-deletion query for sentinel content in the removed source must return zero results.

### 8.4.7 Concurrency

[M] The storage subsystem must safely support one ingestion process running alongside one retrieval process (typical during watch mode). SQLite WAL mode and LanceDB's transactional model are relied on for this. [C] Multi-writer concurrency (two ingestions at once) is not required for v0.1.

### 8.4.8 Portability

[M] The entire `~/.contextd` directory is portable. A user may tar it, move it to a new machine, and resume operation, assuming the same embedding model is available. No absolute paths are baked into the index.

### 8.4.9 Footprint discipline

[M] Index size on disk must remain reasonable relative to source size. See §9.2.6 for the authoritative target; in brief, combined SQLite + LanceDB size must stay under 1 GB for a 100 MB source corpus in v0.1. This is a design constraint, not a guess.

## 8.5 Surface subsystem

### 8.5.1 Responsibilities

The surface subsystem exposes retrieval and ingestion to external callers: MCP-compatible agents, shell users, and humans via a web browser. All three surfaces call the same retrieval and ingestion APIs; no surface duplicates logic.

### 8.5.2 MCP server (primary surface)

[M] The tool runs as an MCP server reachable via stdio transport for local agents (Claude Code, Codex CLI) and optionally via HTTP+SSE for remote-configured agents. [M] The server exposes at minimum: a search tool, a source-fetch tool, and a corpus-listing tool. [M] Tool schemas return structured JSON with citations, not freeform text. [M] The server handles MCP protocol versioning gracefully: requests from clients using older MCP versions either work or fail with a clear error, not silently. [S] The server supports MCP elicitation (the calling agent asking follow-up questions) for ambiguous queries.

### 8.5.3 CLI (secondary surface)

[M] A `contextd` binary installable via `pipx install <published-name>` or equivalent. [M] Subcommands: `ingest`, `query`, `list`, `forget`, `status`. [S] Additional subcommands: `watch`, `serve`, `serve-ui`, `stats`, `doctor`. [M] Output formatting defaults to human-readable; a `--json` flag produces machine-parseable output for scripting. [M] A help system (`contextd --help`, `contextd <subcommand> --help`) covers every flag and subcommand.

### 8.5.4 Web UI (tertiary surface)

[S] A minimal web UI served by `contextd serve-ui` on `localhost:8787` (or a user-chosen port). [S] The UI consists of a single query box, streaming results with numbered citations, and click-through source chunks. [S] The UI binds to localhost only by default; binding to 0.0.0.0 requires explicit configuration and a warning. [M] The UI is not authenticated; exposing it to a network is the user's responsibility.

### 8.5.5 Configuration

[M] A single config file at `~/.contextd/config.toml` controls: data directory, embedding model, reranker choice, default corpus, MCP server transport and port, web UI port, and opt-in toggles for each cloud integration. [M] All configuration keys have sensible defaults; a fresh install works with no config edits. [M] Configuration is human-readable and hand-editable; no GUI-only settings.

### 8.5.6 Logging

[M] The system produces two log streams: a user-facing stream at INFO level (what was ingested, how long it took, any failures) and an audit log at the corpus level (every ingestion, deletion, and configuration change, append-only, timestamped). [M] Log files are rotated and capped so they don't fill disk. [M] No log line contains chunk content or user queries by default; both are available at DEBUG level for troubleshooting but off by default to preserve privacy.

## 8.6 Cross-cutting requirements

### 8.6.1 Determinism

[M] Retrieval over an unchanged corpus with an unchanged query returns the same ranked results across runs, given the same random seeds (where applicable). Non-determinism in the pipeline is limited to reranker LLM calls if the reranker is non-deterministic; this is documented.

### 8.6.2 Observability

[S] Every retrieval carries a trace ID propagated through query rewriting, hybrid retrieval, and reranking. The trace is available in the audit log for post-hoc debugging.

### 8.6.3 Graceful degradation

[M] If the embedding model fails to load, dense retrieval is disabled and the system falls back to sparse-only with a warning. [M] If the reranker is unavailable, retrieval returns the pre-rerank results with a warning. [M] If a corpus is corrupted, the other corpora remain usable. Failures are localized, not cascading.

### 8.6.4 Upgrade path

[S] A version bump that changes the storage schema must ship with a migration script invoked automatically on first run. The migration must be idempotent and reversible where possible. [M] Major version bumps that break the MCP tool schema are not allowed in v0.1.x — they require a v0.2 designation.

---

# Section 9 — Non-Functional Requirements

## 9.1 Framing

Non-functional requirements (NFRs) describe properties the system must exhibit across all functional behaviors — how fast, how private, how reliable, how pleasant, how portable, how cheap. Each NFR has a target, a rationale, and where useful, a verification method. Targets are chosen to be achievable in a 2-day build on the author's hardware while remaining meaningful enough that missing them would materially hurt the user experience.

Targets are tiered: **baseline** (must hold for v0.1 to ship), **stretch** (desired, achievable if time permits), and **aspirational** (v0.2+).

## 9.2 Performance

### 9.2.1 Retrieval latency

- **Baseline (v0.1):** Hybrid retrieval without reranking returns results in under 800ms at p95 on the 30-query eval set, with a corpus size of up to 50,000 chunks. Measured on the author's workstation (RTX-class GPU, WSL2, SSD).
- **Stretch (v0.1):** p95 under 500ms at the same scale.
- **Aspirational (v0.2):** p95 under 300ms at 500,000 chunks.

### 9.2.2 Retrieval with reranking

- **Baseline (v0.1):** With Haiku-class reranking over the top 50 hybrid candidates, p95 end-to-end under 3 seconds.
- **Stretch:** Under 2 seconds.
- **Aspirational (v0.2):** Local cross-encoder reranker with p95 under 1 second.

### 9.2.3 Retrieval with query rewriting

- **Baseline (v0.1):** With rewriting enabled, p95 under 4 seconds.
- **Stretch:** Under 3 seconds.

### 9.2.4 Ingestion throughput

- **Baseline (v0.1):** 50 research PDFs (≈15 MB, average 12 pages each) ingest end-to-end in under 5 minutes. One medium git repo (≤50k LOC) ingests in under 3 minutes. A full Claude.ai export (≈100 conversations, ≈5k messages) ingests in under 2 minutes.
- **Stretch:** Halve each of the above.

### 9.2.5 Memory footprint

- **Baseline (v0.1):** Resident memory during steady-state operation (MCP server idle) under 500 MB. Peak during ingestion of a 50-PDF batch under 3 GB including the loaded embedding model.
- **Stretch:** Steady-state under 250 MB.

### 9.2.6 Disk footprint

- **Baseline (v0.1):** For a 100 MB source corpus, combined index (SQLite chunks + BM25 FTS + LanceDB vectors) stays under 1 GB.
- **Stretch:** Under 600 MB for the same corpus.

### 9.2.7 Cold start

- **Baseline (v0.1):** `contextd query "..."` returns first results within 5 seconds of invocation on a cold cache (embedding model not yet loaded).
- **Stretch:** Under 3 seconds.

### 9.2.8 MCP server steady-state

- **Baseline (v0.1):** The MCP server handles 10 queries per second against the eval corpus without degrading p95 latency beyond the targets above.
- **Stretch:** 50 QPS.

## 9.3 Privacy

### 9.3.1 No outbound network by default

**Baseline (v0.1):** A fresh install with default configuration produces zero outbound network requests during ingestion, retrieval, or idle operation. The only exception is the one-time embedding model download on first run, which is clearly surfaced to the user. Verified by a CI test that runs the full lifecycle under `strace -e network` or equivalent, and by a manual `tcpdump` verification during the release checklist.

### 9.3.2 No telemetry by default

**Baseline (v0.1):** No anonymous usage reporting, no crash reporting, no phone-home of any kind ships enabled. If any analytics capability is ever added, it is off by default and requires explicit opt-in via config. The default `config.toml` shipped in the package has no analytics keys set.

### 9.3.3 Data minimization on logs

**Baseline (v0.1):** Default log level is INFO. At INFO, log lines contain: operation names, file paths being ingested, chunk counts, timings, and errors. Log lines do NOT contain: chunk content, user queries, or retrieved results. These are available at DEBUG level for troubleshooting but off by default.

### 9.3.4 Source non-mutation

**Baseline (v0.1):** No ingestion operation mutates the source file. This is §8.2.8 restated as an NFR; the target is 100% (zero tolerance). Verified by a CI test that hashes all source files before and after ingestion.

### 9.3.5 Cascading deletion

**Baseline (v0.1):** `contextd forget <path>` leaves zero traces of the source content in the index within 10 seconds. Measured by sentinel-string search on the storage files after deletion.

### 9.3.6 Egress visibility

**Stretch (v0.1):** A `contextd doctor --network` mode reports every network endpoint the current configuration will contact, before it contacts it.

## 9.4 Reliability

### 9.4.1 Crash recovery

**Baseline (v0.1):** A crash mid-ingestion leaves the storage in a consistent state — either all chunks of the partially-ingested file are committed, or none are.

### 9.4.2 Partial failure isolation

**Baseline (v0.1):** Failure on one file during a batch ingest does not abort the batch. Failure on one corpus does not corrupt or lock others. Failure of the reranker does not block retrieval.

### 9.4.3 Startup resilience

**Baseline (v0.1):** Starting the MCP server or CLI against a corrupted-but-detectable storage state produces a clear diagnostic message and does not silently return wrong results. Starting against a healthy storage state succeeds or fails within 5 seconds.

### 9.4.4 Configuration validation

**Baseline (v0.1):** Malformed `config.toml` produces a specific error identifying the offending key and line number. Missing required fields produce a clear error naming the field. Invalid values (unknown model name, unparseable port) are caught at startup.

### 9.4.5 Upgrade safety

**Baseline (v0.1):** Installing a new `contextd` version over an existing install does not corrupt the existing index. Schema migrations (if any) are idempotent and logged.
**Aspirational (v0.2):** Schema migrations are reversible; a user can downgrade to a prior version without data loss.

## 9.5 Usability

### 9.5.1 Installation

- **Baseline (v0.1):** `pipx install <published-name>` or `uv tool install <published-name>` succeeds on a fresh Ubuntu 22.04, Ubuntu 24.04, macOS 14+, and WSL2 with Ubuntu 22.04 environment. Install completes in under 3 minutes on a reasonable broadband connection.
- **Stretch (v0.1):** A one-liner `curl | sh` installer that handles Python version setup and pipx install in one step.

### 9.5.2 Time to first successful query

- **Baseline (v0.1):** A new user, following only the README, reaches a first successful retrieval in under 5 minutes.
- **Stretch:** Under 3 minutes.

### 9.5.3 Error messages

**Baseline (v0.1):** Every user-facing error message identifies: what was being attempted, what went wrong, and (where possible) what the user should do next. No stack-trace-only failures on normal error paths.

### 9.5.4 Help and documentation

- **Baseline (v0.1):** `contextd --help` and every subcommand's `--help` cover all flags with a one-line description each. README covers install, ingest, query, MCP setup for Claude Code, and forget. Known limitations are honestly listed.
- **Stretch:** Short tutorial covering the five workflow narratives from Section 7.

### 9.5.5 Output readability

**Baseline (v0.1):** CLI output on an 80-column terminal is readable without horizontal scrolling. Chunk previews truncate sensibly. Citations are visually distinct from content. Progress indicators appear for operations over 2 seconds.

### 9.5.6 Configuration discovery

**Baseline (v0.1):** `contextd config show` prints the active configuration (with any secrets redacted). `contextd config path` prints the config file's location.

## 9.6 Portability

### 9.6.1 Operating systems

- **Baseline (v0.1):** Works on Linux (Ubuntu 22.04 and 24.04), macOS 14+ (Intel and Apple Silicon), and Windows via WSL2 Ubuntu.
- **Deferred (v0.2):** Native Windows without WSL.

### 9.6.2 Python versions

**Baseline (v0.1):** Python 3.11 and 3.12 supported. 3.13 tested best-effort. 3.10 and earlier not supported.

### 9.6.3 Hardware targets

**Baseline (v0.1):** Works on CPU-only for ingestion and retrieval, with degraded ingestion throughput (roughly 3x slower than GPU-accelerated). Accelerates with any CUDA-capable GPU or Apple Silicon GPU via `mlx` or Metal Performance Shaders.

### 9.6.4 Corpus portability

**Baseline (v0.1):** The entire `~/.contextd` directory is movable across machines of the same architecture without re-ingestion, assuming the same embedding model is available.

### 9.6.5 MCP client compatibility

- **Baseline (v0.1):** Works with Claude Code, Codex CLI, and Cursor out of the box, tested against the MCP SDK versions current as of v0.1 ship date.
- **Stretch:** Works with Gemini CLI, Continue, Windsurf, and any other MCP-compliant client.

## 9.7 Cost

### 9.7.1 Zero marginal cost at idle

**Baseline (v0.1):** An idle `contextd` server costs zero dollars. No polling, no keep-alive calls, no cloud dependency in steady state.

### 9.7.2 Marginal cost per query with reranking

**Baseline (v0.1):** A query with default reranking (Haiku-class LLM, top-50 candidates) costs under $0.002 per call at current API pricing. A power user running 100 queries per day costs under $0.20/day, or under $6/month.

### 9.7.3 Cost transparency

**Baseline (v0.1):** `contextd status` prints the per-query cost estimate of the current configuration.

### 9.7.4 Option to run fully free

**Baseline (v0.1):** A fully free configuration (local embedding model, no reranker, no LLM-powered query rewriting) exists and is documented.

## 9.8 NFR summary

| Category | Baseline target | Stretch target |
|---|---|---|
| Retrieval p95 (no rerank) | ≤ 800ms @ 50k chunks | ≤ 500ms |
| Retrieval p95 (with Haiku rerank) | ≤ 3s | ≤ 2s |
| Ingestion, 50 PDFs | ≤ 5 min | ≤ 2.5 min |
| Memory, idle | ≤ 500 MB | ≤ 250 MB |
| Disk, 100 MB corpus | ≤ 1 GB | ≤ 600 MB |
| Cold start first query | ≤ 5s | ≤ 3s |
| Zero outbound network on default config | 100% | 100% |
| No telemetry by default | 100% | 100% |
| Time to first query | ≤ 5 min | ≤ 3 min |
| Cost per rerank query | ≤ $0.002 | — |

Failing a baseline target blocks the v0.1 release. Falling short on a stretch target is acceptable but logged for v0.2 prioritization.

---

# Section 10 — System Architecture

## 10.1 Framing

Three views of the same system: logical architecture (components and relationships), data flow (information movement), deployment topology (processes and coordination). A worked example then traces a retrieval request end-to-end.

## 10.2 Logical architecture

Six logical components. Arrows indicate direction of dependency, not data flow.

```
                  ┌──────────────────────┐
                  │  MCP server          │
                  │  CLI                 │
                  │  Web UI              │
                  └──────────┬───────────┘
                             │ (all three call the same API)
                             ▼
                  ┌──────────────────────┐
                  │  Retrieval engine    │
                  │  (hybrid + rerank)   │
                  └─────┬──────────┬─────┘
                        │          │
            ┌───────────▼──┐    ┌──▼────────────┐
            │  Query       │    │  Storage      │
            │  rewriter    │    │  layer        │
            └──────────────┘    └──▲─────────▲──┘
                                   │         │
                       ┌───────────┘         └──────────┐
                       │                                │
            ┌──────────┴───────┐              ┌─────────┴────────┐
            │  Ingestion       │              │  Watcher         │
            │  pipeline        │              │  (incremental)   │
            └────────┬─────────┘              └────────┬─────────┘
                     │                                 │
                     ▼                                 ▼
            ┌──────────────────┐              ┌──────────────────┐
            │  Source          │              │  Filesystem      │
            │  adapters        │              │  events          │
            │  (PDF, Claude,   │              │                  │
            │   git, md, ...)  │              │                  │
            └──────────────────┘              └──────────────────┘
```

### 10.2.1 Surface layer (MCP / CLI / UI)

Thin presentation layer. The MCP server is the primary surface. The CLI and Web UI are alternative clients of the same internal API. None contains retrieval or ingestion logic; they translate external calls into internal API calls and format responses.

### 10.2.2 Retrieval engine

Pure function over the storage layer plus configuration. Given a query and parameters, returns ranked chunks with provenance. Internally composes: query preprocessing, optional query rewriting, dense search, sparse search, reciprocal-rank-fusion merge, optional reranking, and result formatting.

### 10.2.3 Query rewriter

Optional pre-retrieval step that expands a single query into multiple sub-queries via an LLM call. Gated by configuration. Isolated as its own component so it can be replaced without touching retrieval.

### 10.2.4 Storage layer

Abstraction over the physical stores (SQLite for chunks and metadata, SQLite FTS5 for BM25, LanceDB for vectors). Exposes four primary operations: `upsert_chunks`, `delete_source`, `search_dense`, `search_sparse`, plus lookups by source path.

### 10.2.5 Ingestion pipeline

The write path. Given a source path and source type, dispatches to the appropriate source adapter, receives parsed chunks, computes embeddings, and writes everything to storage atomically per source.

### 10.2.6 Watcher

Long-running process that monitors configured paths for filesystem changes and triggers incremental re-ingestion. Stretch goal per §6.2 S4; deferrable to v0.2 without blocking v0.1.

### 10.2.7 Source adapters

One per source type. Each knows: how to read its source format, how to chunk it, and what metadata to extract. Adapters are the extension point.

## 10.3 Data flow

### 10.3.1 Ingestion flow

```
User: contextd ingest papers/ 
  → CLI dispatch
  → Source adapter (PDF, Claude, git, md)
  → Parser (extract text + structure)
  → Chunker (source-specific strategy)
  → Metadata extractor
  → Embedder (BGE-M3 or configured)
  → Storage writer (atomic per source)
      ├── SQLite: chunks + FTS5 + sources
      ├── LanceDB: vectors
      └── Audit log
```

### 10.3.2 Retrieval flow

```
Client (MCP agent / CLI / UI)
  → Surface layer (parse request)
  → Query rewriting? yes → Query rewriter (LLM expand 1→N) ──┐
                     no ──────────────────────────────────────┤
                                                              ▼
                                                      Retrieval engine
                                                              │
                                             ┌────────────────┴──────────────┐
                                             ▼                               ▼
                                     Dense search                    Sparse search
                                   (vector similarity)               (BM25 / FTS5)
                                             │                               │
                                             └──────────┬────────────────────┘
                                                        ▼
                                              Reciprocal rank fusion
                                                        │
                               ┌────────────────────────┴─────────────────────┐
                               ▼                                              ▼
                       Reranker enabled? yes → LLM reranker (Haiku)        no
                                              │                               │
                                              └──────────┬────────────────────┘
                                                         ▼
                                          Format results with citations
                                                         │
                                                         ▼
                                                  Return to client
```

### 10.3.3 Watch-mode flow

Filesystem event → Watcher process (inotify/FSEvents) → Debounce (30s quiet period) → Enqueue job to SQLite queue → Ingestion pipeline polls queue → event-type dispatch (create/modify/delete) → Storage write.

## 10.4 Deployment topology

### 10.4.1 Process inventory

Three processes in the process-per-component topology:

**Process A: MCP server (`contextd serve`).** Long-running. Handles MCP stdio or HTTP+SSE transport. Serves retrieval requests. Does not ingest.

**Process B: Watcher (`contextd watch`).** Long-running. Observes configured paths and enqueues incremental ingestion jobs. Does the actual ingestion itself. Optional.

**Process C: CLI (`contextd <subcommand>`).** Short-lived. Spawned per invocation. Does bulk ingestion, one-off queries, diagnostics, forget operations.

### 10.4.2 Shared resources

All three processes read and write the same storage directory (`~/.contextd/corpora/<n>/`). Coordination is entirely through storage — no RPC, no message bus, no shared memory.

- **SQLite:** WAL mode enables one writer plus many concurrent readers safely.
- **LanceDB:** transactional writes with MVCC semantics.
- **Model weights cache:** read-only after first download.
- **Config file:** read at process startup.

### 10.4.3 Inter-process coordination via the job queue

The watcher enqueues ingestion jobs to a SQLite-backed queue table.

```sql
CREATE TABLE ingestion_queue (
    id          INTEGER PRIMARY KEY,
    source_path TEXT NOT NULL,
    event_type  TEXT NOT NULL,  -- 'create' | 'modify' | 'delete'
    enqueued_at TIMESTAMP NOT NULL,
    started_at  TIMESTAMP,
    completed_at TIMESTAMP,
    status      TEXT NOT NULL,  -- 'pending' | 'running' | 'done' | 'failed'
    error       TEXT
);
```

### 10.4.4 Lifecycle management

**MCP server startup:** opens storage in read-only mode, loads the embedding model, verifies MCP protocol version, starts serving.

**MCP server shutdown:** flushes any pending log writes, closes storage handles, exits.

**Watcher startup:** opens storage in read-write mode, reads watched paths from config, initializes filesystem notification, rescans for missed changes, starts the event loop.

**Watcher shutdown:** SIGTERM drains the in-flight ingestion job, commits storage, exits.

**CLI invocation:** opens storage in the appropriate mode, performs the operation, commits and closes, exits.

### 10.4.5 Failure isolation

- MCP server crashes do not affect the watcher or ongoing ingestion.
- Watcher crashes do not affect retrieval.
- CLI crashes during bulk ingestion leave the storage in a consistent state.
- Storage corruption affects all three processes but is detectable at startup.

### 10.4.6 When the watcher is not running

v0.1 supports fully-manual ingestion without the watcher. The watcher is an optimization, not a requirement.

## 10.5 Worked example: end-to-end retrieval trace

A Claude Code session calls `search_corpus` with the query "compare Fu and Kaster negation handling."

**Step 1: Claude Code issues the MCP call** (JSON-RPC over stdio).

```json
{
  "jsonrpc": "2.0",
  "id": 47,
  "method": "tools/call",
  "params": {
    "name": "search_corpus",
    "arguments": {
      "query": "compare Fu and Kaster negation handling",
      "limit": 10,
      "corpus": "research"
    }
  }
}
```

**Step 2: MCP server receives and parses.** Validates schema, dispatches to retrieval engine. Budget: 1–3ms.

**Step 3: Query rewriting (if enabled).** Haiku expands to 3–5 sub-queries. Budget: 600–1200ms.

**Step 4: Embedding the query.** BGE-M3 in-memory. Budget: 30–80ms on GPU.

**Step 5: Hybrid search, per sub-query.** Dense + sparse searches in parallel. Budget: 40–80ms total for all sub-queries.

**Step 6: Reciprocal rank fusion.** Pure Python merge. Budget: 2–5ms.

**Step 7: Reranking (if enabled).** Haiku scores the top 50. Budget: 800–1800ms.

**Step 8: Format results.** Hydrate chunks with provenance. Budget: 1–3ms.

**Step 9: Send response.** JSON-RPC serialization. Budget: 1–3ms.

**Total budget:**

- With rewriting + reranking: 1500–3100ms typical.
- Without rewriting, with reranking: 900–2000ms (§9.2.2 baseline ≤ 3s).
- Without either (fully local): 50–150ms (comfortably under §9.2.1).

## 10.6 What the architecture does not include

- No message bus. SQLite-backed job queue is sufficient.
- No service mesh, no container orchestration.
- No distributed tracing.
- No horizontal scaling.
- No built-in backup/sync.

---

# Section 11 — Data Model

## 11.1 Framing

Data lives in three stores:

1. **SQLite** — chunks, sources, edges, configuration, audit log, ingestion queue. One database file per corpus.
2. **LanceDB** — embeddings, keyed by chunk ID. One LanceDB table per corpus, per embedding model.
3. **Filesystem** — raw source files (referenced by path only) and model weights cache.

Chunks are the atomic unit of retrieval. Sources are the atomic unit of ingestion, deletion, and provenance. Edges are the atomic unit of relationships between chunks.

## 11.2 Entity summary

```
CORPUS
  ├── has many SOURCE (1:N)
  ├── has many CHUNK (1:N)
  ├── has many EDGE (1:N)
  └── has many AUDIT_LOG entries (1:N)

SOURCE
  ├── has many CHUNK (1:N)
  └── has many SOURCE_META (1:N)

CHUNK
  ├── has 1 EMBEDDING (1:1)
  ├── has many CHUNK_META (1:N)
  ├── is source of many EDGE (1:N)
  └── is target of many EDGE (1:N)

INGESTION_JOB operates on SOURCE (N:1)
```

## 11.3 The `CORPUS` table

| Column | Type | Nullable | Description |
|---|---|---|---|
| `name` | TEXT PK | no | Corpus name (`research`, `personal`, `coursework`). |
| `root_path` | TEXT | yes | Optional default path. |
| `embed_model` | TEXT | no | e.g., `BAAI/bge-m3`. |
| `embed_dim` | INTEGER | no | Dimension of embedding vectors. |
| `created_at` | TIMESTAMP | no | First ingestion timestamp. |
| `schema_version` | INTEGER | no | Integer schema version for migrations. |

## 11.4 The `SOURCE` table

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | INTEGER PK | no | Auto-increment. |
| `source_type` | TEXT | no | `pdf`, `claude_export`, `git_repo`, `markdown`, `notion`, `gmail`, `arxiv_bookmark`, `web_page`. |
| `path` | TEXT | no | Canonical absolute path at ingest time. |
| `content_hash` | TEXT | no | SHA-256 of canonical source content. |
| `title` | TEXT | yes | Human-readable title. |
| `ingested_at` | TIMESTAMP | no | When ingestion completed. |
| `source_mtime` | TIMESTAMP | yes | Source file mtime. |
| `chunk_count` | INTEGER | no | Denormalized count of chunks. |
| `status` | TEXT | no | `active`, `deleted`, `failed`. |

Indices: `UNIQUE (corpus, path)`, `INDEX (source_type)`.

## 11.5 The `CHUNK` table

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | INTEGER PK | no | Auto-increment. |
| `source_id` | INTEGER FK | no | Cascade delete. |
| `ordinal` | INTEGER | no | Zero-based position within source. |
| `offset_start` | INTEGER | yes | Byte offset. |
| `offset_end` | INTEGER | yes | Byte offset. |
| `token_count` | INTEGER | no | Measured by embedder tokenizer. |
| `content` | TEXT | no | Chunk text. |
| `section_label` | TEXT | yes | For PDFs / markdown. |
| `scope` | TEXT | yes | For code (function/class name). |
| `role` | TEXT | yes | For conversations (`user` / `assistant`). |
| `chunk_timestamp` | TIMESTAMP | yes | Time associated with content. |

Indices: `INDEX (source_id, ordinal)`, `INDEX (chunk_timestamp)`.

The `CHUNK_FTS` virtual table is a SQLite FTS5 virtual table over `CHUNK.content` for BM25 search.

## 11.6 The `EDGE` table

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | INTEGER PK | no | Auto-increment. |
| `source_chunk_id` | INTEGER FK | no | Originating chunk. |
| `target_chunk_id` | INTEGER FK | yes | Destination chunk. |
| `target_hint` | TEXT | yes | For unresolved targets. |
| `edge_type` | TEXT | no | `wikilink`, `conversation_next`, `conversation_prev`, `code_imports`, `pdf_cites`, `email_reply_to`, `email_thread`. |
| `label` | TEXT | yes | Human-readable label. |
| `weight` | REAL | yes | Edge strength. |

Indices: `INDEX (source_chunk_id, edge_type)`, `INDEX (target_chunk_id, edge_type)`, `INDEX (target_hint)`.

**v0.1 edge population:**

- `wikilink`: markdown adapter. Deferred resolution supported.
- `conversation_next` / `conversation_prev`: Claude export adapter.

**v0.2+ edge population:** `code_imports`, `pdf_cites`, `email_reply_to`, `email_thread`.

## 11.7 `CHUNK_META` and `SOURCE_META`

Key-value extension tables.

`CHUNK_META`:

| Column | Type |
|---|---|
| `chunk_id` | INTEGER FK |
| `key` | TEXT |
| `value` | TEXT |

Example keys by source type:

- PDF: `pdf_page`, `pdf_figure_caption`, `arxiv_id`, `doi`.
- Code: `commit_hash`, `commit_author`, `commit_date`.
- Conversation: `conversation_url`, `model_name`, `message_id`.
- Email: `from`, `to`, `subject`, `thread_id`.

## 11.8 The `EMBEDDING` table (LanceDB)

| Column | Type | Description |
|---|---|---|
| `chunk_id` | INTEGER PK | Cross-store key. |
| `vector` | FLOAT32[embed_dim] | BGE-M3 is 1024-dim. |
| `model_name` | STRING | Redundant with `CORPUS.embed_model`. |

## 11.9 The `INGESTION_JOB` table

(See §10.4.3 for DDL.)

## 11.10 The `AUDIT_LOG` table

| Column | Type | Description |
|---|---|---|
| `id` | INTEGER PK | Auto-increment. |
| `occurred_at` | TIMESTAMP | Event time. |
| `actor` | TEXT | `cli`, `watcher`, `mcp_server`. |
| `action` | TEXT | `ingest`, `reingest`, `delete`, `config_change`, `corpus_create`, `corpus_rename`. |
| `target` | TEXT | Path, corpus name, or config key. |
| `details_json` | TEXT | Action-specific structured detail. |

Never updated or deleted.

## 11.11 JSON shape for interchange (MCP return payload)

```json
{
  "results": [
    {
      "chunk_id": 48217,
      "score": 0.874,
      "rank": 1,
      "content": "Fu et al. handle negation via a dedicated tag inserted into the input sequence...",
      "source": {
        "id": 412,
        "type": "pdf",
        "path": "~/papers/fu_2024_clinical_nlp.pdf",
        "title": "Fine-tuning FLAN-T5 for pediatric clinical note extraction",
        "ingested_at": "2026-04-02T14:22:11Z",
        "content_hash": "sha256:3a2f...ef91"
      },
      "section_label": "methods",
      "chunk_timestamp": null,
      "offset_start": 8421,
      "offset_end": 9103,
      "token_count": 186,
      "metadata": {
        "pdf_page": 4,
        "arxiv_id": "2412.09184"
      },
      "edges": [
        { "type": "pdf_cites", "target_chunk_id": 50032, "label": "Mykowiecka 2009" }
      ]
    }
  ],
  "query": {
    "original": "compare Fu and Kaster negation handling",
    "rewritten": ["...", "..."],
    "corpus": "research",
    "filters_applied": {}
  },
  "trace": {
    "trace_id": "01JH8K7Q...",
    "latency_ms": 1842,
    "dense_candidates": 50,
    "sparse_candidates": 50,
    "reranker_used": "haiku-4.5"
  }
}
```

## 11.12 Migrations and schema evolution

- v0.1 ships schema_version=1. Migrations in `migrations/NNN_description.sql`, tracked in `SCHEMA_VERSION` table.
- Additive changes preferred over destructive.
- `CHUNK_META` and `SOURCE_META` absorb schema growth without DDL.
- MCP tool schema changes: major version bump. Data schema changes: minor.

---

# Section 12 — Interface Specs

## 12.1 Framing

Three public surfaces: **MCP** (primary), **CLI** (human-facing), **HTTP** (Web UI and scripts). All three call the same internal APIs. JSON Schemas follow JSON Schema Draft 2020-12.

## 12.2 MCP tools — overview

Seven tools, grouped by intended usage.

**Search:**

- `search_corpus` — hybrid retrieval.
- `list_sources` — enumerate sources.

**Fetch & expand (chaining primitives):**

- `fetch_chunk` — retrieve by ID.
- `expand_context` — get surrounding chunks.
- `get_edges` — traverse relationships.

**Navigation:**

- `get_source` — full source registry entry.
- `list_corpora` — enumerate corpora.

Chaining model: `search_corpus` returns chunk IDs; `fetch_chunk`, `expand_context`, `get_edges` all accept chunk IDs.

## 12.3 MCP tool: `search_corpus`

### 12.3.1 Input schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "additionalProperties": false,
  "required": ["query"],
  "properties": {
    "query": {
      "type": "string",
      "minLength": 1,
      "maxLength": 2000,
      "description": "Natural language query."
    },
    "corpus": {
      "type": "string",
      "description": "Named corpus."
    },
    "limit": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "default": 10
    },
    "source_types": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": ["pdf", "claude_export", "git_repo", "markdown", "notion", "gmail", "arxiv_bookmark", "web_page"]
      }
    },
    "date_range": {
      "type": "object",
      "properties": {
        "start": { "type": "string", "format": "date-time" },
        "end":   { "type": "string", "format": "date-time" }
      }
    },
    "source_path_prefix": {
      "type": "string"
    },
    "metadata_filters": {
      "type": "object",
      "additionalProperties": { "type": "string" }
    },
    "rewrite": {
      "type": "boolean",
      "default": true,
      "description": "Enable LLM query rewriting."
    },
    "rerank": {
      "type": "boolean",
      "default": true,
      "description": "Enable LLM reranking."
    }
  }
}
```

### 12.3.2 Output schema

```json
{
  "type": "object",
  "required": ["results", "query"],
  "properties": {
    "results": {
      "type": "array",
      "items": { "$ref": "#/$defs/ChunkResult" }
    },
    "query": {
      "type": "object",
      "required": ["original", "corpus"],
      "properties": {
        "original":        { "type": "string" },
        "rewritten":       { "type": "array", "items": { "type": "string" } },
        "corpus":          { "type": "string" },
        "filters_applied": { "type": "object" }
      }
    },
    "trace": {
      "type": "object",
      "required": ["trace_id", "latency_ms"],
      "properties": {
        "trace_id":          { "type": "string" },
        "latency_ms":        { "type": "integer" },
        "dense_candidates":  { "type": "integer" },
        "sparse_candidates": { "type": "integer" },
        "reranker_used":     { "type": ["string", "null"] }
      }
    }
  },
  "$defs": {
    "ChunkResult": {
      "type": "object",
      "required": ["chunk_id", "score", "rank", "content", "source"],
      "properties": {
        "chunk_id":        { "type": "integer" },
        "score":           { "type": "number" },
        "rank":            { "type": "integer" },
        "content":         { "type": "string" },
        "section_label":   { "type": ["string", "null"] },
        "scope":           { "type": ["string", "null"] },
        "role":            { "type": ["string", "null"] },
        "chunk_timestamp": { "type": ["string", "null"], "format": "date-time" },
        "offset_start":    { "type": ["integer", "null"] },
        "offset_end":      { "type": ["integer", "null"] },
        "token_count":     { "type": "integer" },
        "source":          { "$ref": "#/$defs/SourceRef" },
        "metadata":        { "type": "object", "additionalProperties": { "type": "string" } },
        "edges":           { "type": "array", "items": { "$ref": "#/$defs/EdgeRef" } }
      }
    },
    "SourceRef": {
      "type": "object",
      "required": ["id", "type", "path", "ingested_at", "content_hash"],
      "properties": {
        "id":            { "type": "integer" },
        "type":          { "type": "string" },
        "path":          { "type": "string" },
        "title":         { "type": ["string", "null"] },
        "ingested_at":   { "type": "string", "format": "date-time" },
        "content_hash":  { "type": "string" }
      }
    },
    "EdgeRef": {
      "type": "object",
      "required": ["type", "target_chunk_id"],
      "properties": {
        "type":            { "type": "string" },
        "target_chunk_id": { "type": ["integer", "null"] },
        "target_hint":     { "type": ["string", "null"] },
        "label":           { "type": ["string", "null"] },
        "weight":          { "type": ["number", "null"] }
      }
    }
  }
}
```

### 12.3.3 Errors

- `INVALID_CORPUS` — JSON-RPC -32602.
- `EMPTY_CORPUS` — returns empty `results`.
- `REWRITER_UNAVAILABLE` / `RERANKER_UNAVAILABLE` — graceful degradation.
- `INTERNAL` — -32603.

## 12.4 MCP tool: `fetch_chunk`

### Input

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["chunk_id"],
  "properties": {
    "chunk_id": { "type": "integer" },
    "include_edges":    { "type": "boolean", "default": false },
    "include_metadata": { "type": "boolean", "default": true }
  }
}
```

### Output

```json
{
  "type": "object",
  "required": ["found"],
  "properties": {
    "found": { "type": "boolean" },
    "chunk": { "$ref": "search_corpus#/$defs/ChunkResult" }
  }
}
```

## 12.5 MCP tool: `expand_context`

### Input

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["chunk_id"],
  "properties": {
    "chunk_id": { "type": "integer" },
    "before":   { "type": "integer", "minimum": 0, "maximum": 20, "default": 2 },
    "after":    { "type": "integer", "minimum": 0, "maximum": 20, "default": 2 }
  }
}
```

### Output

```json
{
  "type": "object",
  "required": ["anchor_chunk_id", "chunks"],
  "properties": {
    "anchor_chunk_id": { "type": "integer" },
    "chunks": {
      "type": "array",
      "items": { "$ref": "search_corpus#/$defs/ChunkResult" }
    }
  }
}
```

## 12.6 MCP tool: `get_edges`

### Input

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["chunk_id"],
  "properties": {
    "chunk_id":  { "type": "integer" },
    "direction": { "type": "string", "enum": ["outbound", "inbound", "both"], "default": "both" },
    "edge_types": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": ["wikilink", "conversation_next", "conversation_prev",
                 "code_imports", "pdf_cites",
                 "email_reply_to", "email_thread"]
      }
    },
    "include_target_chunks": { "type": "boolean", "default": false },
    "limit": { "type": "integer", "minimum": 1, "maximum": 200, "default": 50 }
  }
}
```

### Output

```json
{
  "type": "object",
  "required": ["chunk_id", "edges"],
  "properties": {
    "chunk_id": { "type": "integer" },
    "edges": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["edge_id", "type", "direction"],
        "properties": {
          "edge_id":         { "type": "integer" },
          "type":            { "type": "string" },
          "direction":       { "type": "string", "enum": ["outbound", "inbound"] },
          "target_chunk_id": { "type": ["integer", "null"] },
          "target_hint":     { "type": ["string", "null"] },
          "label":           { "type": ["string", "null"] },
          "weight":          { "type": ["number", "null"] },
          "target":          { "$ref": "search_corpus#/$defs/ChunkResult" }
        }
      }
    }
  }
}
```

## 12.7 MCP tool: `list_sources`

### Input

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "corpus":         { "type": "string" },
    "source_types":   { "type": "array", "items": { "type": "string" } },
    "ingested_since": { "type": "string", "format": "date-time" },
    "limit":          { "type": "integer", "minimum": 1, "maximum": 1000, "default": 100 },
    "offset":         { "type": "integer", "minimum": 0, "default": 0 }
  }
}
```

### Output

```json
{
  "type": "object",
  "required": ["sources", "total"],
  "properties": {
    "sources": {
      "type": "array",
      "items": {
        "allOf": [
          { "$ref": "search_corpus#/$defs/SourceRef" },
          { "type": "object",
            "properties": {
              "chunk_count": { "type": "integer" },
              "status":      { "type": "string" }
            }
          }
        ]
      }
    },
    "total":    { "type": "integer" },
    "has_more": { "type": "boolean" }
  }
}
```

## 12.8 MCP tool: `get_source`

### Input

```json
{
  "type": "object",
  "additionalProperties": false,
  "oneOf": [
    { "required": ["source_id"] },
    { "required": ["path"] }
  ],
  "properties": {
    "source_id": { "type": "integer" },
    "path":      { "type": "string" },
    "corpus":    { "type": "string" }
  }
}
```

### Output

```json
{
  "type": "object",
  "required": ["found"],
  "properties": {
    "found":  { "type": "boolean" },
    "source": {
      "allOf": [
        { "$ref": "search_corpus#/$defs/SourceRef" },
        { "type": "object",
          "properties": {
            "chunk_count":  { "type": "integer" },
            "status":       { "type": "string" },
            "source_mtime": { "type": ["string", "null"], "format": "date-time" },
            "metadata":     { "type": "object", "additionalProperties": { "type": "string" } }
          }
        }
      ]
    }
  }
}
```

## 12.9 MCP tool: `list_corpora`

### Input

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {}
}
```

### Output

```json
{
  "type": "object",
  "required": ["corpora"],
  "properties": {
    "corpora": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "source_count", "chunk_count"],
        "properties": {
          "name":         { "type": "string" },
          "embed_model":  { "type": "string" },
          "embed_dim":    { "type": "integer" },
          "source_count": { "type": "integer" },
          "chunk_count":  { "type": "integer" },
          "created_at":   { "type": "string", "format": "date-time" }
        }
      }
    }
  }
}
```

## 12.10 Chaining patterns

**A. Search → broaden context:** `search_corpus → pick chunk → expand_context(before=3, after=3)`.

**B. Search → follow edges:** `search_corpus → pick chunk → get_edges(type=wikilink, include_target_chunks=true)`.

**C. Search → retrieve conversation thread:** `search_corpus → pick conversation chunk → get_edges(type=conversation_next) ×N`.

**D. Source-anchored drill-down:** `list_sources(source_types=[pdf], ingested_since=...) → get_source → search_corpus(source_path_prefix=...)`.

All patterns complete in 2–4 MCP calls.

## 12.11 CLI specification

### 12.11.1 Command inventory

| Command | MoSCoW | Purpose |
|---|---|---|
| `contextd ingest <path>` | M | Ingest. |
| `contextd query <query>` | M | Single retrieval. |
| `contextd list` | M | List sources. |
| `contextd forget <path>` | M | Remove source. |
| `contextd status` | M | Print status. |
| `contextd serve` | M | Start MCP server. |
| `contextd serve-ui` | S | Start Web UI. |
| `contextd watch <path>` | S | Run watcher. |
| `contextd stats` | S | Corpus statistics. |
| `contextd doctor` | S | Diagnostic checks. |
| `contextd config {show,path}` | M | Config introspection. |
| `contextd eval <eval.json>` | C | Evaluation harness. |
| `contextd version` | M | Print version. |

### 12.11.2 Global flags

| Flag | Default | Purpose |
|---|---|---|
| `--corpus <n>` | `default` | Select corpus. |
| `--config <path>` | `~/.contextd/config.toml` | Override config. |
| `--json` | off | Machine-readable output. |
| `--quiet` / `-q` | off | Suppress progress. |
| `--verbose` / `-v` | off | DEBUG logs. |
| `--no-color` | auto | Force-disable ANSI. |
| `--help` / `-h` | — | Help. |

### 12.11.3 `contextd ingest`

```
contextd ingest <path> [--type <source_type>] [--corpus <n>] [--force]
```

Exit codes: 0 success, 1 partial failure, 2 total failure, 3 invalid arguments.

### 12.11.4 `contextd query`

```
contextd query <query> [--limit N] [--since DATE] [--type TYPE] [--json] [--no-rewrite] [--no-rerank]
```

Default output: human-readable list with rank, source filename, score, section label, truncated content preview.

### 12.11.5 `contextd forget`

```
contextd forget <path> [--corpus <n>] [--dry-run]
```

Requires confirmation unless `--yes`.

### 12.11.6 `contextd status` output

```
contextd 0.1.0 (local-only)
Config:         ~/.contextd/config.toml
Default corpus: research
  - 412 sources, 48,217 chunks, 1.3 GB on disk
  - Embed model: BAAI/bge-m3 (loaded)
  - Reranker: claude-haiku-4-5 (API, est. $0.0016/query)
MCP server:     not running
Watcher:        not running
Network:        idle; last call: 11 min ago (reranker)
```

### 12.11.7 Exit codes

- `0` success.
- `1` partial failure / expected problem.
- `2` unexpected failure.
- `3` invalid arguments.

## 12.12 HTTP API specification

### 12.12.1 Scope

Not the primary interface. Serves the Web UI and scripted integrations.

- **Default bind:** `127.0.0.1:8787`.
- **Auth:** none.
- **CORS:** disabled.
- **Content-Type:** `application/json`.

### 12.12.2 Endpoints

| Method | Path | MCP equivalent |
|---|---|---|
| POST | `/v1/search` | `search_corpus` |
| GET | `/v1/chunks/{chunk_id}` | `fetch_chunk` |
| GET | `/v1/chunks/{chunk_id}/context` | `expand_context` |
| GET | `/v1/chunks/{chunk_id}/edges` | `get_edges` |
| GET | `/v1/sources` | `list_sources` |
| GET | `/v1/sources/{source_id}` | `get_source` |
| GET | `/v1/corpora` | `list_corpora` |
| GET | `/v1/status` | (CLI `status`) |
| GET | `/v1/healthz` | Liveness. |

### 12.12.3 Streaming

`POST /v1/search` supports `Accept: text/event-stream`. Events: `rewrite`, `candidates`, `result`, `done`.

### 12.12.4 Errors

```json
{
  "error": {
    "code": "INVALID_CORPUS",
    "message": "Corpus 'reserach' not found. Did you mean 'research'?",
    "trace_id": "01JH8K7Q..."
  }
}
```

### 12.12.5 Rate limiting

None in v0.1.

### 12.12.6 Versioning

- URL-prefixed: `/v1/*`.
- `Contextd-API-Version` response header.
- Additions don't bump; removals/renames require `/v2/`.

## 12.13 Cross-surface consistency

Every MCP tool has a CLI equivalent and an HTTP endpoint. JSON shapes identical across MCP and HTTP. CLI `--json` matches exactly.

## 12.14 Stability contract

**Stable in v0.1.x:** tool names, required fields, CLI subcommand names, HTTP paths under `/v1/`, exit codes.

**May change in v0.1.x:** new tools, new optional fields, new subcommands, human CLI formatting, log formats.

**Breaks only at v0.2.0:** removal/rename of any of the above.

---

# Section 13 — Tech Stack & Dependencies

## 13.1 Framing

Dependencies are organized by layer. Every entry has: a chosen version (pinned), a rationale, alternatives considered, and the risk of the choice. The language boundary runs between the MCP server (TypeScript) and everything it depends on (Python). The TS MCP server communicates with a local Python backend over a Unix domain socket (POSIX) or a localhost HTTP port.

## 13.2 Runtime layer

### 13.2.1 Python

- **Choice:** CPython 3.12 (baseline), 3.11 supported, 3.13 best-effort.
- **Alternatives:** PyPy (rejected — ML wheels), 3.10 (rejected — typing).

### 13.2.2 Node.js

- **Choice:** Node 22 LTS.
- **Alternatives:** Bun (rejected — MCP SDK compatibility), Deno (rejected — npm ecosystem).

### 13.2.3 Package managers

- **Python:** `uv`, pinned to `0.5.x`. Lockfile `uv.lock`.
- **Node:** `pnpm`, pinned via `packageManager` field.

### 13.2.4 Process supervisor

None in v0.1. OS service manager (systemd user, launchd, etc.) is sufficient.

## 13.3 Storage layer

### 13.3.1 SQLite

- **Pin:** `pysqlite3-binary==0.5.3`.
- **Features:** WAL mode, FTS5, JSON1, foreign keys.
- **Alternatives:** DuckDB (overkill for single-user), PostgreSQL (rejected — separate server).

### 13.3.2 LanceDB

- **Pin:** `lancedb==0.17.0`.
- **Alternatives:** Qdrant embedded (heavier), ChromaDB (weaker scaling), Faiss (no metadata), pgvector (requires Postgres), sqlite-vec (v0.2 candidate).

### 13.3.3 Filesystem layout

Data: `~/.contextd/`. Model cache: `~/.contextd/cache/models/` (respects `HF_HOME`). Backups: user responsibility.

## 13.4 Embeddings layer

### 13.4.1 Primary embedding model

- **Choice:** BGE-M3 (`BAAI/bge-m3`).
- **Pin:** `sentence-transformers==3.3.1`, `FlagEmbedding==1.3.4`.
- **Alternatives:** nomic-embed-text-v1.5 (v0.2 candidate), jina-embeddings-v3, OpenAI text-embedding-3-large (cloud-only, opt-in), voyage-3-large.

### 13.4.2 Embedding runtime

- **Pin:** `torch==2.5.1+cpu` default; `contextd[gpu]` extra for `+cu124`.
- **Alternatives:** ONNX Runtime (v0.2), MLX on Apple Silicon (deferred).

### 13.4.3 Tokenization

- **Pin:** `tokenizers==0.21.0`.

## 13.5 LLM layer (reranking, query rewriting)

### 13.5.1 Default reranker and rewriter

- **Choice:** `claude-haiku-4-5` via Anthropic SDK.
- **Pin:** `anthropic==0.50.0`.
- **Alternatives:** `gpt-4o-mini`, Gemini 2.5 Flash (via config), local Llama (v0.2).

### 13.5.2 Cross-encoder reranker (deferred to v0.2)

`BAAI/bge-reranker-v2-m3` via `FlagEmbedding`.

### 13.5.3 LLM client abstractions

Thin wrapper in `contextd.rerankers` / `contextd.rewriters`. No LangChain.

## 13.6 Retrieval layer

### 13.6.1 Hybrid fusion

Pure-Python RRF with `k=60`.

### 13.6.2 Parsing and chunking

- **PDFs:** `pymupdf4llm==0.0.17`, `pymupdf==1.25.1`. Fallback: `pypdf==5.1.0`. Note: pymupdf AGPL — see §19.3.3.
- **Code:** `tree-sitter==0.23.2` + grammars for Python, TypeScript, JavaScript, Rust, C, C++, Go, Java.
- **Markdown:** `markdown-it-py==3.0.0` + `mdit-py-plugins==0.4.2`.
- **Claude exports:** custom parser.
- **Git:** `pygit2==1.16.0`.

### 13.6.3 Tokenization for chunking budgets

Same `tokenizers` package.

### 13.6.4 Query expansion and synonyms

No static lexicon. LLM-based rewriting.

## 13.7 Surface layer

### 13.7.1 MCP server (TypeScript)

- **Runtime:** Node 22 LTS.
- **MCP SDK:** `@modelcontextprotocol/sdk`, pinned `1.27.x`.
- **Transport:** stdio default; HTTP+SSE.
- **Schema validation:** `zod==3.23.8`.

### 13.7.2 Python backend HTTP API

- **Framework:** `fastapi==0.115.4` + `uvicorn==0.32.1`.
- **Schema validation:** `pydantic==2.10.0`.

### 13.7.3 TS ↔ Python bridge

- **Default:** Unix domain socket on POSIX; localhost HTTP fallback.
- **TS client:** `undici==6.21.0`.
- **Python server:** same FastAPI instance as `/v1/*`.

### 13.7.4 CLI framework

- **Choice:** `typer==0.13.0` + `rich==13.9.4`.

### 13.7.5 Web UI

- **Pin:** `vite==5.4.11`, `react==18.3.1`.
- **Styling:** Tailwind CSS 3.4.x.
- **Streaming:** SSE via `EventSource`.

## 13.8 Dev tooling

### 13.8.1 Python lint/format

- **Pin:** `ruff==0.8.2`, format via `ruff format`.
- **Type checker:** `mypy==1.13.0` or `pyright==1.1.390`.

### 13.8.2 TypeScript lint/format

- **Pin:** `biome==1.9.4`.

### 13.8.3 Testing

- **Python:** `pytest==8.3.4`, `pytest-asyncio==0.24.0`, `pytest-benchmark==5.1.0`.
- **TypeScript:** `vitest==2.1.8`.
- **Coverage targets (v0.1):** storage 70%, adapters 60%, overall 50%.

### 13.8.4 CI

GitHub Actions. Matrix: Ubuntu 22.04/24.04, macOS 14, WSL2. Python 3.11, 3.12.

Per-push jobs: lint, type check, unit tests, install smoke, privacy test, non-mutation test.

### 13.8.5 Release

`uv build` for sdist + wheel. PyPI trusted publishing. npm for TS package.

### 13.8.6 Docs

Markdown in-repo. README primary. `docs/` for extended guides.

### 13.8.7 Observability

- **Logging:** `structlog==24.4.0` (Python), `pino` (TS).
- **Trace IDs:** `python-ulid==3.0.0`.
- **Metrics:** none in v0.1. `py-spy` ad-hoc.

## 13.9 Dependency inventory summary

**Python runtime:** `fastapi==0.115.4`, `uvicorn==0.32.1`, `pydantic==2.10.0`, `pysqlite3-binary==0.5.3`, `lancedb==0.17.0`, `sentence-transformers==3.3.1`, `FlagEmbedding==1.3.4`, `tokenizers==0.21.0`, `torch==2.5.1+cpu`, `anthropic==0.50.0`, `pymupdf4llm==0.0.17`, `pymupdf==1.25.1`, `pypdf==5.1.0`, `tree-sitter==0.23.2` + grammars, `markdown-it-py==3.0.0`, `mdit-py-plugins==0.4.2`, `pygit2==1.16.0`, `typer==0.13.0`, `rich==13.9.4`, `structlog==24.4.0`, `python-ulid==3.0.0`.

**Python dev:** `uv==0.5.5`, `ruff==0.8.2`, `mypy==1.13.0`, `pytest==8.3.4`, `pytest-asyncio==0.24.0`, `pytest-benchmark==5.1.0`, `coverage==7.6.9`.

**TypeScript runtime:** `@modelcontextprotocol/sdk@1.27.1`, `zod@3.23.8`, `undici@6.21.0`.

**TypeScript dev:** `biome@1.9.4`, `vitest@2.1.8`, `typescript@5.7.2`.

**Web UI:** `react@18.3.1`, `vite@5.4.11`, `tailwindcss@3.4.17`.

## 13.10 Dependency risk register

1. **LanceDB** — Critical path; migrate to sqlite-vec or pgvector if maintenance stalls.
2. **BGE-M3** — Replaceable; requires full re-embed.
3. **MCP TypeScript SDK** — Breaking changes controlled via pin.
4. **pymupdf4llm** — AGPL underlying; `pypdf` fallback exists.
5. **Anthropic API availability** — Graceful degradation if unreachable.

## 13.11 "Why not X?" footnotes

- **LlamaIndex / Haystack:** framework overhead doesn't pay off for custom chunking.
- **Vercel / Next.js for UI:** overkill for localhost single-user.
- **Single language:** MCP ecosystem is TS-centric; ML ecosystem is Python-centric.
- **SQLite alone for vectors:** sqlite-vec is tempting; reconsider for v0.2.
- **Pinning every dep:** supply-chain safety > maintenance cost.

---

