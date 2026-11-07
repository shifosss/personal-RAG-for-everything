# Phase 5 — Polish, Eval, Privacy CI, Demo, Release Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans`.

**Goal:** Harden v0.1 for release — grow the eval set from 10 to 30 queries and clear the Recall@5 ≥ 0.80 gate (M4), add privacy + non-mutation CI, land the 90-second demo (S6), polish the README, and tag `v0.1.0`.

**Architecture:** No new subsystems. This phase adds two pytest suites marked `privacy` that run in CI and enforce the two invariants that make `contextd` trustworthy (zero outbound network by default; zero source-file mutation during ingest). It extends the eval harness with LLM-as-judge scoring for aggregate quality. README + demo are content work.

**Tech Stack:** Existing stack only. Demo recorded with whatever AlexZ prefers (QuickTime / OBS / asciinema → mp4). Screenshots via macOS screenshot tool. No new dependencies.

**Prereqs:** Phase 4 complete; all 10 Must-haves green; `ANTHROPIC_API_KEY` available for LLM-as-judge scoring (CI job will skip judge if missing and fall back to Recall@5 + MRR only).

**Exit gate (PRD §16.7):**
- README walks a new user from `pipx install <published-name>` → first successful query in under 5 minutes (manually verified on a clean VM or container)
- 90-second demo video recorded, checked in at `docs/demo/v0.1.mp4` (or linked externally; either way the README embeds it)
- 30-query eval: **Recall@5 ≥ 0.80**, Recall@10 ≥ 0.90, MRR ≥ 0.60, LLM-judge aggregate ≥ 6.5 — all passing (M4 final)
- Privacy CI: `tests/privacy/test_no_outbound_network.py` and `tests/privacy/test_non_mutation.py` green
- Release: `git tag v0.1.0` pushed; `uv build` produces sdist + wheel; optionally published to PyPI via trusted publishing

---

## File Structure

Create:
- `contextd/eval/judge.py` — LLM-as-judge scorer
- `contextd/eval/seed_queries.json` — grow to 30 entries
- `contextd/eval/run.py` — one-command harness runner (`uv run python -m contextd.eval.run`)
- `contextd/cli/commands/eval.py` — `contextd eval` subcommand (PRD C-tier; tiny wrapper)
- `tests/privacy/__init__.py`
- `tests/privacy/test_no_outbound_network.py`
- `tests/privacy/test_non_mutation.py`
- `tests/privacy/test_no_content_at_info.py`
- `docs/demo/v0.1-script.md`
- `docs/demo/v0.1.mp4` (or a URL in README if hosted)
- `README.md` — full rewrite
- `.github/workflows/ci.yml` — add a `privacy` and `eval` job

---

## Task 1: Grow eval seed set to 30 queries

**Files:**
- Modify: `contextd/eval/seed_queries.json`

- [ ] **Step 1:** Curate 30 queries sourced from AlexZ's real work (research papers, past Claude conversations, code). Each entry:

```json
{
  "id": "q01",
  "query": "how does Fu et al. handle negation",
  "corpus": "research",
  "expected_keywords": ["negation", "tag", "sequence"],
  "expected_source_types": ["pdf", "claude_export"],
  "tags": ["paraphrase", "cross_source"]
}
```

Distribute across three tag dimensions (mirrors PRD §15.4 directional quality claims):
- 10 `direct` (keyword queries — sparse-friendly)
- 10 `paraphrase` (semantic queries — dense-friendly)
- 10 `code_identifier` (exact function/class names — sparse-critical)

All 30 must exist in AlexZ's real corpus. Do not use synthetic data.

- [ ] **Step 2: Commit**

```bash
git add contextd/eval/seed_queries.json
git commit -m "eval: grow seed set to 30 queries (10 direct / 10 paraphrase / 10 code)"
```

---

## Task 2: LLM-as-judge scorer

**Files:**
- Create: `contextd/eval/judge.py`
- Test: `tests/integration/eval/test_judge.py`

- [ ] **Step 1: Test with monkey-patched Anthropic client**

