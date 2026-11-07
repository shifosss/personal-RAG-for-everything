# Section 20 — Competitive Analysis

## 20.1 Framing

`contextd` occupies the intersection of three adjacent categories: personal knowledge management, AI memory systems, and retrieval infrastructure. No existing tool covers all three equally, which is precisely the opportunity. This section identifies twelve competitors across those adjacencies, rates each along ten dimensions, and discusses the top six in prose.

The point of this exercise is not to declare `contextd` a winner — it's to identify (a) where differentiation is real and defensible, (b) where overlap means `contextd` must be meaningfully better to justify its existence, and (c) what each competitor does well that `contextd` can learn from.

## 20.2 Competitor set

Twelve products selected to span the relevant quadrants of §5.6. The set is deliberately mixed: closed-source commercial products, open-source projects, and in-app vendor features.

| # | Name | Category | License / model |
|---|---|---|---|
| 1 | **Rewind** | Screen capture + AI memory | Closed, paid, Mac-only |
| 2 | **Screenpipe** | Screen capture + AI memory | Open source (partial paid tiers) |
| 3 | **Microsoft Recall** | Screen capture (OS-integrated) | Closed, bundled with Copilot+ PCs |
| 4 | **Mem** | AI note-taking with memory | Closed, SaaS subscription |
| 5 | **Reflect** | Networked-thought notes + AI | Closed, SaaS subscription |
| 6 | **Notion AI** | Note app + AI overlay | Closed, SaaS subscription |
| 7 | **Memory Palace** | Open-source personal RAG | Open source, self-hosted |
| 8 | **second-brain-agent** (flepied) | Open-source Obsidian RAG + MCP | Open source, self-hosted |
| 9 | **Knowledge Nexus** | Open-source GraphRAG for notes | Open source, self-hosted |
| 10 | **Obsidian + Smart Connections plugin** | Self-organized notes + semantic search | Free / freemium plugin |
| 11 | **Claude Projects** / **ChatGPT Memory** | In-app memory from a vendor | Bundled with subscription |
| 12 | **Cursor / Claude Code built-in memory** | IDE-embedded agent memory | Bundled |

## 20.3 Evaluation dimensions

Ten dimensions covering the axes that matter for `contextd`'s positioning. Each scored on 0–3: 0 absent, 1 partial, 2 good, 3 best-in-class.

1. **MCP-native** — first-class MCP server, designed from day one for agent consumption.
2. **Local-first** — runs entirely on user hardware; no required cloud.
3. **Open source** — auditable, forkable, community-owned.
4. **Multi-source ingestion** — PDFs, conversations, code, notes together.
5. **Corpus-specific chunking** — section/turn/function awareness vs. generic token windows.
6. **Cross-agent portability** — same memory usable across Claude, Codex, Cursor, Gemini.
7. **Privacy posture** — verifiable data handling, no telemetry-by-default.
8. **Install simplicity** — time from discovery to first successful query.
9. **Developer extensibility** — adapter API, plugin system, scriptable.
10. **Active maintenance** — recent commits, real user base, not stagnant.

## 20.4 The matrix

| Competitor | MCP-native | Local-first | Open source | Multi-source | Corpus chunking | Cross-agent | Privacy | Install ease | Extensible | Active |
|---|---|---|---|---|---|---|---|---|---|---|
| **contextd (v0.1 target)** | 3 | 3 | 3 | 2 | 3 | 3 | 3 | 2 | 2 | — (new) |
| Rewind | 0 | 2 | 0 | 1 | 0 | 0 | 2 | 3 | 0 | 3 |
| Screenpipe | 1 | 3 | 3 | 1 | 0 | 1 | 2 | 3 | 2 | 3 |
| Microsoft Recall | 0 | 2 | 0 | 1 | 0 | 0 | 1 | 3 | 0 | 2 |
| Mem | 0 | 0 | 0 | 1 | 1 | 0 | 1 | 3 | 0 | 2 |
| Reflect | 0 | 0 | 0 | 1 | 1 | 0 | 1 | 3 | 1 | 2 |
| Notion AI | 0 | 0 | 0 | 2 | 1 | 0 | 1 | 3 | 1 | 3 |
| Memory Palace | 1 | 2 | 3 | 2 | 1 | 1 | 2 | 1 | 2 | 2 |
| second-brain-agent | 2 | 2 | 3 | 1 | 1 | 2 | 2 | 1 | 2 | 2 |
| Knowledge Nexus | 0 | 2 | 3 | 1 | 1 | 0 | 2 | 1 | 1 | 1 |
| Obsidian + Smart Connections | 0 | 3 | 2 | 1 | 1 | 0 | 2 | 2 | 2 | 3 |
| Claude Projects / ChatGPT Memory | 0 | 0 | 0 | 1 | 1 | 0 | 1 | 3 | 0 | 3 |
| Cursor / Claude Code memory | 1 | 1 | 0 | 1 | 1 | 0 | 1 | 3 | 0 | 3 |

