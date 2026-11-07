# Before-publish checklist — `v0.1.0`

All code work for Phase 5 is landed on `master`. These remaining items
need AlexZ's hands (content creation, environment testing, or state
that affects the public release).

---

## Must-do (gate on tag)

- [ ] **Record the demo video** — follow [`docs/demo/v0.1-script.md`](../docs/demo/v0.1-script.md) verbatim. Target 90 s flat, 1080p / H.264. Commit to `docs/demo/v0.1.mp4` (or link to an external host in the README).
- [ ] **Clock TTFQ on a clean VM/container.** From `pipx install <published-name>` to first successful query, under 5 minutes. If over, fix the README onboarding path or the install path. This is the PRD §16.7 usability gate.
- [ ] **Version bump.** `0.1.0.dev0 → 0.1.0` in both `pyproject.toml` and `contextd/__init__.py`. Single commit: `chore(release): bump version to 0.1.0`.
- [ ] **Tag + push.**
  ```bash
  git tag v0.1.0 -m "contextd v0.1.0 — local-first MCP personal RAG"
  git push origin master --tags
  ```

## Nice-to-have (non-blocking)

- [ ] **PyPI trusted publish** via `.github/workflows/release.yml` — the skeleton is in [`docs/plans/05-phase5-polish.md`](../docs/plans/05-phase5-polish.md) Task 10 Step 5. Required only if the README's `pipx install <published-name>` path is meant to work on day one.
- [ ] **File the v0.1.1 backlog as GitHub issues** — see the audit report for the 13 non-blocking items (mutable defaults in frozen dataclasses, VectorStore dim verification, `_split_by_budget` sentence overflow, serve.py startup race, etc.).

---

## State at time of writing

- **Branch:** `master` at `5f891ce` (same as `phase-5-polish`).
- **Tests:** 114 pass, 2 deselected (slow).
- **Gates:** ruff format + check + mypy all clean.
- **Real eval on fixtures:** Recall@5 = 1.0, MRR = 0.878, gate_passed = true (with `--no-rerank --no-judge`; rerank + judge untested locally because no `ANTHROPIC_API_KEY`).
- **Version:** `0.1.0.dev0`.
