# Section 14 — Ingestion Adapter Specs

## 14.1 Framing

An adapter is the single extension point. Each implements a common Protocol:

```python
class Adapter(Protocol):
    source_type: ClassVar[str]

    def can_handle(self, path: Path) -> bool: ...
    def sources(self, path: Path) -> Iterable[SourceCandidate]: ...
    def parse(self, source: SourceCandidate) -> Iterable[Chunk]: ...
    def metadata(self, source: SourceCandidate) -> dict[str, str]: ...
    def edges(self, chunks: list[Chunk]) -> Iterable[Edge]: ...
```

All adapters must: never mutate source, produce deterministic output, fail gracefully, populate mandatory CHUNK columns.

## 14.2 PDF adapter (Must-have)

### 14.2.1 Responsibilities

Ingest research-paper PDFs. Textbooks and scanned PDFs best-effort.

### 14.2.2 Source discovery

- Single file: one source.
- Directory: recursive walk, `.pdf` only.
- Files < 4KB skipped.

### 14.2.3 Parsing strategy

1. **Primary:** `pymupdf4llm.to_markdown`.
2. **Fallback:** `pypdf` text extraction if primary fails or output suspicious.
3. **Scanned detection:** mark `status='failed'` with error. No OCR in v0.1.

### 14.2.4 Section detection

- Classify headings via keyword map into: `abstract`, `introduction`, `methods`, `results`, `discussion`, `conclusion`, `references`, `other`.
- Body text inherits preceding heading's label.
- Reference sections flagged; excluded from retrieval by default.

### 14.2.5 Chunking strategy

- Target: 512 tokens. Max: 1024.
- Split on paragraph boundaries, then sentence boundaries for oversized.
- Never cross section boundaries.

### 14.2.6 Metadata extraction

**SOURCE_META:** `pdf_authors_list`, `arxiv_id`, `doi`.
**CHUNK_META:** `pdf_page`, `pdf_figure_caption`.

### 14.2.7 Edge production (v0.1)

None. `pdf_cites` deferred to v0.2.

### 14.2.8 Failure modes

| Condition | Behavior |
|---|---|
| Password-protected | Skip with error. |
| Corrupt | Skip with error. |
| Scanned | Skip with "likely scanned" error. |
| > 500MB | Skip. |
| Mixed text + scanned | Best-effort ingest. |

### 14.2.9 Concrete example

**Input** (first page):

```
                  A Hybrid Method for Clinical Negation Detection
                         Li Fu, Jane Smith, Mohammed Khan
                             University of Example, 2024

Abstract. We propose a tagged-sequence approach for negation handling
in pediatric clinical text. Our FLAN-T5 model, fine-tuned on MIMIC-IV,
achieves 0.91 F1 on the i2b2 2010 benchmark.

1. Introduction
Negation is a persistent challenge in clinical NLP pipelines...
```

**Output:**

```python
# ordinal 0 — abstract
Chunk(
  content="Abstract. We propose a tagged-sequence approach...",
  section_label="abstract",
  token_count=56,
  metadata={"pdf_page": "1"},
)

# ordinal 1 — introduction
Chunk(
  content="Negation is a persistent challenge...",
  section_label="introduction",
  token_count=33,
  metadata={"pdf_page": "1"},
)

# SOURCE
Source(
  path="~/papers/fu_2024.pdf",
  source_type="pdf",
  title="A Hybrid Method for Clinical Negation Detection",
  metadata={"pdf_authors_list": "Li Fu, Jane Smith, Mohammed Khan"},
)
```

### 14.2.10 Test fixtures required

5 arXiv papers, 1 scanned PDF, 1 password-protected, 1 non-standard headings, 1 conference paper.

### 14.2.11 Quality bar

- Section labels: ≥ 90% correct on 20-chunk sample from 10 papers.
- Title extraction: ≥ 8/10 match actual.
- Ingestion completeness: every table-of-contents section appears.

