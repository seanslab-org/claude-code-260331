# Brain Memory v8 — lessons from Claude Code's memory subsystem

**Date:** 2026-05-04
**Author:** Lead (research note)
**Source artifacts:**
- Leaked Claude Code source at `~/seanslab/Research/cc-learn/src/memdir/` (memdir.ts, memoryTypes.ts, memoryScan.ts, findRelevantMemories.ts, memoryAge.ts, paths.ts, teamMemPaths.ts)
- Claude Code extraction pipeline at `~/seanslab/Research/cc-learn/src/services/extractMemories/`
- CLAUDE.md loader at `~/seanslab/Research/cc-learn/src/utils/claudemd.ts`
- Docknote target: REQ-070 (Brain Memory v8), `tests/eval-brain-memory/persona-corpus.yaml`, mockups under `ux-mockups/brain-memory/`

**Status:** Reference note for Phase-3 dispatch. Not binding; informs SDD work when v8 unfreezes.

---

## 1. Why this comparison

Brain Memory v8 is the "second top-level scene" — a queryable cross-meeting brain organized by **name / date / concept** pivots, with persona-shaped dossier cards and theme-level synthesis. Claude Code ships a working file-based memory system that has been validated at user-scale across millions of sessions. Different product, adjacent shape. Worth mining for transferable patterns and for the deliberate non-choices Anthropic made.

This note extracts the lessons; it does **not** prescribe v8 architecture. Architecture work resumes at Phase-3 SDD.

---

## 2. Patterns worth borrowing

### 2.1 Index + content split

`MEMORY.md` is a pure index — one line per entry, capped at 200 lines / 25KB. Each topic lives in its own `.md` file with YAML frontmatter (`name`, `description`, `type`, body). The index is always-in-context; details are loaded on demand.

→ Brain Memory analog: a per-user `entities.md` (people + recurring concepts) and a `timeline.md` (date pivots), with each person/concept as its own dossier file. Pivot card content (`role_line`, `day_headline`, `stat_line`) maps cleanly onto frontmatter.

Reference: `src/memdir/memdir.ts:35-102` (cap constants + truncation), `src/memdir/memdir.ts:199-265` (instruction template), `src/memdir/memoryTypes.ts:261-271` (frontmatter shape).

### 2.2 Two-stage retrieval, no embeddings

- **Stage 1 (always-on):** dump `MEMORY.md` index into the system prompt, cached at session start.
- **Stage 2 (lazy, query-time):** an LLM classifier scans the header manifest (filename + description + mtime) and selects ≤5 dossiers to inflate.

No vector DB, no embeddings, no semantic search. Sonnet does the ranking from a flat manifest.

→ For 24 personas × ~50 entities each, an LLM-over-manifest pass is fast, auditable, and aligns with Phase-2/3 privacy posture (no semantic fingerprint persists). Maps directly onto v8's name/date/concept pivot routing.

Reference: `src/memdir/findRelevantMemories.ts:39-82` (≤5-selection contract), `src/memdir/memoryScan.ts:35-94` (manifest shape).

### 2.3 LLM-driven write decision

Claude Code does not enumerate "what's memorable." It puts the rubric in the system prompt and lets the model decide. Extraction runs as a **forked agent** sharing the parent's prompt cache (cheap), once per query loop close.

→ Docknote analog: at meeting close, fork the summarizer with a "what's worth promoting to long-term memory" overlay. Cheaper than a separate pipeline; reuses already-cached transcript context.

Reference: `src/services/extractMemories/extractMemories.ts:6-9` (forked-agent pattern), `src/services/extractMemories/prompts.ts:29-43` (extraction overlay).

### 2.4 Typed-but-prompt-enforced taxonomy

Four types — `user / feedback / project / reference` — are hardcoded in `MEMORY_TYPES`, but only validated at the prompt layer. `parseMemoryType()` permits undefined; legacy/missing `type:` degrades gracefully.