Reading the matrix:

- **The MCP-native column is where `contextd` is structurally different.** Only `second-brain-agent` reaches a 2; `contextd`'s 3 is the entire wedge.
- **Corpus-specific chunking is the quality differentiator.** Every competitor scores 0 or 1 here — most RAG tools use generic token-window chunking. This is where retrieval quality is won.
- **Cross-agent portability is basically empty.** Only `second-brain-agent` (via MCP) reaches a 2. This is the "works with every AI you already use" story that `contextd` can credibly tell.
- **`contextd` starts at 0 on active maintenance.** The honest weakness: brand-new, unproven. Months of consistent releases will close this.

## 20.5 Prose teardowns: the top six

Six competitors deserve deeper discussion because they represent the strongest alternative for some plausible user.

### 20.5.1 Rewind

**One-line positioning:** the polished Mac-native memory app that records everything on your screen and makes it searchable.

**Strengths that matter:**
- 92% recall accuracy on queries like "email from Sarah about budget," compared to 88% for Limitless and 78% for Memex — among the best actual recall numbers in the personal memory space.
- Local-first: no cloud upload unless you enable team sync; EFF audited the tool in September 2024 and found zero privacy leaks in simulated attack scenarios.
- Genuinely delightful install experience. One click, runs.
- Strong brand recognition.

**Where `contextd` wins:**
- Cross-OS. Rewind only works on Macs running macOS 13 or later. If you use Windows at work or Android on mobile, Rewind is useless, and the company has shown no signs of expanding beyond Apple's ecosystem as of April 2026.
- Open source, auditable.
- Structured source types vs. indiscriminate screen capture. Rewind has excellent recall over "what was on screen" but weaker relevance over "what does this paper's methods section actually say" because the data is OCR'd screen frames, not structured document extraction.
- No battery impact — `contextd` only runs when invoked. Rewind's battery impact at 20 to 30 percent drain on laptops is the worst in its category.
- MCP-native agent integration. Rewind exposes search via its own Mac-only API, not via a cross-vendor protocol.

**Where Rewind wins:**
- Ease of capture. `contextd` requires the user to point it at specific sources; Rewind captures automatically.
- Breadth of capture by default. Rewind sees everything that was on screen, including things `contextd` would miss (Slack DMs, browser tabs, ephemeral windows).
- UX polish. Rewind looks like a product; `contextd v0.1` looks like a CLI.

**Likely user overlap:** low. Rewind is bought by knowledge workers on Macs who value convenience over structure. `contextd`'s user is a technical researcher on mixed hardware who values structure and cross-agent integration.

**What we can learn:** the install-to-first-value pipeline. Rewind's "install and go" gets a user to value in under two minutes. `contextd`'s 5-minute target for time-to-first-query (§9.5.2) is inspired by this bar.

### 20.5.2 Screenpipe

**One-line positioning:** open-source Rewind. Cross-OS. Developer-friendly.

**Strengths that matter:**
- 17,500+ GitHub stars, 10,000+ Discord members. Real community.
- Genuinely cross-OS: works on macOS, Windows & Linux.
- Open source. Auditable. Forkable.
- Pay-once, use-forever pricing model — $400 lifetime / $600 with pro included. No subscription tax.
- Strong developer extensibility via "pipes" (scheduled AI agents) and plugin ecosystem.