## 14.3 Claude export adapter (Must-have)

### 14.3.1 Responsibilities

Ingest Claude.ai conversation exports at turn granularity.

### 14.3.2 Export format assumptions

```json
{
  "conversations": [
    {
      "uuid": "abc-123",
      "name": "...",
      "created_at": "...",
      "chat_messages": [
        { "uuid": "msg-001", "text": "...", "sender": "human", "created_at": "..." },
        { "uuid": "msg-002", "text": "...", "sender": "assistant", "created_at": "..." }
      ]
    }
  ]
}
```

### 14.3.3 Source discovery

One `SOURCE` per conversation, path `<export>#conversations/<uuid>`. Hash over canonical JSON.

### 14.3.4 Parsing strategy

Validate shape → iterate conversations → strip messages → skip empties.

### 14.3.5 Chunking strategy

**One chunk per message.** Exceptions: oversized (>2048 tok) split on `\n\n` with `split_of` metadata.

### 14.3.6 Metadata extraction

**SOURCE_META:** `conversation_url`, `created_at`, `updated_at`, `message_count`.
**CHUNK_META:** `message_id`, `split_of`, `model_name`.

### 14.3.7 Edge production

- `conversation_next` and `conversation_prev` edges within each conversation.

### 14.3.8 Failure modes

| Condition | Behavior |
|---|---|
| JSON parse error | Abort file, continue batch. |
| Unknown format version | Abort loudly. |
| Zero-message conversation | Skip silently. |
| Missing `sender` | Default `user` with warning. |
| Missing `created_at` | Use conversation timestamp. |

### 14.3.9 Concrete example

Input (excerpt):

```json
{
  "conversations": [{
    "uuid": "8fa3-...",
    "name": "MoE quantization incompatibility",
    "chat_messages": [
      {
        "uuid": "msg-001",
        "sender": "human",
        "created_at": "2026-03-18T14:22:11Z",
        "text": "Why is BitsAndBytes not working with Baichuan-M3?"
      },
      {
        "uuid": "msg-002",
        "sender": "assistant",
        "created_at": "2026-03-18T14:22:38Z",
        "text": "Fused 3D expert tensors in MoE architectures are incompatible with BitsAndBytes, HQQ, and AWQ quantization."
      }
    ]
  }]
}
```

Output:

```python
# SOURCE
Source(
  path="~/claude-exports/2026-03.json#conversations/8fa3-...",
  source_type="claude_export",
  title="MoE quantization incompatibility",
)

# ordinal 0
Chunk(
  content="Why is BitsAndBytes not working with Baichuan-M3?",
  role="user",
  chunk_timestamp="2026-03-18T14:22:11Z",
)

# ordinal 1
Chunk(
  content="Fused 3D expert tensors in MoE architectures are incompatible...",
  role="assistant",
  chunk_timestamp="2026-03-18T14:22:38Z",
)

# Edges
Edge(source=chunk_0, target=chunk_1, type="conversation_next")
Edge(source=chunk_1, target=chunk_0, type="conversation_prev")
```

### 14.3.10 Test fixtures required

10-conversation mixed-length export, edge-case export (empty, control chars, >4k tok messages), malformed export, sanitized real export.

### 14.3.11 Quality bar

- 50-message conversation → 50 chunks with correct role/timestamps.
- Edges form complete linked lists.
- Re-ingest idempotent.
- Known-content search returns expected message in top-3.

## 14.4 Git repo adapter (Must-have)

### 14.4.1 Responsibilities

Ingest tracked files with code structure and commit metadata.

### 14.4.2 Source discovery

- One `SOURCE` per repository.
- Reject non-git directories.

### 14.4.3 Parsing strategy

- Enumerate via `git ls-files` equivalent. Skip binaries, files > 1MB, ignored directories (configurable).
- Tree-sitter-supported files → AST-aware. Others → plain text.

### 14.4.4 Chunking strategy

