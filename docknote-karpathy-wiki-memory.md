# Karpathy's "LLM Wiki" — research dossier and Docknote v8 alignment

**Date:** 2026-05-04
**Author:** Lead (research subagent)
**Purpose:** Locate Karpathy's actual published statements on LLM-maintained personal knowledge bases ("LLM Wiki" / "LLM Knowledge Bases"), survey the implementations that grew up around them in April–May 2026, and consolidate findings against the locked Docknote v8 architecture and three sibling projects (Memio, askserver, BoscoV4).

**Companion docs (read first for grounding):**
- `/Users/seansong/seanslab/Research/cc-learn/docknote-brain-story.md`
- `/Users/seansong/seanslab/Research/cc-learn/docknote-brainmemory-design.md`
- `/Users/seansong/seanslab/Memio/memio-memory-research.md`
- `/Users/seansong/seanslab/autoresearch/askserver/CLAUDE.md`
- `/Users/seansong/seanslab/BoscoV4/API.md`

---

## A. Karpathy's actual framing

Karpathy's framing is **not** called "Wiki LLM Memory." His exact term is **"LLM Knowledge Bases"** (the tweet) and **"LLM Wiki"** (the gist). The tweet went viral on April 4, 2026; the gist followed two days later. The gist is the canonical artifact and is explicitly an "idea file" intended to be copy-pasted into a coding agent.

### Primary sources

| Source | URL | Date |
|---|---|---|
| Original tweet ("LLM Knowledge Bases") | https://x.com/karpathy/status/2039805659525644595 | 2026-04-04 |
| Follow-up ("idea file") tweet | https://x.com/karpathy/status/2040470801506541998 | 2026-04-06 |
| Gist (`llm-wiki.md`) | https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f | 2026-04-06 |

Tweet text (verbatim, quoted via X-search excerpts):

> "**LLM Knowledge Bases.** Something I'm finding very useful recently: using LLMs to build personal knowledge bases for various topics of research interest. In this way, a large fraction of my recent token throughput is going less into manipulating code, and more into manipulating knowledge."

Follow-up tweet:

> "Wow, this tweet went very viral! I wanted [to] share a possibly slightly improved version of the tweet in an 'idea file'. The idea of the idea file is that in this era of LLM agents, there is less of a point/need of sharing the specific code/app, you just share the idea, then the [agent instantiates it]."

### What he actually argues (gist, verbatim quotes)

1. **The wiki is a persistent, compounding artifact.**
   > "The wiki is a persistent, compounding artifact. The cross-references are already there. The contradictions have already been flagged. The synthesis already reflects everything you've read."

2. **Three layers, hard ownership.**
   > "**Raw sources** … These are immutable — the LLM reads from them but never modifies them. This is your source of truth. **The wiki** — a directory of LLM-generated markdown files. … The LLM owns this layer entirely. … **The schema** — a document (e.g. CLAUDE.md for Claude Code or AGENTS.md for Codex) that tells the LLM how the wiki is structured."

3. **LLM is the writer; you are the curator.**
   > "You never (or rarely) write the wiki yourself — the LLM writes and maintains all of it. You're in charge of sourcing, exploration, and asking the right questions."

4. **Three operations: ingest, query, lint.** Ingest reads source, updates ~10–15 wiki pages, appends `log.md`. Query searches wiki, synthesizes with citations, **and crucially "good answers can be filed back into the wiki as new pages."** Lint detects contradictions, orphan pages, stale claims, missing cross-refs.

5. **Two special files** — `index.md` (content-oriented catalog, LLM updates on every ingest) and `log.md` (chronological, append-only, parseable with `grep`).

6. **Indexing scales without RAG up to ~hundreds of pages.**
   > "This works surprisingly well at moderate scale (~100 sources, ~hundreds of pages) and avoids the need for embedding-based RAG infrastructure."

7. **The wiki is a git repo.**
   > "The wiki is just a git repo of markdown files. You get version history, branching, and collaboration for free."

8. **Why it works.**
   > "The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. … LLMs don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass."

9. **The Memex lineage.** Karpathy explicitly connects the pattern to Vannevar Bush's 1945 Memex.

10. **The IDE metaphor.**
    > "Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase."

### What he is precise about, and what he leaves open