**Where `contextd` wins:**
- Retrieval quality on technical corpora. Screenpipe retrieves over screen frames and transcribed audio; `contextd` retrieves over structured source material (PDFs with section labels, code with function scopes, conversations with turn boundaries). For a research query, `contextd` surfaces a paper's methods section directly; Screenpipe surfaces screenshots of whatever page was visible when.
- MCP-first. Screenpipe's primary interface is its own API and pipe system, not MCP.
- No screen capture. `contextd` explicitly does not watch the screen — meaningful for privacy-sensitive users.
- Lighter resource footprint. Continuous screen capture is expensive; `contextd` is idle by default.

**Where Screenpipe wins:**
- Breadth of capture, same as Rewind.
- Community and mind-share. Screenpipe has a 18-month head start in the open-source personal-memory space.
- Revenue model validated. Proves users will pay $400 one-time for this class of tool.

**Likely user overlap:** medium. Technical users who value open source and cross-OS could choose either. `contextd`'s MCP-native design and structured-source focus should distinguish for the research-leaning subset.

**What we can learn:** the pay-once pricing model is a genuine distinguisher vs. SaaS fatigue. `contextd` is currently free/open — but if monetization ever becomes relevant, Screenpipe's model (optional paid features, no subscription on the core) is the template to copy.

### 20.5.3 Mem

**One-line positioning:** AI-first note-taking where the AI does the organization.

**Strengths that matter:**
- Self-organizing notes — users don't build folder structures; the AI handles categorization.
- Best for teams and individuals who take lots of meeting notes and want automatic organization.
- Polished UX.
- Strong retrieval within its own corpus.

**Where `contextd` wins:**
- Multi-source ingestion vs. notes-only. Mem expects users to migrate into Mem; `contextd` ingests where the content already lives.
- Local-first. Mem is cloud-SaaS; users' notes are on Mem's servers.
- Open source.
- Cross-agent. Mem's retrieval serves Mem's UI, not arbitrary agents.

**Where Mem wins:**
- Zero-friction onboarding for the "notes-only" use case.
- Cleaner UX for users who want to browse, not just query.
- Collaboration features (teams).

**Likely user overlap:** low. Mem targets users who want a notes app; `contextd` targets users who want a context layer under their agents. A user who picks one is unlikely to also need the other.

**What we can learn:** automatic organization (inferring tags, clusters, and connections from content) is a feature `contextd` does not have in v0.1. For v0.2+, lightweight automatic clustering at ingestion time (based on embeddings) could make the `list_sources` output more useful.

### 20.5.4 Memory Palace (open-source)

**One-line positioning:** NotebookLM meets Obsidian, but open source, self-hostable, and powered by whatever LLM you want.

**Strengths that matter:**
- Dual brain design — handles both external research (PDFs, articles, documents) AND internal thoughts (notes, ideas, memories).
- Combines vector similarity with PostgreSQL full-text search for hybrid retrieval.
- Answers stream in real-time, token by token, with real-time status updates during search.
- Uses Supabase + OpenRouter — users can bring any LLM.
- Fully open source; BYOK for cost control.

**Where `contextd` wins:**
- MCP-native. Memory Palace is a web app with its own UI; it's not designed to be queried by arbitrary agents.
- Deployment simplicity. Memory Palace requires Supabase + Redis + three Node processes. `contextd` is one `pipx install`.
- Corpus-specific chunking. Memory Palace uses generic chunking per its own documentation.
- Fully local option. Memory Palace assumes cloud Supabase; `contextd` runs entirely on the user's machine.
- Extensibility — Memory Palace's architecture is monolithic web app; `contextd`'s adapter model invites contributions.

**Where Memory Palace wins:**
- Has a UI. Memory Palace is usable by non-technical users; `contextd` CLI isn't.
- More mature retrieval UX (streaming, multi-query expansion).
- Longer track record on real use (2024–2025 releases).

**Likely user overlap:** medium. A user who wants "notebook LM-like experience, self-hosted" will pick Memory Palace. A user who wants "my agent calls my corpus" will pick `contextd`.

**What we can learn:** streaming UX with real-time status matters for perceived quality. The Web UI (Should-have S2) should adopt this pattern — users see retrieval happening, not a blank spinner.

