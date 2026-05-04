# Brain Memory v8 — Tier 2 deep-dives

**Date:** 2026-05-04
**Companions:**
- `docknote-memory-learn-from-cc.md` (Claude Code lessons)
- `docknote-memory-leads.md` (broad-sweep leads, ~155 entries)
- `docknote-memory-deepdive-tier1.md` (Letta, Graphiti, Granola, Minutes, Anthropic memory tool)

**Targets covered (Tier 2):**
1. mem0 — extract-and-store pipeline (the canonical extract reference)
2. MemMachine — keep-raw-and-project (the counterargument)
3. HippoRAG 2 + RAPTOR — synthesis-pass architecture without graph DBs
4. Letta sleep-time agents (deeper) — actual prompts, code, concurrency model
5. Cline + cursor memory bank — explicit-edit UX from coding-tool memory

---

## Synthesis — what these five collectively tell us

### 1. Extract-vs-raw is decided: hybrid, with raw as source of truth

Both deep-dives converge from opposite poles to the same architecture:

- **MemMachine evidence:** ground-truth-preserving raw episodes outperform mem0's extracted facts at chat scale (~80% fewer input tokens), but the saving is largely a *write-time* effect that narrows as corpus grows. MemMachine's "1 prev + 2 next episode" expansion rule fails at meeting scale where each episode is 10K+ words.
- **mem0 evidence:** the two-phase extract → reconcile shape is sound, but per-message extraction over-fragments at meeting scale, LLM-only dedup produces the lost-creation/flaky-cross-session bugs Deshraj himself documented, and treating extraction *as* truth means errors are unrecoverable.

**Decision:** Keep the raw transcript verbatim as ground truth (Markdown + sentence-level embeddings). Layer **one bounded typed-extraction pass per meeting close** producing `decisions`, `action_items`, `people`, `themes`, each with `sources: [{meeting_id, ts_span}]` pointers back to the transcript. Use mem0's ADD/UPDATE/DELETE/NONE event vocabulary mapped onto Graphiti's bi-temporal `valid_at`/`invalid_at` so UPDATE closes the old interval and opens a new one — never destructive delete. Gate the update LLM with an embedding-similarity check (cosine ≥ 0.92) before invoking the reconciler.

### 2. Synthesis pass: RAPTOR, skip HippoRAG 2 (for now)

Five reasons RAPTOR wins for v8:

1. **Query type matches.** "Recurring concerns about pricing across meetings" is sense-making (RAPTOR's QuALITY +20pp SOTA), not multi-hop factoid (HippoRAG 2's MuSiQue strength).
2. **Build economics are decisive at single-user scale.** ~$15-25 one-time RAPTOR build for 1K meetings vs ~$300+ for HippoRAG 2's OpenIE-on-every-passage step.
3. **Storage and engineering simplicity.** RAPTOR is two SQLite tables (`nodes`, `edges`) plus a vector index. HippoRAG 2 adds a phrase-KG, custom PPR runtime, OpenIE worker.
4. **Synthesis layer fit.** RAPTOR's internal nodes *are* proto-themes; the final theme-extraction prompt has dramatically less work to do over RAPTOR nodes than over raw passages.
5. **Optionality preserved.** HippoRAG 2's recognition-memory filter can be grafted later as a 50-line re-ranker if eval shows multi-hop factoid recall is the failure mode.

**Concrete shape:** `meeting → chunks(100tok) → UMAP+GMM soft cluster → GPT-4o-mini summary at each level → 3-level tree per meeting → weekly rollup cluster → quarterly rollup`. Query: `embed → ANN over flattened forest, top-20 within 2K-token budget → optional recognition-filter LLM call → theme-extraction prompt → {themes: [{name, citations: [meeting_ids]}]}`.

### 3. Sleep-time consolidation: two cadences, mandatory audit trail

Letta's pattern is right but the execution has zero audit trail and last-write-wins concurrency — both unacceptable for a user-facing dossier. Brain Memory v8's nightly pass diverges:

- **Two firings**, not every-N-steps:
  - **Per-meeting close** (incremental): per-card rewrites for entities mentioned in just-ended meeting. Fast model.
  - **Nightly 03:00** (batch): theme rollups, age-banding, cross-card reconciliation, assertion-equivalence dedup. Strong model (Opus / GPT-5-class).
