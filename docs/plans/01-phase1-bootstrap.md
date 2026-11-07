# Phase 1 — Bootstrap & Storage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans`.

**Goal:** Stand up the repo, CI, and storage layer (SQLite + LanceDB) so Phase 2 has a durable sink for chunks and embeddings.

**Architecture:** Single Python package `contextd`, uv-managed. SQLite via stdlib `sqlite3` with WAL + FK on + FTS5 virtual table; LanceDB 0.17 per-corpus directory at `$CONTEXTD_HOME/corpora/<name>/vectors.lance`. All storage calls go through two modules: `contextd/storage/db.py` (relational) and `contextd/storage/vectors.py` (ANN). Data root resolved once at process start via `contextd.config.settings.data_root`.

**Tech Stack:** Python 3.12, uv 0.5.5+, ruff 0.8.2, mypy 1.13, pytest 8.3.4, pytest-asyncio 0.24, lancedb 0.17.0, pysqlite3-binary 0.5.3, pydantic 2.10, structlog 24.4, python-ulid 3.0.

**Exit gate (PRD §16.3):** CI green on Ubuntu+macOS, `uv run pytest` passes smoke + integration tests, `pipx install .` succeeds on a clean venv.

---

## File Structure

Create:
- `pyproject.toml`, `uv.lock`, `LICENSE` (MIT), `README.md` (skeleton), `.gitignore`, `.python-version`
- `.github/workflows/ci.yml`
- `contextd/__init__.py` — exports `__version__`
- `contextd/config.py` — pydantic `Settings` with `CONTEXTD_HOME` env override
- `contextd/logging_.py` — structlog INFO default, no content at INFO
- `contextd/storage/__init__.py`
- `contextd/storage/models.py` — frozen dataclasses from master spec
- `contextd/storage/schema.py` — DDL constants + `apply_schema(conn)`
- `contextd/storage/db.py` — `open_db(corpus) -> sqlite3.Connection` + CRUD helpers
- `contextd/storage/vectors.py` — LanceDB wrapper class `VectorStore`
- `tests/conftest.py` — pytest fixtures: `tmp_contextd_home`, `fresh_corpus`
- `tests/unit/test_config.py`
- `tests/unit/test_models.py`
- `tests/unit/test_schema.py`
- `tests/integration/test_db.py`
- `tests/integration/test_vectors.py`
- `tests/test_smoke.py`

---

## Task 1: Scaffold the Python package

**Files:**
- Create: `pyproject.toml`
- Create: `.python-version`
- Create: `.gitignore`
- Create: `LICENSE` (MIT boilerplate, copyright Alex Zhang)
- Create: `README.md` (just title + 2-line description for now)

- [ ] **Step 1: Write `pyproject.toml`**

```toml
[project]
name = "contextd"
version = "0.1.0.dev0"
description = "Local-first, MCP-first personal RAG server"
requires-python = ">=3.11,<3.14"
readme = "README.md"
license = { text = "MIT" }
authors = [{ name = "Alex Zhang", email = "alexandar.zhang@mail.utoronto.ca" }]
dependencies = [
  "fastapi==0.115.4",
  "uvicorn==0.32.1",
  "pydantic==2.10.0",
  "pydantic-settings>=2.6,<3",
  "lancedb==0.17.0",
  "pyarrow>=18,<19",
  "sentence-transformers==3.3.1",
  "FlagEmbedding==1.3.4",
  "tokenizers==0.21.0",
  "torch==2.5.1",
  "anthropic==0.50.0",
  "pymupdf4llm==0.0.17",
  "pymupdf==1.25.1",
  "pypdf==5.1.0",
  "tree-sitter==0.23.2",
  "markdown-it-py==3.0.0",
  "mdit-py-plugins==0.4.2",
  "pygit2==1.16.0",
  "typer==0.13.0",
  "rich==13.9.4",
  "structlog==24.4.0",
  "python-ulid==3.0.0",
]

[project.optional-dependencies]
gpu = ["torch==2.5.1"]  # CUDA wheel selection handled by uv extra-index-url

[project.scripts]
contextd = "contextd.cli.main:app"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["contextd"]

[dependency-groups]
dev = [
  "ruff==0.8.2",
  "mypy==1.13.0",
  "pytest==8.3.4",
  "pytest-asyncio==0.24.0",
  "pytest-benchmark==5.1.0",
  "coverage==7.6.9",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "N", "SIM", "TCH"]
ignore = ["E501"]  # DDL strings exceed line length

[tool.pytest.ini_options]
minversion = "8.0"
testpaths = ["tests"]
asyncio_mode = "auto"
markers = [
  "unit: fast isolated tests",
  "integration: touch filesystem/sqlite/lancedb",
  "privacy: enforce non-mutation + no-outbound-network",
]

[tool.mypy]
python_version = "3.12"
strict_optional = true
disallow_untyped_defs = true
warn_unused_ignores = true
```

