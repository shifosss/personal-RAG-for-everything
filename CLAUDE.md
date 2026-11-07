# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`contextd` — a local-first, MCP-first personal RAG server that unifies PDFs, AI conversation exports, and git repos behind a single MCP endpoint any agent can query. Currently at `0.1.0.dev0` on branch `phase-5-polish`; all v0.1 Phases 1–5 shipped.

Key references:

- [README.md](./README.md) — pitch + competitive positioning
- [docs/USER_GUIDE.md](./docs/USER_GUIDE.md) — complete end-user walkthrough (install, ingest, query, MCP, config, troubleshooting)
- [docs/PRDs/](./docs/PRDs/) — v0.1 product spec (§13 tech stack, §14 adapters, §15 retrieval, §16 build plan)
- [docs/plans/](./docs/plans/) — per-phase implementation plans (bootstrap → polish)

The PRDs remain the source of truth for intent; when code diverges, the reason belongs in the commit message or the corresponding plan doc.

## Scope & guardrails

- **v0.1 scope is fixed by PRD §16** (Phases 1–5, Must-haves M1–M10, two Should-haves S5/S6). Out-of-scope work is deferred to v0.2+ unless explicitly reopened. Flag scope creep, don't silently accumulate it.
- **Named-corpus separation is non-negotiable.** Personal corpus and any PHIPA-regulated clinical data must never share an index or ingestion path. Default `contextd` usage here is personal-only; clinical work stays inside the institutional environment.
- **Local-first, no telemetry.** No outbound network calls by default, no analytics, no crash reporting. Network is opt-in per source (e.g., Anthropic API for reranking is gated on user config).
- **Python ↔ TypeScript boundary.** Python owns storage, embeddings, retrieval, ingestion; runs FastAPI on `127.0.0.1:8787`. TypeScript owns the MCP stdio server (`mcp-server/`) and forwards to Python over localhost HTTP via `undici`. They never import each other.

## Tech stack (pinned per PRD §13)

| Layer            | Choice                                             | Notes                                      |
| ---------------- | -------------------------------------------------- | ------------------------------------------ |
| Python           | CPython 3.12 (3.11 supported)                      | `uv` for packaging, lockfile `uv.lock`     |
| Node             | 22 LTS                                             | `pnpm` pinned via `packageManager` field   |
| Storage          | SQLite (WAL, FTS5, JSON1) + LanceDB 0.17           | data root `~/.contextd/` in prod           |
| Embeddings       | BGE-M3 via `sentence-transformers` 3.3.1           | `torch>=2.6,<2.7`, `[gpu]` extra for CUDA  |
| LLM              | `anthropic` 0.50 (`claude-haiku-4-5` default)      | graceful degradation if unreachable        |
| MCP              | `@modelcontextprotocol/sdk` 1.27.1 + `zod` ^3.25   | stdio default transport                    |
| HTTP             | `fastapi` 0.115 + `uvicorn` 0.32 + `pydantic` 2.10 |                                            |
| CLI              | `typer` 0.13 + `rich` 13.9                         |                                            |
| Lint/format (Py) | `ruff` 0.8 (format + check)                        |                                            |
| Lint/format (TS) | `biome` 1.9                                        |                                            |
| Tests            | `pytest` 8.3 + `pytest-asyncio` 0.24; `vitest` 2.1 |                                            |

Pins are authoritative in `pyproject.toml` + `uv.lock` and `mcp-server/package.json` + `pnpm-lock.yaml`. The table above is a quick reference; always check the lockfile for the exact version in use.

## Commands

```bash
# Python
uv sync                                      # install deps
uv run ruff format .                         # format
uv run ruff check .                          # lint
uv run mypy contextd/                        # type check
uv run pytest                                # unit + integration (excludes `slow` by default)
uv run pytest -m privacy                     # privacy invariants (no outbound + non-mutation)

# TypeScript (inside mcp-server/)
pnpm install
pnpm format                                  # biome format --write src/
pnpm lint                                    # biome check src/
pnpm test                                    # vitest
pnpm build                                   # emit dist/

# CLI (console script `contextd` → contextd.cli.main:app)
uv run contextd ingest ~/papers/ --corpus research
uv run contextd query "..." --corpus research --limit 5
uv run contextd serve                        # MCP stdio + HTTP on 127.0.0.1:8787
uv run contextd eval contextd/eval/seed_queries.json --corpus eval
```

Full CLI surface: `ingest`, `query`, `serve`, `list`, `forget`, `status`, `config`, `version`, `eval`.

## Local setup notes

- **Dev machine is macOS**; CI matrix targets Ubuntu 22.04/24.04, macOS 14, WSL2. Avoid macOS-only syscalls in production paths (e.g., launchd, FSEvents without Linux fallback).
- **`pnpm` and `uv` are required.** `uv` at `~/.local/bin/uv` (installed); `pnpm` via `brew install pnpm` or `npm i -g pnpm`.
- **Data root is `~/.contextd/`** by default (override with `CONTEXTD_HOME`). Per-corpus layout: `~/.contextd/corpora/<name>/` holds that corpus's SQLite DB + LanceDB dir; corpora are fully isolated on disk.
- **`.env` is auto-loaded** at CLI entry via `python-dotenv`. Put `ANTHROPIC_API_KEY` there — only needed for `--rerank`, `--rewrite`, and `eval --judge`. **Never read `.env` directly** in code or from shell (`cat`, `head`, etc.); verify via `contextd status` (shows `api_key_present`) or `dotenv_values('.env').keys()`.

## Code style

See the language rules in `~/.claude/rules/` (auto-loaded): Python follows PEP 8 + ruff; TypeScript follows biome defaults. Project-specific additions:

- **Immutability by default.** Dataclasses with `frozen=True` for DTOs, `NamedTuple` for value objects. No in-place mutation of ingestion records or retrieval results.
- **Many small files.** 200–400 lines typical, 800 max. Organize by subsystem (`contextd/storage/`, `contextd/ingest/`, `contextd/retrieve/`, `contextd/mcp/`).
- **Provenance preserved.** Every chunk carries `source_id`, `source_path`, `ingest_ts`, and a content hash (see PRD §2.8.1). Don't drop these in intermediate representations.
- **No LangChain / LlamaIndex.** Thin custom wrappers only (PRD §13.5.3, §13.11).

## Testing

- **Markers** (configured in `pyproject.toml`): `unit` (fast, isolated), `integration` (touches SQLite/LanceDB/filesystem), `privacy` (no outbound + non-mutation), `slow` (model downloads). Default `pytest` run excludes `slow`.
- **Eval harness is the ship gate**, not an afterthought: `contextd eval contextd/eval/seed_queries.json --corpus eval`. Gates: Recall@5 ≥ 0.80, Recall@10 ≥ 0.90, MRR ≥ 0.60, judge ≥ 6.5. The `eval` corpus must be pre-populated from `tests/fixtures/`.
- **TDD** for retrieval, ingestion, and storage subsystems. Regression tests accompany every bug fix — see `tests/integration/` for the established pattern.
- **Privacy suite** (`pytest -m privacy`) asserts no non-loopback sockets open during full ingest + query cycles, and that ingestion never mutates source files (sha256 before/after).
- Coverage targets for v0.1: storage 70%, adapters 60%, overall 50% (PRD §13.8.3).

## Communication

Senior-level, terse. Surface architectural/library tradeoffs when making non-obvious choices. Skip basics on Python, TS, RAG, MCP, embeddings.