- **Field-level tools** instead of Letta's blob `rethink_memory`: `update_card_field(card_id, field, new_value, sources=[{meeting_id, ts_span}])`, `demote_field(field)`, `reconcile_flag(field, reason)`.
- **Mandatory `sources` on every write.** Empty-source rewrites rejected before commit.
- **Transactional `start/finish_consolidation`** so partial intermediate states never escape (Letta's biggest defect).
- **Persistent `revisions/<card_id>/<iso8601>.json` ledger** with `{before, after, sources, model, prompt_hash}`. Powers UI hover ("why does it say this?") and one-click rollback.
- **Concurrency:** advisory per-card lock during meeting-close writes; nightly batch refuses to run if any card is mid-write. The day window and 03:00 window don't overlap in practice.

### 4. Edit UX: invert Cline's pattern

Adopt Cline's file taxonomy and "read on every turn" discipline; reverse the edit model.

- **File taxonomy:** `persona.md`, `active.md`, `patterns.md`, `commitments.md`, `person/<slug>.md`, `project/<slug>.md`, `meeting/<date-slug>.md`.
- **Auto-load rule:** `persona.md` + `active.md` always; entity cards loaded by retrieval against the current turn's named entities (cap ~6). This is cursor-memory-bank's mode-driven loading insight applied to entity scope rather than complexity level.
- **Human edits always win.** Every field carries `source: auto|human` + `last_edited`. Human-touched fields are immutable to the consolidation pass, which must propose changes into a separate `proposed_updates` queue rather than overwrite.
- **"Brain inbox" UI** listing every auto-extracted change since last review, with sentence-level diff, per-field accept/reject, and one-click revert backed by 5-deep field history.
- **Trigger phrase:** `update brain` forces a full consolidation sweep, like Cline's `update memory bank`.

The Cline pattern's load discipline transfers cleanly; its edit anarchy (no edit-boundary, git as the only audit, unbounded growth) does not.

### 5. Stage-0 coarse filter is now specified

The leads doc flagged "stage-0 coarse filter at meeting scale" as unresolved. Tier 2 dives resolve it through the union of mem0's scoping triple, Cline's selective load, and Cursor's mode-driven loading:

- **Always-loaded** (zero retrieval cost): `persona.md` + `active.md` + the user's pinned-projects list.
- **Stage 0 (deterministic, sub-ms):** substring + normalized-name match against entity index (Person table indexed on `aliases` + email + Zoom ID); date-range filter on `meeting/` index; scope filter (workspace/project). Cap candidate set to ~50.
- **Stage 1 (LLM-over-manifest):** pick ≤5 dossier cards / meeting summaries from filtered manifest. Sonnet-class model, header-only manifest (filename + description + mtime).
- **Stage 2 (RAPTOR retrieval):** collapsed-tree top-K within 2K-token budget over the chosen subset.
- **Stage 3 (recognition-filter, optional):** HippoRAG 2-style LLM call to prune irrelevant nodes before synthesis.
- **Stage 4 (synthesis):** theme-extraction prompt → `{themes: [{name, citations: [meeting_ids]}]}`.

---

## Updated decision-axis status — all five resolved

| Axis | Verdict | Source |
|---|---|---|
| Extract-and-store vs. keep-raw-and-project | **Hybrid:** raw transcripts as truth, typed extraction at meeting close as derived index, mem0 ADD/UPDATE/DELETE/NONE → Graphiti bi-temporal mutations | Tier 2 §1 |
| Filesystem vs. KG vs. vector vs. hybrid | **Markdown source-of-truth + SQLite/sqlite-vec FTS5 + dense vectors. RAPTOR forest as synthesis substrate. No graph DB.** | Tier 1 + Tier 2 §2 |
| Implicit auto-extract vs. explicit user-edited | **Both, with human-wins:** auto-extract at meeting close + nightly consolidation; per-field human-lock + Brain Inbox for proposed updates | Tier 2 §4 |
| Per-session blank slate vs. always-loaded global | **Always-loaded persona/active.md + Anthropic memory tool as scratchpad** | Tier 1 + Tier 2 §4 |
| Cloud vs. local-first | **Local-first by default** (SnapGPU/Qwen). Memory contents never auto-leave the tailnet. | Phase-2 thesis |

The architecture is **~95% specified**. What remains is SDD-level detail — exact frontmatter schemas, exact prompts for each pass, eval harness against the 24-persona corpus.

---

## Final v8 architecture (one-page summary)

```
~/Docknote/brain/
├── persona.md              # always-loaded
├── active.md               # always-loaded (rolling 14d)
├── patterns.md             # cross-meeting themes
├── commitments.md          # open action items / decisions
├── person/<slug>.md        # one per person, frontmatter-typed
├── project/<slug>.md       # one per project
├── meeting/<YYYY-MM-DD-slug>.md   # one per meeting (Minutes-compatible)
├── revisions/<card>/<iso8601>.json  # audit ledger
└── .agent-memory/<session-id>/      # Anthropic memory-tool sandbox

~/Library/Application Support/Docknote/brain.db
├── nodes (RAPTOR forest)
├── edges (RAPTOR parent-child)
├── facts (bi-temporal: subject, predicate, object, fact,
│         valid_at, invalid_at, created_at, episode_id)
├── entities (Person/Org/Topic, with aliases/email/zoom_id)
├── meeting_index (FTS5 + dense vectors)
└── sentence_index (sentence-level embeddings → meeting_id, ts_span)
```

**Write path** (meeting close):
1. Save full transcript to `meeting/<date-slug>.md` (Minutes-compatible frontmatter).
2. Sentence-level embeddings → `sentence_index`.
3. **Topic-segment extraction** (one bounded LLM call per ~3-min segment): emit `{decisions, action_items, claims, people, themes}` with span citations.
4. **Update reconciler** (mem0 two-phase): for each new fact, embedding-similarity gate → if below threshold, ADD; otherwise LLM-decide ADD/UPDATE/NONE → mutate `facts` table with bi-temporal interval semantics.
5. **Per-card incremental consolidation** (Letta sleep-time pattern + audit): rewrite touched dossier cards via field-level tools, mandatory sources, transactional commit, ledger entry.
6. **RAPTOR build** for the meeting: chunks → UMAP+GMM → 3-level tree.

**Nightly 03:00 batch:**
1. Cross-card reconciliation + age-banding.
2. RAPTOR rollups: weekly + quarterly summary clusters.
3. Theme rollup pass (`patterns.md` update).
4. Assertion-equivalence dedup (cosine ≥ 0.92 + LLM reconciliation).

**Read path:**
1. Always: `persona.md` + `active.md` injected into system prompt.
2. Stage 0 (deterministic): substring/normalized-name match + date filter + scope → ≤50 candidates.
3. Stage 1 (LLM-over-manifest): pick ≤5 cards/summaries.
4. Stage 2 (RAPTOR collapsed-tree): top-20 nodes within 2K-token budget.
5. Stage 3 (recognition-filter, optional): prune irrelevant nodes.
6. Stage 4 (synthesis): theme-extraction prompt → cited themes.

**Two retrieval surfaces, never conflated:**
- `memory` tool (Anthropic wire format) sandboxed to `.agent-memory/` — agent scratchpad only.
- `brain.search` (MCP, Minutes-compatible tool names) — canonical knowledge base.

---

## Tier 3 candidates

The architecture is settled. Remaining open questions are mostly SDD-level. If we want one more research round, candidates from the leads doc:

- **HippoRAG 2 recognition-memory filter as a re-ranker** — read carefully so we know how to graft it onto stage 3 if eval surfaces multi-hop factoid weakness.
- **Plastic Labs Honcho** — the "user-modeling-first" framing might inform persona dossier shape; @courtlandleer is the most rigorous critic of mem0's claims.
- **MIRIX (arXiv 2507.07957)** — six-memory-types taxonomy (Core/Episodic/Semantic/Procedural/Resource/Knowledge Vault). Most modular taxonomy in the literature; useful for v8 schema review.
- **MemoryBank (Ebbinghaus forgetting)** — concrete decay model for age-banding policy.
- **Eval harness design** — how to score against the 24-persona corpus rigorously (this is implementation work, not research).

My recommendation: **stop research, start SDD.** The decisions are firm enough to draft a v8 SDD; further reading risks paralysis. Hold the Tier 3 candidates as targeted reads during SDD review when specific questions surface.

---

# 1. mem0 — deep-dive (Tier 2)

mem0 is the canonical "extract-and-store" memory system: a two-phase LLM pipeline that (1) compresses each conversation turn into atomic facts and (2) reconciles those facts against existing memory via an ADD/UPDATE/DELETE/NONE decision per record. This section quotes the actual prompts shipped on `main`, calibrates the LoCoMo numbers, and decides what Brain Memory v8 should and should not lift.

## The extraction prompt — verbatim

`mem0/configs/prompts.py` ships three extraction variants. The default `FACT_RETRIEVAL_PROMPT` opens:

> "You are a Personal Information Organizer, specialized in accurately storing facts, user memories, and preferences. Your primary role is to extract relevant pieces of information from conversations and organize them into distinct, manageable facts."

It enumerates seven categories (preferences, personal details, plans, activity/service preferences, health/wellness, professional, miscellaneous), and pins the output schema with **six few-shot examples**:

```
Input: Hi, my name is John. I am a software engineer.
Output: {"facts" : ["Name is John", "Is a Software engineer"]}

Input: Yesterday, I had a meeting with John at 3pm. We discussed the new project.
Output: {"facts" : ["Had a meeting with John at 3pm", "Discussed the new project"]}
```

Output contract: `{"facts": [string, ...]}`. Today's date is injected at template time. There are also `USER_MEMORY_EXTRACTION_PROMPT` and `AGENT_MEMORY_EXTRACTION_PROMPT` variants that scope to one role only, plus a much richer V3 `ADDITIVE_EXTRACTION_PROMPT` (~600 lines) that adds Observation Date grounding, `linked_memory_ids`, photo handling, and a 15–80-word target per memory. The V3 prompt is the production-platform direction and is far more useful as a Docknote reference than the OSS default.

## The update prompt — verbatim

`DEFAULT_UPDATE_MEMORY_PROMPT` is the second-phase reconciler:

> "You are a smart memory manager which controls the memory of a system. You can perform four operations: (1) add into the memory, (2) update the memory, (3) delete from the memory, and (4) no change."

It receives the existing memory list (id+text) plus the freshly extracted facts and must return:

```json
{"memory":[{"id":"...","text":"...","event":"ADD|UPDATE|DELETE|NONE","old_memory":"..."}]}
```

Key contracts in the prompt: IDs **must be reused** for UPDATE/DELETE (no new IDs); `old_memory` is required only on UPDATE; semantic equivalence ("Likes cheese pizza" vs "Loves cheese pizza") should resolve to NONE; contradictions trigger DELETE; partial overlap merges to longer fact via UPDATE. `get_update_memory_messages()` inlines `retrieved_old_memory_dict` and the new `response_content` between triple-backticks.

## Per-message vs. per-session extraction

The mem0 paper states the pipeline "initiates upon ingestion of a new message pair" — extraction fires **per user/assistant turn pair**, not per session. Two LLM calls per pair: extraction, update. For a Docknote 1-hour transcript chunked at speaker-turn granularity (~50–80 turns), that is **100–160 LLM calls per meeting**, before retrieval costs.

## Storage layout

```
{ id: uuid, memory: str, hash: md5(memory),
  metadata: dict, score: float (only on search),
  user_id, agent_id, run_id,
  created_at, updated_at }
```

`hash` is content fingerprint for cheap exact-dup short-circuit. **Semantic dedup is delegated entirely to the update-prompt LLM** — no embedding-similarity gate before the LLM call. Vector embedding happens only after the event resolves to ADD/UPDATE.

## Retrieval mechanics

`m.search(query, user_id=, agent_id=, run_id=, filters=)` does single-vector top-k against the embedding store, hard-filtered by scoping IDs. Results return `{id, memory, score, metadata, ...}`. Hybrid (keyword+vector) is opt-in per backend. No reranker in default path.

## Graph mode

Optional Neo4j/Memgraph/Neptune backend. Adds entity/relation extraction (third LLM call) for multi-hop questions. Paper: graph helps **temporal reasoning** (J 58.13 vs 55.51) and **open-domain** (75.71 vs 72.93) but **hurts multi-hop** (47.19 vs 51.15) and roughly **doubles tokens (~14k vs ~7k)** and **latency (p95 2.59s vs 1.44s)**. Does not earn its weight at meeting scale.

## Benchmark numbers — calibrated

mem0's headline LoCoMo scores (LLM-as-judge "J", scale 0–100, GPT-4o-mini agent):

| Category | Mem0 J | Mem0^g J | LangMem | OpenAI Mem |
|---|---|---|---|---|
| Single-hop | 67.13 | 65.71 | 62.23 | 63.79 |
| Multi-hop | 51.15 | 47.19 | 47.92 | 42.92 |
| Temporal | 55.51 | 58.13 | 23.43 | 21.71 |
| Open-domain | 72.93 | 75.71 | 71.12 | 62.29 |

Letta's rebuttal (Dec 2025): plain filesystem agent on GPT-4o-mini hits **74.0%** overall on LoCoMo vs mem0-graph's **68.5%**, and Letta notes "Mem0 did not respond to requests for clarification on how the benchmarking numbers were computed." Plastic Labs separately benchmarks Honcho on LongMemEval where mem0 is around **49% on temporal reasoning**. Net: wins are real but narrower than the paper implies and evaporate against a competently-built file-tool agent.

## Failure modes — from the field

Deshraj Yadav (mem0 co-founder) on X: *"We ran into a bunch of memory quirks with @openclaw — lost context after resets, history getting truncated, rising memory usage, and generally flaky behavior across sessions."* GitHub issue #3009 ("3 out of 5 memory creations lost — Fact extraction inconsistently returns empty results") confirms the extraction prompt silently drops data when the LLM returns `{"facts": []}` on ambiguous input.

## Cost per meeting (Docknote scale)

1-hour transcript, 60 speaker turns, mem0-OSS pipeline:
- **GPT-4o-mini self-hosted**: 60 turns × 2 calls ≈ 120 calls × ~1.5k input/0.3k output tokens ≈ 200k tokens. ~$0.04/meeting + ~$0.01 embedding.
- **Qwen-32B self-hosted**: ~120 sequential calls × ~600ms = ~70s wall-clock per meeting on a single A100; batchable to ~15s.
- **Graph mode** adds third call per turn (≈1.5×). Skip.

The dominant cost isn't dollars — it's **latency to "memory ready"** after a meeting ends.

## Patterns Docknote should borrow

1. **Two-phase shape:** extract → reconcile, with model split (cheap extract / smarter reconcile).
2. **ADD/UPDATE/DELETE/NONE event vocabulary** with strict ID-preservation contract — maps perfectly onto Graphiti bi-temporal `valid_from`/`valid_to`: UPDATE closes old interval, opens new; DELETE just closes.
3. **Scoping triple** `user_id / agent_id / run_id` → for Docknote: `user_id / workspace_id / meeting_id`. Hard-filter every retrieval.
4. **V3 ADDITIVE_EXTRACTION_PROMPT's "Observation Date" discipline** — every relative time reference grounded to meeting start timestamp at extract time.
5. **Few-shot-driven JSON schema** with explicit empty-list fallback (`{"facts":[]}`).

## Patterns Docknote should NOT borrow

1. **Per-turn synchronous extraction.** Batch by topic-segment (~3-min window).
2. **LLM-only dedup.** Add embedding-similarity gate (≥0.92) before invoking update prompt.
3. **Vector store as primary substrate.** Markdown + SQLite + sqlite-vec is strictly better at meeting scale.
4. **Graph mode by default.** Earn it when query class actually needs multi-hop.
5. **Hosted-only premium features.**
6. **Lossy "delete" semantics.** Use Graphiti-style invalidation, never destructive delete.

## Verdict

**Partial yes.** Brain Memory v8's write path should adopt mem0's two-phase shape (extract → ADD/UPDATE/DELETE/NONE reconcile) and its scoping triple, but not its operational defaults. Specifically: keep markdown source-of-truth, gate the update LLM with embedding similarity check, batch extraction at topic-segment rather than turn granularity, encode events as Graphiti bi-temporal interval mutations (never destructive), cap graph-mode to query classes that demonstrably need multi-hop. The extraction-vs-raw question resolves to "extract for the brain, keep raw in the meeting markdown" — mem0's mistake is treating extraction as the source of truth; ours treats it as a derived, regenerable index over the markdown corpus, which sidesteps every failure mode Deshraj himself catalogued.

---

# 2. MemMachine — deep-dive (Tier 2)

## The core claim, stated precisely

From the abstract (arXiv 2604.04853): MemMachine is "a ground-truth-preserving memory system" that "stores complete conversational episodes rather than relying solely on LLM-based extraction." The headline number — "approximately 80 percent fewer input tokens compared to Mem0 under matched conditions" — comes from a single benchmark: **LoCoMo with gpt-4.1-mini, 1,540 questions in memory mode**. MemMachine consumed ~4.2M input tokens vs. Mem0's ~19.2M (~78% reduction). The comparison appears to be **cumulative input tokens across the full benchmark run**, aggregating both write-time and read-time. The paper does not separately decompose these.

## Storage shape

MemMachine ingests every conversational turn as an **Episode** (producer, timestamp, session ID, custom metadata). At write time it does **not** call an LLM to extract facts. Instead:

1. NLTK Punkt tokenizes each episode into sentences.
2. Each sentence is embedded and indexed in **Postgres + pgvector**.
3. Relational structure stored in **Neo4j**.
4. A short-term memory tier holds recent episodes plus an LLM-generated rolling summary; a profile memory tier (SQL) holds distilled user facts (the only place where extraction-style LLM calls happen).

Episode boundaries are **not Bayesian-surprise–detected** — natural conversational turn unit. Sentence-level indexing gives finer-grained retrieval than message-level. No published disk-footprint number.

## The projection mechanism

Query time is where MemMachine spends compute. Key trick is **contextualized retrieval / "episode clusters"**:

1. Vector search finds nucleus matches.
2. Each match is **expanded with one preceding + two following episodes** so multi-turn evidence reassembles.
3. Cross-encoder reranks clusters.
4. Top-k clusters go to the LLM; k=30 is the ablation sweet spot (k=50 degrades to 0.890 — explicit "lost in the middle" failure).

A **Retrieval Agent** layer routes harder queries through parallel decomposition or iterative chain-of-query.

## Why fewer tokens — is the 80% real?

Partly real, partly framing:

- **Real**: mem0 burns input tokens on every message via per-turn extraction prompts. On a 1,540-question LoCoMo run, those extraction prompts dominate. MemMachine skips them entirely.
- **Framing risk**: comparison counts **input** tokens. Read-time, MemMachine packs larger episode clusters (raw sentences + neighbors) into the prompt vs. mem0's terse extracted facts. So query-prompt size is *higher* per-question but the per-message extraction tax is *zero*. Paper does not isolate write vs. read tokens.

Bottom line: 80% is credible on chat-length corpora but is largely a **write-time** saving. As corpus grows, read-time term grows too (more candidates, more rerank work, larger clusters), and the gap narrows.

## Benchmark numbers — calibrated

- **LoCoMo (gpt-4.1-mini)**: 0.9169 overall.
- **LongMemEval-S**: 93.0% accuracy.
- **HotpotQA-hard**: 93.2%, **WikiMultiHop**: 92.6% (Retrieval Agent under noise).
- **Token figure** is LoCoMo-only.

LoCoMo conversations are **chat-length**. No benchmark on long-form transcripts.

## Trade-offs

Acknowledged: temporal reasoning lags Memobase (0.735 vs 0.851), multi-session reasoning caps at 0.872, k>30 degrades. **Not seriously addressed:** query-time latency at scale, disk footprint over months, privacy of persisted raw transcripts, what happens when an "episode" is 10K words.

## Architectural fit for meeting transcripts

This is where the chat-shaped assumption breaks. Meeting episode = ~7-10K words, two orders of magnitude larger than a chat turn:

1. **Sentence-level pgvector index explodes**. 1,000 meetings × ~600 sentences = 600K vectors. Tractable, but ~100× a chat corpus.
2. **The "1 preceding + 2 following episodes" expansion rule is wrong-shaped.** If episode = meeting, expansion grabs entire neighboring meetings. If episode = sentence-within-meeting, you lose cross-meeting recall.
3. **"Find what Alex said about pricing in March"** under MemMachine → vector search across 600K sentences, expand to clusters, cross-encoder rerank, pack into prompt. Retrieval shape is fine; missing piece is **typed structure** (speaker=Alex, topic=pricing, time-window=March). MemMachine has no first-class speaker/topic index.

So MemMachine **scales sub-linearly on storage** but **fails to exploit meeting structure**.

## Maturity assessment

[GitHub MemMachine/MemMachine](https://github.com/MemMachine/MemMachine): ~3.5K stars, 173 forks, 895 commits, v0.3.7 released May 2026. Apache 2.0. Active. Stack: Python SDK, Docker/Helm, Postgres+pgvector + Neo4j, MCP server. **Production caveat**: "requires a running MemMachine Server" — service, not library. Citations beyond authors are thin (paper is April 2026).

## Patterns Docknote should borrow

- **Don't extract at write time.** Persist the raw transcript verbatim.
- **Sentence-level embeddings with provenance back to parent episode.**
- **Contextualized retrieval / episode clusters**: when sentence hits, return neighboring sentences (or surrounding minutes of transcript). Maps to "show me the 60-second window around the hit."
- **Cross-encoder rerank** before context packing.
- **Tiered memory**: working / episodic (raw + vector+graph) / profile (people, projects, preferences).
- Tune **retrieval depth (k)** explicitly; k≈30 is published sweet spot.

## Patterns Docknote should NOT borrow

- **Treating a meeting as single Episode and using "1 prev + 2 next" expansion.** Designed for chat turns. For meetings, episodes must be sub-meeting (sentence or 30-second window) with parent meeting as containing scope.
- **Skipping typed extraction entirely.** Meeting recall queries are **typed**. MemMachine's pure-raw stance leaves these to ad-hoc vector search.
- **Neo4j as hard dependency.**
- **The 80% token claim as planning input.** Chat-length, write-dominated number.

## Verdict on extract-vs-raw axis

**Brain Memory v8 should be hybrid, with raw as source of truth.** MemMachine evidence settles the write-path question against pure-extract: mem0-style per-message extraction is the wrong default for meeting transcripts because (a) extraction error compounds across hundreds of meetings, (b) you can never recover original phrasing once distilled, (c) dominant token cost (extraction × every utterance) is what makes mem0 expensive at meeting scale. Keep raw transcript verbatim, indexed at sentence granularity with cross-encoder rerank — that is the MemMachine pattern and it is the right write path. **But** layer a thin mem0-style typed-extraction pass at meeting close, producing `decisions`, `action_items`, `people`, `themes`, each with pointer back to a transcript span. This costs one bounded LLM call per meeting (not per utterance), gives the typed query surfaces meetings actually need, and keeps the raw store as ground truth so any extraction error is recoverable.

---

# 3. HippoRAG 2 + RAPTOR — deep-dive (Tier 2)

## HippoRAG 2 mechanism

HippoRAG 2 is the ICML 2025 successor to HippoRAG (NeurIPS 2024). Frames retrieval as three memory functions — **factual recall**, **sense-making**, **associative**. Pipeline:

1. **Offline indexing.** LLM runs OpenIE over each passage emitting `(subject, predicate, object)` triples. Phrases become *concept nodes*; passages become *passage nodes*; synonym edges connect near-duplicate phrases (cosine over NV-Embed-v2). Result: heterogeneous KG with both phrase nodes and passage nodes.
2. **Online query.** Query encoded; system does *query-to-triple* link (v2's key change) to pick seed triples; their endpoints become **personalization vector** for Personalized PageRank (PPR). PPR diffuses probability mass across the KG. Top-ranked passage nodes returned to reader LLM.

## HippoRAG 2's improvements over v1

- **Query-to-triple linking** replaces v1's NER-to-node linking. Recall@5 on MuSiQue +12.5%.
- **Passage nodes added to KG** so PPR walks see dense passage signal alongside sparse triple structure.
- **Recognition-memory filter**: LLM call prunes irrelevant retrieved triples online (~7% miss rate, big precision win).
- **Online LLM loop** unifies extraction-side filtering with answer generation.

Headline numbers (Llama-3.3-70B reader):
- **MuSiQue F1**: 44.8 → 51.9 (Recall@5 69.7 → 74.7)
- **2WikiMultihopQA Recall@5**: 76.5 → 90.4
- **Mean +7 F1** over NV-Embed-v2 across associative tasks.

Indexing cost on MuSiQue: **~9M LLM tokens** vs ~115M for GraphRAG/LightRAG at parity corpus.

## RAPTOR mechanism

Stanford, ICLR 2024. Purely tree-shaped:

1. **Chunk** at ~100 tokens (sentence-boundary-respecting), embed with SBERT (`multi-qa-mpnet-base-cos-v1`). Leaves.
2. **Soft cluster** at each level: UMAP for dim-reduction, GMM for soft membership, BIC to pick k. Soft means a chunk can land in >1 cluster.
3. **Summarize** each cluster with `gpt-3.5-turbo`. Output ~131 tokens average; compression ratio 0.28 (72% shrink).
4. **Recurse** until corpus collapses into a small handful of root summaries. Empirically 3–4 levels for book-scale corpora.

At query time:
- **Tree traversal**: top-k at each level, descend.
- **Collapsed tree**: flatten *all* nodes (leaves + every summary at every level), retrieve top-k by cosine until 2,000-token budget (~top-20 nodes). **Collapsed tree wins** — lets retriever pick the *right level of abstraction* per query.

For sense-making queries, retrieved nodes skew toward summaries (18.5–57% non-leaf depending on dataset).

## Storage shape

- **HippoRAG 2**: dense vector index (one per phrase + one per passage) **+** graph of `(node_id, node_id, edge_type)` rows. Reference repo uses in-memory graph (NV-Embed embeddings + Python graph; PPR via `igraph`/`scipy.sparse`). Persistence is files on disk, not graph DB.
- **RAPTOR**: single dense vector index over leaves *and* every summary node. Each node carries `{id, parent_id, level, text, embedding}`. For 1,000 meetings × 10K words ≈ 10M words ≈ 100K leaves of 100 tokens. With 0.28 compression: 100K + 28K + 8K + 2K ≈ **140K nodes**, ~30M tokens of stored text plus ~140K × 768-dim float32 = ~430MB embeddings. Trivially fits SQLite + flat-file ANN index.

## Build-time cost (1,000 meetings × 10K words)

- **RAPTOR**: only LLM calls are summaries. ~5K clusters total. At ~1.5K input + 130 output tokens per call on GPT-4o-mini → **~$15-25 one-time**, plus embedding cost (~$1).
- **HippoRAG 2**: OpenIE on every passage (~100K passages → ~100K LLM calls just for extraction). Even on GPT-4o-mini at ~2K in / 500 out per passage: **~$300-500 one-time** indexing.

RAPTOR is roughly **20× cheaper to index** than HippoRAG 2 at this corpus shape, **>1000× cheaper** than GraphRAG.

## Query-time cost and latency

- **RAPTOR**: one ANN query over flattened tree → top-k → reader. Same latency as vanilla RAG. No per-query LLM call before retrieval.
- **HippoRAG 2**: ANN over triples → recognition-filter LLM call (extra ~300ms, $0.0001) → PPR over small per-query graph. Paper reports HippoRAG-family is 10–30× cheaper *and* 6–13× faster than IRCoT-style iterative retrieval.

## Benchmarks — calibrated

| System | Benchmark | Reader | Score |
|---|---|---|---|
| RAPTOR | QuALITY | GPT-4 | **82.6%** acc (+20pp over prior SOTA 62.3%) |
| RAPTOR | QASPER F1 | GPT-4 | 55.7 (vs DPR 51.2) |
| RAPTOR | NarrativeQA | UnifiedQA | METEOR 19.1 / ROUGE-L 30.8 |
| HippoRAG 2 | MuSiQue F1 | Llama-3.3-70B | **51.9** (v1: 44.8) |
| HippoRAG 2 | 2Wiki Recall@5 | — | **90.4** (v1: 76.5) |
| HippoRAG 2 | avg associative | NV-Embed-v2 reader | **+7 F1** over baseline |

**HippoRAG 2 wins multi-hop factual** (MuSiQue, 2Wiki); **RAPTOR wins sense-making over long narrative documents** (QuALITY, QASPER). For Docknote, "recurring concerns about pricing" is sense-making — squarely RAPTOR's sweet spot.

## Architectural fit for synthesis pass

Acceptance test wants: K dossiers → 3-4 themes → cite source meetings. Neither system *is* a synthesis layer — both are retrievers. But:

- **RAPTOR** pre-computes summaries-of-summaries at build time. Mid-tree nodes already *are* mini-themes. v8 synthesis prompt over top-K retrieved RAPTOR nodes naturally inherits multi-meeting context.
- **HippoRAG 2** returns passages, not themes. Improves passage *recall*, but theme extraction is still entirely separate prompt downstream. KG buys nothing for "give me 3 themes" output shape.

So RAPTOR fits as **retrieval+pre-synthesis substrate**; HippoRAG 2 fits only as **retrieval booster** for multi-hop factoid queries.

## No-graph-DB constraint

HippoRAG 2's reference impl does **not** require graph DB — uses in-memory graph and persists as flat files. PPR is sparse matrix-vector iteration; runs over CSR matrices loaded from SQLite. So constraint isn't strictly a blocker. **However**, you'd still pay LLM-extraction cost (OpenIE per passage), disk cost of phrase-level KG, engineering cost of custom PPR runtime. RAPTOR is a tree of `(parent_id, child_id)` rows in SQLite plus vector index — strictly simpler.

## Combine vs choose

Practical two-stage:
1. **Build-time** (meeting close): RAPTOR-style hierarchical pre-summarization. Each meeting becomes its own little tree. Weekly/quarterly **roll-up** nodes summarize across meetings.
2. **Query-time**: stage-1 retrieval = collapsed-tree top-K over global RAPTOR forest. Stage-2 = dedicated *theme-extraction* prompt over top-K nodes.

HippoRAG 2's PPR can be added later as *re-ranker* for known-multi-hop factoid queries if eval shows RAPTOR underperforms there. Don't build the KG on day one.

## Patterns Docknote should borrow

- **RAPTOR's collapsed-tree retrieval** — flatten every level into one ANN index; cosine picks right granularity. (Avoid inferior tree-traversal mode.)
- **RAPTOR's UMAP+GMM soft clustering** — meeting topic legitimately overlaps with two parents; soft membership preserves that.
- **RAPTOR's BIC-driven k** — auto-fit cluster count per level.
- **HippoRAG 2's recognition-memory filter** — borrow as *re-rank step* after stage-1 retrieval.
- **HippoRAG 2's query-to-triple framing** as inspiration for *query-to-theme-prototype* link: embed query against pre-computed theme vectors at week/quarter rollup level.
- **Explicit roll-up cadence** — weekly + quarterly summaries cheap, queryable, double as user-facing recap surfaces.

## Patterns Docknote should NOT borrow

- **HippoRAG 2's full OpenIE-on-every-passage step.** Net negative.
- **Phrase-level KG with synonym edges.**
- **RAPTOR's `gpt-3.5-turbo` summarizer** — use GPT-4o-mini or small local model.
- **RAPTOR's tree-traversal query mode.**
- **GraphRAG-style global community summaries.**
- **Re-summarizing entire forest on every new meeting.** Append-only: new meeting → its own subtree → attach to existing weekly cluster (re-cluster only at rollup level on schedule).

## Verdict — synthesis-pass architecture for v8

**Adopt RAPTOR-shaped hierarchical pre-summarization at meeting close, with weekly and quarterly rollup levels, queried via collapsed-tree top-K, followed by a dedicated theme-extraction prompt over the retrieved nodes. Skip HippoRAG 2 for v8.**

Concrete shape: `meeting → chunks(100tok) → UMAP+GMM → GPT-4o-mini summaries → 3-level tree per meeting → weekly rollup cluster → quarterly rollup`; `query → embed → ANN over flattened forest, top-20 within 2K-token budget → recognition-filter LLM call → theme-extraction prompt → {themes: [{name, citations:[meeting_ids]}]}`.

---

# 4. Letta sleep-time agents — deep-dive (Tier 2)

## The sleep-time system prompt — verbatim

`letta/prompts/system_prompts/sleeptime_v2.py` assigns persona "Limnal Corporation," memory-block mental model, "be selective but high recall" directive:

> "You are Letta-Sleeptime-Memory, the latest version of Limnal Corporation's memory management system, developed in 2025. You run in the background, organizing and maintaining the memories of an agent assistant who chats with the user."
>
> "Use your precise tools to make narrow edits, as well as broad tools to make larger comprehensive edits. … When writing to memory blocks, make sure to be precise when referencing dates and times (for example, do not write 'today' or 'recently', instead write specific dates and times, because 'today' and 'recently' are relative, and the memory is persisted indefinitely)."
>
> "If there are no meaningful updates to make to the memory, you call the finish tool directly. Not every observation warrants a memory edit, be selective in your memory editing, but also aim to have high recall."

`sleeptime_doc_ingest.py` runs when documents are uploaded; distinguishes read-only persona/instructions sub-blocks from read-write data-source blocks. `voice_sleeptime.py` enforces two-phase tool sequence: `store_memories` → `rethink_user_memory` → `finish_rethinking_memory`.

## `rethink_memory()` — verbatim

From `letta/functions/function_sets/base.py`:

```python
def rethink_memory(agent_state: "AgentState", new_memory: str, target_block_label: str) -> None:
    """
    Rewrite memory block for the main agent, new_memory should contain all current
    information from the block that is not outdated or inconsistent, integrating any
    new information, resulting in a new memory block that is organized, readable, and
    comprehensive.
    """
    if agent_state.memory.get_block(target_block_label) is None:
        new_block = Block(label=target_block_label, value=new_memory)
        agent_state.memory.set_block(new_block)
    agent_state.memory.update_block_value(label=target_block_label, value=new_memory)
    return None
```

Contract is brutally simple: full-block overwrite, no diff, no version bump, no return value. v2 set adds `memory_replace`, `memory_insert`, `memory_apply_patch`, `memory_rethink`.

## `finish_rethinking_memory()` semantics

```python
def finish_rethinking_memory(agent_state: "AgentState") -> None:
    """This function is called when the agent is done rethinking the memory."""
    return None
```

No-op marker. In `voice_sleeptime_agent.py` enforced as `TerminalToolRule`. **No atomic commit, no diff, no audit trail, no version history.** Each `rethink_memory` call is immediate `update_block_value` mutation; if agent makes five rewrites before calling `finish_rethinking_memory`, primary readers between calls see partial intermediate states.

## Trigger policy

In `letta/groups/sleeptime_multi_agent_v3.py`, sleep-time fires **strictly every-N-steps** after foreground agent's `step()` or `stream()` completes:

```python
if self.group.sleeptime_agent_frequency is None or (
    turns_counter % self.group.sleeptime_agent_frequency == 0
):
    ... await self._issue_background_task(...)
```

`sleeptime_agent_frequency` defaults to **5**. **Not idle-triggered.** Dispatch uses `safe_create_task(...)` — fire-and-forget asyncio task. Foreground response returns to user immediately. Transcript handed to sleep-time is slice between `last_processed_message_id` and latest response.

## Concurrency model

**No locking.** `update_block_value` is direct mutation on shared SQL row; `Memory` blocks shared across agents in group via `agent_state.memory`. If primary calls `core_memory_replace` while sleep-time is mid-`rethink_memory`, **last write wins silently.** No optimistic locking, no version column read in rewrite path, no merge conflict surfaced.

## Model used

Per-agent override via `llm_config`. Letta's pattern: fast model for primary, stronger model for sleep-time. No hardcoded default.

## What it rewrites

Strictly **core memory blocks** (in-context, size-bounded text blocks like `human`, `persona`, custom labels). Blog: "transforming raw context into learned context" — flat string in `human` block changes from chat log into structured profile. Archival memory passages **not** rewritten by sleep-time today.

## Cost / wall-clock

Letta hasn't published numbers. From code: at frequency=5, runs once per 5 foreground turns, ingests 5-turn transcript plus current block contents, emits 1–N tool calls. With Sonnet-class models, expect 3–10s wall-clock and 2k–8k tokens per pass.

## `consolidate_archival_memory()` — issue #3116

Open feature request (filed 2025, last comment 2026-05-03). Proposes embedding-similarity dedup + LLM reconciliation merge for archival passages. Status: **not implemented.** Maintainer Charles Packer: prototype as user-supplied custom tools first. Strongest external comment (borjamoskv) argues for two-pass dedup (cosine ≥ 0.92 then LLM "assertion equivalence" check) and **provenance traces from canonical assertion back to episodic sources** — point we should adopt.

## Failure modes

In `_participant_agent_step`: just `print(f"Sleeptime agent processing failed: {e!s}"); raise e`. Run marked failed via `RunUpdate(status=...)`, but no rollback of partial block writes, no retry, no quarantine of bad rewrites. Because rewrites are not versioned, hallucinated rewrite is permanent unless user manually corrects it.

## Adapt to Brain Memory v8

Concrete shape for nightly dossier consolidation:

- **Trigger**: cron-style, not every-N. Two firings: (a) post-meeting close (per-meeting incremental update of touched dossier cards), (b) nightly 03:00 local (theme rollups, age-banding, cross-meeting reconciliation). No turn counter.
- **Inputs**: today's transcript chunks, today's extracted entity facts, current dossier card frontmatter, prior theme set.
- **Outputs**: structured frontmatter diff per card (field-level: `role_line.before`, `role_line.after`, `sources: [meeting_id, span]`); new/updated theme entries; age-band transitions.
- **Audit trail**: every rewrite writes `revisions/<card_id>/<iso8601>.json` with `{before, after, sources, model, prompt_hash}`. UI hovers show diff and source spans. **Single biggest deviation from Letta — we keep full revision log; Letta does not.**
- **Concurrency**: meeting-close writes lease per-card row; nightly batch acquires advisory lock, refuses to run if any meeting is mid-write.

Prompt skeleton (Brain Memory v8 sleep-time):

```
You are Brain-Memory-Consolidator, running nightly at 03:00 local on a Mac
meeting-recording app. Your job is to rewrite person dossier cards from
today's meetings while preserving an auditable diff.

You receive:
- card_current: existing frontmatter (YAML, fields role_line/day_headline/
  stat_line/differentiation_bar, plus body sections).
- today_facts: structured extractions from today's transcripts touching this
  person, each with {meeting_id, ts_span, claim, confidence}.
- prior_theme_set: cross-card themes the person belongs to.

Rules:
1. Emit one tool call per changed field via update_card_field(card_id, field,
   new_value, sources=[{meeting_id, ts_span}]). Never rewrite unchanged fields.
2. Use absolute dates ("2026-05-04"), never "today" or "recently".
3. If a fact contradicts the current card, prefer the newer assertion only if
   confidence >= 0.7; otherwise emit reconcile_flag(field, reason) for review.
4. Age-band: any claim older than 90 days with no corroborating fact today
   gets demoted to body section "Background" via demote_field(field).
5. When done, call finish_consolidation(card_id). All field writes between
   start and finish are committed atomically as one revision.
```

## What NOT to copy from Letta

- **Every-N-steps trigger**: wrong shape — meetings are episodic, not turn-counted.
- **No audit trail**: unacceptable for user-facing dossier UI.
- **Flat-string `rethink_memory` overwrite**: v8 needs field-level rewrites against typed YAML frontmatter.
- **No atomic commit**: must batch all field writes between `start_consolidation` and `finish_consolidation` into one transaction.
- **No retry / no rollback**: keep `revisions/` ledger so bad rewrite is reversible.
- **No archival dedup**: implement assertion-equivalence dedup ourselves.

## Verdict — shape of v8's nightly consolidation

Run two consolidators on different cadences: (a) per-card incremental rewriter on meeting-close (touches only dossier cards mentioned in that meeting), (b) nightly cross-card reconciler at 03:00 (theme rollups, age-banding, dedup). Both use Letta-style "rewrite as tool calls" loop — but with **field-level tools** (`update_card_field`, `demote_field`, `reconcile_flag`) instead of blob `rethink_memory`, **mandatory `sources=[{meeting_id, ts_span}]`** on every write, **transactional `start/finish_consolidation` framing** so partial states never escape, and **persistent `revisions/<card>/<ts>.json` ledger** that powers UI's "why does it say this?" hover and one-click rollback. Strong model (Opus/GPT-5-class) for nightly pass, fast model for per-meeting incremental. Failure handling: rewrite that fails JSON-schema validation or whose `sources` list is empty is rejected before commit; prior revision stays canonical and error toast surfaces in UI next morning.

---

# 5. Cline / cursor memory bank — deep-dive (Tier 2)

## The canonical file set

Cline's Memory Bank defines six core markdown files in strict hierarchy:

- **`projectbrief.md`** — "Foundation document that shapes all other files" and "source of truth for project scope". Written once, edited rarely.
- **`productContext.md`** — "Why this project exists, problems it solves, how it should work, user experience goals."
- **`activeContext.md`** — "Current work focus, recent changes, next steps, active decisions." Updated nearly every session.
- **`systemPatterns.md`** — "System architecture, key technical decisions, design patterns, component relationships."
- **`techContext.md`** — "Technologies used, development setup, technical constraints, dependencies."
- **`progress.md`** — "What works, what's left to build, current status, known issues." Updated alongside `activeContext.md`.

Hierarchy: `projectbrief.md → {productContext, systemPatterns, techContext} → activeContext.md → progress.md`. Update cadence: "after significant milestones or direction changes," "every few sessions" for active work, full sweep when user types `update memory bank`.

## The custom-instructions prompt — verbatim

> "I am Cline, an expert software engineer with a unique characteristic: my memory resets completely between sessions. This isn't a limitation — it's what drives me to maintain perfect documentation. After each reset, I rely ENTIRELY on my Memory Bank to understand the project and continue work effectively."
>
> **"I MUST read ALL memory bank files at the start of EVERY task — this is not optional."**
>
> "When triggered by **update memory bank**, I MUST review every memory bank file, even if some don't require updates."
>
> "The Memory Bank is my only link to previous work. It must be maintained with precision and clarity, as my effectiveness depends entirely on its accuracy."

That "MUST read ALL … at start of EVERY task" line is the entire pattern. Without it, agents skip the load step on short prompts and the files become decorative.

## Mode-driven loading (cursor-memory-bank)

vanzan01's variant splits workflow into six commands, each loading different rule subset:

- **`/van`** — initialize, detect platform, classify task complexity (Levels 1–4). Loads only `main.mdc` + `memory-bank-paths.mdc` + `van-mode-map.mdc`.
- **`/plan`** — implementation plan. Loads `workflow-level{N}.mdc` matched to complexity.
- **`/creative`** — design exploration (Level 3–4 only); writes `creative/creative-[feature].md`.
- **`/build`** — execute the plan.
- **`/reflect`** — review and write `reflection/reflection-[task_id].md`.
- **`/archive`** — generate `archive/archive-[task_id].md` and update Memory Bank.

README cites "**~70% reduction in initial token usage** compared to loading all rules at once." Level-1 tasks skip `/plan` and `/creative` entirely; Level-3–4 traverse all six.

## Update mechanics

Three triggers: (a) automatic — when "discovering new project patterns" or "after implementing significant changes"; (b) explicit — user types `update memory bank` (full audit); (c) end-of-task — `/archive` in cursor variant. The user-typed phrase is the contractual command — what guarantees a full sweep rather than partial.

## User-edit vs. agent-edit boundary

**Boundary is non-existent in the spec** — both edit the same plain markdown. No `<!-- human-only -->` regions, no frontmatter locks. In practice users rely on git: edit a file, commit, then agent diffs against HEAD before rewriting. When users skip git, agent silently overwrites human prose. **This is the pattern's biggest weakness.**

## Audit trail

No in-tool diff UI. Audit = `git diff` on `cline_docs/` (or `memory-bank/`). Implicit assumption that project is already a git repo. For non-git contexts (and meeting apps), this entirely punts the audit problem.

## Failure modes from the field

- **Agent claims update, doesn't write.** [cline/cline issue #1911](https://github.com/cline/cline/issues/1911): "No files are being edited or updated, but CLine is saying it has done so." Tool-call dispatch failure dressed up as success.
- **Unbounded growth.** `activeContext.md` and `progress.md` accrete every session; nothing prunes them.
- **Outdated as truth.** Once `systemPatterns.md` is wrong, agent cites it as canonical and propagates errors.
- **Mode confusion.** Cursor variant gets blamed for "wrong mode at wrong time."

## Why it works (and where it breaks)

Works because: (a) plain markdown is grep-able, diff-able, human-readable; (b) strict "read at start of EVERY task" prompt is short enough to actually get followed; (c) file hierarchy maps to real engineer's mental model. Breaks at scale: single project, single workspace, every-task-touches-everything model. Tokens balloon, drift accelerates, lack of edit-boundary means careless agent run can erase careful human edit.

## Brain Memory v8 — the dossier-card analog

Borrowing file taxonomy but pivoting from "one project" to "many entities":

- **`person/<name>.md`** — analog of `productContext.md`, one per person.
- **`project/<slug>.md`** — analog of `projectbrief.md`, one per project.
- **`active.md`** — analog of `activeContext.md`. Last 7–14 days of meetings. **Always loaded.**
- **`patterns.md`** — analog of `systemPatterns.md`. Recurring themes.
- **`commitments.md`** — analog of `progress.md`. Open action items, decisions.
- **`persona.md`** — user's own profile. **Always loaded.**

**Auto-load rule**: `persona.md` + `active.md` + dossier cards for entities mentioned in current chat or just-ended meeting. Everything else lazy-loads on retrieval. Cursor-memory-bank insight applied to entity scope rather than complexity level.

## User-edit UX for v8

- **Edit affordance**: each dossier card field (role_line, current_focus, communication_style) is click-to-edit text field in dossier panel — not "open file in markdown editor."
- **Lock-after-edit**: any field a human edits gets `human_edited: true` flag and is excluded from consolidation pass's auto-rewrite. Auto-pass may *append* to separate `proposed_updates` array but cannot mutate locked field.
- **Provenance badges**: every field shows `auto-extracted • 2 meetings ago` or `you edited • Apr 12`. Hover shows source meeting(s).
- **Revert**: every auto-rewrite stores prior value in per-field history (cap at 5). "Revert Alex's role_line" is single click; full version history is side drawer.
- **Diff UI**: sentence-level highlight with strikethrough on removed text and underline on added — not paragraph swap. Frontmatter field changes render as two-line side-by-side. Multi-field updates from one consolidation pass batched into single review card user can accept-all / reject-all / cherry-pick.

## What to copy verbatim

- **The "read at start of EVERY task" discipline** — for v8 becomes "load `persona.md` + `active.md` + mentioned-entity cards on every assistant turn, no exception."
- **Markdown-on-disk + frontmatter.**
- **Explicit trigger phrase** — `update brain` to force full consolidation sweep.
- **Hierarchy with clear roots** — `persona` and `project` are foundations; `active` and `commitments` are derived state.

## What to differ on

- **Scope**: cursor/Cline assume one project where every task touches everything. Meetings touch ~3 entities out of hundreds — v8 must do **selective load**, not full load. Cline's "I MUST read ALL" becomes "I MUST load all *relevant* cards" with retrieval doing the work.
- **Edit boundary**: cursor/Cline have no edit-boundary and bleed into git for audit. v8 is consumer software with no git escape hatch — human-edit lock and per-field provenance must be first-class.
- **Audit trail**: git-diff doesn't exist for users. v8 needs explicit "what changed" inbox showing every auto-update from last consolidation pass, accept/reject per field.
- **Pruning**: Cline ignores growth; v8 must age out `active.md` entries (>14 days move to per-meeting archive) and cap dossier history.

## Verdict — dossier-card edit UX for v8

Adopt Cline's file taxonomy and "read on every turn" discipline, but invert the edit model. **File naming**: `persona.md`, `active.md`, `patterns.md`, `commitments.md`, `person/<slug>.md`, `project/<slug>.md` — frontmatter for structured fields, body for free prose. **Auto-load rule**: `persona.md` + `active.md` always; entity cards loaded by retrieval against current turn's named entities (cap ~6). **Edit-vs-auto boundary**: every field carries `source: auto|human` + `last_edited`; human-touched fields are immutable to consolidation pass, which must propose changes into separate `proposed_updates` queue rather than overwrite. **Audit trail**: "Brain inbox" UI listing every auto-extracted change since user last reviewed, with sentence-level diff, per-field accept/reject, one-click revert backed by 5-deep field history. **Conflict policy**: human edits always win; on conflict, consolidation pass appends to field's history with `superseded_by_human` marker rather than discarding observation, so model can still cite meeting evidence on later turns without rewriting user's prose.

---

## Source index

**mem0**: [paper](https://arxiv.org/html/2504.19413v1) · [prompts.py](https://github.com/mem0ai/mem0/blob/main/mem0/configs/prompts.py) · [docs](https://docs.mem0.ai/core-concepts/memory-operations) · [Letta benchmark rebuttal](https://www.letta.com/blog/benchmarking-ai-agent-memory) · [Plastic Labs honcho-benchmarks](https://github.com/plastic-labs/honcho-benchmarks) · [Deshraj on memory quirks](https://x.com/deshrajdry/status/2020046706684096854) · [issue #3009](https://github.com/mem0ai/mem0/issues/3009) · [AWS production blog](https://aws.amazon.com/blogs/database/build-persistent-memory-for-agentic-ai-applications-with-mem0-open-source-amazon-elasticache-for-valkey-and-amazon-neptune-analytics/)

**MemMachine**: [arXiv](https://arxiv.org/abs/2604.04853) · [HTML](https://arxiv.org/html/2604.04853) · [GitHub](https://github.com/MemMachine/MemMachine) · [memmachine.ai](https://memmachine.ai/) · [alphaXiv overview](https://www.alphaxiv.org/overview/2604.04853v1)

**HippoRAG 2 + RAPTOR**: [HippoRAG 2 paper](https://arxiv.org/abs/2502.14802) · [HippoRAG 1](https://arxiv.org/abs/2405.14831) · [RAPTOR paper](https://arxiv.org/abs/2401.18059) · [OSU-NLP-Group/HippoRAG](https://github.com/OSU-NLP-Group/HippoRAG) · [parthsarthi03/raptor](https://github.com/parthsarthi03/raptor) · [Superlinked VectorHub](https://superlinked.com/vectorhub/articles/improve-rag-with-raptor) · [GraphRAG](https://arxiv.org/abs/2404.16130)

**Letta sleep-time**: [Sleep-time compute blog](https://www.letta.com/blog/sleep-time-compute) · [docs](https://docs.letta.com/guides/agents/architectures/sleeptime) · [v1 agent blog](https://www.letta.com/blog/letta-v1-agent) · [sleeptime_v2.py](https://github.com/letta-ai/letta/blob/main/letta/prompts/system_prompts/sleeptime_v2.py) · [base.py](https://github.com/letta-ai/letta/blob/main/letta/functions/function_sets/base.py) · [sleeptime_multi_agent_v3.py](https://github.com/letta-ai/letta/blob/main/letta/groups/sleeptime_multi_agent_v3.py) · [issue #3116](https://github.com/letta-ai/letta/issues/3116)

**Cline / cursor memory bank**: [Cline docs](https://docs.cline.bot/prompting/cline-memory-bank) · [cline/prompts memory-bank.md](https://github.com/cline/prompts/blob/main/.clinerules/memory-bank.md) · [nickbaumann98/cline_docs](https://github.com/nickbaumann98/cline_docs/blob/main/prompting/custom%20instructions%20library/cline-memory-bank.md) · [issue #1911](https://github.com/cline/cline/issues/1911) · [vanzan01/cursor-memory-bank](https://github.com/vanzan01/cursor-memory-bank) · [alioshr/memory-bank-mcp](https://github.com/alioshr/memory-bank-mcp) · [thedotmack/claude-mem](https://github.com/thedotmack/claude-mem) · [Cline blog](https://cline.bot/blog/memory-bank-how-to-make-cline-an-ai-agent-that-never-forgets) · [Cursor forum](https://forum.cursor.com/t/cursor-memory-bank/49776)