### 20.5.5 second-brain-agent (flepied)

**One-line positioning:** Automatically indexes markdown files and their linked content (PDFs, YouTube videos, web pages), with MCP-powered retrieval.

**Strengths that matter:**
- MCP-powered retrieval — built-in MCP server pulls the most relevant context from notes into the LLM or workflow of your choice. Closest existing project to `contextd`'s philosophy.
- Built on LangChain and ChromaDB vector store; automatic indexing via filesystem watch.
- Domain-based document classification — supports filters like Workout, Journal, Project.
- systemd services to manage automatically the different scripts when the operating system starts.

**Where `contextd` wins:**
- Source-type breadth. second-brain-agent is Obsidian-and-linked-content-centric. `contextd` treats PDFs, code, and conversations as first-class, not link targets.
- Corpus-specific chunking. second-brain-agent uses LangChain defaults (generic token windows with overlap).
- Structured MCP schema. second-brain-agent's MCP tools (`search_documents`, `get_document_count`) are minimal; `contextd`'s 7-tool set supports chaining patterns.
- Cross-OS install. second-brain-agent requires inotify-tools; tested under Fedora Linux 42; Ubuntu latest in CI workflows — macOS and Windows are secondary.

**Where second-brain-agent wins:**
- First to market with MCP-native personal RAG. Has a user base and real feedback.
- Proven architecture with systemd integration — genuine "set and forget."
- Simpler dependency graph (LangChain + Chroma is well-understood).
- Tighter Obsidian integration than `contextd v0.1` will have.

**Likely user overlap:** high. This is the closest direct competitor. Users who maintain Obsidian vaults and want MCP access may prefer second-brain-agent's tighter integration.

**What we can learn:** (a) systemd service integration is a polish item `contextd` should adopt quickly; (b) domain-based filtering (their equivalent of named corpora) is a feature users reach for; (c) shipping an MCP-native RAG in 2024 before the ecosystem was ready is impressive and proves the niche exists.

### 20.5.6 Claude Projects / ChatGPT Memory (vendor in-app)

**One-line positioning:** the vendor's own attempt at persistent context, scoped to their product.

**Strengths that matter:**
- Claude excels at project-scoped context.
- Zero setup. Already there when you open the app.
- Integrated with the vendor's core model so retrieval shapes reasoning without explicit tool calls.
- The user already pays the subscription.

**Where `contextd` wins:**
- Cross-vendor. The user's memory in Claude is invisible to Codex, Cursor, Gemini. `contextd` is the shared layer.
- No data retention by a vendor. A user's Claude Projects content lives on Anthropic's servers. `contextd` content stays on the user's machine.
- File-system-scale corpora. Claude Projects caps what you can attach; `contextd` has no cap other than disk.
- Ingestion from where content lives. Users don't migrate PDFs into Claude; `contextd` ingests the downloads folder directly.

**Where vendor memory wins:**
- Native model integration. When the user says "continue what we were doing," Claude Memory has zero latency and deep model-level integration. `contextd`'s MCP round-trip is a weaker substitute for the vendor's native context injection.
- Reliability via the vendor's infrastructure.
- Doesn't require running anything locally.

**Likely user overlap:** high on a per-vendor basis, but the overlap evaporates when the user operates multiple vendors — which is the `contextd` thesis.

**What we can learn:** the "it just works" feel of in-app memory is the UX bar. `contextd` must feel equally weightless via MCP integration — any friction here destroys the cross-agent value proposition. If an agent invocation feels heavier than using Claude Projects natively, users won't bother.

## 20.6 The not-really-competitors

Four tools worth mentioning briefly because users may confuse them with competitors; they aren't.

- **Obsidian itself:** a note-taking app. `contextd` ingests from Obsidian; does not replace it.
- **Pinecone / Weaviate / Qdrant:** vector databases for enterprise production. `contextd` uses LanceDB for similar capability at single-user scale but isn't competing for enterprise workloads.
- **LlamaIndex / Haystack:** RAG frameworks for developers to build their own RAG apps. `contextd` is the built-RAG-app, not a framework to build one from.
- **Perplexity / You.com / ChatGPT Search:** web search with RAG. Answer questions over the public web. `contextd` answers questions over the user's private corpus.

