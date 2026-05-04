# Brain Memory — clarity pass

**Date:** 2026-05-05
**Companion to:** `docknote-brainmemory-design.md` (the locked 68K spec), `docknote-brain-philosophy.md` (the two principles).
**Purpose:** Resolve the one vocabulary collision that makes the locked design feel slippery, and walk the system end-to-end in one short read. **No decision in §15 of the design spec is changed.** This unifies language; it does not revise architecture.

---

## The collision

The word **engram** does two jobs in the spec:

- In §4 (Cortex), an engram is a **file** — `cortex/person/alex.md`, the Karpathy-style wiki page.
- In §4.3 and §8 (Hippocampus + recall), an engram is a **sentence vector** — one of thousands of dots in `sentence_index`, the substrate of the galaxy.

Both are right. They're the same memory at different levels of consolidation, exactly mirroring biological memory (hippocampal traces → cortical pages). But the doc never names the relationship, so the read path, the galaxy UI, and the four people-lens queries all read as slightly slippery: *which engram is a galaxy dot? which one does Brain Inbox surface? which one does Geomi propose to update?*

This doc fixes that with three named atoms.

---

## The three atoms

| Atom | Layer | Where it lives | What it is | Principle served |
|---|---|---|---|---|
| **Capture** | raw substrate | `raw/meeting/<id>/transcript.md` | Immutable transcript chunk with `[SPEAKER_n M:SS]` markers. Source-of-truth, never modified. | 1 (markdown is truth) |
| **Trace** | hippocampal index | `brain.db` → `sentence_index` (sqlite-vec) | One row per ~3-sentence passage of a transcript. Holds: passage span (`meeting_id`, `start_offset`, `len`), vector, speaker, tentative kind label, entity refs, `confirmed_at` (nullable). **Not text.** Just a pointer + attributes. | 1 (index ≠ truth) |
| **Engram** | cortical page | `cortex/person/<slug>.md`, `cortex/topic/<slug>.md`, `cortex/meeting/<id>.md` | A rendered Karpathy-style wiki page, frontmatter + body, citing back into `raw/`. One file = one engram. Edited via Geomi-confirmation flow (with direct-edit as escape hatch). | 2 (action catalyzes) |

**Mental rule of thumb:**

> A capture is what was said. A trace is *that* it was said and where. An engram is what it means.

The trace is the bridge: every claim on an engram page (`person/alex.md` says "Alex pushes timeline back, holds line on quality") is sourced to one or more traces, which point back to the capture. Pull on an engram and you can always recover the original utterance.

This is the v8-Karpathy reconciliation in one sentence: **the LLM proposes engram pages, the user confirms them through Geomi, and the trace layer is the audit trail underneath**.

---

## End-to-end (one walk through the whole system)

Each step says which atom is produced and which principle it serves.

```
1. CAPTURE        meeting ends → transcript.md + audio.opus
   ─ produces:    capture (raw)
   ─ principle:   1 — source preserved, never overwritten

2. ENCODING       per-meeting close, ~1 minute later
   a. extract_traces.py:
      - split transcript into ~3-sentence passages
      - embed each (CoreML local) → trace vector
      - LLM labels kind {observation, commitment, decision, concern, quote}
      - resolve entities eagerly against entity_index
      - write trace rows to sentence_index, link to entity_index
   b. render_meeting_engram.py:
      - LLM writes cortex/meeting/<id>.md (summary + frontmatter:
        decisions[], action_items[], claims[], attendees[])
      - cites back into raw/meeting/<id>/transcript.md

   ─ produces:    traces + one meeting engram
   ─ principle:   1 — claims live where they were said (meeting frontmatter),
                  not in a parallel facts table

3. CONSOLIDATION  nightly 03:00, batch
   - re-cluster RAPTOR forest (incremental, append-only per Decision #5)
   - lint pass: regenerate cortex/index.md, append cortex/log.md, surface
     orphans / contradictions / age-band transitions
   - propose updates to person/topic engrams from yesterday's traces
     (proposed_updates queue — NOT auto-committed to engram pages)

   ─ produces:    RAPTOR rollups + queued proposals (no new engrams yet)
   ─ principle:   2 — no engram changes without user verdict

4. CONFIRMATION   morning, the ritual that actually encodes the memory
   - Brain Inbox surfaces 3-5 proposed updates as Geomi conversational prompts
   - User: Confirm | Reject | Modify | Defer
   - Confirmed proposals commit to engram pages (with user as git author)
   - Confirmed traces get confirmed_at = now (outranks unconfirmed in recall)

   ─ produces:    updated engram pages + confirmed-trace rankings
   ─ principle:   2 — every yes is the user's own remembering act

5. RECALL         user asks (search bar, people lens, Geomi pre-meeting)
   - Stage 0: working memory (persona.md + Recent) preloaded
   - Stage 1: deterministic filter on entity_index + meeting frontmatter
   - Stage 2: LLM picks ≤5 engrams from a manifest
   - Stage 3: RAPTOR over traces within those engrams
   - Stage 4: LLM synthesizes from retrieved traces, citing meeting_ids

   ─ reads:       traces (the galaxy) + engram pages (the wiki)
   ─ principle:   1 — meaning reconstructed at query time, not pre-stored
```

This is the whole loop. Capture → trace → engram → confirmation → recall, with the consolidation pass as the bridge between yesterday's traces and tomorrow's engram updates.

---

## The four people-lens queries — concrete shapes

The locked landing (`v4.3-memory-galaxy-floating-ask.html`) commits to these four. Here is what each one *actually does* against the trace + engram layers. The killer-move (scope × person, e.g. "everything Alex said in February") falls out of #1.

