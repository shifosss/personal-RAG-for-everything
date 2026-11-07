# `contextd` — Product Requirements Document

**Version:** 0.1.0
**Date:** April 19, 2026
**Author:** Alex Zhang (Chen Zhang)
**Status:** Initial complete draft

---

## Table of contents

1. [Executive Summary](#section-1--executive-summary)
2. [Background & Significance](#section-2--background--significance)
3. [Problem Statement](#section-3--problem-statement)
4. [Target Users & ICP](#section-4--target-users--icp)
5. [Product Vision & Positioning](#section-5--product-vision--positioning)
6. [Goals & Non-Goals](#section-6--goals--non-goals)
7. [User Stories & Use Cases](#section-7--user-stories--use-cases)
8. [Functional Requirements](#section-8--functional-requirements)
9. [Non-Functional Requirements](#section-9--non-functional-requirements)
10. [System Architecture](#section-10--system-architecture)
11. [Data Model](#section-11--data-model)
12. [Interface Specs](#section-12--interface-specs)
13. [Tech Stack & Dependencies](#section-13--tech-stack--dependencies)
14. [Ingestion Adapter Specs](#section-14--ingestion-adapter-specs)
15. [Retrieval Pipeline Spec](#section-15--retrieval-pipeline-spec)
16. [2-Day Build Plan](#section-16--2-day-build-plan)
17. [Evaluation Methodology](#section-17--evaluation-methodology)
18. [Success Metrics & KPIs](#section-18--success-metrics--kpis)
19. [Risks & Mitigations](#section-19--risks--mitigations)
20. [Competitive Analysis](#section-20--competitive-analysis)
21. [Go-to-Market & YC Positioning](#section-21--go-to-market--yc-positioning)
22. [Roadmap Beyond v0.1](#section-22--roadmap-beyond-v01)
23. [Open Questions & Decisions Log](#section-23--open-questions--decisions-log)
24. [Appendix](#section-24--appendix)

---

# Section 1 — Executive Summary

## 1.1 One-sentence pitch

`contextd` is a local-first, open-source context layer that unifies your PDFs, past AI conversations, code, and notes behind a single MCP endpoint any AI agent can query.

## 1.2 Problem in one paragraph

Knowledge workers who use AI heavily now operate across multiple assistants — Claude, ChatGPT, Codex, Cursor, Gemini — each with isolated memory. Their most valuable working context (papers read, decisions made, code written, prior conversations with each AI) is scattered across local files, app databases, exports, and browser history. Every new session starts cold, and users spend tokens and time re-establishing context that already exists elsewhere on their machine. Existing solutions either lock data into one app (Mem, Notion AI, Obsidian), require a specific OS (Rewind on macOS, Recall on Windows), or focus on general screen capture rather than structured technical corpora.

## 1.3 Solution in one paragraph

`contextd` runs locally as an MCP server. It ingests a user-configured set of corpus sources — PDFs, Claude conversation exports, git repositories, Obsidian vaults, email, arXiv/bioRxiv bookmarks — chunks and embeds them with source-aware strategies, and exposes hybrid retrieval over the MCP protocol. Any MCP-compatible agent (Claude Code, Codex, Cursor, custom agents) can query the user's unified corpus on demand. Data never leaves the user's machine. A CLI and a minimal web UI wrap the same retrieval layer for direct human use.

## 1.4 Key differentiators

- **MCP-first:** works with any compliant agent; not trapped inside one chat app.
- **Local-first:** no cloud dependency, no subscription, no telemetry, no privacy surface.
- **Corpus-native:** section-aware chunking for papers, function-aware for code, turn-aware for conversations — not one-size-fits-all.
- **Cross-OS:** Linux, macOS, and Windows (WSL) from day one, unlike Rewind or Microsoft Recall.
- **Open source (MIT/Apache):** forkable, extensible, community-owned.

## 1.5 Primary user (v0.1)

Alex Zhang (the author): ML/AI research students and engineers who read papers, run experiments, maintain multiple repositories, and depend on AI assistants daily. The tool is built for Alex's workflow first; public ICP is a secondary goal.

## 1.6 Public ICP (v0.2+)

Knowledge workers who use AI tools heavily and feel the friction of re-explaining context to every AI session — researchers, engineers, PMs, analysts, writers. Deliberately broader than v0.1's internal user to validate that the abstraction generalizes.

## 1.7 v0.1 scope (2-day build)

- **Ingestion:** PDFs, Claude.ai conversation exports, local git repositories.
- **Retrieval:** hybrid dense (BGE-M3) + sparse (BM25) with LLM reranking.
- **Surfaces:** MCP server (primary), CLI (secondary), minimal web UI (tertiary).
- **Storage:** SQLite + LanceDB, single-user, no auth.
- **Deployment:** one-command install on Linux / WSL / macOS.

## 1.8 Success criteria for v0.1

- Daily personal usage: `contextd` invoked ≥5 times/day within one week of shipping.
- Retrieval quality: top-5 recall ≥ 80% on a 30-query eval set derived from real research questions.
- Demo readiness: 90-second video showing cross-AI context portability (Claude Code → Codex, same corpus).
- Distribution: public GitHub repo, working `pipx install` or equivalent.

## 1.9 Longer-term vision (humble framing)

`contextd` begins as a personal tool that the author uses daily. If the abstraction proves useful beyond one person, it may generalize into an open primitive for personal context in AI-native workflows — but that ambition is downstream of the tool working well for one user first. Phase 2 work (shared corpora, enterprise access control) is explicitly deferred until v0.1 has demonstrated personal utility over at least three months.

## 1.10 Explicit non-goals for v0.1

- **Not a note-taking app.** `contextd` does not replace Obsidian, Notion, or your text editor. It ingests from them.
- **Not a screen recorder.** No always-on screen capture, no keystroke logging. Ingestion is per-source and explicit.
- **Not a chat app.** The UI is a retrieval lens, not a conversational agent. The conversation happens in Claude/Codex/etc., with `contextd` as a queried backend.
- **Not a team/collaboration tool.** Single-user only; no sharing, no permissions, no multi-tenant concerns.
- **Not a cloud service.** No server to deploy, no account to create, no telemetry to opt out of.
- **Not a commercial product in v0.1.** Open source, no monetization surface until after personal use validates the abstraction.

---

# Section 2 — Background & Significance

## 2.1 Core thesis

AI agents have become competent reasoners but lack persistent memory of their users across sessions and across tools. Every new Claude session, Cursor workspace, or Codex task begins without knowledge of what the user has previously read, written, decided, or discussed. Users compensate by manually re-establishing context: pasting excerpts, retyping project background, reattaching files, summarizing past decisions. The cost scales linearly with the number of AI tools the user operates. A researcher using four assistants a day performs four separate context reconstructions per day.

This cost is observable as: wasted tokens (repaying for the same context), inconsistent reasoning across agents (each sees a partial slice of the user's work), and a ceiling on how much agentic workflows can compound — since each new agent invocation starts from zero.

## 2.2 Why the problem is acute now (2024–2026)

Three shifts have converged, each now measurable.

**Agent proliferation.** WRITER's 2026 AI Adoption in the Enterprise survey found that 97% of executives say their company deployed AI agents in the past year, with 52% of employees already using them. 38% of knowledge workers use generative AI tools daily in their work, up from 11% in 2024, and 65% of organizations now use generative AI in at least one business function. The single-assistant user is now the minority; the multi-assistant user is typical. 51% of enterprises already run AI agents in production as of 2026, with another 23% actively scaling them, and 50% of AI agents currently operate in isolated silos rather than as part of a multi-agent system — creating redundant workflows and shadow AI risk. That siloing is precisely the context fragmentation `contextd` targets.

**MCP standardization.** The MCP TypeScript and Python SDKs reached 97 million monthly downloads by March 2026, a trajectory faster than React's first three years. MCP server downloads grew from roughly 100,000 in November 2024 to over 8 million by April 2025, with 5,800+ MCP servers and 300+ MCP clients now available, and MCP was donated to the newly formed Agentic AI Foundation under the Linux Foundation in December 2025, ensuring vendor-neutral governance. OpenAI, Google DeepMind, Microsoft, and thousands of developers building production agents have adopted MCP, making it the de facto protocol for connecting AI systems to real-world data and tools. Before MCP, a personal context layer would have required writing custom integrations for each vendor; today a single MCP server reaches every major agent.

**The shift from chat to agents.** Gartner projects that by the end of 2026, 40% of enterprise applications will include task-specific AI agents, up from less than 5% in 2025. The global AI agents market hits $10.91 billion in 2026 and is projected to reach $50.31 billion by 2030 at a 45.8% CAGR. Agents amplify the context problem because they chain reasoning steps; missing context at step one compounds by step five.

**The RAG market tailwind.** The global retrieval-augmented generation market is projected to grow from $2.76 billion in 2026 to approximately $67.42 billion by 2034, expanding at a CAGR of 49.12% from 2025 to 2034. MarketsandMarkets projects the RAG market from $1.94 billion in 2025 to $9.86 billion by 2030 at 38.4% CAGR, with enterprises increasingly adopting hybrid AI models that combine generative AI with retrieval, reasoning, and memory modules. The numbers differ across analysts but directionally agree: retrieval-over-personal-context is one of the fastest-growing categories in enterprise AI. `contextd` is the personal, local, open analog of that trend.

## 2.3 Existing approaches and why each falls short

Current solutions cluster into four patterns, each with a structural limit.

**In-app memory (ChatGPT memory, Claude Projects, Cursor rules).** Locked to one vendor. Switching tools means losing memory. Fine for users loyal to one assistant, useless for power users who route tasks across several.

**Screen-capture tools (Rewind, Screenpipe, Microsoft Recall).** Rewind achieves 92% top-1 recall on user benchmarks but is Mac-only and consumes 20–30% of laptop battery due to continuous screen capture and real-time transcription. Screenpipe is cross-OS and open source with 17,500+ GitHub stars, priced at $400–600 one-time. These tools capture indiscriminately and structure minimally. Retrieval over screen frames is weaker than retrieval over source documents for technical research.

**Note-apps with AI (Mem, Reflect, Notion AI).** Mem pioneered self-organizing notes; Reflect focuses on networked thought; Notion AI adds AI to existing Notion databases. These require users to first move everything into the app. Papers read in a browser, code written in an IDE, and conversations with AI assistants never enter the system.

**DIY second-brain projects (Memory Palace, second-brain-agent, Knowledge Nexus).** Memory Palace is an open-source RAG system over documents and notes using Supabase and OpenRouter. second-brain-agent indexes Obsidian-style markdown with LangChain and ChromaDB, and exposes an MCP server. Technically capable, but each is optimized for a single ingestion paradigm (usually Obsidian markdown), and none addresses cross-agent portability as the primary design goal.

None of the existing solutions combine (a) multi-source ingestion with source-specific chunking, (b) MCP as the primary interface, (c) local-first privacy, and (d) open-source distribution. That combination is the gap `contextd` targets.

Notably, the thriving open-source second-brain ecosystem validates the demand but none of the existing projects treat MCP as the primary surface. Most were designed before MCP existed (late 2024) and bolted it on later. `contextd` is designed MCP-first from day one, which shapes every downstream decision — schema, chunking strategy, tool naming, auth model.

## 2.4 The contrarian pattern worth acknowledging

Karpathy's "LLM Knowledge Base" argues that fancy RAG infrastructure introduces more latency and retrieval noise than it solves for personal research, and proposes instead an LLM-maintained markdown wiki. This is the strongest argument against building `contextd` as specified. The response: the wiki pattern optimizes for quality over recall at small corpus sizes and breaks once the corpus exceeds a model's reliable context window (roughly 200–500 documents for current frontier models). `contextd` treats the wiki pattern as compatible rather than competing — an LLM-curated wiki is one ingestible source among many.

## 2.5 Why local-first

The data `contextd` touches — unpublished research, private code, personal conversations, clinical notes — is sensitive by nature. Any cloud-hosted solution faces a trust ceiling for exactly the users who would benefit most. The hardware trend also supports local deployment: consumer GPUs now run BGE-M3 embeddings and 7B–13B rerankers comfortably, and SQLite plus LanceDB handle single-user retrieval at corpus sizes well beyond what a single person generates.

## 2.6 Why open source

For any personal-context substrate to become standard, users must audit what it does with their data. Closed-source tools in this space face permanent adoption skepticism. Open source also unlocks the long tail of ingestion adapters (Kindle highlights, Apple Notes, Slack exports, Roam, Logseq, Zotero) that a single author cannot realistically build and maintain alone.

## 2.7 Significance if the project succeeds

Three outcomes, in descending order of ambition:

1. **A useful personal tool.** The author's research productivity compounds, other researchers fork it for their own corpora, and it reaches a small but engaged open-source audience. This is the humble success.
2. **A de facto reference implementation.** `contextd` becomes the example people point to when discussing personal MCP context layers, even if they use a different implementation. Similar trajectory to projects like `ripgrep` or `fzf` — a primitive tool that shapes expectations.
3. **An emerging standard.** If the ingestion formats and MCP schemas prove durable, they become informal conventions others adopt. This is speculative and not a v0.1 goal.

If none of the above materializes, v0.1 still delivers the first outcome for the author personally, which is sufficient justification for the build.

## 2.8 Legal & regulatory context

Personal context layers touch three distinct legal surfaces. `contextd`'s local-only design avoids the hardest version of each, but users deploying the tool still need to understand what they're responsible for.

### 2.8.1 Copyright of ingested material

Users will ingest copyrighted content — PDFs of published papers, books, articles, excerpts pasted into notes. The tool does not reproduce or redistribute any ingested material. All derived artifacts (embeddings, BM25 indices, chunks) stay on the user's machine. Chunks are never transmitted to third parties unless the user explicitly configures a cloud LLM for query-time reranking or generation.

Relevant legal framework: fair use in the United States and text-and-data-mining exceptions in the EU (notably the DSM Directive Article 3 for research and Article 4 for commercial TDM with opt-out) generally permit computational analysis of lawfully accessed works for personal research. `contextd` is not a lawyer and makes no legal determination; users remain responsible for the licensing status of anything they ingest. The project's documentation will explicitly note this rather than leave it implicit.

Engineering consequence: the system must preserve provenance (source path, ingestion timestamp, original file hash) for every chunk, so a user can audit and purge material that turns out to be outside fair use / TDM scope.

### 2.8.2 PII and GDPR posture

Much of what `contextd` ingests is personal data under GDPR — emails, conversations, notes naming third parties. The GDPR only applies to personal data — data from which a living individual can be identified or is identifiable.

The key GDPR posture for `contextd`:

- **User is both data subject and data controller.** A solo user ingesting their own emails into `contextd` is processing personal data for "purely personal or household activity," which under Article 2(2)(c) of the GDPR falls outside the Regulation's scope. This is the "household exemption" and is the simplest posture.
- **No third-party data processor.** Because the tool runs locally and transmits nothing to cloud services by default, `contextd` itself does not trigger processor obligations. A software company acts as a processor when it stores or has access to the personal data of its customers' customers, staff or contacts — e.g., it provides a SaaS solution and hosts its customers' data. `contextd` hosts nothing.
- **User deploys in a shared/enterprise context.** If a user installs `contextd` on work hardware and ingests work email, the household exemption no longer applies; the employer becomes the controller. Documentation will flag this scenario and recommend against ingesting employer-owned data without authorization.

Engineering consequences:

- **Right to erasure** should be trivial: deleting a source must cascade-delete all chunks, embeddings, and index entries derived from it.
- **Data minimization** means ingestion should be opt-in per source, not default-crawl-everything.
- **Transparency** means the CLI must always be able to answer "what do you currently have indexed and where did it come from?"

### 2.8.3 HIPAA and sensitive sector data

For the author specifically: the author's regulated clinical NLP work must not be ingested into `contextd` as configured for personal use, because the data is covered by HIPAA-equivalent obligations (Canadian PHIPA in Ontario) and leaves the institution's controlled environment only under specific protocols. The tool must make it easy to maintain strict separation between a personal corpus and any regulated professional corpus. This is handled via separate named corpora with no cross-corpus retrieval by default, rather than a single unified index.

Engineering consequence: the data model treats "corpus" as a first-class partition with its own directory, its own index, and its own MCP endpoint, so a user can run one `contextd` for personal work and separately handle regulated data under institutional systems without commingling.

### 2.8.4 AI-specific regulation

European adoption of AI agents prioritizes auditability, explainability, and compliance under GDPR and emerging AI regulations. The EU AI Act's obligations on general-purpose AI systems target model providers, not tools like `contextd` that are thin wrappers over retrieval plus user-selected LLMs. However, any downstream model the user plugs in (Claude, GPT, Gemini, local Llama) inherits its own regulatory posture. `contextd` documentation will note this separation: we are a retrieval layer, not a model provider.

### 2.8.5 Summary

The local-only, single-user, open-source design chosen in Section 1 is not a coincidence — it's the combination that minimizes legal exposure for the v0.1 target users. Any future cloud, multi-user, or team version will reintroduce processor obligations, DPAs, and cross-border transfer concerns that are explicitly out of scope for v0.1.

---

# Section 3 — Problem Statement

## 3.1 The problem in one sentence

Users of multiple AI assistants lose their working context at every tool boundary, and the effort to reconstruct it falls entirely on the user.

## 3.2 The underlying conditions

Four conditions together produce the problem. None is new on its own; the combination is recent.

**Condition A: Users now operate several AI assistants concurrently.** A researcher reads in a browser-based chat, writes code with a terminal agent, refactors with an IDE-embedded agent, and drafts with a general-purpose assistant. Each tool has its own session boundary, its own memory system, and its own definition of "project."

**Condition B: Each assistant's memory is private to itself.** Claude memory, ChatGPT memory, Cursor rules, and Codex project context do not interoperate. The user's mental model of their own work is unified; the tools' models are not.

**Condition C: The source material that would serve as shared context already exists — but in scattered, un-unified form.** Papers sit in a downloads folder. Past conversations are inside vendor chat histories. Code is in git. Notes are in Obsidian or Notion. No tool queries across all of these, because no tool owns them all.

**Condition D: Reconstruction is manual and expensive.** The user retypes project background, re-pastes excerpts, re-attaches files, and re-summarizes decisions at the start of each new session. The cost scales with the number of tools, sessions per day, and corpus depth.

## 3.3 The specific pain, qualitatively

The pain is not "my AI forgot." The pain is the cumulative friction of being the only component of your workflow that remembers everything. Users describe it as:

- Opening a new Claude session and instinctively beginning with "as I mentioned before..." — knowing Claude didn't hear it before.
- Finding a paper in their downloads folder they have a dim memory of reading, and being unable to recall what they concluded about it.
- Switching from a coding agent to a writing assistant mid-project and losing the thread of which decisions were already made.
- Knowing that something they want is somewhere — a conversation, a note, a commit message, a PDF — and having no single search surface that covers all of them.
- Watching context windows consume tokens on material the user has already paid for the model to process, once, in a different session.

Each instance is small. The aggregate over a working week is meaningful, and the aggregate over a year of research is substantial — but the cost is invisible because it never surfaces as a single event.

## 3.4 Why users haven't solved it themselves

Three reasons the problem persists despite being obvious to anyone who feels it.

**Tool-specific solutions don't generalize.** A user can build a Claude Project with attached files. It works only in Claude. A user can write Cursor rules. They work only in Cursor. Every per-tool solution is throwaway work from the moment the user adopts the next tool.

**Generic RAG projects require ingestion discipline the user lacks.** The existing second-brain projects assume the user already maintains an Obsidian vault or equivalent. Most heavy AI users don't — their "second brain" is the sum of their downloads folder, their chat histories, their repos, and their working memory. Tools that demand a single canonical note store impose a precondition the target user hasn't met.

**The protocol to solve it generally didn't exist until recently.** Before MCP (late 2024), a cross-agent context layer would have required a custom integration per vendor — not realistic as a side project. MCP's 2025–2026 adoption is what makes this project tractable in 2026 that wasn't in 2023.

## 3.5 Day-in-the-life: without `contextd`

*A composite day drawn from the author's actual workflow, edges smoothed for clarity.*

**9:15 AM.** Alex opens Claude to start on the MedGemma LoRA pipeline. He pastes four paragraphs of context: what the project is, what was decided last week, which paper's approach is being compared, which repo the code lives in. Claude asks a clarifying question about the dataset; Alex pastes another paragraph. Ten minutes in, real work begins.

**10:40 AM.** A reviewer comment references a technique Alex read about last month. He knows which paper — roughly — but not its title, and the PDF is in a downloads folder with seventy other PDFs. He searches arXiv, re-downloads, re-reads the abstract, gives up on finding the exact claim, and writes a weaker response than he wanted.

**12:20 PM.** Alex switches to Codex in the terminal to restructure an ingestion script. Codex has no idea what the project is. Alex re-explains. Codex suggests an approach that Alex already tried and rejected three weeks ago in a Claude conversation he can no longer find.

**3:00 PM.** A professor emails asking about Alex's approach. To answer, Alex needs to synthesize: the three comparison papers, his own experimental results, a conversation he had with a collaborator in Slack, and a decision he made in a Claude session two Thursdays ago. Synthesis takes forty-five minutes. Most of it is locating the source material.

**6:00 PM.** Alex opens a fresh Claude session to draft an outreach email. He wants it to reference specific findings from his pipeline. He pastes the findings again. Claude drafts. Alex edits. Alex wonders if this is a reasonable way to spend his life.

Each friction point is survivable. The day contains perhaps ninety minutes of pure context reconstruction, distributed across twenty events, none salient enough to fix individually.

## 3.6 Day-in-the-life: with `contextd`

**9:15 AM.** Alex opens Claude to start on the MedGemma LoRA pipeline. He types: "pick up where we left off on the LoRA work." Claude calls `contextd.search_corpus("MedGemma LoRA pipeline")`, retrieves the last three relevant conversations, the current approach document, and the most recent experimental results. It summarizes the state in two sentences and asks whether the goal today is still the negation-handling improvement. Real work begins in forty seconds.

**10:40 AM.** Alex asks Claude about "that paper on handling negation in clinical extraction we were comparing to." Claude queries `contextd`, retrieves the Fu paper's methods section, quotes the specific claim, and returns the source filename and ingestion date. Alex verifies it matches his memory in fifteen seconds.

**12:20 PM.** Alex switches to Codex. Codex is configured to query the same `contextd` MCP server. It sees the project's recent commits, the relevant conversations about the ingestion script's structure, and the prior-rejected approach. It proposes a different approach and flags the one Alex already rejected.

**3:00 PM.** The professor's email. Alex asks Claude for a synthesis. `contextd` returns the three comparison papers' relevant sections, Alex's own experimental results, the Slack conversation, and the Claude session from two Thursdays ago. Claude drafts a response with citations pointing to source files Alex can verify. Synthesis takes nine minutes.

**6:00 PM.** Outreach email. Alex describes the recipient. `contextd` surfaces his prior successful cold emails, relevant findings, and the recipient's known interests from past research on them. Draft is produced in thirty seconds. Alex edits.

The day still contains real work. The difference is that the user is the researcher, not the archivist.

## 3.7 What success looks like

Success is when Alex stops noticing the tool. Not because it's invisible — he invokes it constantly — but because asking it for context has become as cheap as asking his own memory, and he trusts the answers enough to stop re-verifying. That is the bar.

---

# Section 4 — Target Users & ICP

## 4.1 Tiering structure

Three concentric circles, Alex-first.

- **Primary (v0.1):** Alex Zhang. The author. The tool is built for his workflow, validated against his use, iterated on based on his daily friction.
- **Secondary (v0.2):** Research students with a workflow structurally similar to Alex's — ML/AI/applied research, reads papers daily, writes code daily, operates multiple AI assistants, has a corpus that exceeds what any one chat app can hold in context.
- **Tertiary (v0.3+):** Heavy AI users broadly — PMs, analysts, writers, consultants who use 3+ AI tools daily and feel the context-fragmentation pain, even if their corpus composition differs.

Each tier is a release gate, not a marketing tier. The tool only expands outward when the previous ring has demonstrated sustained use.

## 4.2 Primary user profile: Alex Zhang

**Role:** Third-year CS undergraduate, Data Science Specialist at University of Toronto (ASIP co-op stream). Currently a research student in clinical NLP.

**AI tool footprint:** Claude Code, Codex CLI, Claude.ai web, ChatGPT, occasional Gemini. Multi-agent Claude Code setup with 13 subagents across opus/sonnet/haiku tiers. Uses MCP servers extensively.

**Corpus composition:**

- Academic PDFs (arXiv, bioRxiv, PubMed) — hundreds, with a high-density subset of ~50 that anchor current work.
- Past Claude.ai conversations — months of accumulated context across research, engineering, and coursework.
- Git repositories — personal projects (Stellaris advisor, clinical NLP scaffolds), forks, coursework.
- Obsidian / markdown notes — lecture notes, experiment logs.
- Email — professor correspondence, co-op outreach.
- Claude Code / Codex session transcripts — the "conversation with machine" layer that existing tools ignore.

**Hardware:** Windows 11 + WSL2 workstation with consumer RTX-class GPU; Slurm-accessible HPC cluster for heavy jobs.

**Daily workflow shape:** morning deep work on research (clinical NLP pipeline), afternoon coursework (GLMs, approximation algorithms), evening personal projects and co-op prep. Transitions across three or four AI tools per day.

**Dominant frictions (derived from actual workflow):**

1. Re-explaining the clinical NLP project at the start of each Claude session.
2. Losing prior-art context (Bannett, Fu, Kaster pipelines) mid-conversation.
3. Disconnect between Claude Code sessions and his own git history.
4. Relitigating architectural decisions (MedGemma vs Qwen, LoRA vs full fine-tune) that were already resolved weeks ago.
5. Missed co-op interview windows because relevant context on each company wasn't surfaceable.

**What Alex is not:** a note-taking purist. He does not maintain a clean Obsidian vault. His "system" is the sum of downloads, chats, commits, and working memory. `contextd` must fit this state, not demand a cleaner one as a precondition.

## 4.3 Secondary users: research students

The generalization of Alex without Alex-specific details. Characteristics shared with the primary user:

- Daily paper reading as a core work input.
- Active codebase they contribute to.
- Multi-assistant AI workflow.
- Corpus that exceeds any single app's context window.
- Technical enough to install a CLI tool and configure MCP servers.
- Privacy-conscious — working with unpublished research, draft theses, or unreleased code.

Variations that don't disqualify:

- Non-biomedical research (physics, systems, pure ML, humanities with code).
- Mac, Linux, or Windows/WSL primary hardware.
- Python, TypeScript, Rust, or Julia stack.

Variations that do disqualify for v0.2:

- No comfort with CLI installation.
- Research that cannot leave an institutional environment (clinical data under HIPAA/PHIPA, classified work, regulated industry data under strict governance).

## 4.4 Tertiary users: heavy AI users broadly

The generalization beyond research. Characteristics:

- 3+ AI tools used daily.
- Corpus of reference material relevant to ongoing work (could be market reports, legal filings, product specs, editorial archives, client briefs).
- Enough technical comfort to run a local tool, or willingness to follow a one-command install.
- Pain with context fragmentation acute enough to motivate adoption of a new tool.

Examples: product managers maintaining spec libraries, analysts with research archives, writers with source-material libraries, consultants with client-context libraries, independent researchers, OSS maintainers tracking their own issue histories.

## 4.5 Anti-personas

Explicit users the tool is not for, to prevent scope creep.

- **Users looking for a note-taking app.** They should use Obsidian, Notion, or Reflect.
- **Users looking for screen-capture recall.** They should use Screenpipe or Rewind.
- **Users who want an "AI personality" or companion.** `contextd` has no personality; it is retrieval infrastructure.
- **Users who won't install a CLI.** v0.1 is opinionated toward technical users; a point-and-click installer is not in scope.
- **Team and enterprise users.** Single-user; no sharing, no access control, no admin surface.
- **Users whose corpus cannot leave their institution.** The household-exemption posture assumes the user owns or is personally authorized to process the data. Clinical, classified, or regulated corpora belong in institutional systems, not in a personal tool.

## 4.6 User sophistication assumptions

For v0.1, we assume the user:

- Can run a shell and install a CLI tool via `pipx` or `uv tool install`.
- Has an MCP-compatible agent already (Claude Code, Codex CLI, or similar) and understands how to point it at a local MCP server.
- Can provide a directory path when asked to ingest a source.
- Reads a README.

For v0.2, we assume the user can still do the above but may need clearer documentation, better error messages, and optional one-command ingestion recipes.

For v0.3+, we may need a GUI installer and a setup wizard. Explicitly deferred.

## 4.7 Recruiting for later phases

v0.2 recruiting: the author's own research network (U of T, lab collaborators), plus organic discovery via the GitHub README, plus a Show HN / r/LocalLLaMA post timed with a working demo.

v0.3+ recruiting: blog posts targeted at each tertiary segment, YC AI Startup School audience as a concentrated gathering of the ICP.

---

# Section 5 — Product Vision & Positioning

## 5.1 One-sentence pitch

`contextd` is a local-first, open-source MCP server that unifies a user's PDFs, past AI conversations, code, and notes into a single corpus any AI agent can query.

## 5.2 Extended pitch (≈150 words)

Knowledge workers who use AI heavily now operate across several assistants — Claude, ChatGPT, Codex, Cursor, Gemini — each with isolated memory. Their most valuable working context is already on their own machine, but scattered across downloads, chat histories, git repos, and note apps. `contextd` runs as a local MCP server that ingests these sources with source-aware chunking strategies, indexes them with hybrid dense-plus-sparse retrieval, and exposes them over the Model Context Protocol. Any MCP-compatible agent can query the user's unified corpus on demand, with full citations back to source files. Data never leaves the user's machine unless the user explicitly routes a query through a cloud LLM. A CLI and minimal web UI wrap the same retrieval layer for direct use. The project is MIT-licensed and designed to be forkable, extensible, and private by default.

## 5.3 Positioning statement

*Using the Geoffrey Moore template — forced discipline, not a marketing flourish.*

**For** technical knowledge workers who use three or more AI assistants daily,
**who** lose their working context at every tool boundary and spend meaningful time reconstructing it manually,
**`contextd`** is an open-source, local-first personal context layer
**that** unifies their scattered source material (PDFs, past AI conversations, code, notes) behind a single MCP endpoint any agent can query.
**Unlike** in-app memory systems (Claude Projects, ChatGPT memory, Cursor rules), which are locked to one vendor, or screen-capture tools (Rewind, Screenpipe), which capture indiscriminately and struggle with structured technical corpora, or note-apps-with-AI (Mem, Reflect, Notion AI), which demand the user first migrate everything into the app,
**`contextd`** is MCP-native from day one, cross-OS, works with the corpus the user already has (not a cleaner one they haven't built), and runs entirely on local hardware.

## 5.4 Three differentiators

### 5.4.1 MCP-first, not MCP-retrofitted

Every other mature second-brain project was designed before MCP existed in late 2024, and integrates MCP as one of several output surfaces. `contextd` is built MCP-first: the MCP tool schemas are the product's contract, the CLI and UI are thin lenses over the same endpoints, and every design decision — from chunking to metadata to query rewriting — is evaluated by how well it serves agent consumption rather than human eyeballs.

The practical consequence: `contextd` fits naturally into the way power users now work, which is not "chat with my notes" but "let my coding agent silently pull the context it needs." The tool is invisible to the user most of the time and the agent is the primary consumer.

### 5.4.2 Corpus-native, not one-size-fits-all

A PDF of a research paper, a turn-by-turn Claude conversation, and a Python module are three different data shapes. Most RAG systems pretend they are the same and chunk everything with a fixed 512-token window. `contextd` uses source-specific strategies: section-aware chunking for papers (abstract, methods, results as distinct units with metadata), turn-aware chunking for conversations (user/assistant turns with speaker context), function-aware chunking for code (via tree-sitter ASTs, preserving scope).

The practical consequence: retrieval quality is materially higher for the queries researchers and engineers actually ask ("what method did Fu use for negation?", "what did I conclude about MoE fine-tuning three weeks ago?") than it would be with generic chunking.

### 5.4.3 Local-first with no trust ceiling

Existing tools force a choice between privacy (Rewind, but Mac-only and screen-capture-heavy) and capability (cloud RAG services, but your data is on their servers). `contextd` runs on the user's machine, embeddings compute locally on consumer GPUs, storage is SQLite and LanceDB in a user-chosen directory, and no network request is made unless the user explicitly configures one. The audit trail is the filesystem.

The practical consequence: users working with unpublished research, draft code, or sensitive correspondence can adopt the tool without the usual risk calculation that blocks cloud solutions. Open source reinforces this — the user can verify the claim, not just trust the vendor.

## 5.5 The wedge: why `contextd` wins (a specific theory)

Five conditions must hold for a project in this category to succeed. `contextd` is designed around the conjunction, not any single factor.

**Condition 1: The protocol exists.** MCP's standardization through 2025 and governance transfer to the Linux Foundation's Agentic AI Foundation in December 2025 means a single server implementation now reaches every major agent. Before late 2024, a cross-agent context layer was not feasible as a solo project. This is the single most important precondition, and it is new.

**Condition 2: Local compute is sufficient.** BGE-M3 embeddings, BM25 indices, and Claude Haiku–class rerankers run comfortably on the hardware the target user already owns. In 2021 this would have required a cloud dependency; in 2026 it does not. The hardware curve has crossed the threshold where local-only is a real choice, not a compromise.

**Condition 3: The pain is acute enough for users to act.** Agent proliferation has reached a tipping point. The single-assistant user is now the minority among technical workers; the multi-assistant user feels the context tax every working day. Users who would have shrugged at this project in 2023 are actively searching for a solution in 2026.

**Condition 4: The incumbent solutions have structural limits.** Vendor-memory solutions cannot interoperate across vendors without the vendors agreeing, which they will not. Screen-capture tools cannot produce high-quality retrieval from screen frames over technical corpora. Note-app-with-AI tools cannot ingest what users don't already put into them. None of these limits is a temporary product gap — each is an architectural choice that blocks the full solution.

**Condition 5: The cultural moment favors open-source, local-first infrastructure.** The 2024–2025 wave of privacy concerns around Microsoft Recall, the success of local-first projects like Ollama and Screenpipe, and Karpathy's public advocacy for user-owned context systems have normalized the posture `contextd` takes by default. A closed SaaS version of this product would face permanent trust skepticism from exactly the users who need it most.

The wedge is the intersection. Any one condition alone is insufficient: MCP without local compute forces cloud; local compute without the pain produces a toy nobody adopts; the pain without open source produces another Mem or Rewind — useful but ceiling-limited. `contextd` is specifically the project that only makes sense when all five hold simultaneously, which is now.

The flip side is honest: if any of the five conditions reverses — MCP fragments, consumer GPUs stop keeping pace, one vendor solves cross-agent memory unilaterally (unlikely), incumbent tools close the quality gap (possible), or the open-source/local-first moment passes — the wedge narrows. The project bet is that all five hold for at least the next 24 months.

## 5.6 Competitive landscape (high-level here; detailed matrix in Section 20)

`contextd` sits in a four-quadrant landscape:

- **Closed, cloud (ChatGPT memory, Claude Projects, Notion AI):** high ease of use, locked to one vendor, cloud-only.
- **Closed, local (Rewind):** strong privacy posture, Mac-only, screen-capture paradigm.
- **Open, cloud (Memory Palace, Knowledge Nexus):** forkable but deployment requires Supabase/Postgres plumbing.
- **Open, local (`contextd`, Screenpipe, second-brain-agent):** the quadrant `contextd` occupies.

Within the fourth quadrant, the differentiation is MCP-first design and corpus-specific chunking. Section 20 lays out the detailed teardown.

## 5.7 What `contextd` is not competing with

Important to state explicitly to prevent mission creep.

- Not competing with Obsidian, Notion, or any note-taking app. `contextd` ingests from them and makes them more useful; it does not replace them.
- Not competing with Claude, ChatGPT, or any LLM. It feeds them.
- Not competing with MCP itself. It is an MCP server, the way `ripgrep` is not a competitor to the POSIX shell but a citizen of it.
- Not competing with Pinecone, Weaviate, or enterprise vector DBs. Those serve teams and production workloads; `contextd` serves one user on one machine.

---

# Section 6 — Goals & Non-Goals

MoSCoW prioritization. Every goal has an attached success criterion — something observable that determines whether the goal was met. Success criteria are binary where possible; where a threshold is used, the threshold is chosen to be both achievable in 2 days and meaningful enough that falling short indicates a real gap.

## 6.1 Must-have (v0.1 ships without these and the product fails)

**M1. Ingest PDFs with section-aware chunking.** *Success:* A folder of 50 research PDFs ingests end-to-end in under 5 minutes on the author's workstation. Every chunk has extracted metadata (title, first-page authors, ingestion timestamp, source filename) and a section label (abstract / introduction / methods / results / discussion / other). A manual spot-check of 10 random chunks shows section labels correct on ≥8 of 10.

**M2. Ingest Claude.ai conversation exports.** *Success:* A full Claude.ai JSON export ingests without errors. Each message becomes a retrievable chunk with role (user/assistant), conversation title, timestamp, and conversation URL. Spot-check on 10 random retrievals shows the conversation context (surrounding turns) is reconstructable.

**M3. Ingest a local git repository.** *Success:* A repository of ≤50k lines of Python/TypeScript ingests in under 3 minutes. Chunks are function- or class-scoped via tree-sitter where the language is supported, whole-file for unsupported languages. Every chunk has file path, commit hash at ingest time, and language metadata.

**M4. Hybrid retrieval (dense + sparse + rerank).** *Success:* On a 30-query hand-built eval set drawn from the author's real research questions, top-5 recall ≥ 80% and top-1 precision ≥ 60%. Hybrid must demonstrably outperform dense-alone and sparse-alone on the same eval set.

**M5. MCP server exposing core tools.** *Success:* A running `contextd` MCP server is callable from both Claude Code and Codex CLI. At minimum, `search_corpus`, `get_source`, and `list_corpora` tools are implemented and return results under the Section 9 latency targets.

**M6. CLI for ingestion and query.** *Success:* `contextd ingest <path>`, `contextd query "..."`, `contextd list`, and `contextd status` all work end-to-end. A new user can go from `pipx install <published-name>` to a first successful query in under 5 minutes following the README.

**M7. Local-only by default, no telemetry.** *Success:* Running `contextd` with no configuration makes zero outbound network requests. Verified with `tcpdump` or equivalent during a 10-minute ingestion + query session. Any cloud call is opt-in via explicit configuration.

**M8. Provenance on every retrieved chunk.** *Success:* Every chunk returned by retrieval includes: source file path, ingestion timestamp, original file hash, chunk offset within source. A user can locate the exact source passage on disk from any retrieval result.

**M9. Deletion cascades.** *Success:* Removing a source via `contextd forget <path>` deletes all chunks, embeddings, BM25 entries, and metadata derived from it. A subsequent `contextd query` cannot return any content from the removed source. Verified on a test corpus with inserted sentinel strings.

**M10. Public GitHub repo with working README.** *Success:* Repo exists at an MIT license, passes its own install instructions on a clean WSL2 environment, and has a runnable demo script that showcases cross-AI context portability (same MCP server queried from two different agents). README is honest about v0.1 scope and known limitations.

## 6.2 Should-have (v0.1 ships without these but is meaningfully worse)

**S1. Ingest Obsidian / markdown directories.** *Success:* A directory of markdown files ingests with wikilink graph preserved (each `[[link]]` becomes an edge in the metadata). Frontmatter is parsed and indexed as metadata.

**S2. Minimal web UI.** *Success:* A single-page UI served by `contextd serve-ui` that provides a query box, streaming results with numbered citations, and click-through to source chunks with timestamps. No auth, localhost-only binding by default.

**S3. Query rewriting via LLM.** *Success:* A user query is expanded into 3–5 semantically diverse sub-queries before retrieval. On the same 30-query eval set, query rewriting improves top-5 recall by ≥ 5 percentage points over raw-query retrieval.

**S4. Watch-mode incremental ingestion.** *Success:* `contextd watch <path>` detects new and modified files under the path and re-ingests them within 30 seconds without requiring restart. Deletions of watched files are also reflected.

**S5. Named corpora (partitioning).** *Success:* A user can maintain separate corpora (e.g., `research`, `personal`, `coursework`) with no cross-corpus retrieval by default. The MCP server exposes the active corpus selection per request.

**S6. Demo video and screenshots in README.** *Success:* A 60–90 second Loom or asciinema recording embedded in the README showing end-to-end flow, plus three static screenshots of the CLI and UI in action.

## 6.3 Could-have (nice if time permits, explicitly optional)

**C1. Ingest Notion export.** *Success:* A Notion HTML/Markdown export ingests page-by-page with page title as metadata.

**C2. Ingest Gmail via MCP.** *Success:* With user-provided Gmail MCP configuration, emails in a user-specified label are ingestible. Attachments deferred.

**C3. Embedding model choice via config.** *Success:* User can swap embedding model (BGE-M3 default, alternatives: nomic-embed, voyage-3-large via API) via a single config line. Re-index handled gracefully.

**C4. Benchmarking harness.** *Success:* `contextd eval <eval-set.json>` runs a fixed query set against the current index, prints recall@k, MRR, and latency distribution. Useful for regression testing.

**C5. Summary view of ingested corpus.** *Success:* `contextd stats` prints source counts by type, total chunks, total embeddings, disk footprint, and date range of ingested material.

**C6. Simple `contextd doctor` diagnostic.** *Success:* `contextd doctor` checks: Python version, tree-sitter grammars installed, embedding model downloaded, SQLite DB integrity, LanceDB integrity. Reports pass/fail for each.

## 6.4 Won't-have in v0.1 — deliberately deferred to v0.2+

These are things I actively want later but explicitly punt to keep v0.1 shippable in 2 days.

**D1. Screen capture / OCR of arbitrary windows.** Deferred to v0.3+. Rewind/Screenpipe do this well; differentiation is not worth the complexity cost in v0.1.

**D2. Automatic ingestion from browser history.** Deferred to v0.2. Requires browser extension or history DB reader per browser; non-trivial and not day-one critical.

**D3. Biomedical-specialized embeddings (SPECTER2, MedCPT).** Deferred to v0.2. BGE-M3 is good enough on a general eval set; domain specialization is a quality improvement, not a capability gate.

**D4. Advanced ranking with cross-encoder reranker.** Deferred to v0.2. v0.1 uses LLM-as-reranker via Haiku; cross-encoder reranker (bge-reranker-v2-m3) is a measurable quality upgrade but adds a second model dependency.

**D5. GraphRAG / knowledge graph over the corpus.** Deferred to v0.3. Interesting research direction, not a day-one necessity.

**D6. Conversational memory across `contextd` queries themselves.** Deferred to v0.2. The primary surface is MCP; conversational continuity belongs to the calling agent.

**D7. Multi-user / shared-corpus support.** Deferred to v0.3+. Requires auth, access control, and a server model — all of which violate the v0.1 local-first posture.

**D8. Packaged desktop app (Electron / Tauri).** Deferred to v0.3. CLI + web UI is enough for the ICP of v0.1.

**D9. Automatic source-type detection.** Deferred to v0.2. v0.1 requires the user to specify source type at ingest time; later versions should infer from path/content.

**D10. Windows-native support without WSL.** Deferred to v0.2. WSL is the supported Windows path for v0.1; native Windows paths introduce sqlite/tree-sitter/pyenv issues not worth solving in 2 days.

## 6.5 Won't-have — permanent non-goals

These are architecturally excluded from the project. Not deferred — *never*.

**N1. Never a cloud SaaS hosted by the project.** The project may offer optional bring-your-own-cloud-LLM routing for reranking or generation, but there will be no "contextd.ai" server holding user data. If a cloud version is ever needed, it is a separate product, not `contextd`.

**N2. Never telemetry-on-by-default.** Usage analytics, crash reporting, and "anonymous" phone-home behavior are permanently opt-in, and the default config must not include them. Verified by the repo-level policy and a CI test.

**N3. Never a wrapper around a single proprietary model.** The project must work with open embedding models and open (or user-chosen) LLMs. A version that hard-depends on one vendor's API as a required dependency is out of scope.

**N4. Never a chat app.** The UI is a retrieval surface with citations, not a conversational agent. Conversational state belongs to the calling MCP agent.

**N5. Never silent data modification.** The tool reads user data and writes only to its own directory. Ingesting a PDF must never modify the PDF. Ingesting a git repo must never mutate the working tree. Verified by a CI test that checks filesystem hashes before and after ingestion.

**N6. Never a "trust us" security posture.** The audit trail is the filesystem and the source code. Claims about data handling must be verifiable by inspection or by running the tool offline.

## 6.6 Success criteria aggregation

The v0.1 release ships when:

- All 10 Must-haves (M1–M10) meet their success criteria.
- At least 4 of 6 Should-haves (S1–S6) meet their success criteria.
- 0–2 Could-haves (C1–C6) — not a blocker either way.
- All Won't-haves (D1–D10, N1–N6) are either implemented or explicitly documented as out-of-scope.

Falling short of this bar means v0.1 is not ready. Exceeding it means v0.1 is slightly over-engineered for the 2-day window and we should have cut more aggressively.

---

# Section 7 — User Stories & Use Cases

## 7.1 Framing

The primary lens is Alex's workflow as the representative power user. Five workflow narratives follow, each capturing a distinct query pattern the system must handle well. After the narratives, a condensed user story list catalogs the full set of behaviors the system is expected to support, including ones that don't merit a full narrative.

The narratives are written in present-tense, descriptive prose — not demo scripts. They deliberately include friction that remains after `contextd` (retrievals that fail, moments where the user intervenes) to stay honest.

## 7.2 Workflow narrative 1: synthesizing prior art for a paper comparison

Alex is preparing a section of a write-up that compares his MedGemma-27B LoRA approach against three recent pipelines in pediatric clinical NLP: Bannett's embedding-classifier, Fu's Flan-T5 fine-tuned extractor, and Kaster's zero-shot GPT pipeline. He has read all three papers at different points over the last two months, annotated some of them, and discussed two of them with Claude in separate sessions.

Without `contextd`, this would mean: locating each PDF in the downloads folder, reopening each, searching within the PDFs for the specific methodology details, checking Obsidian for notes he wrote while reading them, and trying to reconstruct what he concluded from the Claude discussions. The reconstruction would take 40–60 minutes before any actual writing.

With `contextd`, Alex asks Claude: "Compare how Bannett, Fu, and Kaster handle negation in their pipelines, and where my MedGemma-27B LoRA approach diverges. Give me source citations." Claude calls `search_corpus` with a query rewriter that expands the request into four sub-queries (one per pipeline, one for the author's own project). `contextd` returns chunks from all four source types — the three PDFs' methods sections, Alex's prior Claude conversations where he articulated his own approach, and his repository's design doc. Claude drafts a comparison table with inline citations pointing to source filenames and chunk offsets. Alex verifies three of the citations against the original PDFs in one click each, rewrites two sentences where Claude's phrasing was imprecise, and moves on.

Critical behaviors this narrative requires:

- Multi-source retrieval across PDFs, conversations, and code docs within a single query.
- Section-aware chunking for PDFs so the methods section is retrievable distinctly from the abstract.
- Turn-aware chunking for conversations so Alex's own articulations are retrievable as atomic units.
- Query rewriting that expands a compound comparison into retrievable sub-queries.
- Provenance on every chunk so verification is one click.

## 7.3 Workflow narrative 2: recovering a prior debugging session

Alex is hitting a CUDA architecture mismatch on an HPC cluster while running a training job on H100 nodes. He has a vague memory that he encountered something similar in March during the Baichuan-M3-235B fine-tuning work — he solved it, committed the fix, and moved on. He does not remember which repo it was in, which Claude session discussed it, or what the specific fix was.

Without `contextd`, this means: grepping through `~/.claude/projects/` for keywords like "sm_90" or "H100", scrolling through git log across three candidate repos, and possibly re-deriving the solution from scratch because the search didn't find the right artifact.

With `contextd`, Alex asks Claude Code: "Last time I hit a CUDA arch mismatch on H100, what did I change?" Claude calls `search_corpus("CUDA architecture mismatch H100 sm_90")`. `contextd` returns two hits: a Claude conversation from March 14 where the problem was diagnosed, and a git commit from March 16 with the message "Pin cuda-toolkit==12.8.1 for H100 compatibility." The commit diff is retrievable because the repo was ingested. Claude summarizes both in two sentences and asks whether Alex wants to apply the same pin here. The entire recall takes under 20 seconds.

Critical behaviors this narrative requires:

- Code-aware chunking that preserves commit metadata alongside file-level content.
- Cross-corpus retrieval that treats past conversations and past commits as a unified search surface.
- Retrieval latency low enough to feel native (sub-second, ideally sub-500ms).
- Robustness to imprecise queries — the user remembers "CUDA mismatch," not the exact error string.

## 7.4 Workflow narrative 3: preparing for a meeting with a professor

Alex has a meeting scheduled with Prof. Gao to pitch the Stellaris strategic advisor project as a potential CSC494/495 supervised collaboration. Prof. Gao has published in game AI and RL; Alex has exchanged emails with the lab about general interests. Alex wants to walk in with: a clear summary of Prof. Gao's recent relevant work, the specific hooks in that work where the Stellaris advisor fits, the history of his prior correspondence with the lab, and a memory of what he already said in a cold-outreach email three weeks ago so he doesn't repeat himself.

Without `contextd`, this means: Google-searching Prof. Gao's recent papers one by one, re-reading Alex's sent email in Gmail, and trying to remember what talking points he has already deployed. Twenty-plus minutes of manual aggregation.

With `contextd`, Alex asks Claude: "Prep me for the Prof. Gao meeting — pull the most relevant of his recent papers I've read, my prior outreach, and the current state of the Stellaris advisor PRD." `contextd` returns: two of Prof. Gao's papers that Alex had downloaded and ingested, the sent outreach email from three weeks ago (via the optional Gmail MCP integration if configured), and the current version of the Stellaris advisor PRD from his notes directory. Claude produces a one-page brief: three bullet points from Prof. Gao's work most relevant to the pitch, the current ask in Alex's own words, and a callout of what the prior outreach already covered so the meeting doesn't repeat ground.

Critical behaviors this narrative requires:

- Optional email ingestion (gated behind user-explicit Gmail MCP configuration, respecting the privacy posture).
- Ingestion of third-party PDFs (papers Alex did not author but has read).
- Recency-weighted retrieval — the latest version of the PRD matters more than drafts from a month ago.

## 7.5 Workflow narrative 4: catching a decision Alex already made and forgot

Alex is in a Codex CLI session, restructuring the ingestion script for the clinical pipeline. Codex proposes using BitsAndBytes for 4-bit quantization of the MoE layers. Alex has a faint sense that this isn't the right approach but can't remember why.

Without `contextd`, Alex accepts Codex's suggestion, discovers the incompatibility two hours in, and relitigates a decision he'd already made with Claude three weeks ago.

With `contextd`, Codex — configured with the same MCP server — automatically queries `search_corpus` as part of its reasoning about the proposal. `contextd` returns a Claude conversation from March 18 where Alex wrote, in his own words, that "fused 3D expert tensors in MoE architectures are incompatible with BitsAndBytes, HQQ, and AWQ quantization — this is why we're moving away from Baichuan-M3 toward MedGemma-27B." Codex surfaces this in its response, flags the conflict with its own suggestion, and proposes an alternative path.

Critical behaviors this narrative requires:

- MCP-native design so the agent calls `contextd` as part of its routine reasoning, not only when the user explicitly asks.
- Corpus portability so a tool other than Claude (in this case Codex) queries the same corpus.
- Retrieval that surfaces negative results and prior rejections, not only affirmative facts.
- Conversations as a first-class ingestion target — the decision lived in a chat, not in a commit message or note.

## 7.6 Workflow narrative 5: end-of-week retrospective

At the end of a working week, Alex wants a brief retrospective: what he worked on, what decisions he made, which papers he actually engaged with, and which threads are still open. Not for a performance review — for his own planning the following Monday.

Without `contextd`, this is either skipped entirely or performed as a 30-minute manual review across git log, Claude chat history, and an Obsidian daily notes folder he maintains inconsistently.

With `contextd`, Alex runs from the CLI: `contextd query "summarize what I worked on this week, grouped by project" --since "7 days ago"`. The tool returns a timeline view: commits by repo, Claude conversations clustered by topic, papers ingested in the window, and any notes touched. Alex spends five minutes reading it and decides Monday's first task in under ten minutes total.

Critical behaviors this narrative requires:

- Time-range filtering as a first-class query parameter.
- Clustering or grouping of retrieved results (by project, by topic, by corpus) rather than a flat list.
- CLI-native query interface usable without invoking an LLM — the summary is over Alex's own data.

## 7.7 Condensed user story catalog

The narratives above capture flagship workflows. The following list catalogs the full set of expected behaviors, including shorter ones that don't merit narrative treatment. Format: *As a user, I want [action], so that [outcome].* Stories are tagged with the MoSCoW tier from Section 6.

### Ingestion stories

- **[M]** As a user, I want to point the tool at a folder of PDFs and have them indexed without manual per-file setup, so that onboarding my existing downloads folder is trivial.
- **[M]** As a user, I want to import my Claude.ai conversation export file, so that my history of AI conversations becomes part of my searchable corpus.
- **[M]** As a user, I want to ingest a local git repository and have code chunked function-by-function where possible, so that my own code is retrievable at a useful granularity.
- **[S]** As a user, I want to ingest an Obsidian vault with wikilinks preserved, so that my manually curated notes are integrated without losing their graph structure.
- **[S]** As a user, I want incremental re-ingestion as files change, so that I don't re-run a full ingest after every edit.
- **[C]** As a user, I want to ingest from Notion, Gmail, and arXiv bookmarks via optional MCP adapters, so that I can broaden coverage without reinstalling the tool.
- **[C]** As a user, I want ingestion to be interruptible and resumable, so that a failed ingestion on a 500-PDF folder doesn't require starting over.

### Retrieval stories

- **[M]** As a user, I want semantic search across my full corpus by natural-language query, so that I don't need to remember exact filenames or phrasings.
- **[M]** As a user, I want keyword search to work when I remember an exact phrase, so that precise queries aren't diluted by semantic approximation.
- **[M]** As a user, I want hybrid dense + sparse retrieval with reranking, so that both vague and precise queries return relevant results.
- **[S]** As a user, I want to filter retrievals by source type (PDFs only, conversations only, code only), so that I can scope queries when the corpus is large.
- **[S]** As a user, I want to filter retrievals by date range, so that I can ask "what was I working on last month."
- **[S]** As a user, I want query rewriting that expands my question into related sub-queries, so that compound questions don't require manual decomposition.
- **[C]** As a user, I want to filter by named corpus (personal, research, coursework), so that different working contexts stay separated.

### Agent-integration stories

- **[M]** As a user, I want `contextd` exposed as an MCP server, so that Claude Code, Codex, Cursor, and other MCP-compatible agents can query my corpus.
- **[M]** As a user, I want MCP tool schemas that return structured results with citations, so that calling agents can reason over provenance, not just raw text.
- **[S]** As a user, I want MCP retrieval latency to meet the Section 9 targets, so that agents don't stall while waiting on my context layer.
- **[C]** As a user, I want agent-specific query patterns (e.g., a tool specifically for "find a prior decision") in addition to generic search, so that agents can signal intent and get ranking tuned accordingly.

### Direct-use stories

- **[M]** As a user, I want a CLI with `ingest`, `query`, `list`, `status`, and `forget` commands, so that I can use the tool without any agent in the loop.
- **[S]** As a user, I want a minimal web UI at `localhost:XXXX` for ad-hoc browsing, so that occasional non-agent queries have a nicer interface than the terminal.
- **[S]** As a user, I want the web UI to stream results with numbered citations that expand to source chunks, so that I can verify retrievals inline.
- **[C]** As a user, I want a `contextd stats` command that summarizes my corpus by type and size, so that I can understand what's indexed without opening a database.

### Provenance and trust stories

- **[M]** As a user, I want every retrieved chunk to include source file, ingestion timestamp, and chunk offset, so that I can always locate the original passage.
- **[M]** As a user, I want `contextd forget <path>` to cascade-delete all derived data, so that removal is a real operation, not a soft hide.
- **[M]** As a user, I want zero outbound network requests by default, so that I can verify the tool is local-only by inspection.
- **[S]** As a user, I want a diagnostic command that verifies no ingestion step mutated source files, so that I can trust the tool hasn't touched my originals.
- **[C]** As a user, I want an append-only audit log of ingestions and deletions, so that I can reconstruct what was in my corpus at any past point.

### Deployment and ops stories

- **[M]** As a user, I want a single-command install (`pipx install <published-name>` or `uv tool install <published-name>`), so that setup isn't a project.
- **[M]** As a user, I want the tool to work on Linux, macOS, and Windows via WSL2, so that my actual development environment is supported.
- **[S]** As a user, I want a watch-mode daemon that re-ingests changed files automatically, so that my corpus stays current without manual action.
- **[C]** As a user, I want a `contextd doctor` command that checks dependencies and configuration, so that setup problems are self-diagnosable.

### Anti-stories (what we are deliberately not supporting)

- *NOT supported:* As a user, I want to record my screen continuously and OCR it. *(Use Screenpipe.)*
- *NOT supported:* As a user, I want to chat with my notes conversationally inside `contextd`. *(The calling agent is the chat; `contextd` is retrieval.)*
- *NOT supported:* As a user, I want to share my corpus with teammates. *(v0.1 is single-user; team support is a permanent non-goal for this codebase.)*
- *NOT supported:* As a user, I want the tool to run without me installing anything. *(v0.1 is CLI-first; GUI installers are deferred to v0.3+.)*

## 7.8 Coverage check

Every Must-have goal in Section 6 is covered by at least one user story above. Every user story above maps to at least one goal or is explicitly tagged as an anti-story. This is the coverage discipline: no stories that don't serve goals, no goals without a story that would exercise them.

---