## 20.7 Composite picture

Three takeaways from the matrix and prose:

**Takeaway 1: The MCP-first + corpus-specific combination is genuinely uncovered.** Every competitor scores at most a 2 on MCP-native and most score 1 on corpus-specific chunking. `contextd`'s combined score of 6 across these two dimensions is the highest. That's the wedge.

**Takeaway 2: The strongest direct competitor is second-brain-agent.** Closest philosophy, closest architecture, earliest to MCP. `contextd`'s differentiation is broader source types, deeper structural chunking, and cross-OS first-class support.

**Takeaway 3: Vendor in-app memory is the hardest to beat per-vendor, and the easiest to beat across vendors.** A user loyal to one assistant has no reason to use `contextd`. A user bouncing between three assistants has a compelling reason. This is why the ICP tiers in §4 explicitly target multi-agent users.

## 20.8 What this analysis does not claim

Honest boundaries:

- **`contextd` is not "better" than any of these on all axes.** It is better on the axes that matter for the stated ICP.
- **Scores in the matrix are my assessment, not empirical benchmarks.** A user benchmarking for their specific use case might score differently.
- **The competitive landscape moves monthly.** This matrix reflects April 2026; by the YC AI Startup School in July 2026, one or more tools may have materially changed. A re-evaluation is scheduled for June 2026.
- **"Active maintenance" of 3 on multiple competitors means those teams have capital and attention `contextd` does not yet have.** Catching up on raw product polish is a multi-quarter effort.

---

# Section 21 — Go-to-Market & YC Positioning

## 21.1 Framing

Two distinct motions:

- **YC AI Startup School track (§21.2–§21.4):** application, acceptance (or not), event. A specific deadline and format; requires tailored materials.
- **Broader GTM (§21.5–§21.9):** open-source launch, community building, early-adopter acquisition. Independent of YC outcomes.

The two motions share underlying materials (demo video, README, positioning) but differ in audience, tone, and success metric. YC evaluators read thousands of applications and skim; broader GTM audiences self-select and go deeper.

A principle governing both: **the product is the pitch.** A working `contextd` that Alex uses daily is a stronger argument than any slide. The GTM plan is about surfacing that evidence, not manufacturing it.

## 21.2 YC AI Startup School — application strategy

### 21.2.1 Event context

YC Startup School 2026 is July 25–26 in San Francisco, hand-selecting the most promising builders in the world. Benefits include direct access to YC partners and alumni, $25K+ in compute credits from OpenAI, Anthropic, and more, up to $500 flight reimbursement, targeted at undergraduate and graduate students in CS, software engineers, and ML engineers. The event serves as a filtering mechanism and a direct pipeline into YC's funding batches.

### 21.2.2 Application deadline and cadence

- Application open date: typically announced 2–3 months before the event.
- Alex's readiness milestones:
  - **By April 30, 2026:** v0.1.0 shipped, README polished, demo video live.
  - **By May 15, 2026:** 30 days of dogfood data; personal-use KPIs (§18.2) collected for at least 3 weeks.
  - **By June 1, 2026:** some external evidence of use (at minimum GitHub stars, PyPI installs, or a Show HN thread).
  - **Application submitted:** as soon as it opens, likely May–June 2026.

### 21.2.3 Application positioning — structural talking points

YC partners reading the application will skim. The order of information matters.

**Lead (first sentence):** what `contextd` is, in concrete terms. Not "we're building the context layer for agentic AI" (jargon-heavy, abstract) but something like "`contextd` is a local MCP server that lets your AI assistants query your PDFs, past conversations, and code from a single place."

**The demo (second sentence or link):** the 90-second video from §16.8. The moment where the same query flows through Claude Code and Codex with identical results is the visual proof of the thesis.

**The builder story (third paragraph):** Alex's background matters here — YC partners review thousands of applications; write what is true, not what sounds impressive. The authentic version: third-year CS student at UofT, current research student doing clinical NLP, built `contextd` because he was personally losing context across Claude, Codex, and Cursor every day. The specificity is the credibility.