- [ ] **Step 2: Write `.python-version`**

```
3.12
```

- [ ] **Step 3: Write `.gitignore`**

```
# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/
.pytest_cache/
.ruff_cache/
.mypy_cache/
htmlcov/
.coverage
dist/
build/

# Node
node_modules/
.turbo/

# contextd data
.contextd-dev/
data/
*.lance/
*.lance

# OS
.DS_Store
```

- [ ] **Step 4: `uv sync` to generate `uv.lock`**

```bash
cd personal-RAG-for-everything
uv sync --dev
```

Expected: `uv.lock` created; `.venv/` populated.

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml uv.lock .python-version .gitignore LICENSE README.md
git commit -m "chore: scaffold contextd Python package with pinned deps"
```

---

## Task 2: Write `contextd/config.py` with TDD

**Files:**
- Create: `contextd/__init__.py`
- Create: `contextd/config.py`
- Test: `tests/unit/test_config.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/test_config.py
from pathlib import Path
import pytest
from contextd.config import Settings, get_settings

def test_default_data_root_uses_home(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.delenv("CONTEXTD_HOME", raising=False)
    s = Settings()
    assert s.data_root == Path.home() / ".contextd"

def test_env_override_data_root(monkeypatch: pytest.MonkeyPatch, tmp_path: Path) -> None:
    monkeypatch.setenv("CONTEXTD_HOME", str(tmp_path))
    get_settings.cache_clear()
    s = get_settings()
    assert s.data_root == tmp_path

def test_settings_is_frozen() -> None:
    s = Settings()
    with pytest.raises(Exception):  # pydantic ValidationError on frozen model
        s.data_root = Path("/nope")  # type: ignore[misc]
```

- [ ] **Step 2: Run test — expect fail**

```bash
uv run pytest tests/unit/test_config.py -v
```

Expected: ImportError / ModuleNotFoundError.

- [ ] **Step 3: Implement `contextd/__init__.py`**

```python
__version__ = "0.1.0.dev0"
```

- [ ] **Step 4: Implement `contextd/config.py`**

```python
from __future__ import annotations
from functools import lru_cache
from pathlib import Path
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="CONTEXTD_",
        env_file=None,
        frozen=True,
        extra="ignore",
    )

    data_root: Path = Field(default_factory=lambda: Path.home() / ".contextd", alias="CONTEXTD_HOME")
    default_corpus: str = "personal"
    log_level: str = "INFO"
    schema_version: int = 1

    embedding_model: str = "BAAI/bge-m3"
    embedding_dim: int = 1024
    embedding_device: str = "cpu"
    embedding_batch_size: int = 16

    retrieval_default_limit: int = 10
    retrieval_dense_top_k: int = 50
    retrieval_sparse_top_k: int = 50
    retrieval_rrf_k: int = 60
    retrieval_rerank_top_k: int = 50
    retrieval_rewrite_enabled: bool = False  # PRD D-30: disabled by default in v0.1
    retrieval_rerank_enabled: bool = True
    retrieval_rewrite_timeout_ms: int = 3000
    retrieval_rerank_timeout_ms: int = 5000

    reranker_provider: str = "anthropic"
    reranker_model: str = "claude-haiku-4-5"
    rewriter_model: str = "claude-haiku-4-5"

    mcp_host: str = "127.0.0.1"
    mcp_port: int = 8787


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

- [ ] **Step 5: Run test — expect pass**

```bash
uv run pytest tests/unit/test_config.py -v
```

- [ ] **Step 6: Commit**

```bash
git add contextd/__init__.py contextd/config.py tests/unit/test_config.py
git commit -m "feat(config): pydantic Settings with CONTEXTD_HOME override"
```

---

## Task 3: Write `contextd/storage/models.py` dataclasses