**Supported languages:** per top-level declaration (function, class, struct, enum, interface, trait, impl). Methods → own chunks with `scope`. Module-top grouped unless > 512 tok.

**Unsupported:** whole-file if < 1024 tok; else 512-tok windows with 64-tok overlap.

**Config files (YAML/TOML/JSON):** whole-file.

### 14.4.5 Metadata extraction

**SOURCE_META:** `repo_remote_url`, `repo_head_commit`, `repo_branch`.
**CHUNK_META:** `file_path`, `language`, `commit_hash`, `commit_author`, `commit_date`, `is_test`.

### 14.4.6 Edge production (v0.1)

None. `code_imports` deferred to v0.2.

### 14.4.7 Failure modes

| Condition | Behavior |
|---|---|
| Not a git repo | Fail fast. |
| Bare repo | Unsupported in v0.1. |
| Submodules | Ingest parent, skip submodule contents. |
| Merge-conflicted tree | Proceed with warning. |
| Detached HEAD | Proceed; `repo_branch='(detached)'`. |
| Missing grammar | Fall back to plain-text. |
| Parse error | Fall back to plain-text for that file. |
| Untracked files | Skipped in v0.1. |

### 14.4.8 Concrete example

Input (`src/adapters/pdf.py`):

```python
"""PDF ingestion adapter."""
from pathlib import Path
import pymupdf4llm

CHUNK_TARGET_TOKENS = 512

class PDFAdapter:
    source_type = "pdf"

    def can_handle(self, path: Path) -> bool:
        return path.suffix.lower() == ".pdf"

    def parse(self, source):
        text = pymupdf4llm.to_markdown(str(source.path))
        return self._chunk(text)
```

Output:

```python
# SOURCE
Source(
  path="~/code/contextd",
  source_type="git_repo",
  title="contextd",
  metadata={
    "repo_remote_url": "https://github.com/alexzhang/contextd",
    "repo_head_commit": "7a3f1e2...",
    "repo_branch": "main",
  },
)

# ordinal 142 — module-top
Chunk(
  content='"""PDF ingestion adapter."""\nfrom pathlib...',
  scope="",
  metadata={
    "file_path": "src/adapters/pdf.py",
    "language": "python",
    "commit_hash": "7a3f1e2...",
  },
)

# ordinal 143 — class
Chunk(
  content='class PDFAdapter:\n...',
  scope="PDFAdapter",
  metadata={"file_path": "src/adapters/pdf.py", "language": "python"},
)
```

### 14.4.9 Test fixtures required

Python repo (10 files), TypeScript repo (5 files), mixed-language repo, repo with `.gitignore` patterns, malformed-syntax repo.

### 14.4.10 Quality bar

- ≥ 90% of top-level declarations as distinct chunks with correct `scope`.
- `.gitignore`-excluded files → zero chunks.
- 10 random chunks have correct `commit_hash`.
- Scope unique within file (except `split_of`).
- 50k-LOC repo ingests in ≤ 3 min.

## 14.5 Markdown / Obsidian adapter (Should-have) — summary spec for v0.2

**Note: this is a design sketch for v0.2. v0.1 ships with only a minimal stub; full implementation is deferred per §16.2.**

### 14.5.1 Responsibilities

Ingest markdown directories with heading hierarchy and wikilinks.

### 14.5.2 Parsing & chunking

- Parser: `markdown-it-py` + frontmatter plugin.
- Chunking: by heading. `section_label` is heading path.
- Target: 512 tokens; split oversized on paragraphs.

### 14.5.3 Metadata

- Frontmatter → `SOURCE_META` with `md_frontmatter_` prefix.
- Heading path per chunk in `section_label`.
- File mtime in `SOURCE.source_mtime`.

### 14.5.4 Edge production

- `wikilink` edges from `[[Link]]` syntax.
- Target resolution: match filename, else deferred with `target_hint`.
- Sweep resolves deferred edges on each ingest.

### 14.5.5 Failure modes