**The traction (fourth paragraph):** whatever is true at application time. GitHub stars, PyPI installs, external users if any. If traction is thin, acknowledge it honestly and compensate with evidence of personal use depth ("I've used it for N queries across M sessions over K weeks").

**The vision (fifth paragraph):** the humble-ambitious framing from §1.9. Phase 1: useful personal tool. Phase 2: open primitive if it generalizes. Phase 3: standard if it becomes foundational. Not "we will dominate the $11B RAG market" — that pattern is exhausting to read.

**The ask (if required by the form):** attendance at AI Startup School. Not "funding" — that's a YC batch application, which is a separate motion.

### 21.2.4 What to avoid in the application

Common failure modes for student applications:
- **Jargon-saturated abstractions** ("agentic context infrastructure for the multi-model era"). Replaceable with concrete nouns.
- **Market-size arguments as the lead.** YC partners know the RAG market is large; they need evidence of a specific approach working.
- **Overclaiming traction.** 83 stars is 83 stars, not "strong early traction."
- **Apologizing for being a student.** Plenty of YC founders were students; the right framing is "I'm at the ICP of the tool, using it daily."
- **Hypotheticals about future features.** Stay in the world of what exists.

### 21.2.5 If accepted

- Attend in person. Both days. Non-negotiable.
- Prepare a 10-second pitch and a 60-second version.
- Have the demo ready to run from a laptop on any network — no live internet dependency. Pre-downloaded everything.
- Go with specific questions for three partners, matched to their expertise. Not "what do you think of my project" — a specific engineering or strategy question.
- Treat external users met at the event as the highest-value output of the trip, not partner introductions.

### 21.2.6 If rejected

The rejection path is the base case to plan for — AI Startup School is selective. Contingencies:

- **Apply to other programs.** On Deck, Buildspace, university-linked accelerators.
- **Apply directly to YC batch** as a next step; AI Startup School is one gate, not the only one to YC.
- **Continue the broader GTM (§21.5).** The project's health does not depend on YC.
- **Revisit in January 2027** if YC announces an AI Startup School 2027 or equivalent.
- **Use the rejection as signal to refine positioning.** Ask for feedback; if none is given, review the application dispassionately for weak points.

## 21.3 YC-bound materials

A specific set of deliverables that must exist by application time.

### 21.3.1 90-second demo video

The primary artifact. Script in §16.8 is the target. Constraints:
- Runs in a real terminal, not in a slide mockup.
- No narration that's not in the voiceover; every on-screen event is real.
- Uploaded to YouTube (unlisted or public) and embedded in README.
- Mirror on Loom for network-flexible playback.

### 21.3.2 README at application-ready quality

Specific checklist:
- Single-sentence description at the top.
- Demo video embedded.
- Install in 3 commands or fewer.
- Quickstart showing first successful query in under 5 minutes of reading.
- "Known limitations" section. Honest, short, dated.
- MCP setup snippets for Claude Code and Codex CLI.
- License (MIT), contributor count badge, CI badge, PyPI version badge.

### 21.3.3 Metrics page

A public `METRICS.md` (or the §18.7 dashboard made public with personal-use KPIs redacted) so an evaluator can verify traction independently.

### 21.3.4 A single blog post

One substantive post on why personal AI memory needs to be MCP-native. Technical, opinionated, clearly written. 1,500–2,500 words. Linkable in the application under "writing." Establishes that Alex has thought deeply about the problem.

## 21.4 YC event plan

### 21.4.1 Days before