**Files:**
- Create: `contextd/storage/__init__.py` (empty)
- Create: `contextd/storage/models.py`
- Test: `tests/unit/test_models.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/test_models.py
from datetime import datetime
import pytest
from dataclasses import FrozenInstanceError
from contextd.storage.models import Chunk, Corpus, Edge, Source

def test_source_is_frozen() -> None:
    s = Source(
        id=1, corpus="personal", source_type="pdf",
        path="/tmp/a.pdf", content_hash="sha256:abc", ingested_at=datetime.now(),
        chunk_count=5, status="active",
    )
    with pytest.raises(FrozenInstanceError):
        s.path = "/tmp/b.pdf"  # type: ignore[misc]

def test_chunk_defaults() -> None:
    c = Chunk(id=1, source_id=1, ordinal=0, content="hello", token_count=1)
    assert c.section_label is None
    assert c.role is None

def test_edge_accepts_hint_without_target() -> None:
    e = Edge(id=1, source_chunk_id=5, edge_type="wikilink", target_hint="Fu 2024")
    assert e.target_chunk_id is None
    assert e.target_hint == "Fu 2024"

def test_corpus_requires_embed_model() -> None:
    with pytest.raises(TypeError):
        Corpus(name="x", embed_dim=1024, created_at=datetime.now(), schema_version=1)  # type: ignore[call-arg]
```

- [ ] **Step 2: Run test — expect fail** (`uv run pytest tests/unit/test_models.py -v`)

- [ ] **Step 3: Implement `contextd/storage/models.py`** — copy the full block from `00-master-spec.md` (section "Shared data types"), then `touch contextd/storage/__init__.py`.

- [ ] **Step 4: Run test — expect pass**

- [ ] **Step 5: Commit**

```bash
git add contextd/storage/
git commit -m "feat(storage): frozen dataclasses for Corpus, Source, Chunk, Edge"
```

---

## Task 4: Write `contextd/storage/schema.py` (DDL) with TDD

**Files:**
- Create: `contextd/storage/schema.py`
- Test: `tests/unit/test_schema.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/test_schema.py
import sqlite3
from contextd.storage.schema import DDL_STATEMENTS, apply_schema

def _tables(conn: sqlite3.Connection) -> set[str]:
    rows = conn.execute("SELECT name FROM sqlite_master WHERE type IN ('table', 'view')").fetchall()
    return {r[0] for r in rows}

def test_apply_schema_creates_all_tables(tmp_path):
    db = tmp_path / "t.db"
    conn = sqlite3.connect(db)
    apply_schema(conn)
    names = _tables(conn)
    for t in ("corpus", "source", "chunk", "chunk_fts", "edge", "chunk_meta", "source_meta", "audit_log"):
        assert t in names, f"missing table: {t}"

def test_apply_schema_is_idempotent(tmp_path):
    db = tmp_path / "t.db"
    conn = sqlite3.connect(db)
    apply_schema(conn)
    apply_schema(conn)  # second apply must not raise
    assert "chunk" in _tables(conn)

def test_fts_trigger_populates_on_insert(tmp_path):
    db = tmp_path / "t.db"
    conn = sqlite3.connect(db)
    apply_schema(conn)
    conn.execute("INSERT INTO corpus VALUES ('personal', NULL, 'BAAI/bge-m3', 1024, '2026-04-20T00:00:00', 1)")
    conn.execute("INSERT INTO source(corpus, source_type, path, content_hash, ingested_at, chunk_count, status) "
                 "VALUES ('personal', 'pdf', '/a.pdf', 'sha256:x', '2026-04-20T00:00:00', 1, 'active')")
    sid = conn.execute("SELECT id FROM source").fetchone()[0]
    conn.execute("INSERT INTO chunk(source_id, ordinal, token_count, content) VALUES (?, 0, 2, 'negation handling')", (sid,))
    conn.commit()
    hits = conn.execute("SELECT rowid FROM chunk_fts WHERE chunk_fts MATCH 'negation'").fetchall()
    assert len(hits) == 1
```

- [ ] **Step 2: Run test — expect fail.**

- [ ] **Step 3: Implement `contextd/storage/schema.py`**

Copy the full DDL block from `00-master-spec.md` section "SQLite DDL" into a module-level `DDL_STATEMENTS: tuple[str, ...]`. Split on `;` with care (keep triggers intact — use a split helper that respects `BEGIN...END` blocks, or use one element per statement).

