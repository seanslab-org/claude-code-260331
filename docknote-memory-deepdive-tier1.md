# Brain Memory v8 — Tier 1 deep-dives

**Date:** 2026-05-04
**Companions:**
- `docknote-memory-learn-from-cc.md` (Claude Code lessons)
- `docknote-memory-leads.md` (broad-sweep leads, ~155 entries)

**Targets covered (Tier 1):**
1. Letta (formerly MemGPT) — OS-style memory blocks
2. Graphiti / Zep — temporal knowledge graph
3. Granola — closest competitive analog
4. Minutes (silverstein) — Markdown-on-disk + MCP, closest architectural analog
5. Anthropic memory tool + context engineering — wire-format interop standard

---

## Synthesis — what these five collectively tell us

Five deep-dives, six convergences worth pinning before SDD work resumes:

### 1. Filesystem-as-memory wins on the cheap path

- **Letta's own benchmarks** (LoCoMo) put a plain `grep`/`search_files`/`open` filesystem agent at **74.0% vs Mem0's graph-based 68.5%** with the same model. Their conclusion: "memory is more about how agents manage context than the exact retrieval mechanism."
- **Minutes ships it as the product.** Markdown-on-disk + SQLite FTS5 derived index, 1.2k stars, weekly cadence, no embedding requirement.
- **Anthropic's memory tool *is* a filesystem** — six commands (`view`/`create`/`str_replace`/`insert`/`delete`/`rename`) rooted at `/memories`, line-numbered output, no other primitives. The closest thing to an interop standard.
- **Granola is the counter-example, and it's a live wound.** Their March 2026 SQLite encryption broke every community MCP/Obsidian/scraper integration; Guido Appenzeller publicly trashed it; "Use MCP instead" was the user demand and Granola refused. Plain Markdown is the strategic counter-positioning.
- **Graphiti is the only outlier**, and that's because graph DBs are the wrong physics for single-user Mac apps (Kuzu archived, FalkorDB Lite pre-GA, Neo4j JVM, Neptune cloud-only).

**Decision implication:** Brain Memory v8's source-of-truth should be **plain Markdown in `~/Docknote/brain/`** with SQLite + sqlite-vec as a derived, disposable index. Match Minutes' frontmatter conventions and file-naming for ecosystem compatibility. Skip the graph DB.

### 2. Bi-temporal facts are the right abstraction; the implementation is the question

- **Graphiti's bi-temporal edges** (`valid_at` / `invalid_at` / `created_at`) are the cleanest model for "what was Alex's pricing position in March?" pivots and dossier evolution. Soft-invalidation, never delete, episode-back-pointers for provenance.
- **Letta achieves the same effect** via `tags[]` + `start_datetime`/`end_datetime` filters on archival search. Less rigorous, simpler.
- **Minutes' frontmatter** has `date:` and per-decision/action-item arrays, which is the lightweight version.
- **Graphiti's LongMemEval numbers** are the persuasive evidence: temporal reasoning **+38-48%**, preference tracking **+78-184%**, multi-session synthesis **+17-31%** vs full-context baseline. Exactly the categories Brain Memory v8 cares about.

**Decision implication:** Steal Graphiti's edge schema vocabulary verbatim and put it in a SQLite `facts` table: `(subject, predicate, object, fact, valid_at, invalid_at, created_at, episode_id)`. Build a deterministic resolver (email/calendar-id/canonical-name keys) and skip the LLM-per-episode dedup ($0.10–$0.40/meeting and 3–10min wall-clock at Graphiti's pipeline cost).

### 3. Two retrieval surfaces, not one

The Anthropic deep-dive surfaced the cleanest architectural split:

- **Memory tool (`/memories`)** = agent scratchpad / session continuity. Agent-owned, agent-pruned. Sandbox to `~/Docknote/brain/.agent-memory/<session-id>/`.
- **MCP `brain.search`** = canonical knowledge base. User-owned, schema-validated, indexed. The Brain Memory v8 surface.

Routing prompt for the agent:

> Use `memory` for things YOU recorded (todos, scratch, where-was-I). Use `brain.search` for things the USER recorded (meetings, transcripts, notes). "What did I discuss with X" → `brain.search`. "Where was I in this task" → `memory.view`.

**Decision implication:** Don't map `~/Docknote/brain/` directly behind the memory tool's `/memories` root — that conflates agent-owned and user-owned data and exposes the brain to `delete`/`str_replace` from any agent. Implement the memory-tool wire format as a sandboxed scratchpad surface, keep canonical brain writes behind MCP.

### 4. Synthesis is where Brain Memory v8 differentiates — none of the prior art does it well

- **Minutes** stops at a filtered list per person + commitment tracker + consistency report. No narrative, no `role_line`, no `day_headline`.
- **Letta** memory blocks are flat strings — schema is the developer's problem.
- **Graphiti** has no narrative layer; ranks edges, that's it.
- **Granola's People view** is reportedly a chronological note list, not a synthesized profile.
- **Anthropic memory tool** is a CRUD protocol; semantics are the developer's problem.

The persona-corpus acceptance bar (≥80%, with `expected_synthesis_themes` of 3–4 themes per query that must all appear) is asking for *cross-document theme extraction* none of these ship.

**Decision implication:** The synthesis pass — the `step 5` from the lessons doc — is the architecturally novel work. Everything below it is borrowable from these five.

### 5. Citations with jump-to-source are the single most-praised feature in the category

Granola users uniformly love inline `[1][2]` citations with line-level jumps to source meeting + timestamp. Reviewers across `tldv.io`, `wondertools.substack.com`, and `themeridiem.com` flag it as the standout. This is non-negotiable for v8 — every dossier card claim and every chat answer must cite the originating meeting and span.

### 6. The privacy posture is now a competitive wedge, not just a thesis

- **Granola** stores transcripts in US AWS VPC, calls OpenAI/Anthropic/Google for inference, opts users into training by default, encrypted the local DB to enforce lock-in (March 2026).
- **Letta Cloud** is per-agent billed and cloud-shaped.
- **Graphiti's pipeline is LLM-heavy** — every episode hits an LLM 4–8 times for extract/dedup/summarize.
- **Anthropic memory tool** ships file *contents* to Anthropic on every recall.

Docknote's Phase-2 SnapGPU thesis (spark1+spark2 over Tailscale, no cloud retention) is now a direct counter-positioning to Granola's vulnerable moment. The encrypted-DB episode is the strategic opening: lead with **"your audio, transcripts, and inference never leave your tailnet"** + **Markdown-on-disk you own** + **MCP-native data plane**.