```python
# tests/integration/eval/test_judge.py
import pytest
from contextd.eval.judge import judge_result

pytestmark = pytest.mark.integration


async def test_judge_returns_0_to_10_integer(monkeypatch):
    class FakeMessages:
        def create(self, **kw):
            class R: content = [type("b", (), {"text": '{"score": 7, "rationale": "on topic"}'})()]
            return R()
    class FakeClient: messages = FakeMessages()
    monkeypatch.setattr("contextd.eval.judge._anthropic_client", lambda: FakeClient())
    score = await judge_result(query="q", result_text="...")
    assert score == 7


async def test_judge_skips_when_api_unavailable(monkeypatch):
    class FakeMessages:
        def create(self, **kw): raise ConnectionError("down")
    class FakeClient: messages = FakeMessages()
    monkeypatch.setattr("contextd.eval.judge._anthropic_client", lambda: FakeClient())
    score = await judge_result(query="q", result_text="...")
    assert score is None
```

- [ ] **Step 2-3: Implement + run**

```python
# contextd/eval/judge.py
from __future__ import annotations
import asyncio
import json
import os
from functools import lru_cache
from anthropic import Anthropic

_SYS = (
    "You are a retrieval-quality judge. Score how well the retrieved text answers the query. "
    "Scores 0-10: 10 = directly answers, 7-9 strongly on-topic, 4-6 tangential, 1-3 keyword overlap only, 0 irrelevant. "
    'Reply ONLY with JSON: {"score": <int>, "rationale": "<short>"}'
)


@lru_cache(maxsize=1)
def _anthropic_client() -> Anthropic:
    return Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))


async def judge_result(*, query: str, result_text: str) -> int | None:
    try:
        res = await asyncio.wait_for(
            asyncio.to_thread(
                _anthropic_client().messages.create,
                model="claude-haiku-4-5", max_tokens=200, temperature=0.0,
                system=_SYS,
                messages=[{"role": "user", "content": f"Query: {query}\n\nRetrieved:\n{result_text[:2000]}"}],
            ),
            timeout=10.0,
        )
        data = json.loads(res.content[0].text.strip())
        return int(data.get("score", 0))
    except Exception:
        return None
```

- [ ] **Step 4: Commit**

```bash
git add contextd/eval/judge.py tests/integration/eval/test_judge.py
git commit -m "feat(eval): LLM-as-judge scorer with graceful skip"
```

---

## Task 3: Full eval runner

**Files:**
- Create: `contextd/eval/run.py`
- Create: `contextd/cli/commands/eval.py`
- Test: `tests/integration/eval/test_run.py`

- [ ] **Step 1: Implement runner with Recall@k + MRR + judge aggregate**

```python
# contextd/eval/run.py
from __future__ import annotations
import asyncio
import json
import statistics
from dataclasses import asdict, dataclass
from pathlib import Path
from contextd.eval.judge import judge_result
from contextd.retrieve.pipeline import retrieve
from contextd.retrieve.preprocess import build_request


@dataclass(frozen=True)
class EvalReport:
    n_queries: int
    recall_at_5: float
    recall_at_10: float
    mrr: float
    judge_mean: float | None
    per_tag: dict[str, dict[str, float]]
    gate_passed: bool


async def run(seed_path: Path, corpus: str, rerank: bool = True, judge: bool = True) -> EvalReport:
    queries = json.loads(seed_path.read_text())
    per_tag_hits: dict[str, list[int]] = {}
    r5 = r10 = mrr_sum = 0
    judge_scores: list[int] = []

    for q in queries:
        req = build_request(query=q["query"], corpus=q.get("corpus", corpus), limit=10, rerank=rerank)
        results, _ = await retrieve(req)
        kw = [k.lower() for k in q.get("expected_keywords", [])]
        types_ok = not q.get("expected_source_types") or any(r.source.source_type in q["expected_source_types"] for r in results)
        positions = [i for i, r in enumerate(results, start=1)
                     if any(k in r.chunk.content.lower() for k in kw) and types_ok]
        pos = positions[0] if positions else None
        if pos and pos <= 5: r5 += 1
        if pos and pos <= 10: r10 += 1
        if pos: mrr_sum += 1.0 / pos
        for tag in q.get("tags", []):
            per_tag_hits.setdefault(tag, []).append(1 if pos and pos <= 5 else 0)
        if judge and results:
            top_text = "\n---\n".join(r.chunk.content for r in results[:5])
            s = await judge_result(query=q["query"], result_text=top_text)
            if s is not None: judge_scores.append(s)

    n = len(queries)
    per_tag = {
        tag: {"recall_at_5": sum(xs) / len(xs), "n": len(xs)}
        for tag, xs in per_tag_hits.items()
    }
    judge_mean = statistics.mean(judge_scores) if judge_scores else None
    report = EvalReport(
        n_queries=n, recall_at_5=r5 / n, recall_at_10=r10 / n, mrr=mrr_sum / n,
        judge_mean=judge_mean, per_tag=per_tag,
        gate_passed=(r5 / n >= 0.80 and r10 / n >= 0.90 and mrr_sum / n >= 0.60 and (judge_mean is None or judge_mean >= 6.5)),
    )
    return report


def main() -> None:
    import argparse, sys
    ap = argparse.ArgumentParser()
    ap.add_argument("seed", type=Path)
    ap.add_argument("--corpus", default="personal")
    ap.add_argument("--no-rerank", action="store_true")
    ap.add_argument("--no-judge", action="store_true")
    args = ap.parse_args()
    report = asyncio.run(run(args.seed, args.corpus, rerank=not args.no_rerank, judge=not args.no_judge))
    print(json.dumps(asdict(report), indent=2))
    sys.exit(0 if report.gate_passed else 1)


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: `contextd eval` CLI wrapper**

```python
# contextd/cli/commands/eval.py
from __future__ import annotations
import asyncio
import json
from dataclasses import asdict
from pathlib import Path
import typer
from contextd.eval.run import run