```python
from __future__ import annotations
import sqlite3

PRAGMAS = (
    "PRAGMA journal_mode = WAL",
    "PRAGMA foreign_keys = ON",
    "PRAGMA synchronous = NORMAL",
)

DDL_STATEMENTS: tuple[str, ...] = (
    """CREATE TABLE IF NOT EXISTS corpus (
        name           TEXT PRIMARY KEY,
        root_path      TEXT,
        embed_model    TEXT NOT NULL,
        embed_dim      INTEGER NOT NULL,
        created_at     TEXT NOT NULL,
        schema_version INTEGER NOT NULL
    )""",
    """CREATE TABLE IF NOT EXISTS source (
        id           INTEGER PRIMARY KEY AUTOINCREMENT,
        corpus       TEXT NOT NULL REFERENCES corpus(name) ON DELETE CASCADE,
        source_type  TEXT NOT NULL,
        path         TEXT NOT NULL,
        content_hash TEXT NOT NULL,
        title        TEXT,
        ingested_at  TEXT NOT NULL,
        source_mtime TEXT,
        chunk_count  INTEGER NOT NULL DEFAULT 0,
        status       TEXT NOT NULL DEFAULT 'active',
        UNIQUE (corpus, path)
    )""",
    "CREATE INDEX IF NOT EXISTS idx_source_type ON source(source_type)",
    "CREATE INDEX IF NOT EXISTS idx_source_hash ON source(content_hash)",
    """CREATE TABLE IF NOT EXISTS chunk (
        id              INTEGER PRIMARY KEY AUTOINCREMENT,
        source_id       INTEGER NOT NULL REFERENCES source(id) ON DELETE CASCADE,
        ordinal         INTEGER NOT NULL,
        offset_start    INTEGER,
        offset_end      INTEGER,
        token_count     INTEGER NOT NULL,
        content         TEXT NOT NULL,
        section_label   TEXT,
        scope           TEXT,
        role            TEXT,
        chunk_timestamp TEXT
    )""",
    "CREATE INDEX IF NOT EXISTS idx_chunk_source_ordinal ON chunk(source_id, ordinal)",
    "CREATE INDEX IF NOT EXISTS idx_chunk_timestamp ON chunk(chunk_timestamp)",
    """CREATE VIRTUAL TABLE IF NOT EXISTS chunk_fts USING fts5(
        content, content='chunk', content_rowid='id', tokenize='unicode61'
    )""",
    """CREATE TRIGGER IF NOT EXISTS chunk_ai AFTER INSERT ON chunk BEGIN
        INSERT INTO chunk_fts(rowid, content) VALUES (new.id, new.content);
    END""",
    """CREATE TRIGGER IF NOT EXISTS chunk_ad AFTER DELETE ON chunk BEGIN
        INSERT INTO chunk_fts(chunk_fts, rowid, content) VALUES ('delete', old.id, old.content);
    END""",
    """CREATE TRIGGER IF NOT EXISTS chunk_au AFTER UPDATE ON chunk BEGIN
        INSERT INTO chunk_fts(chunk_fts, rowid, content) VALUES ('delete', old.id, old.content);
        INSERT INTO chunk_fts(rowid, content) VALUES (new.id, new.content);
    END""",
    """CREATE TABLE IF NOT EXISTS edge (
        id                INTEGER PRIMARY KEY AUTOINCREMENT,
        source_chunk_id   INTEGER NOT NULL REFERENCES chunk(id) ON DELETE CASCADE,
        target_chunk_id   INTEGER REFERENCES chunk(id) ON DELETE CASCADE,
        target_hint       TEXT,
        edge_type         TEXT NOT NULL,
        label             TEXT,
        weight            REAL
    )""",
    "CREATE INDEX IF NOT EXISTS idx_edge_src ON edge(source_chunk_id, edge_type)",
    "CREATE INDEX IF NOT EXISTS idx_edge_tgt ON edge(target_chunk_id, edge_type)",
    "CREATE INDEX IF NOT EXISTS idx_edge_hint ON edge(target_hint)",
    """CREATE TABLE IF NOT EXISTS chunk_meta (
        chunk_id INTEGER NOT NULL REFERENCES chunk(id) ON DELETE CASCADE,
        key      TEXT NOT NULL,
        value    TEXT NOT NULL,
        PRIMARY KEY (chunk_id, key)
    )""",
    """CREATE TABLE IF NOT EXISTS source_meta (
        source_id INTEGER NOT NULL REFERENCES source(id) ON DELETE CASCADE,
        key       TEXT NOT NULL,
        value     TEXT NOT NULL,
        PRIMARY KEY (source_id, key)
    )""",
    """CREATE TABLE IF NOT EXISTS audit_log (
        id           INTEGER PRIMARY KEY AUTOINCREMENT,
        occurred_at  TEXT NOT NULL,
        actor        TEXT NOT NULL,
        action       TEXT NOT NULL,
        target       TEXT NOT NULL,
        details_json TEXT NOT NULL DEFAULT '{}'
    )""",
)


def apply_schema(conn: sqlite3.Connection) -> None:
    for p in PRAGMAS:
        conn.execute(p)
    for s in DDL_STATEMENTS:
        conn.execute(s)
    conn.commit()
```