---

## Updated decision-axis status

The five axes from `docknote-memory-leads.md` §C, after Tier 1:

| Axis | Verdict | Confidence |
|---|---|---|
| Extract-and-store vs. keep-raw-and-project | **Hybrid:** keep raw transcripts as the source of truth (MemMachine-style), extract typed facts (people, decisions, action items, themes) at meeting close. Both surfaces queryable. | High |
| Filesystem vs. KG vs. vector vs. hybrid | **Markdown source-of-truth + SQLite-derived FTS5 + sqlite-vec for stage-2.** No graph DB. Borrow Graphiti's *schema*, not its *engine*. | High |
| Implicit auto-extract vs. explicit user-edited | **Both:** auto-extract at meeting close (Letta sleep-time pattern), user can edit any dossier card. Audit trail required. | Medium |
| Per-session blank slate vs. always-loaded global | **Always-loaded persona/project context** (ChatGPT-style for the user-facing assistant), **agent memory tool as scratchpad** (Claude-style for agent continuity). Two surfaces, different policies. | High |
| Cloud vs. local-first vs. hybrid | **Local-first by default** (SnapGPU/Qwen), explicit consent for any cloud route. Memory contents never auto-leave the tailnet. | High (already locked by Phase-2 thesis) |

Three of five axes are now firm. The remaining two — the implicit/explicit balance, and exactly what "stage-0 coarse filter" looks like at meeting scale — need Tier 2 deep-dives.

---

## Suggested Tier 2 picks

Pulled from the broad-sweep leads with the open questions in mind:

1. **mem0 + Mem0 paper (arXiv 2504.19413)** — extract-then-store at scale; what their two-phase extract+update pipeline actually does, and where it breaks (per @deshrajdry's own admissions). Resolves: extraction-as-write-path quality bar.
2. **HippoRAG 2 / RAPTOR** — cross-meeting multi-hop synthesis without graph DB. Resolves: synthesis pass architecture for theme extraction.
3. **MemMachine paper (arXiv 2604.04853)** — keep-raw-and-project, 80% fewer tokens than Mem0. Resolves: extraction-vs-raw fork rigorously.
4. **Letta sleep-time agents (deeper)** — read the actual prompts and consolidation policies. Resolves: nightly synthesis pass mechanics.
5. **Cline memory bank + cursor-memory-bank** — coding-tool memory in production at scale, very different write loop than meeting tools. Resolves: explicit-edit UX patterns.

Pick by which decision is most blocking. My recommendation: **mem0 + MemMachine first** — those two arguing in opposite directions on the most consequential axis (extract vs. raw) is the highest-leverage read.

---

# 1. Letta — deep-dive

Letta (formerly MemGPT) is the spiritual ancestor of every "agent with memory" architecture shipping today. The original 2023 paper ([arXiv 2310.08560](https://arxiv.org/abs/2310.08560)) framed LLM context as OS-style virtual memory: a fixed "main context" the model sees plus a tiered "external context" the model pages in via function calls. The current OSS project ([github.com/letta-ai/letta](https://github.com/letta-ai/letta), Apache-2.0, ~99% Python) keeps that framing but has rebuilt the agent loop twice since the paper. For Brain Memory v8 they are the most directly relevant prior art, so it is worth being precise.

## Data model

Letta splits memory into three tiers ([letta.com/blog/agent-memory](https://www.letta.com/blog/agent-memory)):

- **Core memory** — in-context "memory blocks". Each block is `{label, description, value, limit}` where `limit` is a character cap. Blocks are always rendered into the system prompt, so they cost tokens every turn. Defaults include `persona` and `human` blocks. Blocks can be `read_only` and shared across agents ([memory-blocks docs](https://docs.letta.com/guides/agents/memory-blocks)).
- **Recall memory** — full message history, persisted automatically to the DB and queryable but not auto-rendered. Eviction kicks in around context-window pressure (recursive summarization of ~70% of older messages).
- **Archival memory** — semantically searchable passages with `{content, tags[], embedding}`. The agent's long-term notebook for things it explicitly chose to remember.

## Write path

Memory writes are **agent-driven via tool calls**, not background extraction by default. Base function set ([letta/functions/function_sets/base.py](https://github.com/letta-ai/letta/blob/main/letta/functions/function_sets/base.py)):

- `core_memory_append(label, content)`, `core_memory_replace(label, old_content, new_content)`
- Newer line-aware variants: `memory_insert(label, new_string, insert_line)`, `memory_replace(label, old_string, new_string)`, `memory_rethink(label, new_memory)`
- `archival_memory_insert(content, tags=None)`

The agent decides when to call these as part of normal tool-use. There is no implicit "after every turn, extract entities" hook — consolidation is delegated to sleep-time agents (below).

## Read path

Core blocks are **always loaded** — zero retrieval cost, but they spend tokens on every prompt. Everything else is on-demand:

- `archival_memory_search(query, tags=None, tag_match_mode="any"|"all", top_k, start_datetime, end_datetime)` — embedding-based semantic search backed by pgvector, with tag filters and a real time-range filter, paginated via `page=N`.
- `conversation_search(query, roles, limit, start_date, end_date)` — hybrid (text + semantic) over recall memory.

`top_k` and page size are agent-supplied; the underlying store is pgvector, so latency is fine into the millions ([AWS Aurora pgvector blog](https://aws.amazon.com/blogs/database/how-letta-builds-production-ready-ai-agents-with-amazon-aurora-postgresql/)).

## Persistence

Postgres + pgvector is the only first-class backend. `LETTA_PG_URI` configures it; the Docker image bundles `pgvector/pgvector:pg16` if you don't bring your own ([selfhosting/postgres docs](https://docs.letta.com/guides/selfhosting/postgres)). Schema is managed by Alembic. SQLite exists for trivial dev runs but is not the supported path. Multi-tenant is org/user-scoped at the API layer; self-host is single-tenant by default.

## Sleep-time agents / consolidation

The interesting recent work. A *sleep-time agent* is a second agent that **shares memory blocks** with the primary and runs in the background ([sleep-time-compute blog](https://www.letta.com/blog/sleep-time-compute), [sleeptime docs](https://docs.letta.com/guides/agents/architectures/sleeptime/)). It fires every `sleeptime_agent_frequency` steps (default 5), reads the new conversation slice, and rewrites memory blocks via `rethink_memory()` / `finish_rethinking_memory()`. It typically uses a stronger model than the primary. Output is "learned context" — distilled, deduped block content. The primary keeps serving while the sleep-time agent is mid-think; updates land asynchronously. There is open work on `consolidate_archival_memory()` to dedupe passages ([issue #3116](https://github.com/letta-ai/letta/issues/3116)) but it is not yet shipped.

## Filesystem-as-memory finding

Letta's [benchmarking blog](https://www.letta.com/blog/benchmarking-ai-agent-memory) ran agents on **LoCoMo** (long-conversation QA) using GPT-4o-mini and got **74.0%** with a Letta agent that uses plain filesystem-style tools (`grep`, `search_files`, `open`, `close`) over conversation files — beating Mem0's reported **68.5%** with their graph-based variant. Conclusion is deliberately deflationary: "memory is more about how agents manage context than the exact retrieval mechanism." Vector search is still under the hood (`search_files` is semantic), but the *interface* is a filesystem. Letta v1's agent loop ([letta-v1-agent blog](https://www.letta.com/blog/letta-v1-agent)) leans into this — heartbeats and `send_message` are deprecated; the loop is a generic tool-using agent and "Letta Filesystem" / "Context Repositories" become first-class.

## License and operational reality

Apache-2.0, Python 3.11+, runs as a server (FastAPI) on port 8283. Self-host is free; the realistic stack is `letta/letta:latest` + a pgvector Postgres container, ~1–2 GB RAM idle, plus whatever the embedding model needs. Letta Cloud: Pro $20/mo (20 agents), Max Lite $100, Max $200, plus an API plan at $0.10/active-agent/mo + $0.00015/sec tool exec ([letta.com/pricing](https://www.letta.com/pricing)). For a single-user Mac app, **self-host is the only sane path** — Cloud's per-agent billing is wrong shape for "one agent per Docknote user."

## Patterns Docknote should borrow

1. **Memory blocks as the primary abstraction for dossier cards.** Persona-shaped cards map almost 1:1 to `{label=person_id, description, value, limit}`. The character-budget discipline is exactly what dossier UX needs. *Risk:* Letta blocks are flat strings; Docknote dossiers are structured. You will end up with a typed schema *inside* the block value.
2. **Tag-filtered + time-range archival search.** `archival_memory_search(tags, start_datetime, end_datetime)` is exactly the API cross-meeting recall needs. *Risk:* tag granularity matters — auto-tagging quality dominates recall.
3. **Sleep-time consolidation between meetings.** A nightly/post-meeting agent that rewrites person and theme blocks is the right shape for "multi-meeting theme synthesis." *Risk:* sleep-time agents are non-deterministic rewriters. For a user-visible dossier, audit trail (diff before/after) and rollback are required.
4. **Filesystem-as-memory for transcripts.** Store each meeting transcript as a file the agent can `grep`/semantic-search. Cheap, debuggable, embeddings still happen under the hood.

## Patterns Docknote should NOT borrow

- **The whole Letta server.** FastAPI + Postgres + pgvector + Alembic on a Mac, per user, for a single-tenant app, is gross over-engineering. Use SQLite + `sqlite-vec` and skip the daemon.
- **MemGPT-style heartbeats and `send_message` plumbing.** Letta themselves deprecated this in v1.
- **Multi-agent coordination primitives.** Shared blocks, agent-to-agent messaging, org/user scoping — none of it is needed for one user on one Mac.
- **Recursive 70%-eviction summarization of recall.** Meetings already produce a summary artifact upstream; double-summarizing is lossy.
- **Letta's persona/human default blocks.** They are conversational-assistant shaped; meeting-recall needs entity-keyed blocks (people, projects, themes) from day one.

## Verdict

**(b) Borrow specific patterns and build.** Letta's `{label, description, value, limit}` block schema, tag+time-range archival search, sleep-time consolidation pattern, and "filesystem is competitive" finding are all directly transferable. But adopting Letta wholesale means running a Postgres+pgvector server on every user's Mac for a fraction of its multi-agent, multi-tenant feature set — wrong physics for a single-user macOS app. Reimplement the *ideas* on a SQLite + sqlite-vec substrate, treat Letta as the reference architecture you cite in design docs, not the runtime you ship.

---

# 2. Graphiti / Zep — deep-dive

Graphiti is the open-source temporal knowledge-graph engine that backs Zep. Companion paper ([arXiv 2501.13956](https://arxiv.org/abs/2501.13956)) is the canonical technical reference; implementation at [github.com/getzep/graphiti](https://github.com/getzep/graphiti) (Apache 2.0, ~20K stars).

## Bi-temporal model — concretely

Every fact carries two independent time axes: **event time** (when the fact was true in the world) and **transaction time** (when Graphiti learned it). On the wire this surfaces as four datetime fields on `EntityEdge`:

```python
class EntityEdge:
    uuid: str
    source_node_uuid: str
    target_node_uuid: str
    name: str        # relation label, e.g. "PRICED_AT"
    fact: str        # NL description
    created_at: datetime    # transaction time (ingest)
    valid_at:   datetime | None   # event time start
    invalid_at: datetime | None   # event time end (superseded)
    expired_at: datetime | None   # explicit retraction
    attributes: dict
    group_id: str
```

`EpisodicNode` carries `created_at` (ingest) plus `reference_time` (when the source artefact occurred), so a meeting transcript can be stamped with the actual meeting time independent of when you ingested it. `EntityNode` is conventional — `uuid`, `name`, `summary`, `labels`, `name_embedding`, `attributes` — with `summary` mutating over time as new episodes arrive ([help.getzep.com/graphiti](https://help.getzep.com/graphiti)).

## Ingestion pipeline

Episodes (chat messages, JSON, or free text up to a few KB) are added via `add_episode(...)`. Per episode the LLM is called multiple times: entity extraction, edge extraction, deduplication against existing nodes, summary update, and edge invalidation checks. Recent versions added a deterministic IR front-end (BM25 + entropy-gated fuzzy matching) so the LLM is only invoked when the lexical layer is uncertain ([blog.getzep.com/llm-rag-knowledge-graphs-faster-and-more-dynamic](https://blog.getzep.com/llm-rag-knowledge-graphs-faster-and-more-dynamic/)). Default LLM is OpenAI; Anthropic, Gemini, Groq, Ollama supported. Concurrency gated by `SEMAPHORE_LIMIT` (default 10).

**Cost in practice:** a 1-hour transcript is ~7-10K words ≈ 50-80 episodes if you chunk by speaker turn. With GPT-4o-mini, expect 4-8 LLM calls per episode → 200-600 LLM calls per meeting, on the order of $0.10-$0.40 in API spend. Wall-clock ingestion of a 1-hour meeting is typically 3-10 minutes depending on parallelism.

## Query API

```python
edges = await graphiti.search(
    query="what did Alex say about pricing in March",
    num_results=10,
    center_node_uuid=alex_uuid,   # optional: pivot on a node
)
```

Retrieval is cosine-similarity (embeddings) ∪ BM25 ∪ graph-distance from a `center_node_uuid`, fused with RRF/MMR/cross-encoder/episode-mentions reranking. Crucially, **no LLM call at query time** — Neo4j blog cites P95 ≈ 300ms ([neo4j.com/blog](https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/)). Temporal filtering ("in March") is a `valid_at` range predicate on returned edges, applied client-side or via custom Cypher.

## Conflict resolution + invalidation

Soft-invalidate, never hard-delete. When a new episode produces an edge that contradicts an existing one (same source/target/relation type with incompatible facts), the LLM decides whether the old edge is superseded; if so, the old edge gets `invalid_at = new_edge.valid_at` while the new edge becomes current. Provenance is preserved because every edge retains a back-pointer to the originating `EpisodicNode`, and the episode itself is immutable. Headline feature for "what did Alex's pricing position used to be?" queries.

## Backend

Graphiti speaks Neo4j 5.26+, FalkorDB 1.1.2+, Kuzu 0.11.2+, and Amazon Neptune. For a Mac-local app, embedded options are **Kuzu** (`KuzuDriver(db="/path/graphiti.kuzu")`, single file, no server) and **FalkorDB Lite** (subprocess, ARM64, still in-development per [issue #1240](https://github.com/getzep/graphiti/issues/1240)). **Caveat: Kuzu was archived in October 2025** ([issue #1132](https://github.com/getzep/graphiti/issues/1132)) — works but unmaintained. Neo4j Desktop runs locally on Mac but adds 500MB+ JVM.

Footprint at meeting scale (1000 meetings × ~70 episodes × ~5 entities/edges → ~5K nodes, ~25-50K edges + embeddings): on Kuzu/FalkorDB roughly **~500MB-1.5GB** disk; on Neo4j community closer to 2-3GB with indices. RAM dominated by HNSW index — budget 200-400MB resident.

## Benchmark numbers

On **DMR** (Deep Memory Retrieval): Zep 94.8% vs MemGPT 93.4% — narrow, paper itself flags DMR as saturating. The interesting numbers are **LongMemEval** ([blog.getzep.com/state-of-the-art-agent-memory](https://blog.getzep.com/state-of-the-art-agent-memory/)):

- Accuracy: Zep+GPT-4o **71.2%** vs full-context+GPT-4o **63.8%** (+18.5% relative against GPT-4o-mini)
- Median latency: Zep+GPT-4o **2.58s** vs full-context **28.9s** (~90% reduction)
- Tokens: Zep used **<2%** of full-context baseline tokens
- Sub-task lifts: temporal reasoning **+38-48%**, preference tracking **+78-184%**, multi-session synthesis **+17-31%**

These are exactly the categories Brain Memory v8 cares about.

## License + cost

Apache 2.0. Zep Cloud is metered: Flex **$125/mo** for 50K credits, Flex Plus **$375/mo** for 200K, Enterprise custom ([getzep.com/pricing](https://www.getzep.com/pricing/)). 1 credit = 350 bytes per episode → 1-hour meeting (~30KB transcript) ≈ 90 credits. Self-hosted is free of platform fees but you pay LLM bills directly.

## Failure modes

- **Entity disambiguation**: "Alex Smith" vs "A. Smith" vs "Alex" — fuzzy matching helps but breaks on common first names.
- **Coreference**: [issue #1171](https://github.com/getzep/graphiti/issues/1171) — pronouns dropped, edges get wrong subject. Brutal for meeting transcripts.
- **Cost spikes on noisy input**: messy ASR transcripts produce spurious entities, each triggering its own dedup pass.
- **Backend churn**: Kuzu archived, FalkorDB Lite not GA, Neo4j embedded means JVM. None of the embedded options are clearly future-proof.
- **No batch recompute**: incremental ingestion means you cannot cheaply re-extract with a better prompt/model — you'd re-pay the full ingestion bill.

## Patterns Docknote should borrow

- **Bi-temporal edges** (`valid_at` / `invalid_at` / `created_at`) — right shape for pivots and dossier evolution.
- **Episodes as immutable provenance** — every dossier card claim links back to a transcript span. Non-negotiable.
- **Hybrid retrieval (vector ∪ BM25 ∪ graph distance) with no-LLM query path** — sub-second recall.
- **Soft-invalidation, never delete** — supports "what changed about this person/topic over time."
- **Center-node search** (`center_node_uuid`) — natural primitive for persona dossiers.

## Patterns Docknote should NOT borrow

- **LLM-per-episode dedup at ingest time.** For ~10-meetings/day scale, deterministic resolver (email, Zoom display name, canonical-name) wins on cost, latency, reproducibility.
- **Full ontology of typed entity/edge classes.** For meetings you need {Person, Org, Topic, Decision, ActionItem}. Don't import the framework weight.
- **Graph DB infra.** Neo4j too heavy, Kuzu archived, FalkorDB Lite unproven. SQLite + sqlite-vec + a `facts` table gives 90% of the queries.
- **Community detection / clustering.** Pitched at enterprise corpora; one user's meetings have insufficient signal-to-noise.
- **Cloud-coupled architecture.** Zep Cloud is a non-starter for the default tier.

## Verdict

**(b) Borrow the bi-temporal facts pattern, but build the store.** Graphiti is the right *idea* and the LongMemEval numbers prove the model works on exactly the axes v8 targets. But the implementation is over-engineered for single-user Mac meeting recall: LLM-heavy ingestion, graph-DB dependency where every embedded option is compromised, entity-dedup failure modes that scale with transcript noise. Build a thin SQLite+sqlite-vec store with `(subject, predicate, object, fact, valid_at, invalid_at, created_at, episode_id)`, deterministic resolver, no-LLM hybrid query path. Steal Graphiti's schema vocabulary verbatim — it's well-considered — and revisit adoption only if Brain Memory grows into multi-user / shared-org territory.

---

# 3. Granola — deep-dive

Granola is a Mac/Windows/iOS AI notepad: light user notes, background transcription, AI-rewritten clean note, increasingly a memory layer over the corpus. Closest competitive analog to Brain Memory v8 ([granola.ai](https://www.granola.ai/), [reworked.co on $125M raise](https://www.reworked.co/digital-workplace/granola-raises-125m-launches-enterprise-context-tools/)).

## Product surface for memory

Three nouns and one verb:

- **Folders / Team Folders / "Spaces."** Sidebar folders ("Sales Calls," "Customer Feedback," "Hiring Loops"). Private by default; can be shared via URL. Templates exist for User Research, Sales Pipeline, Interview Loop ([Team Folders blog](https://www.granola.ai/blog/say-hello-to-team-folders), [2.0 blog](https://www.granola.ai/blog/two-dot-zero)).
- **People & Companies view.** Two icons in bottom-left of sidebar. Clicking surfaces every note that mentions them. Auto-extracted from calendar attendees. Basic plan: rolling 30-day. Business: full lifetime. **No rich "dossier card"** — effectively a filtered note list, not a synthesized profile ([help.granola.ai/people-and-companies](https://help.granola.ai/article/people-and-companies)).
- **Granola Chat.** `/`-invoked chat against (a) one meeting, (b) folder, (c) Person/Company, (d) entire corpus. "Recipes" are pre-written prompts authored by Lenny Rachitsky, Matt Mochary, users ([Recipes blog](https://www.granola.ai/blog/say-hello-to-recipes)).
- **Citations.** "Inline citations and a jump-to-source link" on every chat answer; "cites the exact lines." The single feature reviewers most consistently praise ([Chat smarter blog](https://www.granola.ai/blog/granola-chat-just-got-smarter)).
- **Pre-meeting brief.** Calendar-linked: a Granola note page is pre-created from each calendar event before the call; "Prep next meeting" recipe pulls prior context. Closer to "open the note, run a recipe" than a true async briefing.

## Storage architecture

Local DB at `~/Library/Application Support/Granola` as SQLite. Until ~March 2026 it was a plain `.db` that the community built scrapers/MCP servers/Obsidian sync against. **March 2026 Granola encrypted the local database** — set off the public backlash. [Shadow.do teardown](https://www.shadow.do/blog/granola-encrypted-its-local-database-heres-why-that-matters----and-what-to-use-instead) and [themeridiem](https://themeridiem.com/enterprise-technology/2026/4/2/ai-meeting-tools-hit-privacy-inflection-as-granola-default-exposes-notes) discuss the file existence and switchover; pre-encryption community tools implied tables for documents/notes, transcripts (segmented), attendee/calendar metadata, but no canonical reverse-engineering post survives publicly post-encryption.

Audio retention is **zero**: "Granola doesn't store the audio from meetings — it transcribes in real time on macOS/Windows, or after your meeting using temporarily cached audio on iOS." Transcripts and notes kept; users can delete individual notes or request full deletion ([granola.ai/security](https://www.granola.ai/security)).

## Cloud dependency

Granola is **not** local-first for inference. Notes and transcripts stored in **US AWS VPC**, encrypted at rest and in transit, daily backups. Inference against "top models across providers (OpenAI, Anthropic, Google, etc)." Granola contractually disallows third-party training. Granola itself trains on anonymized data by default; opt-out is in Settings, Enterprise has it off by default.

## Cross-meeting recall mechanics

Granola is **deliberately opaque** about implementation. Official line: Chat was "rebuilt from the ground up to be agentic," uses "reasoning models" for "detailed analysis across large numbers of meetings," accesses meeting notes, transcripts, and Team Space notes. No vector-DB, embeddings, or RAG pipeline named publicly. User-visible mechanics: (1) scope selector — single meeting / folder / person / company / global; (2) Personal API and Enterprise API expose context to external agents (Claude, Figma Make).

Practically, when user asks "what did Alex say about pricing in March?", Granola appears to (a) restrict candidate set by scope, (b) pass an agentic loop over constrained transcripts to a reasoning model, (c) emit answers with line-level citations. Time-window filtering is implicit through prompt parsing — no documented date facet UI.

## Pricing

- **Basic — $0.** AI notes, AI chat within and across meetings, shared folders, templates, multi-language, opt-out training. **Limited meeting history**, People/Companies rolling-30-day.
- **Business — $14/user/mo.** Unlimited history; "advanced AI thinking models"; Attio/Notion/Slack/HubSpot/Affinity/Zapier; **MCP integration**; Personal API; centralized billing.
- **Enterprise — $35/user/mo.** SSO, admin, Enterprise API, org-wide auto-deletion, team-wide training opt-out by default.

## Critiques and gaps

- **Encrypted-DB lock-in (March 2026).** Guido Appenzeller publicly trashed the change ("the app now has zero value" for AI-native flows); Granola's "use MCP instead" answer was rejected ("We don't use MCP, only APIs"). Community pivoted to plain-Markdown stacks.
- **Privacy defaults.** Notes default link-shareable; users opted into training unless they dig. A TestFlight build shipped a hard-coded API key.
- **Speaker labels.** Transcripts are continuous text; speaker attribution unreliable past 2-3 participants.
- **No audio playback.** Transcript errors unverifiable.
- **Folders are flat.** "Shared / private" is the only axis.
- **Ask-Granola state loss.** "When you go out of a meeting and then re-enter, all of your Ask Granola questions and answers are gone."
- **Export friction.** Copy/paste or Zapier; no native Google Doc sync.
- **No Android.**
- **People view is thin.** Filtered note list, not synthesized dossier card.

## Where Docknote can differentiate

1. **Local LLM via SnapGPU.** No cloud retention, no AWS VPC, no OpenAI/Anthropic egress.
2. **Persona-shaped dossier cards.** Granola's People view is reportedly a note list. Docknote ships real synthesized profile (role, projects, recurring topics, decisions, open threads) with explicit synthesis steps.
3. **Pivot-based recall.** Granola is folder-or-person scoped; Docknote pivots across topic / decision / commitment / artifact.
4. **Open data on disk.** Markdown-on-disk, exportable, scriptable, MCP-native from day one. The encrypted-DB backlash is a live wound.
5. **Persistent chat state.**

## Where Granola is right and Docknote should match

- **Inline citations with jump-to-source.** Non-negotiable.
- **Scope selector on chat.** Same UI affordance.
- **Calendar-pre-created note pages.**
- **Recipes (curated prompt library).** Lenny/Mochary-authored recipes drive adoption.
- **Folder templates** for known workflows.
- **Shareable URLs** to non-users for read-only handoff.

## Verdict

**Copy:** (1) inline jump-to-source citations on every chat answer, (2) scope selector, (3) calendar-pre-created note pages. **Differentiate on:** (1) local-LLM via SnapGPU — turn Granola's encrypted-DB-and-AWS-VPC posture into Docknote's affirmative "your audio, transcripts, and inference never leave your tailnet" story; (2) real persona dossier cards; (3) Markdown-on-disk + MCP-native data plane that explicitly inverts Granola's March-2026 lock-in moment. The encrypted-DB episode is the strategic opening — Granola's product is strong, but its data-ownership story is now a liability and a permanent talking point.

---

# 4. Minutes (silverstein) — deep-dive

A privacy-first conversation-memory layer that writes every meeting to `~/meetings/` as Markdown and exposes the corpus through a 29-tool MCP server. Closest public analogue to the Brain Memory v8 vision.

## File layout

One `.md` per meeting in `~/meetings/`, ISO-date-prefixed slug:

```
~/meetings/2026-03-17-pricing-discussion.md
~/meetings/2026-03-25-standup.md
```

Inbox/voice-memo intake lives in `~/.minutes/inbox/`. Frontmatter is structured YAML with first-class arrays for `action_items` and `decisions` ([repo](https://github.com/silverstein/minutes)):

```yaml
---
title: Q2 Pricing Discussion with Alex
type: meeting
date: 2026-03-17T14:00:00
duration: 42m
context: "Discuss Q2 pricing, follow up on annual billing decision"
action_items:
  - assignee: mat
    task: Send pricing doc
    due: Friday
    status: open
decisions:
  - text: Run pricing experiment at monthly billing with 10 advisors
    topic: pricing experiment
---
## Summary
- Alex proposed lowering API launch timeline ...
## Transcript
[SPEAKER_0 0:00] So let's talk about the pricing...
```

Two body sections — `## Summary` (bullets) and `## Transcript` (timestamped diarized lines). Compatible with Obsidian/Logseq/`grep`.

## Capture pipeline

`Audio → Transcribe → Diarize → Summarize → Markdown → Relationship Graph`. Transcription local (`whisper.cpp` or Parakeet); diarization local (`pyannote-rs`); summarization optional and pluggable (Claude / Ollama / Mistral / any OpenAI-compatible endpoint). Silero VAD against Whisper hallucination. Voice memos < 2min skip diarization. Phone capture rides iCloud/Dropbox sync into `~/.minutes/inbox/`. No calendar integration — manual or hotkey start; v0.13 added Zoom/Teams call-aware auto-start.

## MCP server surface

TypeScript/Node, shipped as `npx minutes-mcp`. **Twenty-nine tools across eight categories** ([useminutes.app/docs/mcp/tools](https://useminutes.app/docs/mcp/tools)):

- Recording: `start_recording`, `stop_recording`, `get_status`, `list_processing_jobs`
- Search/recall: `list_meetings`, `search_meetings`, `get_meeting`, `activity_summary`, `search_context`, `get_moment`, `research_topic`
- Notes/processing: `process_audio`, `add_note`, `open_dashboard`
- People/relationships: `consistency_report`, `get_person_profile`, `track_commitments`, `relationship_map`
- Live: `start_live_transcript`, `read_live_transcript`, `start_dictation`, `stop_dictation`
- Voice ID: `list_voices`, `confirm_speaker`
- Insights: `get_meeting_insights`, `ingest_meeting`, `knowledge_status`
- QMD integration: `qmd_collection_status`, `register_qmd_collection`

Plus stable resource URIs: `minutes://meetings/recent`, `minutes://meetings/{slug}`, `minutes://actions/open`, `minutes://ideas/recent`, `minutes://events/recent`, `minutes://status`, `ui://minutes/dashboard`. Per-tool input/output JSON schemas live only in `capabilities.ts` (not in public docs).

## Retrieval implementation

Three-layer:
1. **SQLite FTS5** index built from the Markdown corpus (added v0.16.0). Powers `minutes search "pricing"` and `search_meetings`.
2. **SQLite relationship index** rebuilt from frontmatter in <50ms — powers `get_person_profile`, `track_commitments`, `relationship_map`, `consistency_report`.
3. **Optional `qmd` engine** swap via `[search] engine = "qmd"` for semantic/embedding search.

Markdown-on-disk is source of truth; both indexes are derived and disposable. No vendor-managed embedding store; no LLM-mediated retrieval pass — retrieval is deterministic FTS+SQL, then the LLM client consumes the structured result.

## Cross-meeting synthesis

Mostly retrieval, with light extraction. Goes only as far as: per-person profile rollup, commitment tracker across meetings, contradiction surface (`consistency_report`). **No narrative dossier, no day-headline, no role-line, no concept clustering.** Relationship graph keyed on names appearing in frontmatter — no entity coreference resolution beyond duplicate-contact suggestion. **This is the gap Docknote v8 fills.**

## Operational footprint

Single Mac (or Win/Linux). Two processes: Tauri menu-bar app (Rust core + WebView) and on-demand `minutes-mcp` Node process. GPU-accelerated transcription (Metal/CUDA/ROCm/Vulkan). Brew cask for desktop, `brew install minutes` for CLI, `cargo install minutes-cli` from source, `npx minutes-mcp` for MCP-only. State under `~/meetings/` and `~/.minutes/` — uninstall is `brew uninstall` plus `rm -rf`.

## License + maturity

MIT, single primary author (Mat Silverstein, X1 Wealth). 1.2k stars, 120 forks, 41 releases, 960+ commits, latest v0.16.0 on 2026-04-29 — weekly cadence. ~15 open issues; pain centers on audio reliability and packaging (binary signing, semver). Issue response fast — author engaged. Codebase: Rust 63% (`crates/core`, `crates/cli`, `crates/whisper-guard`, `crates/reader`), TypeScript ~9% (`crates/mcp`, `crates/sdk`, `tauri/`).

## What Docknote should borrow

- **File naming**: `YYYY-MM-DD-slug.md` for grep/Obsidian compatibility.
- **Frontmatter shape**: arrays of typed objects for `action_items` and `decisions`. Add `type:`, `duration:`, `context:`.
- **Two-section body**: `## Summary` bullets + `## Transcript` with `[SPEAKER_n M:SS]` markers.
- **MCP tool names**: `list_meetings`, `search_meetings`, `get_meeting`, `get_person_profile`, `track_commitments`, `consistency_report` — proven; copying lowers cognitive load.
- **Resource URI scheme**: `minutes://...` style → `docknote://...`.
- **Retrieval stack**: SQLite FTS5 over derived index, rebuilt from Markdown — exactly the right "no magic" pattern. Don't reach for embeddings until FTS5 demonstrably fails.
- **Append-only event stream** with stable cursors — agents need this for resumable workflows.

## What Docknote should improve / replace

- **Synthesis pass**: produce `role_line`, `day_headline`, `stat_line`, persona dossier cards.
- **Pivot routing**: single retrieval surface that distinguishes name vs. date vs. concept queries and rewrites accordingly.
- **Entity coreference**: "Alex" / "Alex Chen" / `@alex` should collapse.
- **Age-banded staleness**: explicit decay tiers with per-band retrieval policy.
- **Structured insight types beyond `decision`/`action_item`**: `question`, `risk`, `claim`, `commitment-to-self`.
- **Cross-source recall** (Slack, email, docs).

## Verdict — fork vs. reference vs. skip

**Reference heavily, do not fork.** The architectural shape (Markdown source-of-truth + SQLite-derived indexes + MCP server) is exactly right and we should copy file format, tool names, and retrieval stack near-verbatim for ecosystem compatibility. But the implementation is a Rust workspace tightly coupled to whisper.cpp/pyannote-rs/Tauri capture — Docknote's center of gravity is synthesis, persona memory, and cross-source intake, none of which Minutes is wired for. Forking inherits audio-capture surface area we don't need and doesn't accelerate the dossier/pivot/coreference work, which is essentially all-new code. Greenfield in our preferred stack, treat the Minutes MCP tool list and frontmatter schema as a de-facto compatibility target so users can swap engines, and cite the project as prior art. MIT means we can lift specific files (e.g. `capabilities.ts`'s tool schemas) verbatim if useful.

---

# 5. Anthropic Memory Tool + Context Engineering — deep-dive

## Wire format: typed CRUD over `/memories`

The memory tool is a server-defined tool with type `memory_20250818`, enabled by adding a single entry to the `tools` array — no JSON schema sent by the developer:

```json
"tools": [{ "type": "memory_20250818", "name": "memory" }]
```

Behind that the model emits six commands, all rooted at `/memories`:

- `view` — `{ command, path, view_range?: [start, end] }` — directories list 2 levels deep with sizes; files come back with 1-indexed, 6-char-wide line numbers separated by tab.
- `create` — `{ command, path, file_text }` — must error if file exists ("Error: File {path} already exists").
- `str_replace` — `{ command, path, old_str, new_str }` — must error on missing or non-unique match.
- `insert` — `{ command, path, insert_line, insert_text }` — `insert_line` valid range `[0, n_lines]`.
- `delete` — `{ command, path }` — recursive for directories.
- `rename` — `{ command, old_path, new_path }` — must NOT overwrite an existing destination.

Activation requires the beta header `context-management-2025-06-27`. SDKs ship a `BetaAbstractMemoryTool` (Python) / `betaMemoryTool` (TS) helper so you only implement the storage backend ([memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)).

## Storage contract

The tool is a **protocol, not a service**. The model produces tool calls; your application executes them. Anthropic's stated requirements:

- **Path traversal protection is mandatory.** Validate every path starts with `/memories`, canonicalize, reject `../`, `..\\`, `%2e%2e%2f`. Use `pathlib.Path.resolve()` + `relative_to()`.
- **Exact error strings matter** — Claude is trained against them. Custom errors degrade retry behavior.
- **Line-numbered, header-prefixed `view` output** is part of the contract. The model prompts itself off the formatting.
- **No atomicity or concurrency guarantees specified.** Developer's problem; for Docknote at minimum file locks per path and write-via-tempfile-then-rename.
- **Sensitive-data filtering is the developer's job.** Anthropic says Claude "usually refuses" to write secrets, which is not a security boundary.
- **Size caps and TTL** are recommended, not enforced.

The system prompt auto-injects when the tool is enabled:

> "IMPORTANT: ALWAYS VIEW YOUR MEMORY DIRECTORY BEFORE DOING ANYTHING ELSE. … ASSUME INTERRUPTION: Your context window might be reset at any moment, so you risk losing any progress that is not recorded in your memory directory."

## Composition with context editing

The "context management" launch ships two paired features: **context editing** auto-clears stale tool results client-side as you approach the window; the **memory tool** persists across sessions ([Anthropic blog](https://claude.com/blog/context-management)). They compose because context editing throws away bulky tool I/O while memory keeps the distilled lesson. Anthropic also recommends pairing with **server-side compaction**: compaction summarizes the live conversation, memory survives the summary boundary.

## Token economics

Anthropic's published numbers (internal eval):
- Memory + context editing combined: **+39% performance** on agentic evals.
- Context editing alone: **+29%**.
- 100-turn web-search workload: **84% token reduction** with context editing.

The 84% figure is **token-consumption reduction on a single long task**, not steady-state savings. Reproducible only for tool-heavy agents where most tokens are tool output. For Docknote's typical "one meeting, ask three questions" usage the win is closer to zero — the value of memory there is correctness, not tokens.

## Difference from ChatGPT memory

| | Claude memory | ChatGPT memory |
|---|---|---|
| Activation | Explicit / on-request; agent calls a tool | Always loaded into system prompt |
| Form | Raw conversation search + file directory; agent decides what's relevant | Extracted facts (Bio), 40-conversation history, profile blocks, behavioral metadata — all injected as text |
| Per-chat default | **Blank slate** unless the model fetches | Pre-populated |
| Latency | One+ round-trip per recall | Free, but pollutes every prompt |

Shlok's reverse-engineering shows Claude.ai uses `conversation_search` and `recent_chats` over the **raw chat log**, no synthesis ([Claude memory](https://www.shloked.com/writing/claude-memory)). ChatGPT instead bets on Sutton's bitter lesson: stuff everything in, the model sorts it out ([ChatGPT memory](https://www.shloked.com/writing/chatgpt-memory-bitter-lesson)). The memory **tool** API is the productized version of Claude's philosophy. Note: Claude.ai's *consumer* memory is different — it auto-syntheses every 24h and pre-loads each chat; the API memory tool is the strict, opt-in version.

## Implications for Brain Memory exposure

Docknote already has an MCP server. Adding the memory tool means **two retrieval surfaces** the agent must route between:

- **Memory tool (`/memories`)** = scratchpad + session continuity. Agent-owned, agent-pruned, ephemeral-ish, written by the agent itself.
- **MCP `brain.search`** = canonical knowledge base. User-owned, indexed, embedding/keyword search over the v8 markdown corpus.

Routing prompt:

> Use `memory` for things YOU recorded in the current project (todos, decisions, scratch state). Use `brain.search` (MCP) for anything the USER recorded — meetings, transcripts, notes, prior chats from other tools. If a user asks "what did I discuss with X" — that's `brain.search`. If you ask yourself "where was I in this task" — that's `memory.view /memories`.

Mapping `~/Docknote/brain/` directly behind the memory tool's `/memories` root is tempting but wrong — it conflates agent-owned and user-owned data and exposes the brain to `delete`/`str_replace` from any agent. Use a sandboxed `~/Docknote/brain/.agent-memory/<session-id>/` directory for the memory tool, keep `brain/` strictly behind MCP read tools.

## Privacy / cloud surface

Every `view`, `create`, `str_replace` ships **the actual file contents** to Anthropic in the next turn — that's how the model uses them. ZDR means no retention, but it's still in transit and processed cloud-side. Not compatible with Docknote's "the brain never leaves the device" claim when routing target is Claude. Posture:

- **SnapGPU + Qwen path:** memory tool fine, all local.
- **Claude API path:** gate which files the memory tool can surface; default-deny for anything tagged sensitive in v8 frontmatter; show user a "this query will send N memory files to Anthropic" preview, like a tool-permission prompt.

## Production-grade considerations

- **Tool calls fail like any other tool call** — return `is_error: true` with canonical error string.
- **Idempotency**: `create` non-idempotent (errors on re-create), `str_replace` non-idempotent (second call won't find `old_str`). Mid-stream retries on your side must not double-apply.
- **Concurrency**: two parallel agent sessions can race. Use per-path advisory locks; on conflict, return error string the model can interpret.
- **Mid-conversation failure** of memory writes is the worst case: model thinks it persisted; user thinks it persisted; reality differs. Make writes synchronous, fsync, return success only after fsync.

## What Docknote should adopt directly

The **wire format**. Six-command, `/memories`-rooted protocol is simple, well-specified, has SDK helpers, and any client that speaks it (Claude API today; likely Cursor, Zed, others tomorrow) can read Docknote scratchpad without a custom adapter. The Anthropic memory-tool format is the closest thing to an interop standard for "agent-writable persistent storage" in 2026, and the cost of implementing it is one well-tested file backend.

## What Docknote should NOT adopt

- **Blank-slate-per-chat semantics.** Docknote's product promise is "your assistant always knows you" — closer to ChatGPT memory. Always-loaded persona context belongs in the system prompt regardless of memory tool, injected by Docknote, not pulled by the model. The memory tool is *additional* recall, not *primary*.
- **The "Claude writes whatever it wants" default.** Anthropic's prompt invites the model to scribble freely. Restrict the memory tool to a sandbox dir; canonical brain writes go through MCP `brain.write` with schema validation.
- **Cloud-by-default routing.** The memory tool is route-agnostic; default reads to SnapGPU/local Qwen and require explicit consent before memory contents leave the device.
- **The auto-injected "ALWAYS VIEW MEMORY DIRECTORY" preamble** verbatim. Docknote's agent has more retrieval surfaces; replace with the routing prompt above.

## Verdict

**(a) Implement the memory-tool wire format as a primary interface — but only as the *agent-scratchpad* surface, not as the canonical brain interface.** Cheap to implement, decouples Docknote from any single LLM vendor, gives session continuity for free. Brain Memory v8 itself stays behind MCP, where Docknote owns schema, indexing, and privacy. Use the memory tool for what it's actually good at — agent-owned ephemeral state — and keep user-owned knowledge on the MCP side where local-first and the privacy moat are non-negotiable. The risk of vendor lock-in is low because the protocol is plain JSON CRUD; the upside of standards-compatibility is high.

---

## Source index

**Letta**: [GitHub](https://github.com/letta-ai/letta) · [agent-memory blog](https://www.letta.com/blog/agent-memory) · [v1 agent](https://www.letta.com/blog/letta-v1-agent) · [benchmarking](https://www.letta.com/blog/benchmarking-ai-agent-memory) · [sleep-time](https://www.letta.com/blog/sleep-time-compute) · [paper](https://arxiv.org/abs/2310.08560) · [memory blocks](https://docs.letta.com/guides/agents/memory-blocks) · [pricing](https://www.letta.com/pricing)

**Graphiti / Zep**: [GitHub](https://github.com/getzep/graphiti) · [paper](https://arxiv.org/abs/2501.13956) · [TKG architecture](https://blog.getzep.com/zep-a-temporal-knowledge-graph-architecture-for-agent-memory/) · [SOTA benchmarks](https://blog.getzep.com/state-of-the-art-agent-memory/) · [Neo4j](https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/) · [pricing](https://www.getzep.com/pricing/) · [Kuzu archived #1132](https://github.com/getzep/graphiti/issues/1132)

**Granola**: [home](https://www.granola.ai/) · [security](https://www.granola.ai/security) · [pricing](https://www.granola.ai/pricing) · [2.0](https://www.granola.ai/blog/two-dot-zero) · [Team Folders](https://www.granola.ai/blog/say-hello-to-team-folders) · [Chat smarter](https://www.granola.ai/blog/granola-chat-just-got-smarter) · [Recipes](https://www.granola.ai/blog/say-hello-to-recipes) · [People & Companies help](https://help.granola.ai/article/people-and-companies) · [$125M raise](https://www.reworked.co/digital-workplace/granola-raises-125m-launches-enterprise-context-tools/) · [encrypted-DB teardown](https://www.shadow.do/blog/granola-encrypted-its-local-database-heres-why-that-matters----and-what-to-use-instead) · [tldv review](https://tldv.io/blog/granola-review/) · [themeridiem](https://themeridiem.com/enterprise-technology/2026/4/2/ai-meeting-tools-hit-privacy-inflection-as-granola-default-exposes-notes)

**Minutes**: [GitHub](https://github.com/silverstein/minutes) · [MCP tools docs](https://useminutes.app/docs/mcp/tools) · [useminutes.app](https://www.useminutes.app/) · [releases](https://github.com/silverstein/minutes/releases)

**Anthropic memory tool**: [docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) · [context engineering blog](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) · [context-management announcement](https://claude.com/blog/context-management) · [Claude memory help](https://support.claude.com/en/articles/11817273-use-claude-s-chat-search-and-memory-to-build-on-previous-context) · [Shlok on Claude memory](https://www.shloked.com/writing/claude-memory) · [Shlok on ChatGPT memory](https://www.shloked.com/writing/chatgpt-memory-bitter-lesson) · [Embrace The Red](https://embracethered.com/blog/posts/2025/chatgpt-how-does-chat-history-memory-preferences-work/) · [Leonie Monigatti](https://www.leoniemonigatti.com/blog/claude-memory-tool.html)