def eval_(
    seed: Path = typer.Argument(..., exists=True),
    corpus: str = typer.Option("personal", "--corpus"),
    rerank: bool = typer.Option(True, "--rerank/--no-rerank"),
    judge: bool = typer.Option(True, "--judge/--no-judge"),
) -> None:
    report = asyncio.run(run(seed, corpus, rerank=rerank, judge=judge))
    typer.echo(json.dumps(asdict(report), indent=2))
    if not report.gate_passed:
        raise typer.Exit(1)
```

Register:

```python
# contextd/cli/main.py — append
from contextd.cli.commands import eval as eval_cmd
app.command(name="eval", help="Run the retrieval eval harness.")(eval_cmd.eval_)
```

- [ ] **Step 3: Test, commit**

```bash
git add contextd/eval/run.py contextd/cli/commands/eval.py contextd/cli/main.py tests/integration/eval/test_run.py
git commit -m "feat(eval): full runner with Recall@k, MRR, judge; `contextd eval` subcommand"
```

---

## Task 4: Privacy CI — no outbound network

**Files:**
- Create: `tests/privacy/__init__.py`
- Create: `tests/privacy/test_no_outbound_network.py`

Approach: monkey-patch `socket.socket.connect` to raise on any non-loopback target, then exercise the happy path (ingest + query with `--no-rerank`, `--no-rewrite`). The only permitted exception is the one-time embedding-model download, which is pre-cached in CI.

- [ ] **Step 1: Test**

```python
# tests/privacy/test_no_outbound_network.py
from __future__ import annotations
import ipaddress
import socket
from pathlib import Path
import numpy as np
import pytest
from typer.testing import CliRunner
from contextd.cli.main import app

pytestmark = pytest.mark.privacy

_ALLOWED_LOOPBACK = {ipaddress.ip_address("127.0.0.1"), ipaddress.ip_address("::1")}


class OutboundBlocked(AssertionError):
    pass


@pytest.fixture
def block_outbound(monkeypatch: pytest.MonkeyPatch):
    real_connect = socket.socket.connect

    def guarded(self, address):
        host = address[0] if isinstance(address, tuple) else address
        try:
            ip = ipaddress.ip_address(host)
        except ValueError:
            # Hostname → resolve via getaddrinfo, then check each result
            import socket as _s
            infos = _s.getaddrinfo(host, None)
            for _, _, _, _, sa in infos:
                ip = ipaddress.ip_address(sa[0])
                if ip not in _ALLOWED_LOOPBACK:
                    raise OutboundBlocked(f"outbound connect to {host} blocked")
        else:
            if ip not in _ALLOWED_LOOPBACK:
                raise OutboundBlocked(f"outbound connect to {host} blocked")
        return real_connect(self, address)

    monkeypatch.setattr(socket.socket, "connect", guarded)


