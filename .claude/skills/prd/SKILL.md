---
name: prd
description: Use before making an implementation choice in this repo. Surfaces the relevant PRD section from docs/PRDs/ so the decision stays grounded in the v0.1 spec. Accepts a topic/keyword (e.g., "chunking strategy", "MCP tools", "eval"). With no argument, prints the PRD table of contents.
---

# /prd — ground an implementation choice in the PRD

The `contextd` PRD in `docs/PRDs/` is the default spec for v0.1. Before choosing a library, schema, module boundary, chunking strategy, or scope-adjacent change, pull the relevant section.

## Usage

- `/prd` — with no argument, read the table of contents from `docs/PRDs/part1_front_s1-s7.md` (lines 1–40) and list section numbers + titles.
- `/prd <topic>` — grep `docs/PRDs/` for `<topic>`, identify the most relevant 1–3 sections, read only those ranges (not the whole file), and summarize the PRD's position in 3–6 bullets.

## Flow

1. Receive `$ARGUMENTS` as the topic (may be empty, multi-word, or a section number like "§15.4").
2. If empty → read the ToC and list section numbers + titles, nothing else.
3. If a section number is given (`§N` or `section N`) → jump directly to that section across the `partX_*.md` files.
4. If a topic is given → run `Grep` for the topic across `docs/PRDs/*.md` with `output_mode=content` and `-n=true`. Rank hits by density (section headings beat prose mentions).
5. Read a tight window around the top 1–3 hits (50–100 lines each) — never the whole file.
6. Return:
   - **PRD says:** 3–6 bullets summarizing the PRD's chosen approach, pins, and rationale for `<topic>`.
   - **Explicit scope note:** whether the topic is in v0.1 (M1–M10 + S5/S6) or deferred to v0.2+ (see §16).
   - **File refs:** `docs/PRDs/partN_*.md:line` for each source bullet so the user can jump to it.
   - **Open flags:** anything PRD §23 (Open Questions & Decisions Log) explicitly left unresolved on this topic.

## Guardrails

- Do not summarize sections you did not read. If grep misses, say so.
- Do not paraphrase scope. Quote the PRD's exact wording on Must-have / Should-have / deferred status.
- If the topic spans the Py↔TS boundary (PRD §13.7.3), flag both sides.
- If the topic touches regulated data (PHIPA), refuse to conflate it with the personal corpus — cite §2.8.3 and §4.2.

## Example

Input: `/prd chunking for PDFs`

Output shape:
```
PRD says:
- PDFs use pymupdf4llm 0.0.17 with pypdf 5.1 fallback (§13.6.2)
- Section-aware chunking, 512-token target, 64-token overlap (§14.2)
- AGPL license on pymupdf → see §19.3.3 for mitigation
- Quality bar: § detection ≥ 80% on fixture set (§14.2 exit criteria)

Scope: v0.1 Must-have M1 (PDF ingestion). Phase 2 deliverable (§16.4).

Refs:
- docs/PRDs/part2_s8-s13.md:1491
- docs/PRDs/part3_s14-s16.md:§14.2

Open flags: none on this topic.
```