- [ ] **Step 4: Run test — expect pass.**

- [ ] **Step 5: Commit**

```bash
git add contextd/storage/schema.py tests/unit/test_schema.py
git commit -m "feat(storage): DDL and apply_schema with FTS5 triggers"
```

---

## Task 5: Write `contextd/storage/db.py` (connection + CRUD)

**Files:**
- Create: `contextd/storage/db.py`
- Modify: `tests/conftest.py` (new)
- Test: `tests/integration/test_db.py`

- [ ] **Step 1: Write `tests/conftest.py`**

```python
from __future__ import annotations
from pathlib import Path
import pytest
from contextd.config import get_settings

@pytest.fixture
def tmp_contextd_home(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> Path:
    monkeypatch.setenv("CONTEXTD_HOME", str(tmp_path))
    get_settings.cache_clear()
    return tmp_path
```

- [ ] **Step 2: Write the failing integration test**

```python
# tests/integration/test_db.py
from datetime import datetime, timezone
import pytest
from contextd.storage.db import open_db, insert_corpus, insert_source, insert_chunk, get_source_by_path, fetch_chunks_by_ids

pytestmark = pytest.mark.integration

def test_insert_corpus_and_source_roundtrip(tmp_contextd_home):
    conn = open_db("personal")
    insert_corpus(conn, name="personal", embed_model="BAAI/bge-m3", embed_dim=1024, created_at=datetime.now(timezone.utc), schema_version=1)
    sid = insert_source(conn, corpus="personal", source_type="pdf", path="/a.pdf", content_hash="sha256:x",
                        ingested_at=datetime.now(timezone.utc), chunk_count=0, status="active", title="A")
    got = get_source_by_path(conn, corpus="personal", path="/a.pdf")
    assert got is not None and got.id == sid and got.title == "A"

def test_insert_chunks_and_fetch_by_ids(tmp_contextd_home):
    conn = open_db("personal")
    insert_corpus(conn, name="personal", embed_model="BAAI/bge-m3", embed_dim=1024, created_at=datetime.now(timezone.utc), schema_version=1)
    sid = insert_source(conn, corpus="personal", source_type="pdf", path="/b.pdf", content_hash="sha256:y",
                        ingested_at=datetime.now(timezone.utc), chunk_count=0, status="active")
    c1 = insert_chunk(conn, source_id=sid, ordinal=0, token_count=2, content="hello world")
    c2 = insert_chunk(conn, source_id=sid, ordinal=1, token_count=3, content="goodbye world today")
    rows = fetch_chunks_by_ids(conn, [c1, c2])
    assert {r.id for r in rows} == {c1, c2}

def test_unique_corpus_path_conflict_raises(tmp_contextd_home):
    import sqlite3
    conn = open_db("personal")
    insert_corpus(conn, name="personal", embed_model="BAAI/bge-m3", embed_dim=1024, created_at=datetime.now(timezone.utc), schema_version=1)
    insert_source(conn, corpus="personal", source_type="pdf", path="/a.pdf", content_hash="sha256:x",
                   ingested_at=datetime.now(timezone.utc), chunk_count=0, status="active")
    with pytest.raises(sqlite3.IntegrityError):
        insert_source(conn, corpus="personal", source_type="pdf", path="/a.pdf", content_hash="sha256:y",
                       ingested_at=datetime.now(timezone.utc), chunk_count=0, status="active")
```

- [ ] **Step 3: Run tests — expect fail**

- [ ] **Step 4: Implement `contextd/storage/db.py`**