→ Brain Memory's three pivot types (`name / date / concept`) want the same shape: frontmatter convention, no schema validator. Lets us add a v9 type (e.g. `decision`, `commitment`) without migrations.

Reference: `src/memdir/memoryTypes.ts:14-30` (type set + lenient parse).

### 2.5 Staleness as first-class

`memoryAge.ts` warns the model that >1-day-old "project state" facts may be stale; a literal `MEMORY_DRIFT_CAVEAT` is injected into the prompt.

→ Brain Memory has the same problem at higher stakes: *"Priya — second-year PhD"* rots in 12 months; *"primary advisor"* might change in 24. Bake age-banded confidence into the dossier card render itself, not only into the prompt. UI may want a faded role_line for >180-day stale dossiers.

Reference: `src/memdir/memoryAge.ts`, `src/memdir/memoryTypes.ts:201-256` (drift caveat copy).

### 2.6 Frontmatter description IS the retrieval surface

`description` is "used to decide relevance in future conversations, so be specific." That single line is what powers stage-2 LLM ranking — it's not UI flavor text; it's the load-bearing query target.

→ For Brain Memory: every dossier needs a one-line **differentiation bar** (role_line equivalent) authored in this register. That line *is* the retrieval surface. Treat it as schema, not as decoration.

Reference: `src/memdir/memoryTypes.ts:113-178` (TYPES_SECTION_INDIVIDUAL prompt explaining `description` semantics).

### 2.7 Truncation has a fallback story

Index over cap → truncate by line, then by byte at last newline boundary, append a warning naming which cap fired. Never silently drop.

→ Same discipline for Brain Memory's always-loaded index. The user must be able to see when entities have rolled off the always-loaded surface.

Reference: `src/memdir/memdir.ts:57-102` (`truncateEntrypointContent`).

### 2.8 Path validation paranoia

`validateMemoryPath()` rejects traversals, UNC shares, and root-near paths. Settings.json overrides are explicitly excluded from `projectSettings` because that source is untrusted (could be a hostile repo writing to `~/.ssh`).

→ Brain Memory's local-only privacy moat means user-supplied corpus paths (calendar imports, contacts CSV, third-party transcript drops) need the same paranoia. Phase-3 SDD must include a path-validation contract before any external-source ingestion lands.

Reference: `src/memdir/paths.ts:109-179`.

---

## 3. Deliberate non-choices worth copying

### 3.1 No vector DB, no embeddings

Files + Sonnet classifier carry the entire system. For Docknote, this aligns with:
- The Phase-2/3 privacy-moat thesis (no semantic fingerprint persists locally).
- Cost: SnapGPU on spark1+spark2 already does the LLM ranking work. No separate embedding model to host.
- Auditability: every retrieval decision is a traceable LLM call against a flat manifest, not a similarity score over an opaque vector.

The persona corpus has 72 queries; brute-force LLM-over-manifest is well within latency budget at that scale.

### 3.2 No mid-session cache invalidation

`MEMORY.md` changes during a session do **not** bust the prompt cache. New entries appear next session. The system trades freshness for cache hit rate.

→ For Brain Memory: accept that "what was just said in this meeting" lives in the meeting object, not the cross-meeting brain — and the brain rebuilds at session boundaries (or on explicit user refresh). Cheaper, simpler, matches user mental model ("close the meeting, then it's in the brain").

---

## 4. Where Docknote must diverge

### 4.1 Scale

Claude Code memory tops out at dozens of entries per project. Brain Memory at year-2 = hundreds of people × hundreds of meetings, possibly thousands of entities. A flat manifest LLM-ranks well up to a few hundred items; past that, latency and prompt cost climb fast.

**Implication:** the two-stage shape stays, but a **stage 0** coarse pre-filter is required:
- Substring/normalized match on person names ("Hinton" → name pivot, restrict manifest to person dossiers whose `aliases` include "Hinton").
- Date-range filter on date pivots ("last committee meeting" → restrict timeline manifest to `meeting_type=committee`, sort by date desc, top-N).
- Concept pivot can stay full-manifest LLM-ranked at v8 scale; revisit at v9.