| Condition | Behavior |
|---|---|
| Invalid YAML frontmatter | Skip frontmatter, ingest body. |
| Broken markdown | markdown-it-py is lenient. |
| Circular wikilinks | Allowed. |
| Empty file | Skipped. |

### 14.5.6 Concrete example

Input (`clinical-notes.md`):

```markdown
---
tags: [clinical, nlp, med]
project: clinical-nlp
---

# Clinical NLP Notes

## Negation handling

Comparing approaches from [[Fu 2024]] and [[Kaster 2024]].
```

Output: 1 source + 1 chunk + 2 wikilink edges (deferred).

### 14.5.7 Quality bar

- 100% of level-2 headings produce boundaries.
- ≥ 95% of wikilinks produce edges.
- Valid frontmatter extracted cleanly.

## 14.6 Cross-adapter guarantees

1. Never mutate source.
2. Deterministic output.
3. Graceful per-source failures.
4. Populate required columns.
5. Adapter-specific metadata in `CHUNK_META`/`SOURCE_META`.
6. Valid offsets.

## 14.7 Ingestion pipeline coordination

1. CLI/watcher invokes `pipeline.ingest(path, corpus)`.
2. Pipeline infers source type.
3. Adapter yields candidates.
4. For each candidate: hash check → parse → metadata → embed → write chunks + vectors → edges → commit.
5. Audit log entry.

---

# Section 15 — Retrieval Pipeline Spec

## 15.1 Framing

Six stages:

```
Preproc → Rewrite → Retrieve (hybrid) → Fuse (RRF) → Rerank → Format
```

## 15.2 Stage 1: Preprocessing

- Strip, NFC normalize, enforce length cap.
- Validate filters.
- Allocate trace_id.
- Output: `QueryRequest`.
- Budget: ≤ 5ms.

## 15.3 Stage 2: Query rewriting (optional)

### 15.3.1 Responsibilities

Expand one query into 3–5 sub-queries.

### 15.3.2 When it runs

- **Architecturally:** on by default for MCP, off for CLI.
- **v0.1 production default:** off everywhere (per §16.2). The `rewrite: true` default activates in v0.1.1 after dogfood confirmation. The spec below remains as the forward-looking design.
- Off when rewriter LLM unreachable.

### 15.3.3 Behavior

- Haiku call; JSON array of 3–5 sub-queries.
- Combined: `[original, *sub_queries]`, deduplicated.
- Hard cap: 6 total queries.

### 15.3.4 The exact prompt

**System prompt:**

```
You are a query-expansion assistant for a personal retrieval system.

Your job: given one user query, produce 3-5 alternative phrasings that
together cover the semantic territory the user might be searching for.

Good expansions vary in one of these ways:
- Synonym substitution (e.g., "LLM fine-tuning" → "language model training")
- Specificity shift (more specific OR more general than the original)
- Reframing (e.g., active → passive, noun-phrase → question)
- Related-concept framing (e.g., "negation handling" → "clinical text
  polarity detection")

Bad expansions include:
- Trivial reorderings of the same words
- Queries that drift off-topic
- Queries that are redundant with the original
- More than 5 expansions (return 3-5)

You MUST respond with only valid JSON, matching this exact schema:

{
  "sub_queries": ["<expansion 1>", "<expansion 2>", ...]
}

No prose, no markdown, no code fences. Just the JSON object.
```

**User message:** `Original query: {query}`.

**Config:** `max_tokens: 400`, `temperature: 0.4`, JSON mode.

### 15.3.5 Failure modes

| Condition | Behavior |
|---|---|
| API unreachable | Skip; use original. |
| Invalid JSON | Retry once. |
| Empty sub_queries | Use original. |
| > 5 sub-queries | Truncate. |
| Sub-query > 2000 chars | Truncate. |
| Latency > 3s | Cancel, use original. |

### 15.3.6 Output

```python
RewrittenQueries(
    original: str,
    sub_queries: list[str],
    rewriter_used: str | None,
)
```