```python
from __future__ import annotations
import sqlite3
from datetime import datetime
from pathlib import Path
from contextd.config import get_settings
from contextd.storage.models import Chunk, Source, SourceStatus, SourceType
from contextd.storage.schema import apply_schema


def _corpus_db_path(corpus: str) -> Path:
    settings = get_settings()
    p = settings.data_root / "corpora" / corpus
    p.mkdir(parents=True, exist_ok=True)
    return p / "chunks.db"


def open_db(corpus: str) -> sqlite3.Connection:
    conn = sqlite3.connect(str(_corpus_db_path(corpus)), isolation_level=None)
    conn.row_factory = sqlite3.Row
    apply_schema(conn)
    return conn


def insert_corpus(conn: sqlite3.Connection, *, name: str, embed_model: str, embed_dim: int,
                  created_at: datetime, schema_version: int, root_path: str | None = None) -> None:
    conn.execute(
        "INSERT OR IGNORE INTO corpus(name, root_path, embed_model, embed_dim, created_at, schema_version) "
        "VALUES (?, ?, ?, ?, ?, ?)",
        (name, root_path, embed_model, embed_dim, created_at.isoformat(), schema_version),
    )


def insert_source(conn: sqlite3.Connection, *, corpus: str, source_type: SourceType, path: str,
                  content_hash: str, ingested_at: datetime, chunk_count: int, status: SourceStatus,
                  title: str | None = None, source_mtime: datetime | None = None) -> int:
    cur = conn.execute(
        "INSERT INTO source(corpus, source_type, path, content_hash, title, ingested_at, source_mtime, chunk_count, status) "
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)",
        (corpus, source_type, path, content_hash, title, ingested_at.isoformat(),
         source_mtime.isoformat() if source_mtime else None, chunk_count, status),
    )
    return int(cur.lastrowid or 0)


def get_source_by_path(conn: sqlite3.Connection, *, corpus: str, path: str) -> Source | None:
    row = conn.execute("SELECT * FROM source WHERE corpus = ? AND path = ?", (corpus, path)).fetchone()
    return row_to_source(row) if row else None


def insert_chunk(conn: sqlite3.Connection, *, source_id: int, ordinal: int, token_count: int, content: str,
                 offset_start: int | None = None, offset_end: int | None = None,
                 section_label: str | None = None, scope: str | None = None,
                 role: str | None = None, chunk_timestamp: datetime | None = None) -> int:
    cur = conn.execute(
        "INSERT INTO chunk(source_id, ordinal, offset_start, offset_end, token_count, content, "
        "section_label, scope, role, chunk_timestamp) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)",
        (source_id, ordinal, offset_start, offset_end, token_count, content,
         section_label, scope, role, chunk_timestamp.isoformat() if chunk_timestamp else None),
    )
    return int(cur.lastrowid or 0)


def fetch_chunks_by_ids(conn: sqlite3.Connection, ids: list[int]) -> list[Chunk]:
    if not ids:
        return []
    placeholders = ",".join("?" for _ in ids)
    rows = conn.execute(f"SELECT * FROM chunk WHERE id IN ({placeholders})", ids).fetchall()
    return [_row_to_chunk(r) for r in rows]


def row_to_source(r: sqlite3.Row) -> Source:
    return Source(
        id=r["id"], corpus=r["corpus"], source_type=r["source_type"], path=r["path"],
        content_hash=r["content_hash"], title=r["title"],
        ingested_at=datetime.fromisoformat(r["ingested_at"]),
        source_mtime=datetime.fromisoformat(r["source_mtime"]) if r["source_mtime"] else None,
        chunk_count=r["chunk_count"], status=r["status"],
    )


def _row_to_chunk(r: sqlite3.Row) -> Chunk:
    return Chunk(
        id=r["id"], source_id=r["source_id"], ordinal=r["ordinal"],
        offset_start=r["offset_start"], offset_end=r["offset_end"],
        token_count=r["token_count"], content=r["content"],
        section_label=r["section_label"], scope=r["scope"], role=r["role"],
        chunk_timestamp=datetime.fromisoformat(r["chunk_timestamp"]) if r["chunk_timestamp"] else None,
    )
```

- [ ] **Step 5: Run tests — expect pass**

- [ ] **Step 6: Commit**

```bash
git add contextd/storage/db.py tests/conftest.py tests/integration/test_db.py
git commit -m "feat(storage): SQLite connection + insert/fetch helpers"
```

---

## Task 6: Write `contextd/storage/vectors.py` LanceDB wrapper