def test_ingest_and_query_no_outbound(tmp_contextd_home, block_outbound, monkeypatch):
    # Use a stub embedder to avoid HF download.
    class StubEmb:
        model_name = "stub"; dim = 4
        def embed(self, texts): return np.tile([1.0, 0.0, 0.0, 0.0], (len(texts), 1)).astype(np.float32)
    monkeypatch.setattr("contextd.ingest.embedder.default_embedder", lambda: StubEmb())

    FIX = Path(__file__).resolve().parents[1] / "fixtures" / "pdfs"
    r = CliRunner().invoke(app, ["ingest", str(FIX), "--corpus", "personal"])
    assert r.exit_code == 0, r.output

    r = CliRunner().invoke(app, ["query", "any", "--corpus", "personal",
                                  "--limit", "1", "--no-rerank", "--no-rewrite", "--json"])
    assert r.exit_code == 0, r.output
```

- [ ] **Step 2: Run locally, expect pass** (embedder stubbed so no HF hit). Commit.

```bash
git add tests/privacy/
git commit -m "test(privacy): ingest + query complete with all non-loopback sockets blocked"
```

---

## Task 5: Privacy CI — non-mutation guarantee

**Files:**
- Create: `tests/privacy/test_non_mutation.py`

- [ ] **Step 1: Test**

```python
# tests/privacy/test_non_mutation.py
from __future__ import annotations
import hashlib
from pathlib import Path
import numpy as np
import pytest
from typer.testing import CliRunner
from contextd.cli.main import app

pytestmark = pytest.mark.privacy


def _tree_hash(path: Path) -> dict[str, str]:
    out: dict[str, str] = {}
    for f in sorted(path.rglob("*")):
        if f.is_file():
            out[str(f.relative_to(path))] = hashlib.sha256(f.read_bytes()).hexdigest()
    return out


def test_ingest_does_not_mutate_source(tmp_contextd_home, monkeypatch, tmp_path):
    # Copy fixtures to a scratch dir so we can modify-check.
    import shutil
    source_dir = Path(__file__).resolve().parents[1] / "fixtures" / "pdfs"
    scratch = tmp_path / "pdfs"
    shutil.copytree(source_dir, scratch)
    pre = _tree_hash(scratch)

    class StubEmb:
        model_name = "stub"; dim = 4
        def embed(self, texts): return np.tile([1.0, 0.0, 0.0, 0.0], (len(texts), 1)).astype(np.float32)
    monkeypatch.setattr("contextd.ingest.embedder.default_embedder", lambda: StubEmb())

    r = CliRunner().invoke(app, ["ingest", str(scratch), "--corpus", "personal"])
    assert r.exit_code == 0, r.output

    post = _tree_hash(scratch)
    assert pre == post, f"sources mutated during ingest: {set(pre) ^ set(post)}"
```

- [ ] **Step 2: Commit**

```bash
git add tests/privacy/test_non_mutation.py
git commit -m "test(privacy): ingestion never mutates source files (sha256 pre/post)"
```

---

## Task 6: No-content-at-INFO logging check

**Files:**
- Create: `tests/privacy/test_no_content_at_info.py`

- [ ] **Step 1: Test**

```python
# tests/privacy/test_no_content_at_info.py
from __future__ import annotations
import logging
from datetime import datetime, timezone
import numpy as np
import pytest
from contextd.logging_ import configure_logging
from contextd.retrieve.pipeline import retrieve
from contextd.retrieve.preprocess import build_request
from contextd.storage.db import insert_chunk, insert_corpus, insert_source, open_db
from contextd.storage.vectors import VectorStore

pytestmark = pytest.mark.privacy