### 15.3.7 Latency

600–1200ms typical; 3-second cancel.

## 15.4 Stage 3: Hybrid retrieval

### 15.4.1 Responsibilities

For each query, top-K from dense + sparse.

### 15.4.2 Dense retrieval

- BGE-M3 query embedding (uses query instruction).
- LanceDB ANN search, `k=50`, cosine similarity.
- Filters pre- or post-applied.

### 15.4.3 Sparse retrieval

- SQLite FTS5 BM25.
- Standard tokenization; no stop-word removal.
- SQL WHERE for filters.

### 15.4.4 Parallelism

`asyncio.gather` across all query×method combinations.

### 15.4.5 Output

```python
RetrievalCandidates(
    per_query: list[QueryResults],
)

QueryResults(
    query: str,
    dense:  list[(chunk_id: int, score: float)],
    sparse: list[(chunk_id: int, score: float)],
)
```

### 15.4.6 Latency

Dense 10–40ms/query; sparse 5–20ms/query; 40–80ms parallel for 6×2.

### 15.4.7 Directional quality expectations

- Hybrid > dense-only on code-identifier queries.
- Hybrid > sparse-only on paraphrase queries.
- Dense degrades on code identifiers; sparse degrades on paraphrases.

## 15.5 Stage 4: Reciprocal rank fusion

### 15.5.1 Algorithm

Score(c) = Σ_L (1 / (k + rank_L(c))), k=60.

### 15.5.2 Pseudocode

```python
from collections import defaultdict

def reciprocal_rank_fusion(
    result_sets: list[QueryResults],
    k: int = 60,
    top_n: int = 50,
) -> list[tuple[int, float]]:
    score_accumulator: dict[int, float] = defaultdict(float)

    for qr in result_sets:
        for rank_idx, (chunk_id, _) in enumerate(qr.dense):
            score_accumulator[chunk_id] += 1.0 / (k + rank_idx + 1)
        for rank_idx, (chunk_id, _) in enumerate(qr.sparse):
            score_accumulator[chunk_id] += 1.0 / (k + rank_idx + 1)

    ranked = sorted(
        score_accumulator.items(),
        key=lambda x: x[1],
        reverse=True,
    )
    return ranked[:top_n]
```

### 15.5.3 Why RRF

- No score calibration needed.
- Deterministic and debuggable.
- Simpler than learned fusion; no labeled data required.

### 15.5.4 Weighted RRF (v0.2)

Per-list weights as v0.2 refinement.

### 15.5.5 Output

`list[(chunk_id, rrf_score)]`, length `top_n=50`.

### 15.5.6 Latency

2–5ms.

## 15.6 Stage 5: Reranking (optional)

### 15.6.1 Responsibilities

Rescore top-50 against original user query.

### 15.6.2 When it runs

- On by default for MCP and CLI.
- Off when LLM unreachable, or `rerank: false` / `--no-rerank`.

### 15.6.3 Behavior

- Fetch full chunk content (batched).
- Single LLM call with query + 50 candidates.
- 0–10 integer scale; resort by reranker score, ties by RRF.

### 15.6.4 The exact prompt

**System prompt:**

```
You are a reranker for a personal retrieval system. You score how
relevant each candidate chunk is to the user's original query.

Your output is a JSON array of {id, score} objects, one per candidate.
Scores are integers 0-10:
- 10: chunk directly answers the query or contains the specific fact
      the user is looking for
- 7-9: chunk is strongly on-topic and provides useful context
- 4-6: chunk is tangentially related; mentions the topic but doesn't
       answer the query
- 1-3: chunk shares vocabulary but is off-topic (false semantic match)
- 0: chunk is irrelevant

Important rules:
- Score based on the ORIGINAL QUERY, not keyword overlap
- A chunk that is short and precise should be preferred over a long
  chunk that mentions the topic incidentally
- Code chunks: if the query is about behavior or intent, prefer the
  function body; if about signature or usage, prefer the signature
- PDF chunks: prefer methods/results sections over abstracts when the
  query is about how something was done
- Conversation chunks: a user message stating intent is often MORE
  useful than the assistant's response restating it

Output format (strict, no prose, no markdown):
[
  {"id": 12345, "score": 8},
  {"id": 67890, "score": 3},
  ...
]

Every id from the input MUST appear in the output exactly once.
```