| Query | Index used | Retrieval | LLM in loop? |
|---|---|---|---|
| **Who said what** | `sentence_index` filtered by `speaker = person` | Traces grouped by meeting, sorted by date. The passage text is fetched from the capture span on demand. | No (deterministic — this is the killer-move's substrate) |
| **Whose progress** | `sentence_index` filtered by `entity = person AND kind ∈ {observation, concern}` | Time-ordered traces. LLM weaves the arc ("first visit reported X, second visit added Y…"). | Yes — Stage 4 synthesis |
| **Who committed** | `sentence_index` filtered by `(speaker = person OR mentioned = person) AND kind = commitment` | Two-column render: their commitments to me vs my commitments to them. | No (deterministic list with passage text) |
| **Who needs care** | `sentence_index` filtered by `entity = person AND kind = concern AND confirmed AND recency < 90d` | Surfaced proactively in pre-meeting brief; in lens, ranked list. | Optional — a one-line "why this matters now" via Stage 4 |

Two things to notice:

1. **Three of four are deterministic.** No LLM hallucination risk on the dominant gestures. The LLM is only invoked for "whose progress" (genuinely synthetic) and optionally for the "why this matters" annotation on "who needs care". Principle 1 in action — structure is enough; the model is reserved for the actually-hard step.

2. **All four are SQL on `sentence_index`** — same query shape, different `WHERE` clauses. Adding a fifth lens (e.g. "who disagreed") is a `WHERE kind = disagreement` away. The lens system is extensible without schema changes.

---

## The galaxy UI — concrete shapes

The galaxy plots **one dot per meeting**, positioned at the centroid of that meeting's traces in vector space. Constellations are RAPTOR clusters (themes spanning meetings). This is the right resolution for a landing surface — sentence-level dots would be a fog; meeting-level dots are scannable while keeping the underlying truth at sentence resolution.

| Visual | Data |
|---|---|
| **Dot** | One meeting. Position = centroid of its traces. Size = trace count (a longer meeting is a bigger dot). |
| **Constellation** | One RAPTOR cluster. Label = LLM-generated theme name. |
| **Opacity** | `created_at` decay. The neglected cluster fades; the active one glows. |
| **Click constellation** | Spotlight: filter to its dots, dim everything else. |
| **Click time column** | Filter dots to that period. Compose with constellation and pinned person. |
| **Click dot** | Drill into per-meeting view (sentence-level traces visible there, not on the galaxy). |
| **Pin person + click time** | The killer move. Equivalent to "Who said what" filtered to the time range. |

**Why meeting-level on the galaxy and not sentence-level:** at typical corpus size (1-3K meetings × ~200 traces/meeting = 200K-600K trace points), sentence-level would render as a smear. Meeting-level dots are the right zoom for orientation, with sentence-level retrieval available the moment the user clicks in.

This *is* a small design call beyond §15 of the spec. See "Open question 1" below.

---

## What does NOT change

Every decision in §15 of the locked spec stands. To name the load-bearing ones:

- Markdown source-of-truth (Decision #1)
- git as audit substrate (Decision #2)
- SQLite + sqlite-vec, no graph DB (Decision #3)
- No `facts` table — reconstruct at recall (Decision #12)
- Karpathy three-layer model: `raw/` + `cortex/` + `cortex/SCHEMA.md` (Decision #13)
- Geomi as confirmation mediator (Decision #14)
- People as first-class lens; scope × person is the load-bearing gesture (Decision #15)

This doc is purely the unification of vocabulary across §4 (cortex/engram) and §4.3 + §8 (hippocampus/sentence_index). The naming "**capture / trace / engram**" replaces the implicit dual usage of "engram"; everything else is downstream of decisions already made.

---

## Genuinely-open questions (additions to §14 of the spec)

§14 listed 8 SDD-routed questions. After this clarity pass, three more surface that the spec didn't name:

**1. Galaxy plotting unit — meeting-level dot from trace-centroid (proposed above), or sentence-level dot with meeting-level visual grouping?**
Recommendation: **meeting-level dot**. Cleaner at corpus scale, sentence-level retrieval is a click away. Sean to confirm or push back.

**2. `meeting_engram ↔ traces` cardinality — is the meeting engram regenerable from traces alone, or is its body authoritative?**
Recommendation: **regenerable.** The meeting engram is a rendered view; if lost, re-render from `raw/transcript.md`. Frontmatter (`decisions[]`, `action_items[]`, `claims[]`) is authoritative because it's user-confirmed; body is derived. This matches §4.1's "meeting engram is a rendered summary; if lost, regenerate" but pushes the boundary clearly to *frontmatter authoritative, body derived*.

**3. Person/topic engram regeneration — same question, harder answer.**
A `person/alex.md` page accumulates user-confirmed claims over months. Body is *not* regenerable from traces alone; the chosen wording is the user's. Recommendation: **frontmatter and body both authoritative on person/topic engrams; only the meeting engram body is derived.** This is the line where principle 2 starts to bind: the user has been encoding via confirmation for months, and that encoding lives in the page's wording.

These three are upstream of the four seam-A storage decisions left over from the seams diagram (engram granularity, embedding compute, consolidation cadence, entity resolution timing). Resolve these first; the storage picks fall out cleanly.

---

## One-line summary

**Three atoms:** capture (what was said) → trace (that it was said, indexed) → engram (what it means, confirmed). The galaxy plots meetings; the four people-lens queries are SQL filters on traces; Geomi confirms proposed engram updates; the LLM only synthesizes when synthesis is actually required. Principle 1 keeps the structure thin. Principle 2 keeps the user encoding.