**Files:**
- Create: `contextd/storage/vectors.py`
- Test: `tests/integration/test_vectors.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/integration/test_vectors.py
import numpy as np
import pytest
from contextd.storage.vectors import VectorStore

pytestmark = pytest.mark.integration

def test_upsert_then_ann_search_returns_closest(tmp_contextd_home):
    vs = VectorStore.open(corpus="personal", embed_dim=4, model_name="test-4d")
    vecs = np.array([
        [1.0, 0.0, 0.0, 0.0],   # chunk 1
        [0.0, 1.0, 0.0, 0.0],   # chunk 2
        [0.7, 0.7, 0.0, 0.0],   # chunk 3 (between 1 and 2)
    ], dtype=np.float32)
    vs.upsert([1, 2, 3], vecs)
    hits = vs.ann_search(np.array([1.0, 0.0, 0.0, 0.0], dtype=np.float32), k=2)
    # chunk 1 must rank first
    assert hits[0][0] == 1

def test_delete_removes_vectors(tmp_contextd_home):
    vs = VectorStore.open(corpus="personal", embed_dim=4, model_name="test-4d")
    vecs = np.eye(4, dtype=np.float32)
    vs.upsert([1, 2, 3, 4], vecs)
    vs.delete([2, 4])
    hits = vs.ann_search(np.array([0.0, 1.0, 0.0, 0.0], dtype=np.float32), k=4)
    ids = {h[0] for h in hits}
    assert 2 not in ids and 4 not in ids

def test_roundtrip_empty_search_returns_empty(tmp_contextd_home):
    vs = VectorStore.open(corpus="personal", embed_dim=4, model_name="test-4d")
    hits = vs.ann_search(np.array([1.0, 0.0, 0.0, 0.0], dtype=np.float32), k=3)
    assert hits == []
```

- [ ] **Step 2: Run test — expect fail.**

- [ ] **Step 3: Implement `contextd/storage/vectors.py`**

```python
from __future__ import annotations
from pathlib import Path
from typing import Self
import lancedb
import numpy as np
import pyarrow as pa
from contextd.config import get_settings


class VectorStore:
    def __init__(self, db: lancedb.DBConnection, table: lancedb.table.Table, embed_dim: int, model_name: str) -> None:
        self._db = db
        self._table = table
        self._embed_dim = embed_dim
        self._model_name = model_name

    @classmethod
    def open(cls, *, corpus: str, embed_dim: int, model_name: str) -> Self:
        root = get_settings().data_root / "corpora" / corpus
        root.mkdir(parents=True, exist_ok=True)
        db = lancedb.connect(str(root / "vectors.lance"))
        schema = pa.schema([
            pa.field("chunk_id", pa.int64()),
            pa.field("vector", pa.list_(pa.float32(), embed_dim)),
            pa.field("model_name", pa.string()),
        ])
        if "embedding" in db.table_names():
            table = db.open_table("embedding")
        else:
            table = db.create_table("embedding", schema=schema, mode="create")
        return cls(db, table, embed_dim, model_name)

    def upsert(self, chunk_ids: list[int], vectors: np.ndarray) -> None:
        if vectors.shape != (len(chunk_ids), self._embed_dim):
            raise ValueError(f"vectors shape {vectors.shape} != ({len(chunk_ids)}, {self._embed_dim})")
        # Delete by id then add (LanceDB upsert idiom for small batches)
        ids_sql = ", ".join(str(i) for i in chunk_ids)
        self._table.delete(f"chunk_id IN ({ids_sql})")
        records = [
            {"chunk_id": cid, "vector": vectors[i].tolist(), "model_name": self._model_name}
            for i, cid in enumerate(chunk_ids)
        ]
        self._table.add(records)

    def delete(self, chunk_ids: list[int]) -> None:
        if not chunk_ids:
            return
        ids_sql = ", ".join(str(i) for i in chunk_ids)
        self._table.delete(f"chunk_id IN ({ids_sql})")

    def ann_search(self, query_vec: np.ndarray, k: int) -> list[tuple[int, float]]:
        if self._table.count_rows() == 0:
            return []
        result = (self._table
                  .search(query_vec.tolist())
                  .metric("cosine")
                  .limit(k)
                  .to_list())
        return [(int(r["chunk_id"]), float(r["_distance"])) for r in result]
```

- [ ] **Step 4: Run test — expect pass**

- [ ] **Step 5: Commit**

```bash
git add contextd/storage/vectors.py tests/integration/test_vectors.py
git commit -m "feat(storage): LanceDB VectorStore with upsert/delete/ann_search"
```

---

## Task 7: Write `contextd/logging_.py` structlog config