**User message:**

```
Query: {original_query}

Candidates:
[id=12345] {chunk_content_truncated_to_800_tokens}
[id=67890] {chunk_content_truncated_to_800_tokens}
...
```

**Config:** `claude-haiku-4-5`, `max_tokens: 1200`, `temperature: 0.0`, JSON mode. Chunk truncation: 800 tokens from start.

### 15.6.5 Failure modes

| Condition | Behavior |
|---|---|
| API unreachable | Skip; RRF order. |
| Invalid JSON | Retry once. |
| Missing IDs | Assigned score 0. |
| Extra IDs | Ignored. |
| Latency > 5s | Cancel, RRF order. |
| Context exceeded | Halve to top-25, retry. |

### 15.6.6 Output

`list[(chunk_id, final_score)]`, length `limit` (default 10).

### 15.6.7 Latency

800–1800ms typical; 5-second cancel.

### 15.6.8 Directional expectations

- Improves top-5 precision at moderate cost to top-10 recall.
- Materially improves false-semantic-match cases.

## 15.7 Stage 6: Formatting and citation

- Batched SQL: chunks + source refs + chunk_meta + edges.
- Assemble `ChunkResult` in rank order.
- Trace object populated.
- No content truncation by default.
- Budget: 1–3ms.

## 15.8 Configuration knobs

| Knob | Default | Range | Effect |
|---|---|---|---|
| `retrieval.default_limit` | 10 | 1–100 | Final result count. |
| `retrieval.dense_top_k` | 50 | 10–200 | Dense candidates. |
| `retrieval.sparse_top_k` | 50 | 10–200 | Sparse candidates. |
| `retrieval.rrf_k` | 60 | 10–200 | RRF denominator. |
| `retrieval.rerank_top_k` | 50 | 5–100 | Rerank candidates. |
| `retrieval.rewrite_max_sub_queries` | 5 | 0–10 | Max sub-queries. |
| `retrieval.rewrite_timeout_ms` | 3000 | 500–10000 | Rewriter cancel. |
| `retrieval.rerank_timeout_ms` | 5000 | 1000–15000 | Reranker cancel. |
| `retrieval.rerank_content_truncate_tokens` | 800 | 100–4000 | Per-chunk truncation for reranker. |
| `retrieval.exclude_reference_sections` | `true` | bool | Exclude PDF references. |
| `models.rewriter` | `claude-haiku-4-5` | any | Rewriter model. |
| `models.reranker` | `claude-haiku-4-5` | any | Reranker model. |
| `models.embedder` | `BAAI/bge-m3` | any | Embedder (corpus-level). |

## 15.9 Observability

Each call: audit log with trace_id, hashed query, corpus, per-stage latency, candidate counts, model names, result count. Raw query/sub_queries at DEBUG only.

`--explain` flag prints full trace.

## 15.10 What the pipeline doesn't do

- No learned ranker (v0.1).
- No query understanding beyond rewriting.
- No result summarization.
- No personalization beyond filters.
- No cross-corpus retrieval by default.

---

# Section 16 — 2-Day Build Plan

## 16.1 Framing

Five phases, strict order. Total budget: 18 hours across 2 days.

```
Phase 1: Bootstrap      (2h) ──▶ Gate 1: repo + CI + storage live
Phase 2: Ingestion      (5h) ──▶ Gate 2: all 3 Must-have adapters work
Phase 3: Retrieval      (4h) ──▶ Gate 3: hybrid retrieval returns results
Phase 4: MCP + CLI      (4h) ──▶ Gate 4: Claude Code can search corpus
Phase 5: Polish + demo  (3h) ──▶ Gate 5: shippable v0.1
```