async def test_query_content_not_logged_at_info(tmp_contextd_home, monkeypatch, caplog):
    configure_logging()
    caplog.set_level(logging.INFO, logger="contextd")
    conn = open_db("personal")
    insert_corpus(conn, name="personal", embed_model="t", embed_dim=4,
                  created_at=datetime.now(timezone.utc), schema_version=1)
    sid = insert_source(conn, corpus="personal", source_type="pdf", path="/a.pdf",
                         content_hash="sha256:x", ingested_at=datetime.now(timezone.utc),
                         chunk_count=0, status="active")
    insert_chunk(conn, source_id=sid, ordinal=0, token_count=2,
                 content="TOP SECRET PHI PATIENT X42")
    conn.commit()
    vs = VectorStore.open(corpus="personal", embed_dim=4, model_name="t")
    vs.upsert([conn.execute("SELECT id FROM chunk").fetchone()[0]],
              np.array([[1, 0, 0, 0]], dtype=np.float32))

    class StubEmb:
        model_name = "t"; dim = 4
        def embed(self, texts): return np.tile([1.0, 0.0, 0.0, 0.0], (len(texts), 1)).astype(np.float32)
    monkeypatch.setattr("contextd.ingest.embedder.default_embedder", lambda: StubEmb())

    req = build_request(query="TOP SECRET PHI", corpus="personal", limit=1, rerank=False)
    await retrieve(req)

    joined = " ".join(rec.getMessage() for rec in caplog.records)
    assert "TOP SECRET" not in joined
    assert "PATIENT X42" not in joined
```

- [ ] **Step 2: Commit.**

```bash
git add tests/privacy/test_no_content_at_info.py
git commit -m "test(privacy): query content never logged at INFO"
```

---

## Task 7: CI job for privacy + eval

**Files:**
- Modify: `.github/workflows/ci.yml` — add two jobs

- [ ] **Step 1: Extend workflow**

```yaml
  privacy:
    runs-on: ubuntu-24.04
    needs: [python]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
        with: { version: "0.5.5" }
      - run: uv sync --dev
      - run: uv run pytest -m privacy -q

  eval:
    runs-on: ubuntu-24.04
    needs: [python]
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
        with: { version: "0.5.5" }
      - run: uv sync --dev
      - name: Seed eval corpus
        run: |
          uv run contextd ingest tests/fixtures/pdfs/ --corpus eval
          uv run contextd ingest tests/fixtures/claude/export.json --corpus eval
      - name: Run 30-query eval (informational; non-blocking)
        run: uv run contextd eval contextd/eval/seed_queries.json --corpus eval --no-judge
        continue-on-error: true
```

Note the gate is enforced locally at release time; CI runs it informationally so flaky LLM calls don't block PR merges on tiny typo fixes.

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add privacy + eval jobs (privacy blocking; eval informational)"
```

---

## Task 8: README

**Files:**
- Rewrite: `README.md`

Sections (in order):
1. **Tagline.** _"Local-first, MCP-first personal RAG. Ingest PDFs, Claude exports, and git repos; query them from any agent."_
2. **90-second demo** — embedded video or GIF + link to `docs/demo/v0.1.mp4`.
3. **Install.**

   ```bash
   pipx install <published-name>           # or: uv tool install <published-name>
   contextd --help
   ```

4. **Ingest.**

   ```bash
   contextd ingest ~/papers/ --type pdf --corpus research
   contextd ingest ~/claude-exports/2026-03.json --corpus research
   contextd ingest ~/code/my-project --corpus research
   ```

5. **Query.**

   ```bash
   contextd query "how did Fu handle negation" --corpus research --limit 3
   ```

6. **Use from Claude Code / Codex / any MCP client.**

   ```bash
   contextd serve
   ```

   Plus the `.mcp.json` / `codex` config snippet.

7. **Design principles:** local-first, zero telemetry, named-corpus isolation, source non-mutation.
8. **Scope table** — what ships in v0.1, what's v0.2+ (pulled from master spec).
9. **Development** — `uv sync --dev`, `uv run pytest`, `cd mcp-server && pnpm build`.
10. **License:** MIT.

- [ ] **Step 1: Write. Step 2: Run through install → query on a clean container (or tmp venv) with a stopwatch; fail the gate if TTFQ > 5 min. Step 3: Commit.**

```bash
git add README.md
git commit -m "docs: full README with TTFQ-measured install path and MCP integration"
```

---

## Task 9: Demo video

**Files:**
- Create: `docs/demo/v0.1-script.md`
- Create: `docs/demo/v0.1.mp4` (or embed a hosted URL in README)