**Precise (locked-in opinions):**
- Markdown files on disk; git for history; `index.md` + `log.md` as navigation.
- Three hard layers with strict ownership (raw immutable, wiki LLM-owned, schema co-evolved).
- LLM maintains everything; human curates sources and asks questions.
- Citations to raw sources required at query time.
- One source touches many pages (≥10).
- Compound knowledge across sessions.

**Deliberately open (he says "intentionally abstract"):**
- Directory layout beyond `raw/` + `wiki/`.
- Page schema / frontmatter conventions.
- Search infrastructure (suggests `qmd` for hybrid BM25+vector at scale, but says index-only is fine for ~100 sources).
- Whether to use entity graphs, knowledge graphs, structured tables.
- How to handle multi-user / team ingest.

The gist is **agnostic about substrate inside the wiki/ folder.** It is **not** agnostic about: markdown source-of-truth, git history, three-layer ownership, LLM-as-maintainer, ingest+query+lint trio.

---

## B. Implementations table

Surveyed via GitHub search, the HN front page (April–May 2026), Karpathy's reply chain on X, and the citation graph backwards from the gist. Stars are as visible during this research session (May 4, 2026).

| Project | Stars | Substrate | Atomic unit | Karpathy ack | Closest match to v8 | Relevant findings |
|---|---|---|---|---|---|---|
| `karpathy/llm-wiki` (the gist itself) | 5,000+ stars / 4,282 forks (per VentureBeat) | markdown + git | wiki page | n/a (the source) | — | The pattern itself; not an implementation |
| `Ar9av/obsidian-wiki` | 921 | markdown in Obsidian vault + `.manifest.json` delta tracking | wiki page (Obsidian note) | "inspired by Andrej Karpathy's LLM Wiki pattern" | Closest — has provenance tracking (extracted/inferred/ambiguous) and a 4-stage ingest pipeline | Multimodal ingest (vision over screenshots); cross-linker skill; does NOT have meeting-shape; 4-stage Ingest→Extract→Resolve→Schema |
| `lucasastorian/llmwiki` | 798 | filesystem + SQLite FTS5 (derived index) + Claude via MCP | markdown wiki page | "Open Source Implementation of Karpathy's LLM Wiki" | Strong — filesystem is source-of-truth, SQLite is derived | Tools are search/read/write/delete; `str_replace` writes; cached artifacts in `.llmwiki/`; treats LLM as remote-MCP collaborator, not a Tauri-style in-process agent |
| `Astro-Han/karpathy-llm-wiki` | 728 | markdown only (`raw/`, `wiki/`, `index.md`, `log.md`) | wiki page | "Unofficial community implementation of the workflow from Karpathy's LLM Wiki idea" | Closest at the directory layout level (literal gist structure) | Production deployment: 94 wiki articles, 99 sources ingested in 1 month — useful real-world data point for v8 scale assumptions |
| `skyllwt/OmegaWiki` (DAIR Lab, Peking U.) | 479 | markdown + 9 typed entity kinds + two knowledge graphs (`graph/edges.jsonl`, `graph/citations.jsonl`) | wiki page (typed: Paper / Concept / Topic / Person / Idea / Experiment / Claim / Summary / Foundation) | "Andrej Karpathy proposed LLM-Wiki … This project takes that idea and runs the full distance" | Nearest to v8's three-engram-kinds taxonomy, but extended to nine | 24 Claude Code skills; failed-experiments-as-anti-repetition memory; this is the most **structured** of the Karpathy-inspired projects |
| `kfchou/wiki-skills` | 116 | markdown + `SCHEMA.md` + `wiki/pages/` flat names | markdown page | Direct: "implements Karpathy's LLM Wiki pattern" | Direct skills-shaped port; very thin | `wiki-ingest`, `wiki-query`, `wiki-update`, `wiki-lint`, `wiki-audit` (citation verification) — `wiki-audit` is a useful new primitive |
| `Pratiyush/llm-wiki` | 218 | markdown + JSON-LD graph + static-site export | session-as-source | "Karpathy's LLM Wiki pattern — implemented and shipped" | Different angle: turns past Claude Code / Codex / Cursor / Gemini sessions into a wiki | 67 releases, dual human/AI-readable HTML+JSON output per page, 16 lint rules |
| `eugeniughelbur/obsidian-second-brain` | (unknown, est. 100–500) | Obsidian vault | note | "Inspired by Andrey Karpathy's LLM-Wiki" | Obsidian-shaped, not file-system-shaped from outside | 31 commands; auto-synthesis; 4 scheduled agents (cron-style consolidation, matches v8's 03:00 nightly idea) |
| `NicholasSpisak/second-brain` | (small, recent) | Obsidian vault | note | "Based on Andrej Karpathy's LLM Wiki pattern" | Obsidian-shaped | Web Clipper integration (matches Karpathy's gist tip section) |
| `shannhk/llm-wikid` | (small) | Obsidian vault | note | "Karpathy-style LLM knowledge base for Obsidian" | Obsidian-shaped | Almost identical to `wiki-skills` but Obsidian-flavored |
| `AgriciDaniel/claude-obsidian` | (small) | Obsidian vault | note | "Persistent, compounding wiki vault based on Karpathy's LLM Wiki pattern" | Obsidian-shaped | Notable: `/wiki`, `/save`, `/autoresearch` slash commands; 11 skills; auto-filing |
| `lewislulu/llm-wiki-skill` | (small) | markdown | wiki page | "Karpathy-style LLM knowledge base Agent Skill for OpenClaw/Codex" | OpenClaw plugin — relevant to Memio | Calls itself "Experimental — will iterate over time" |
| `rohitg00/2067...` (gist) | n/a (gist) | — (extension proposal) | — | "extending Karpathy's LLM Wiki pattern with lessons from building agentmemory" | — | "LLM Wiki v2": adds idle-trigger background entity-extraction, dedup, contradiction detection — closer to v8's nightly-consolidation model |
| Brad Groux's OpenClaw integration (Twitter @BradGroux) | n/a (in-house) | OpenClaw memory + `research/knowledge-system/` directory | wiki page | "@Karpathy nailed it, so I adapted his pattern … directly into my OpenClaw operating system" | OpenClaw-internal | Notable because OpenClaw is the same agent harness Memio uses; Brad's "Corpus-first architecture: Built research/knowledge-system/ with a strict Raw → Compiled loop" — mirrors gist exactly |
| `Programming-With-Maury/Karpathy-LLM-Wiki` | (small) | markdown + AGENTS.md | wiki page | Project name itself | Very thin | Mostly the gist re-instantiated as a starter template |

**Out of scope** (these are pre-Karpathy or independently arrived; covered in v8's other dossiers): Cline memory bank, Cursor memory bank (vanzan01), claude-mem (thedotmack), basic-memory, A-MEM, Letta filesystem-as-memory, Minutes, Granola, Claude Code `src/memdir/`. Note that **Karpathy's gist explicitly does not cite any of these** — the gist's only acknowledged ancestor is Vannevar Bush's 1945 Memex.

---

## C. Architectural patterns that emerge

**Strongly converged across implementations:**

1. **Markdown source-of-truth.** Every single Karpathy-cited implementation uses markdown. Not JSON, not YAML, not SQL, not RDF. The schema lives **inside** markdown frontmatter or `[[wikilinks]]`.
2. **Three layers** mirroring the gist: `raw/` (immutable), `wiki/` (LLM-owned), schema (`CLAUDE.md`, `AGENTS.md`, `SCHEMA.md`).
3. **`index.md` + `log.md`** as the LLM's navigation primitives.
4. **`[[wikilinks]]`** (Obsidian-style) for cross-references. This is universal among the Obsidian-flavored ones and adopted by ΩmegaWiki and others.
5. **Three operations** — ingest, query, lint — show up in all of `Astro-Han/karpathy-llm-wiki`, `kfchou/wiki-skills`, `Ar9av/obsidian-wiki`, OmegaWiki's skill set.
6. **Git for history.** The gist makes this explicit; every implementation that mentions versioning uses git.
7. **Citations to raw sources at query time.** Every implementation does this.
8. **LLM-as-maintainer, human-as-curator.** Universal.

**Contested — patterns that vary:**

1. **Whether to add structured layers on top of markdown.** OmegaWiki goes furthest (typed entities, two knowledge graphs); `lucasastorian/llmwiki` adds SQLite FTS5; `Ar9av/obsidian-wiki` adds `.manifest.json` delta tracking. `Astro-Han/karpathy-llm-wiki` and the Obsidian-vanilla ones stay pure markdown.
2. **Where search lives.** Some delegate to Obsidian's own search; some embed `qmd`-style hybrid retrieval; some use SQLite FTS5; some use embedding-based RAG **on top of** the wiki (the gist explicitly says this is acceptable beyond ~100 pages).
3. **Trigger model.** Most are query-driven (user runs `/wiki-ingest`). A few (rohitg00 v2 gist, OmegaWiki, `obsidian-second-brain`'s "scheduled agents") have **idle-time consolidation** — closer to v8's 03:00 nightly pass.
4. **Provenance fidelity.** `Ar9av/obsidian-wiki` tags every claim as `extracted | inferred | ambiguous`. Most others just keep citation links and rely on git for audit.
5. **Whether to "promote with approval."** HN critique converged on "draft freely, promote on approval is the only thing that works" — but only `Ar9av/obsidian-wiki` and the OpenClaw-flavored ones implement an approval queue. v8's Brain Inbox is rare in the Karpathy-ecosystem.
6. **Episodic vs semantic units.** All Karpathy-inspired projects index by **topic / entity / concept**. None of them have a first-class **meeting** kind. v8's `meeting/` engram is Granola-borrowed, not Karpathy.

**Known critiques (HN, Reddit r/LocalLLaMA, qaadika, others):**
- "Garbage facts in, garbage briefs out" — once a hallucinated claim lands in the wiki, lint can't tell which.
- Long-context attention degrades; index.md grows past comfortable context window.
- No consensus / dispute resolution mechanism (Wikipedia comparison).
- Without mandatory human review, agent-only promotion poisons the wiki.

---

## D. Comparison vs Docknote v8

For each Karpathy principle, where v8 sits:

| Karpathy principle | v8 stance | Notes |
|---|---|---|
| **Markdown source-of-truth** | aligned | Cortex is `~/Docknote/brain/*.md`. v8 §15 row 1 names this explicitly as the "openness wedge." |
| **Three hard layers (raw/wiki/schema)** | partial | v8 has cortex (≈ wiki) + hippocampus indexes (≈ derived). It does NOT have a `raw/` layer separate from meeting engrams — the raw transcript IS the meeting engram body. v8 conflates Karpathy's `raw/` and `wiki/` layers into one cortex tree. |
| **LLM owns the wiki layer entirely** | conflict | v8 explicitly says "human edits always win." The cortex is co-edited; `human_locked: true` makes a field immutable to auto-consolidation. Karpathy's gist says "you never (or rarely) write the wiki yourself"; v8 inverts this priority. |
| **Always-loaded vs query-time-loaded** | mixed / aligned | v8 has Stage 0 (`persona.md` + Recent view always-loaded, capped 4K) AND a 5-stage retrieval path. Karpathy's gist is implicitly query-time (LLM reads `index.md` first, drills into pages). v8 is more aggressive about working memory than the gist requires. |
| **`index.md` + `log.md`** | conflict | v8 has none of these. The hippocampus's `entity_index` (SQLite) is the structural equivalent of `index.md`; git history is the structural equivalent of `log.md`. **v8 is missing a content-oriented catalog file.** This is a real gap — see action items. |
| **Three operations: ingest, query, lint** | partial | v8 has encoding (≈ ingest) and recall (≈ query). v8 has nothing called `lint`. The nightly consolidation does part of lint's job (theme dedup, age-banding, contradiction reconciliation), but there's no first-class "health check the cortex" pass. **Missing.** |
| **Citations to raw sources** | aligned + stronger | v8's per-field provenance with `source: { meeting_id, span }` exceeds the gist. v8 cites span-level (HH:MM:SS); the gist cites at page level. ➕ |
| **Git for history** | aligned + stronger | v8 wraps git2-rs with a typed commit message format and per-field human-lock semantics. ➕ |
| **No structured tables alongside the wiki** | aligned (after §15 row 12) | v8 reversed the Graphiti `facts` table borrow. This is **the most Karpathy-aligned move v8 has made.** It was decided independently (the v8 reasoning was about "fake-precision contradictions"), but it lands exactly where Karpathy's gist sits. |
| **Cross-references (`[[wikilinks]]`)** | partial | v8 has `entity_index` resolver (name → engram path) and aliases, but the prose body of an engram does NOT use `[[wikilinks]]`. **The Karpathy-inspired field has converged on `[[wikilinks]]`; v8 hasn't adopted it.** Either v8 should add wikilink syntax (cheap), or explicitly reason about why aliases-in-frontmatter is sufficient. |
| **Index scales to ~100 sources without RAG** | conflict | v8 has RAPTOR + sentence-vector galaxy from day 1. This is justified by the meeting-recall use case (sentence-level temporal recall is harder than concept lookup), but it's worth noting v8 builds infrastructure Karpathy would call premature for the first ~hundreds of pages. |
| **Forgetting / decay** | partial / aligned | v8 has `age_band: current → recent → background → archived`. Karpathy's gist mentions "stale claims that newer sources have superseded" as a lint target but doesn't specify a decay model. v8's age-banding is a stronger explicit decay mechanism. ➕ |
| **The schema file (`CLAUDE.md`/`AGENTS.md`)** | conflict | v8 has no equivalent. The Brain API trait IS the schema, but it's Rust code, not a markdown contract. Karpathy's gist treats the schema as a **co-evolved markdown document** — a place where the user and LLM negotiate conventions over time. v8 hardcodes conventions in code. **Missing.** This is the second real gap. |
| **Idle-time / scheduled consolidation** | aligned + stronger | v8's nightly 03:00 pass is more aggressive than the gist's "periodic lint." Closer to OmegaWiki and `obsidian-second-brain`'s scheduled agents. ➕ |
| **Approval queue for proposed updates** | aligned + stronger | v8's Brain Inbox is rare in the Karpathy-ecosystem (only `Ar9av/obsidian-wiki` and the OpenClaw fork have it). v8's `proposed_updates` SQLite table + per-field accept/reject is the strongest implementation of this pattern surveyed. ➕ |

**Summary:**
- v8 is **aligned** on: markdown SOT, git audit, citations, decay, no facts-table, query-time reconstructive recall.
- v8 is **stronger** than the field on: per-field provenance, approval queues, span-level citations, scheduled consolidation, working memory.
- v8 has three real **gaps** vs the Karpathy pattern: (1) no `index.md` content catalog file; (2) no `lint` operation; (3) no co-evolved schema markdown. (See §F.)
- v8 has one principled **conflict** with Karpathy: human-edits-win vs LLM-owns-wiki. v8 is right; this conflict is a feature, not a bug, given the doctor/lawyer/salesperson user model where human authority over their professional record is non-negotiable.

---

## E. Cross-reference with Memio / askserver / BoscoV4

Each sibling project represents a different bet on memory substrate. Where does Karpathy land?

### Memio — graph-augmented observational
Memio (`memio-memory-research.md`) explicitly chose a **layered** architecture: OpenClaw's markdown-first observational memory **plus** Mem0-style graph memory for entity relationships. Memio's call:
> "This gives Memio three retrieval signals — keyword matching, semantic similarity, and relationship traversal — covering all query types users will throw at it."

Karpathy's gist does NOT advocate graph memory. It is silent on the question, but the gist's posture ("avoid the need for embedding-based RAG infrastructure" at small scale, "LLMs … can touch 15 files in one pass") favors the **markdown layer alone** as long as scale permits. Memio's graph layer is a hedge against query types Karpathy's pattern doesn't optimize for (relationship traversal across many entities).

**Karpathy's stance vs Memio:** Compatible at the markdown layer (both have it). Karpathy is silent on graphs; Memio adds them. The risk Karpathy implicitly warns against — "the wiki gets stuck at a local maximum without external feedback" — is the same risk Memio's graph layer is trying to mitigate (explicit relationships make staleness detectable).

### askserver — "Less Structure, More Intelligence"
askserver's `CLAUDE.md` says it directly:
> "This follows Andrej Karpathy's auto-research methodology: one mutable artifact, one locked evaluation harness, one authoritative score, keep only improvements, revert regressions."
> "**Less Structure, More Intelligence.** The model does the thinking. Python handles I/O. No hardcoded question parsers, no rigid extraction schemas, no hand-tuned scoring parameters."

This is an interesting case: askserver is citing **a different Karpathy methodology** (auto-research / autoresearch — `research/program.md`), not the LLM Wiki gist. But the philosophical alignment is striking — both are "less structure, more intelligence" bets.

askserver's storage substrate: natural-language meeting summaries (≈600 tokens each), JSONL-stored, model-driven extraction. There is **no wiki layer.** The catalog is metadata-only (date, title, participants, topics). The model does retrieval at query time directly over summaries.

**Karpathy's stance vs askserver:** Strongest alignment of the three siblings. askserver takes the gist's "avoid RAG infrastructure, let the LLM read directly" idea further than the gist does — it doesn't even build a wiki, it just lets the model select-then-answer over raw summaries. Limitation: no compounding artifact. askserver's knowledge does NOT build up across sessions; every query starts from the corpus. v8's compounding cortex is the hybrid askserver lacks.

### BoscoV4 — typed tables + structured citations
BoscoV4 (`API.md`, `Schema.md`, `SDD.md`) is a **structured-table** architecture. Decisions, citations, chunks, meetings, action items — all live as typed SQL rows. The `search` API returns typed `result_type: decision | action_item | claim` with citation anchors pointing to `target_table`, `target_id`, `chunk_id`, `start_ms`, `end_ms`.

This is the **opposite** of Karpathy's posture. Bosco's `decisions` table is exactly the kind of "structured claims accumulating over years" that Karpathy's gist (and v8 §15 row 12) implicitly rejects.

**Karpathy's stance vs BoscoV4:** Direct conflict. Bosco bet on Granola-shape (structured tables, typed results); Karpathy bets on markdown-shape (free-form pages, LLM synthesis). The two architectures answer the same questions but with opposite precision/flexibility tradeoffs. Bosco's API is more queryable by deterministic code; Karpathy's wiki is more queryable by LLMs. v8 explicitly chose Karpathy-shape over Bosco-shape (§15 row 12).

### The four-way landscape

| | Substrate | Compounding artifact? | Karpathy alignment |
|---|---|---|---|
| **askserver** | natural-language summaries (JSONL), no wiki | No (every query re-reads corpus) | Strong philosophy alignment, no wiki |
| **Memio** | markdown observational + graph layer | Yes (markdown) + relationships | Compatible; adds graph |
| **v8 (Docknote)** | markdown engrams + sentence-vector index + RAPTOR | Yes (cortex) | Strong alignment + per-field provenance |
| **BoscoV4** | typed SQL tables (decisions, action_items, citations) | Yes (rows) | Direct conflict — opposite bet |

v8 is the closest of the four to Karpathy's gist, with two enhancements (per-field provenance, approval queue) and one principled inversion (human edits win over LLM updates).

---

## F. Action items for v8

Concrete proposals, ordered by impact-per-effort:

### F1. Add a `cortex/index.md` (and consider `cortex/log.md`) — HIGH impact, LOW effort

The Karpathy-ecosystem has converged on a content-oriented catalog file. v8 has the data (every engram, every entity, every alias) sitting in the SQLite `entity_index` — but no markdown-side reflection of it. This is a real gap.

**Proposal:** Auto-generated `cortex/index.md` updated at every nightly consolidation. Lists every engram (person/topic/meeting) with one-line summary, last-touched date, source count. Updated by `auto-consolidator`. Treated as a derived artifact (regenerable), not a source of truth — but **lives in the cortex tree** so a user `cd ~/Docknote/brain` sees a navigable index.

`log.md` is more debatable; git log already serves this purpose. Recommend skipping.

**Where to add to design doc:** §4.1 (Filesystem layout) — add `index.md` to the cortex tree. §7.2 (Consolidation) — add "regenerate index.md" as step 6.

### F2. Add a `lint` operation to the Brain API — HIGH impact, MEDIUM effort

The nightly consolidation already does **parts** of lint (age-banding, theme dedup, contradiction `reconcile_flag`). But there's no first-class lint operation a power user can invoke on demand, and no health-check that surfaces:
- Engrams with no inbound references (orphans)
- Entities mentioned in transcripts that have no engram (gaps)
- Stale claims (older than N days with no corroboration)
- Missing cross-references

**Proposal:** Add `Hippocampus::lint() -> LintReport` returning categorized issues. Surface a "Brain Health" view in the UI alongside Brain Inbox. Run as part of nightly consolidation; expose ad-hoc invocation via a hidden settings toggle.

**Where to add to design doc:** §5 (Hippocampus trait) — new method. §7.2 (Consolidation) — add lint as step 7. §9 (Edit UX) — add Brain Health surface.

### F3. Add a `cortex/SCHEMA.md` co-evolved schema document — MEDIUM impact, LOW effort

Karpathy's gist treats the schema (`CLAUDE.md`/`AGENTS.md`) as a **co-evolved markdown document** — a place where the user and LLM negotiate conventions ("how should I name things, what aliases should I track for this person, when do I demote vs delete"). v8 currently encodes all conventions in Rust code. This means the user can't teach v8 about their own conventions ("call me Sean, not the user"; "in legal context, 'pleading' means X").

**Proposal:** Ship a default `cortex/SCHEMA.md` that the user and consolidation pass both read at init. User can edit it (it's user-owned, never auto-rewritten). The consolidation pass reads it as a system-prompt contribution. Examples:
- Naming conventions
- Domain vocabulary (legal / medical / sales)
- Per-engram-kind expectations (what fields matter for a person engram in a legal practice)

**Where to add to design doc:** §4.1 — add `SCHEMA.md` to filesystem layout. §7.1 step 3 — schema content joins the encoding prompt. §7.2 step 1 — schema content joins the consolidation prompt.

### F4. Cite Karpathy explicitly in the design rationale — LOW impact, ZERO effort

§1 of the design doc lists prior art (Letta, Graphiti, Granola, Minutes, mem0, MemMachine, RAPTOR, Cline). It does NOT cite Karpathy's LLM Wiki gist. Given that v8 §15 row 12 (no facts table, reconstructive recall) lands exactly on Karpathy's gist, and §15 row 1 (markdown SOT) is the gist's central claim, **Karpathy's gist deserves an explicit citation in §17.**

**Proposal:** Add to §17 References:
> **Karpathy's "LLM Wiki" gist** (https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f, 2026-04-06) — the markdown-source-of-truth, LLM-as-maintainer, three-layer (raw/wiki/schema), index+log+lint pattern v8 independently arrived at via §15 row 12. v8 differs in: per-field human-locks ("human edits win"), span-level citations, scheduled consolidation, approval queue, and a meeting-shaped engram kind.

This is honest engineering — the credit is owed and the differences are real.

### F5. Consider `[[wikilink]]` syntax in engram bodies — MEDIUM impact, LOW–MEDIUM effort

Every Obsidian-flavored Karpathy implementation uses `[[wikilinks]]`. v8's engram bodies are plain markdown prose; cross-engram references are implicit via entity name resolution at render time. Adopting `[[wikilinks]]` in body text would:
- Make engrams browsable as a graph in any markdown editor (Obsidian, Logseq, Foam) the user opens `~/Docknote/brain` in
- Make the relationship structure visible to git diffs (you can see when a new edge appears)
- Match the field's convention — power users will expect it

The cost: the consolidation pass has to emit `[[Alex Chen]]` instead of `Alex Chen`, and the entity_resolver has to treat `[[name]]` as a name-resolution boundary on read. Not hard; just a convention to adopt.

**Recommendation:** Adopt for v8.1 (first iteration after launch); not required for ship. Document the decision in §15 either way.

### F6. Don't add a `raw/` directory — KEEP the design as is

A surface-reading of Karpathy might suggest adding a `raw/` layer separate from `meeting/`. **Reject this.** v8's meeting engrams ARE the raw layer (transcript + frontmatter); separating them would duplicate data with no win. The gist's `raw/` exists because Karpathy ingests articles/PDFs/papers; v8 ingests transcripts which are produced by Docknote itself. Document the difference in §15 to forestall future drift.

**Proposal:** Add a §15 row clarifying "we conflate Karpathy's raw/ and wiki/ into a single cortex/ tree because the meeting engram itself is the raw transcript."

### F7. Don't soften "human edits win" — KEEP the design as is

The Karpathy gist says "the LLM owns the wiki layer entirely." v8 inverts this with `human_locked: true`. The doctor/lawyer/salesperson use case demands this inversion; "the LLM rewrote my note about the patient" is unacceptable in a way that "the LLM rewrote my note about Tolkien lore" is not. **Document the inversion as deliberate.** §10 already implies it; §15 should make it explicit as a decision row.

### F8. Surface contradictions found during nightly consolidation in the Brain Inbox — LOW effort, MEDIUM impact

The consolidation pass already calls `reconcile_flag` for unresolvable conflicts. v8 currently treats these as a `proposed_updates` row. Karpathy's lint operation and the HN critique ("garbage facts in, garbage briefs out") both highlight contradiction-detection as the highest-value lint check. v8 should surface these as a **first-class category** in the Brain Inbox UI, distinct from "auto-extracted change you might want to accept."

**Proposal:** Brain Inbox grouping: (1) Proposed updates, (2) **Contradictions to reconcile** (new), (3) Possible orphans/gaps (from F2 lint). §9.3 should specify this layout.

---

## Closing notes

Karpathy's "LLM Wiki" gist is the single most articulate public statement of what v8 is. v8 was designed independently — its lineage (Letta sleep-time, Graphiti bi-temporal, Granola UX, Cline file taxonomy, Claude Code memdir) doesn't include Karpathy. But §15 row 12 — the no-facts-table reversal — converges on Karpathy's central insight: **the brain isn't tabular, the LLM should reconstruct from prose at recall time, and the prose is the source of truth.**

Three actions are owed: (1) add `index.md`, (2) add `lint`, (3) cite Karpathy. The fourth — schema markdown — is the easiest way to give the user a seat at the design table without reopening the API contract. The fifth — `[[wikilinks]]` — is field-convention-tax; pay it post-launch.

The principled disagreement with Karpathy (human edits win) is the right call for v8's user base. Document it; defend it; don't drift from it.

---

## Sources

- [Andrej Karpathy on X: "LLM Knowledge Bases" (2026-04-04)](https://x.com/karpathy/status/2039805659525644595)
- [Andrej Karpathy on X: "idea file" follow-up (2026-04-06)](https://x.com/karpathy/status/2040470801506541998)
- [Karpathy's `llm-wiki.md` gist (2026-04-06)](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [VentureBeat: Karpathy shares 'LLM Knowledge Base' architecture that bypasses RAG](https://venturebeat.com/data/karpathy-shares-llm-knowledge-base-architecture-that-bypasses-rag-with-an)
- [Brad Groux on X: OpenClaw integration of Karpathy's pattern](https://x.com/BradGroux/status/2040648338589069598)
- [Elvis Saravia on X: Obsidian + Karpathy](https://x.com/omarsar0/status/2039844072748204246)
- [Ole Lehmann on X: plain-English explainer](https://x.com/itsolelehmann/status/2040119257581646030)
- [Ar9av/obsidian-wiki](https://github.com/Ar9av/obsidian-wiki)
- [lucasastorian/llmwiki](https://github.com/lucasastorian/llmwiki)
- [Astro-Han/karpathy-llm-wiki](https://github.com/Astro-Han/karpathy-llm-wiki)
- [skyllwt/OmegaWiki](https://github.com/skyllwt/OmegaWiki)
- [kfchou/wiki-skills](https://github.com/kfchou/wiki-skills)
- [Pratiyush/llm-wiki](https://github.com/Pratiyush/llm-wiki)
- [eugeniughelbur/obsidian-second-brain](https://github.com/eugeniughelbur/obsidian-second-brain)
- [NicholasSpisak/second-brain](https://github.com/NicholasSpisak/second-brain)
- [shannhk/llm-wikid](https://github.com/shannhk/llm-wikid)
- [AgriciDaniel/claude-obsidian](https://github.com/AgriciDaniel/claude-obsidian)
- [lewislulu/llm-wiki-skill](https://github.com/lewislulu/llm-wiki-skill)
- [rohitg00 — LLM Wiki v2 extension gist](https://gist.github.com/rohitg00/2067ab416f7bbe447c1977edaaa681e2)
- [Show HN: lucasastorian/llmwiki discussion](https://news.ycombinator.com/item?id=47656181)
- [Show HN: Karpathy-style LLM wiki discussion](https://news.ycombinator.com/item?id=47899844)
- [VentureBeat coverage](https://venturebeat.com/data/karpathy-shares-llm-knowledge-base-architecture-that-bypasses-rag-with-an)