Each phase's deliverable is a commit on `main`.

## 16.2 Scope locked for the 2-day build

**Must-haves (all 10):** M1–M10.

**Should-haves (2 of 6):**

- **S5 Named corpora.**
- **S6 Demo video and screenshots.**

**Deliberately deferred to v0.2:** S1 Obsidian (stub only), S2 Web UI, S3 Query rewriting, S4 Watch-mode, all C-series.

**Cut from scope:** S3 query rewriting is cut from v0.1 despite being specced. The pipeline supports it architecturally; the default flips from `true` to `false` for v0.1. Re-enable in v0.1.1 after the first week of dogfooding confirms baseline retrieval quality.

## 16.3 Phase 1 — Bootstrap (2 hours)

### Tasks

1. **Repo creation (20 min).** `gh repo create`, `uv init`, TS subpackage, `.gitignore`, LICENSE, README skeleton.
2. **CI pipeline (30 min).** GitHub Actions with lint, typecheck, test, install-smoke. Matrix: Ubuntu + macOS, Python 3.12.
3. **Storage layer (60 min).** `contextd/storage/schema.py` (DDL), `contextd/storage/db.py` (WAL, FKs), `contextd/storage/vectors.py` (LanceDB wrapper). Integration test.
4. **Smoke test (10 min).** `pytest tests/test_smoke.py`.

### Exit gate

- Repo + CI green.
- Smoke test passes.
- `pipx install .` works on clean venv.

## 16.4 Phase 2 — Ingestion (5 hours)

### Tasks

1. **Adapter framework (45 min).** Protocol, registry, pipeline, embedder.
2. **PDF adapter (90 min).** Per §14.2. Fixtures + quality bar check.
3. **Claude export adapter (60 min).** Per §14.3. Fixtures + quality bar.
4. **Git adapter (90 min).** Per §14.4. Fixtures + quality bar. Dogfooding: ingest the `contextd` repo itself.
5. **CLI `ingest` (30 min).** Typer + Rich.
6. **Integration test (15 min).**

### Exit gate

- `contextd ingest ~/papers/` works for 5+ PDFs.
- `contextd ingest ~/claude-exports/my-export.json` creates sources per conversation.
- `contextd ingest ~/code/some-repo` creates function-scoped chunks.
- All adapter quality bars pass.

## 16.5 Phase 3 — Retrieval (4 hours)

### Tasks

1. **Dense search (45 min).**
2. **Sparse search (30 min).** FTS5 triggers verified.
3. **RRF fusion (20 min).** Pseudocode from §15.5.
4. **Reranker (45 min).** Anthropic API + graceful degradation.
5. **Pipeline assembly (30 min).** Trace object.
6. **CLI `query` (30 min).**
7. **Eval harness v0 (30 min).** 10 queries.

### Exit gate

- Query returns relevant chunks in < 2s (no rerank).
- Query with `--rerank` in < 4s.
- 10-query eval: ≥ 60% top-5 recall.
- `--json` matches schema.

## 16.6 Phase 4 — MCP + CLI (4 hours)

### Tasks

1. **Python HTTP backend (45 min).** FastAPI with 7 endpoints.
2. **TypeScript MCP server (90 min).** Zod schemas, 7 tools, stdio transport.
3. **CLI completion (45 min).** `list`, `forget`, `status`, `config`, `version`.
4. **Claude Code integration test (30 min).**
5. **Named corpora (30 min).** S5.
6. **Cross-AI verification (20 min).** Codex CLI with same MCP server.

### Exit gate

- `contextd mcp` runs.
- Claude Code calls all 7 tools.
- Codex CLI calls at least `search_corpus` and `fetch_chunk`.
- `--corpus` separates data.
- All 10 Must-haves meet criteria.

## 16.7 Phase 5 — Polish, eval, demo (3 hours)