- [ ] **Step 1:** Follow the PRD §16.8 script verbatim:
  - Fresh terminal in WSL2 or macOS
  - `pipx install <published-name>`
  - `contextd ingest ~/papers/ --type pdf --corpus research` → 47 PDFs
  - `contextd ingest ~/claude-exports/2026-03.json --type claude_export --corpus research`
  - `contextd query "how did Fu handle negation" --limit 3`
  - Cut to Claude Code: _"Compare Fu and Kaster negation handling in the research corpus"_
  - Cut to Codex CLI: _"What did I conclude about Fu vs Kaster in my own notes?"_
  - Final card: _"Same corpus. Any agent. Local-first."_

Target: 90 s, no pauses, no typing mistakes — re-record until clean. 1080p, H.264.

- [ ] **Step 2: Commit (use LFS or external host if > 10 MB)**

```bash
git add docs/demo/v0.1-script.md docs/demo/v0.1.mp4
git commit -m "docs: 90-second demo — ingest, query, cross-agent (Claude Code + Codex)"
```

---

## Task 10: Release

- [ ] **Step 1: Final lint/type/test sweep**

```bash
uv run ruff format --check .
uv run ruff check .
uv run mypy contextd/
uv run pytest -q
uv run pytest -m privacy -q
uv run contextd eval contextd/eval/seed_queries.json --corpus research
```

All must pass. Eval gate: Recall@5 ≥ 0.80, Recall@10 ≥ 0.90, MRR ≥ 0.60, judge ≥ 6.5.

- [ ] **Step 2: Bump version in `pyproject.toml` and `contextd/__init__.py` to `0.1.0`.**

- [ ] **Step 3: Build + install on a clean venv**

```bash
uv build
python -m venv /tmp/contextd-release
/tmp/contextd-release/bin/pip install dist/contextd-0.1.0-*.whl
/tmp/contextd-release/bin/contextd version
```

- [ ] **Step 4: Tag + push**

```bash
git tag v0.1.0 -m "contextd v0.1.0 — local-first MCP personal RAG"
git push origin master --tags
```

- [ ] **Step 5: (Optional) PyPI trusted publish via GitHub Actions release workflow.** Include this in a separate commit if AlexZ wants it:

```yaml
# .github/workflows/release.yml
name: release
on:
  push:
    tags: [ 'v*.*.*' ]
jobs:
  pypi:
    runs-on: ubuntu-24.04
    environment: pypi
    permissions: { id-token: write }
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
        with: { version: "0.5.5" }
      - run: uv build
      - uses: pypa/gh-action-pypi-publish@release/v1
```

- [ ] **Step 6: Ship decision** — per PRD §16.7:
  - All pass → tag `v0.1.0` (done).
  - Some fail → tag `v0.1.0-rc1`, document issues in GitHub release notes, and iterate Day 3.

---

## Phase 5 Exit Gate Checklist

- [ ] Full `uv run pytest -q` green (unit + integration + privacy)
- [ ] `uv run contextd eval contextd/eval/seed_queries.json --corpus research` passes the M4 gate: **Recall@5 ≥ 0.80, Recall@10 ≥ 0.90, MRR ≥ 0.60, judge ≥ 6.5**
- [ ] Privacy CI green (no outbound network, no source mutation, no content at INFO)
- [ ] README installable-from-scratch accurate — clocked TTFQ under 5 minutes on a clean VM
- [ ] Demo video exists at `docs/demo/v0.1.mp4` or linked in README; shows cross-AI portability
- [ ] `v0.1.0` tag pushed to `origin master`
- [ ] (Optional) Wheel published on PyPI via trusted publishing

---

## Post-release day-3 pattern (if cuts were taken)

If PRD §16.9 scope-cut ladder got invoked:

| Cut taken | Restore path | Ship as |
|-----------|--------------|---------|
| Dropped S6 demo | Record + PR + re-tag `v0.1.1` | `v0.1.0-rc1` → `v0.1.0` once restored |
| Dropped S5 corpora | Add flag back + migrate test data | `v0.1.0-rc1` |
| Dropped TS MCP | Python-only `mcp` SDK fallback, then port to TS for `v0.1.1` | `v0.1.0-rc1` |
| Dropped rerank | Ship RRF-only, add rerank back in `v0.1.1` after dogfooding | `v0.1.0` (acceptable) |
| Dropped git adapter | Add in `v0.1.1` | `v0.1.0-rc1` |
| Dropped PDF sections | Fixed windows are OK; improve later | `v0.1.0` (acceptable) |

Never cut M1–M10.