- Finalize the 10-second and 60-second pitches.
- Research the speaker list; identify 3 partners to meet and prepare one specific question for each.
- Confirm demo runs on the laptop hardware being brought.
- Charge laptop and backup battery.
- Business cards — not for partners (they don't need them), but for other builders (exchange is a high-value outcome).

### 21.4.2 During

- Talk to other builders more than to partners. The partner conversations are 5-minute windows; the builder conversations can be 30 minutes and lead to users, collaborators, or investors later.
- Demo on demand — anyone who asks "what are you building" gets the 60-second pitch + a 30-second demo clip on the laptop.
- Take notes during every session. Questions partners ask speakers are better data than the speakers' answers.
- Observe which builders the partners are gravitating toward; learn from their pitches.

### 21.4.3 After

- Within 48 hours: send thank-you notes to any partner or builder who gave meaningful time.
- Within 1 week: write a post-event reflection (private or public) capturing what worked, what didn't.
- Within 2 weeks: incorporate any concrete product feedback received into the roadmap or close it as not-useful.
- Apply to the YC batch if that's the next step; treat the event as preparation, not the endpoint.

## 21.5 Broader GTM — audiences

Four distinct audiences, each with a different channel and message.

### 21.5.1 Open-source developers (the MCP community specifically)

**Who:** people already using MCP servers, building on MCP, or following the MCP ecosystem.
**Why:** they're the most likely early adopters because the MCP thesis is already sold to them.
**Channels:** MCP-adjacent GitHub discussions, MCP registry / directory submissions, MCP community Discord or forums (wherever the community concentrates), X/Twitter threads tagged `#MCP`.
**Message:** "MCP-native personal context layer. Works with Claude Code, Codex, Cursor out of the box."

### 21.5.2 Technical researchers and ML engineers

**Who:** the tier 2 ICP — researchers with paper-heavy workflows.
**Why:** highest match to Alex's own profile; most likely to get immediate value.
**Channels:** r/MachineLearning, Hacker News, academic Twitter/Bluesky, Hugging Face community posts, `r/LocalLLaMA`.
**Message:** "Local-first retrieval over papers, code, and past AI conversations. Your research, queryable from any agent."

### 21.5.3 Local-first / privacy-conscious developers

**Who:** Ollama users, Obsidian users, self-hosted homelab enthusiasts.
**Why:** the local-first, no-telemetry, open-source posture is exactly what they want.
**Channels:** r/selfhosted, r/LocalLLaMA, self-hosted-focused newsletters, privacy-engineering forums.
**Message:** "Everything local. No cloud, no telemetry, no subscription."

### 21.5.4 AI power users broadly

**Who:** the tier 3 ICP — PMs, analysts, writers who use multiple AI tools daily.
**Why:** largest potential audience but weakest message-market fit at v0.1 due to CLI-first design.
**Channels:** newsletters like Ben's Bites, The Rundown AI; Product Hunt; general X/Twitter.
**Message:** v0.1 messaging to this audience is deliberately soft — focus on making the technical audiences succeed first. Broader messaging starts at v0.2 when a packaged desktop app exists.

## 21.6 Launch sequence

The order of public announcements. Timed to maximize both depth (technical audience who tries the tool) and breadth (wider awareness).

### 21.6.1 T-0: silent ship

`v0.1.0` tagged, PyPI published, README live. No announcement. A 1–2 day quiet period to catch basic install failures from early organic discovery before a coordinated launch. Fix whatever breaks.

### 21.6.2 T+3 days: Show HN

Post title focus: *concrete, not abstract*. Something like "contextd: a local MCP server for your personal corpus" — describes what it is in five words. Not "I built a thing that…"

Post body focus: demo video embedded, three-bullet summary of what it does, one-paragraph story of why, explicit acknowledgment of limitations and related projects (including naming Screenpipe, Rewind, second-brain-agent — HN audiences reward this).

Time of day: 8–10am PT on a Tuesday, Wednesday, or Thursday for best visibility window.

### 21.6.3 T+3 days: r/LocalLLaMA

Same content adapted for Reddit. Emphasize: local-first, open source, BYO embedding model, MCP integration with the tools this community already uses.

### 21.6.4 T+5 days: X/Twitter thread

One thread, 8–12 tweets, built around the demo video as the centerpiece. Include the Show HN link in the thread to cross-pollinate.

### 21.6.5 T+1 week: r/MachineLearning

Focused on the research use cases. Angle: the paper-synthesis narrative from §7.2, with screenshots showing real research retrieval.

### 21.6.6 T+2 weeks: written post

The substantive blog post from §21.3.4. By this point Show HN and Reddit have surfaced real early-user questions; the post can address them in depth and link back to them. Publish on a personal blog or Medium; cross-post to dev.to.

### 21.6.7 T+1 month: Product Hunt

Only if v0.1.1 has shipped with a polished UX (likely some Should-haves converted). Product Hunt's audience is broader and less technical; the launch needs more polish than HN.

## 21.7 Ongoing community building

Beyond the launch moment.

### 21.7.1 Responsiveness bar

- Issues: respond within 48 hours for the first 6 months. Not necessarily resolve — respond. Silent issue queues kill projects.
- PRs: acknowledge within 24 hours; merge or reject with reason within 1 week.
- Questions on HN/Reddit/Twitter: respond as time allows; don't disappear after launch.

### 21.7.2 Regular cadence

- Monthly release notes posted in a discussion thread on GitHub. Short format; "what shipped, what's next." This is a lightweight way to show the project is alive.
- Quarterly longer post on progress, lessons, and roadmap.

### 21.7.3 Extension contributions

The project's adapter model (§14) is the invitation. Actively promote:
- "Wanted adapters" issue with a bounty-like tag for high-demand ones.
- Tutorial for building an adapter.
- First community-contributed adapter gets explicit credit in the README.

### 21.7.4 Conference and meetup presence

Opportunistic, not strategic in v0.1:
- Local Toronto AI meetups — low cost, high signal.
- Any MCP-specific workshop or conference that emerges.
- If a talk opportunity arises, 15–20 minutes on "building a personal MCP context layer" is more useful than pure product pitch.

## 21.8 Messaging guidelines

Rules that apply to every piece of public communication.

### 21.8.1 Use concrete over abstract

"Query your PDFs from Claude Code" > "Cross-surface context retrieval."
"Runs locally" > "Privacy-first."
"7 MCP tools" > "Comprehensive agent integration."

### 21.8.2 Name competitors with respect

Per the analysis in §20, `contextd` differentiates on specific axes. Public messaging acknowledges this:
- "Like Screenpipe but MCP-native and source-structured."
- "Not a replacement for Rewind's screen-capture model; a complement for structured knowledge work."
- "Building on what second-brain-agent started."

This is both honest and tactically smart — HN and Reddit audiences can tell when competitors are dismissed unfairly, and they will push back.

### 21.8.3 Be specific about scope

"v0.1: PDFs, Claude exports, git repos. Obsidian coming in v0.2." > "Ingests everything."
"Works with Claude Code and Codex CLI in v0.1." > "Universal agent support."

Honest scope statements protect against disappointment from users who try the tool expecting features that don't exist yet.

### 21.8.4 Attribute rather than imply

When citing MCP adoption stats, link the source. When claiming retrieval quality, link to the eval harness. When describing the competitive landscape, link to competitors. Readers reward sourcing; the absence of sources is noticed.

### 21.8.5 One principle per post

Each blog post, thread, or video takes one point and makes it clearly. A 2,500-word post on MCP's role in personal AI ≠ a 2,500-word post covering MCP + retrieval quality + privacy + install. The scatter reduces impact.

## 21.9 Success metrics for GTM

Already defined in §18.4 (community KPIs); restated here as the GTM-specific subset:

- **Month 1 post-launch:** ≥ 50 GitHub stars, ≥ 300 unique PyPI installs, ≥ 5 external issues.
- **Month 3:** ≥ 250 stars, ≥ 2,000 installs, ≥ 3 external PRs merged.
- **Month 6:** ≥ 1,000 stars, ≥ 10,000 installs, ≥ 5 external contributors.

Misses trigger GTM-specific investigation before triggering kill criteria:
- Low stars but high installs → the tool is being used but not endorsed; work on making it share-worthy.
- High stars but low installs → the pitch is good but the install experience is broken.
- Low both → either the message isn't landing or the audience isn't being reached; revisit channel mix.

## 21.10 What this GTM plan does not include

Boundaries for honesty:

- **No paid advertising.** Budget is zero; organic channels only.
- **No influencer outreach budget.** Messaging targets channels where quality content reaches users without paid intermediation.
- **No hype cycle gaming.** The plan is honest; no fake accounts, no astroturfing, no coordinated manufactured attention.
- **No committed SLA beyond best-effort responsiveness.** Alex is a student; the project is free. Users get the maintenance budget Alex can sustainably provide.
- **No premature team expansion.** v0.1 is a one-person project; adding "founding engineer" or "community lead" before the project has proven product-market fit is cargo-culting.

---