### Tasks

1. **README completion (60 min).**
2. **Demo video (45 min).** 90-second script from §16.8.
3. **Full eval run (30 min).** 30-query set, target ≥ 80% Recall@5.
4. **Privacy + non-mutation CI tests (30 min).** Replace stubs with real tests.
5. **Final polish (15 min).** Help, errors, version.

### Exit gate

- README installable-from-scratch accurate.
- Demo video shows cross-AI portability.
- Recall@5 ≥ 0.80.
- Privacy and non-mutation tests pass.
- `pipx install` works on clean VM.

### Shipping decision

All pass → tag `v0.1.0`, push, publish.

Some fail → `v0.1.0-rc1` with documented issues, iterate Day 3.

## 16.8 Demo script (for YC + launch)

```
[Terminal opens in WSL2, clean directory]

$ pipx install <published-name>
  installing from PyPI...
  contextd 0.1.0 installed

$ contextd ingest ~/papers/ --type pdf --corpus research
  Ingesting 47 PDFs...
  47 sources, 1,284 chunks, 2.1GB storage

$ contextd ingest ~/claude-exports/2026-03.json --type claude_export --corpus research
  89 sources, 3,102 chunks

$ contextd query "how did Fu handle negation" --limit 3

  [1] 0.91 | fu_2024_clinical_nlp.pdf (methods, p.4)
      Fu et al. handle negation via a dedicated tag inserted into
      the input sequence before fine-tuning FLAN-T5 on...

  [2] 0.87 | claude-export.json (assistant, 2026-03-18)
      Based on your question about negation handling, the Fu
      paper uses tagged sequences while Kaster is zero-shot...

  [3] 0.72 | kaster_2024.pdf (introduction, p.2)
      [...]

[Cut to Claude Code]

> "Compare Fu and Kaster negation handling in the research corpus"

  [search_corpus via MCP → synthesized comparison with citations]

[Cut to Codex CLI]

> "What did I conclude about Fu vs Kaster in my own notes?"

  [same MCP server → same content, different format]

[Text overlay: "Same corpus. Any agent. Local-first."]
```

## 16.9 Scope-cut ladder

If behind schedule, cut in this order:

1. **Drop S6 demo video.** Screenshots only. Recovers 45 min.
2. **Drop S5 named corpora.** Single-corpus default. Recovers 30 min.
3. **Drop TypeScript MCP server.** Python-only via `mcp` Python SDK. Recovers 60 min.
4. **Drop reranking from v0.1.** RRF-only. Recovers 45 min.
5. **Drop git adapter.** PDFs + Claude exports only. Recovers 90 min.
6. **Drop PDF section detection.** Fixed windows. Recovers 30 min.

**Do not cut:** M1–M10 Must-haves.

If cumulative cuts would take the build below M1–M10, ship as `v0.1.0-rc1` with honest acknowledgment, take a third day.

## 16.10 Daily standups (with yourself)

End of Day 1 (write to `DAYLOG.md`):

1. Gates passed/failed?
2. If failed, which cut taken/planned?
3. Single riskiest thing left for Day 2?
4. On track for end-of-Day-2 shipping decision?

End of Day 2:

1. All Must-haves meet criteria?
2. How many Should-haves landed?
3. Top regret or rough edge — visible enough to hurt the demo?
4. Ship as v0.1.0 or v0.1.0-rc1?

## 16.11 What this plan explicitly does not assume

- **Not assuming Claude Code writes every line.** It helps; the plan is budgeted assuming you review and test.
- **Not assuming the first MCP integration just works.** SDK version mismatches, stdio quirks, WSL path handling are realistic risks. Cut 3 exists for this scenario.
- **Not assuming BGE-M3 loads first try.** 2GB model download can fail; pre-download during Phase 1 if bandwidth is tight.
- **Not assuming the eval corpus is representative.** 30 queries from Alex's real work; numbers may not generalize. v0.2 concern.

---