### 4.2 Temporal reasoning

"last committee meeting" / "the call before the deposition" / "what we said two Tuesdays ago" is a date-pivot class Claude Code never has to do. Needs an explicit timeline index keyed by date + meeting_type, separate from the entity index.

### 4.3 Structured dossier schema

Claude Code frontmatter is `name / description / type` and that's it. Brain Memory cards have load-bearing structured fields:
- Name pivot: `role_line` (differentiation bar)
- Date pivot: `day_headline`, `stat_line` (`"1 decision · 2 open questions"`)
- Concept pivot: TBD at v0.5

Frontmatter must carry these as typed fields, not free text. Closer to a small typed dossier than to a Markdown note.

### 4.4 Multi-meeting synthesis

The `expected_synthesis_themes` check in the persona corpus (acceptance bar ≥80%) is asking for *cross-document theme extraction* — themes pulled from multiple meetings, not from a single dossier. Claude Code's per-entry recall doesn't do this.

**Implication:** v8 needs a synthesis pass *on top of* retrieval. Pseudocode:

```
1. classify pivot type (name/date/concept)        — LLM
2. coarse filter manifest by pivot                 — deterministic
3. select ≤K dossiers from filtered manifest       — LLM
4. inflate dossiers + pull linked meeting excerpts  — deterministic
5. synthesize themes across the K dossiers         — LLM (the new step)
6. render: dossier card + theme bullets            — UI
```

Step 5 is the Brain-Memory-specific work that has no analog in Claude Code. It is also the step the persona corpus scores hardest (`expected_synthesis_themes` is a list of 3-4 themes that must all appear).

---

## 5. Headline takeaway

Claude Code's memory is a **boring, auditable, file-system-shaped** system that punts hard work to LLM judgment at well-chosen seams. The Phase-3 win is to copy the *shape* — index + dossier, two-stage retrieval, prompt-as-rubric, no embeddings, frontmatter description as retrieval surface, age-banded staleness — and only add structure where Brain Memory's product surface forces it (typed pivot fields, temporal index, synthesis pass, scale-driven stage-0 filter).

The architecturally novel piece for v8 is **step 5 (multi-dossier synthesis)**. Everything below it is borrowable from a working precedent.

---

## 6. Open questions for Phase-3 SDD

1. Do we persist the post-meeting extraction overlay's *outputs* alongside the meeting (cheap re-run on schema migration), or only the promoted dossier deltas?
2. Stage-0 alias resolution for name pivots: phonetic match (Soundex/Metaphone) or strict normalized-string only? (Affects "Joe" vs "Joey" vs "Joseph" disambiguation in Joe-style team rosters.)
3. Cache invalidation policy: rebuild the always-loaded index lazily on next session start, or eagerly at meeting close? Claude Code chooses lazy; Brain Memory's "I just labeled three speakers and want them in the brain *now*" might force eager.
4. How does the synthesis pass cite its sources? Inline meeting-id badges on theme bullets, or a separate "evidence drawer"?
5. Privacy-moat reconciliation: when SnapGPU runs the synthesis pass on spark, does the dossier manifest leave the device? If yes, what subset? (Probably: yes, but only `description` + linked excerpt IDs; full dossier body stays local.)

---

## 7. Next research step

Web pass on existing solutions — Twitter discourse + GitHub repos. Candidates to investigate: mem0, Letta / MemGPT, Zep, Cognee, A-MEM, EpisodicMem, knowledge-graph-based personal memory projects, Cline / Cursor memory implementations, Notion AI's "AI memory" if public. Goal: confirm the no-embeddings choice is defensible in 2026, surface any synthesis-pass prior art, and check whether any local-first project has already solved stage-0 filtering at Brain-Memory scale.