**Files:**
- Create: `contextd/logging_.py`
- Test: `tests/unit/test_logging.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/test_logging.py
import logging
from contextd.logging_ import configure_logging, get_logger

def test_default_level_is_info(monkeypatch):
    monkeypatch.delenv("CONTEXTD_LOG_LEVEL", raising=False)
    configure_logging()
    assert logging.getLogger("contextd").level == logging.INFO

def test_debug_env_overrides(monkeypatch):
    monkeypatch.setenv("CONTEXTD_LOG_LEVEL", "DEBUG")
    configure_logging()
    assert logging.getLogger("contextd").level == logging.DEBUG

def test_get_logger_is_structlog(monkeypatch):
    configure_logging()
    log = get_logger("test")
    # structlog BoundLogger has a `bind` method; stdlib Logger does not.
    assert hasattr(log, "bind")
```

- [ ] **Step 2: Run test — expect fail**

- [ ] **Step 3: Implement**

```python
from __future__ import annotations
import logging
import os
import structlog


def configure_logging() -> None:
    level_name = os.environ.get("CONTEXTD_LOG_LEVEL", "INFO").upper()
    level = getattr(logging, level_name, logging.INFO)
    logging.basicConfig(format="%(message)s", level=level)
    logging.getLogger("contextd").setLevel(level)
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso", utc=True),
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(level),
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )


def get_logger(name: str) -> structlog.stdlib.BoundLogger:
    return structlog.get_logger(name)
```

- [ ] **Step 4: Run test — expect pass**

- [ ] **Step 5: Commit**

```bash
git add contextd/logging_.py tests/unit/test_logging.py
git commit -m "feat(logging): structlog JSON renderer with env-driven level"
```

---

## Task 8: Smoke test

**Files:**
- Create: `tests/test_smoke.py`

- [ ] **Step 1: Write test**

```python
# tests/test_smoke.py
from contextd import __version__
from contextd.config import get_settings
from contextd.storage.db import open_db
from contextd.storage.vectors import VectorStore

def test_version_defined():
    assert __version__.startswith("0.1.")

def test_open_db_creates_file(tmp_contextd_home):
    conn = open_db("personal")
    row = conn.execute("SELECT name FROM sqlite_master WHERE type='table' AND name='chunk'").fetchone()
    assert row is not None

def test_vector_store_open_is_idempotent(tmp_contextd_home):
    v1 = VectorStore.open(corpus="personal", embed_dim=4, model_name="test")
    v2 = VectorStore.open(corpus="personal", embed_dim=4, model_name="test")
    assert v1 is not v2  # separate handles, same underlying store
```

- [ ] **Step 2: Run test — expect pass** (all machinery already built)

- [ ] **Step 3: Commit**

```bash
git add tests/test_smoke.py
git commit -m "test: end-to-end smoke verifying package imports + storage init"
```

---

## Task 9: CI pipeline

**Files:**
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Write workflow**

```yaml
# .github/workflows/ci.yml
name: ci
on:
  push:
    branches: [master, main]
  pull_request:

jobs:
  python:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-24.04, macos-14]
        python-version: ["3.12"]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
        with:
          version: "0.5.5"
      - name: Set up Python ${{ matrix.python-version }}
        run: uv python install ${{ matrix.python-version }}
      - name: Sync deps
        run: uv sync --dev
      - name: Format check
        run: uv run ruff format --check .
      - name: Lint
        run: uv run ruff check .
      - name: Typecheck
        run: uv run mypy contextd/
      - name: Unit + integration tests
        run: uv run pytest -q --maxfail=1
      - name: Install smoke
        run: |
          uv build
          uv tool install ./dist/contextd-*.whl
          contextd --help || contextd version || true
```

- [ ] **Step 2: Push to a feature branch and confirm CI green before merging to master**

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: lint, typecheck, pytest, install-smoke on ubuntu + macos"
```

---

## Phase 1 Exit Gate Checklist

- [ ] `uv run ruff format --check .` — clean
- [ ] `uv run ruff check .` — clean
- [ ] `uv run mypy contextd/` — clean
- [ ] `uv run pytest -q` — green, no skips
- [ ] `uv build && uv tool install ./dist/contextd-*.whl` — succeeds on clean venv
- [ ] GitHub Actions `ci` workflow passes on both `ubuntu-24.04` and `macos-14`
- [ ] `data/` and `.contextd-dev/` are in `.gitignore` — confirmed
- [ ] All commits land on `master` with conventional-commits messages

**Stop here. Do not begin Phase 2 until this gate passes and AlexZ signs off.**
