# Brain Memory v8 — design vs UI/SDD review

**Date:** 2026-05-04
**Purpose:** Audit the locked v8 design (`docknote-brainmemory-design.md`) against existing Docknote brain-memory artefacts (Monica's SDD, the v8 hi-fi mockup, the 24-persona gallery, the persona-corpus, Geomi-concept). Identify divergences, schema gaps, and required reconciliations before SDD-2 work begins.

**Source artefacts reviewed:**
- `~/seanslab/Docknote/docs/brain-memory-sdd.md` — Monica's Phase-2 SDD lock (visual reference at v6 mockup)
- `~/seanslab/Docknote/ux-mockups/brain-memory/murmur-tide-hifi-v8.html` — canonical v8 hi-fi
- `~/seanslab/Docknote/ux-mockups/brain-memory/personas-gallery-v1.html` — 24-persona dossier-card gallery
- `~/seanslab/Docknote/ux-mockups/brain-memory/scan-and-synthesis.html` — synthesis surface mockup
- `~/seanslab/Docknote/tests/eval-brain-memory/persona-corpus.yaml` — acceptance corpus
- `~/seanslab/Docknote/scripts/eval_brain_memory.py` — eval harness
- `~/seanslab/Docknote/docs/geomi-concept.md` — Geomi companion model spec

**Design under review:**
- `/Users/seansong/seanslab/Research/cc-learn/docknote-brainmemory-design.md`

---

## 1. Headline finding

The design and the existing UI/SDD are **directionally aligned, materially divergent in three places.** The read-side surface (pivot routing, dossier card composition, synthesis-with-fragments) tracks closely. The write-side (Brain Inbox, click-to-edit, time-travel, `human_locked` semantics) and parts of the file taxonomy are entirely absent from existing artefacts.

Three classes of problem to resolve:

1. **File taxonomy collision** — SDD specifies `~/Docknote/index/people/<id>.md`; design specifies `~/Docknote/brain/person/<slug>.md`. New files in design (`active.md`, `patterns.md`, `commitments.md`, `persona.md`, `project/`) have no SDD or UI counterpart.
2. **Synthesis output shape mismatch** — design specifies structured JSON; UI renders prose paragraph; corpus scores keyword substring on the prose.
3. **Three full UI features have zero mockup representation** — Brain Inbox, click-to-edit + provenance badges + lock indicators, time-travel / history / revert.

Plus six schema fields the mockups render that the design schema doesn't name.

---

## 2. Per-dimension findings

### A. Pivot routing

- **Design (§8 Stage 0.5):** `pivot_route(q)` — LLM call classifying query into `name | date | concept`.
- **SDD §5 + mockup + corpus:** Same three pivots. SDD specifies a **deterministic resolver chain** — `dateparser → person fuzzy → concept exact → semantic fallback (≥0.55 cosine)` — with LLM as optional fallback. v8 ships v0 with name + date only; concept = v0.5.
- **Status:** ⚠️ partial.
- **Action:** Adopt SDD's resolver-chain-first as canonical. Demote LLM router to fallback. Cheaper, deterministic, matches the user's published mental model.

### B. Dossier card field schema

- **Design (§4.4):** `role_line`, `day_headline`, `stat_line`, `differentiation_bar` per card with provenance.
- **UI/SDD/corpus:** v8 mockup renders four CSS slots:
  - `dossier-name` — canonical name
  - `dossier-role` — one-line role with accent fragment (e.g., *"Series A investor · <span class='accent'>friendly skeptic</span>"*)
  - `dossier-body` — 2-3 sentence prose paragraph (the most load-bearing field)
  - `dossier-stat` — touchpoint micro-stat (e.g., *"12 meetings · last seen today, 9:17 am"*)
  - `dossier-list` — for date pivots, `<ul>` of `(time, meeting_name)` quick-list entries
- **Persona corpus** scores three fields total: `role_line` (name pivot), `day_headline` + `stat_line` (date pivot). Concept pivots **omit the card** in v0.
- **Status:** ⚠️ partial — naming inconsistent and one design field is unmapped.
- **Action:**
  - Map `dossier-name` → `canonical_name` (already in §4.4).
  - Map `dossier-role` → `role_line` ✓.
  - Map `dossier-stat` → `stat_line` ✓.
  - **Add `body_blurb`** — the 2-3 sentence prose paragraph rendered as `<p class="dossier-body">`. Most load-bearing single field on the card and currently missing from the schema.
  - **Add `day_quick_list: [{time, name}]`** for date-pivot cards.
  - Resolve `differentiation_bar`: it's currently unscored by the corpus and substantively redundant with `role_line.accent`. Either fold into `role_line` (recommended) or drop entirely.

### C. Citations and source attribution

- **Design (§9.2 + §10.4):** every dossier claim and synthesis output cites `meeting_id + span`.
- **UI:** rendered as fragment rows below the synthesis paragraph:
  ```html
  <div class="frag-source">
    from <a href="…">privacy-review-30apr / 12:48</a>
  </div>
  ```
  Synthesis paragraph itself does **not** carry inline `[1]` markers — provenance lives entirely in the fragment list (Granola-style: synthesis on top, receipts beneath).
- **Status:** ⚠️ partial.
- **Action:** Make explicit in the design that synthesis citations are *list-anchored* via fragment rows, **not** inline `[n]`. Current design wording is ambiguous; UI has clearly chosen the Granola pattern.

### D. Synthesis output shape

- **Design (§8 Stage 4):** strict JSON `{themes: [{name, description, citations: [meeting_ids]}]}` × 3-4.
- **UI:** mockup renders synthesis as a single 2-4-sentence prose paragraph (`<div class="synthesis"><p>…</p></div>`), **not** a structured 3-4-themes object.
- **Persona corpus:** scores `expected_synthesis_themes: [string, string, string, string]` — a flat keyword list (4 themes, no descriptions, no citations field). The eval harness (`scripts/eval_brain_memory.py:152-158`, `_judge_synthesis_grounding`) does case-insensitive substring matching: keywords must appear in the prose paragraph.
- **Status:** ❌ divergent.
- **Action:** Resolve. Two options:
  1. Keep the structured JSON internally; render externally as a prose paragraph (themes inlined, fragments as the citation surface). Eval scores against the prose. Recommended.
  2. Drop the structured JSON; Stage 4 emits prose directly per SDD §3.7's `brain_synthesis_prompt.yaml`.
  
  Recommend (1) — preserves a programmatic API surface for future use (e.g. if MCP is revisited post-launch).

### E. Brain Inbox / proposed-updates UI

- **Design (§9.3):** dedicated "Brain Inbox" surface listing auto-extracted changes since last review. Per-field accept/reject. Sentence-level diff. Conflict resolution.
- **UI/SDD/mockups:** **absent entirely.** No Brain Inbox surface in the v8 mockup, persona gallery, scan-and-synthesis, or SDD. Monica's SDD §3-§7 is purely the recall/read surface.
- **Status:** ❌ divergent (full design feature missing from UI).
- **Action:** Brain Inbox needs its own SDD pass and hi-fi mockup. The design's text rendering (lines 486-499) is a verbal sketch, not a buildable spec. Min/Rachel should produce a hi-fi alongside the next murmur-tide iteration.

### F. Edit affordances

- **Design (§9.1):** click-to-edit per field with `human_locked: true` semantics — auto-pass cannot overwrite locked fields, queues to `proposed_updates` instead.
- **UI:** dossier card in mockup is **read-only.** No click-to-edit, no lock indicators, no edit pencil, no provenance badges (`auto-extracted · 2 days ago · qwen-3-32b`).
- **Status:** ❌ divergent.
- **Action:** Either ship edit UI or descope. If shipping, needs (a) `:hover` edit affordance on each `dossier-*` slot, (b) badge sub-component below each field, (c) lock-icon when `human_locked: true`. None of this exists today and there is no mockup precedent.

### G. History / time-travel UI

- **Design (§9.4):** date slider on dossier card, one-click revert, 5-deep field history visible, side drawer for full git history.
- **UI:** absent. The Tide ribbon is a meeting-recency timeline, **not** a card-history time-travel UI.
- **Status:** ❌ divergent.
- **Action:** Net-new mockup work. Could plausibly piggyback on Tide visually (slide a card-state indicator along the same vertical ribbon) but no design exists.

### H. Geomi proactive surfaces (Layer 2.5)

- **Design (§6):** three triggers — `pre_meeting_brief`, `mid_meeting_suggest` (negative-default), `end_of_day` nudge.
- **`geomi-concept.md` §5:** specifies a **ten-touchpoint map** — pre-meeting (T-10 min), cross-meeting cluster forms, speaker-label confirm, vocabulary unknown, transcription complete, summary complete, brain opened, chat opened, plus the design's three. Hard rule §7.3: *"Event-driven bubbles only. Never time-driven. Max one per stage. Dismiss = silent for the remainder of that meeting."*
- **Status:** ✅ aligned on principle (negative-default rule matches REQ-075), ⚠️ partial on coverage — Geomi-concept names *more* surfaces than the design.
- **Action:** Either expand Layer 2.5 to cover all ten Geomi-concept touchpoints, or explicitly scope: *"v8 ships only the three brain-grounded triggers; the other seven are owned by the per-meeting recording pipeline, not the brain layer."* Recommend the latter — keeps Brain Memory's API surface focused.

### I. Always-loaded files

- **Design (§4.1, §8 Stage 0):** `persona.md` + `active.md` always-loaded into system prompt. Combined cap 4K tokens.
- **UI/SDD:** **absent.** SDD §3 enumerates `meta.md`, `summary.md`, NEW PERSON index, NEW CONCEPT index, `brain_synthesis_prompt.yaml` cache — no `persona.md` or `active.md` analog. Geomi-concept §4 ("Memory is visible — what the Geomi has learned is exposed in Brain / Settings as a personality trait") gestures at exposing memory but doesn't reference an `active.md`-style rolling digest.
- **Status:** ❌ divergent.
- **Action:** Decide whether `active.md` and `persona.md` exist as files (design says yes) and where they surface in the UI. Candidate: a Brain settings panel showing *"What I always know about you"* (persona.md) and *"What's on your mind this fortnight"* (active.md preview). Currently invisible to the user.

### J. File taxonomy

- **Design (§4.1):**
  ```
  ~/Docknote/brain/
  ├── persona.md
  ├── active.md
  ├── patterns.md
  ├── commitments.md
  ├── person/<slug>.md
  ├── project/<slug>.md
  └── meeting/<YYYY-MM-DD>-<slug>.md
  ```
- **SDD §3:**
  ```
  ~/Docknote/index/
  ├── people/<person-id>.md
  ├── concepts/<concept-id>.md
  └── aggregate_index.people.md   # existing pattern, extended
  ```
  No `project/`, no `commitments.md`, no `patterns.md`, no `persona.md`, no `active.md`.
- **Status:** ❌ divergent. Major reconciliation required.
- **Action:** Two paths:
  1. **Migrate SDD's `~/Docknote/index/people/` → design's `~/Docknote/brain/person/`** — rename + relocate. Strategic upside: `~/Docknote/brain/` as a git repo is the privacy/portability moat (the wedge against Granola). The SDD's `index/` location reads as derived/internal.
  2. Keep SDD layout; drop design's `brain/` namespace; fold new file types into `index/` tree.
  
  Recommend (1). Add migration note to design §4.1; revise §4.1 to acknowledge SDD's prior choice. Either way, design's `project/<slug>.md`, `commitments.md`, `patterns.md`, `persona.md`, `active.md` are entirely new files; need to be added to SDD or descoped from design.

### K. Acceptance corpus alignment

- **Persona corpus** declares only three dossier fields:
  - `role_line` (name pivot)
  - `day_headline` (date pivot)
  - `stat_line` (date pivot)
  - Concept pivots: *"concept → concept card (v0.5); card omitted from v0 scoring (no card expected)"*
- **Eval harness** (`scripts/eval_brain_memory.py:152-158`) confirms: stub returns `{role_line}` for name, `{day_headline, stat_line}` for date, `{}` for concept.
- **Design's `differentiation_bar`** is **not scored** by the corpus.
- **Design's `body_blurb` / `dossier-body`** (which the persona-gallery renders for every persona) is **not scored** either.
- **`expected_synthesis_themes`** is a flat string list, not a `{name, description, citations}` object (per dimension D).
- **Status:** ⚠️ partial.
- **Action:** Either (a) drop `differentiation_bar` from design schema as unscored, (b) extend `persona-corpus.yaml` to score it (and `body_blurb`). Pick one. Currently the design names a field the eval bar ignores.

### L. Visual / layout patterns (Tide / Hearth / Murmur)

- **Mockup composition:**
  - `aside.tide` (left) — vertical SVG ribbon timeline with ovals (one per meeting), threads (shared-people connectors), today-bubble pulse, pivot-driven lit/dim states.
  - `main.murmur` (right) — `.response-grid` containing synthesis on left + dossier card on right, then full-width `.frag` rows below.
  - "Hearth" — the dossier card itself. v8 mockup header annotation: *"This re-introduces the Hearth surface that SDD v0 cut — but as a sibling card inside Murmur, not a separate scene."*
- **Design treatment (§3):** "Brain scene (cards + chat) · Geomi bubbles · Brain Inbox" — generic naming, no Tide/Hearth/Murmur callout.
- **Status:** ⚠️ partial — design treats Layer 3 generically.
- **Action:** Add to design §3 a Layer 3 sub-section pointing to `ux-mockups/brain-memory/murmur-tide-hifi-v8.html` as the canonical composition and naming the three components (Tide ribbon, Hearth dossier card, Murmur response area).

---

## 3. Schema gaps the design missed

Six fields the mockups render that the design doesn't name:

1. **`body_blurb` / `dossier-body`** — the 2-3 sentence prose paragraph rendered for every persona-gallery card (e.g., *"The voice you reach for when the product story drifts toward features. Cares about data outliving the company…"*). Most load-bearing field visually; absent from §4.4.

2. **Date-card `quick_list`** — the `<ul class="dossier-list">` of `(time, meeting_name)` tuples on a date pivot. Absent from §4.4.

3. **`role_line.accent` substring** — the visually highlighted suffix (`<span class="accent">friendly skeptic</span>`) that carries the differentiator. Either codify as structured `{prefix, accent}` pair or document the trailing `·`-segment styling convention.

4. **Fragment row schema** — `{stripe_color, who, ts, attribution, text, source_link, decoration}`. Used consistently in mockup; design §3.7 mentions fragments only obliquely.

5. **Empty / sparse state copy** — *"Nothing in your brain matches that — try a name or a date"* (negative case), *"Memory needs at least 10 meetings to find patterns. You have 3."* (small-corpus). SDD §6.1 owns these; design references neither.

6. **Concept-card variant** — the persona-gallery shows concept cards using `dossier-avatar.is-glyph` (a `#` mark) with `dossier-role: "Concept · 3 meetings"` and a touchpoint stat. Design §4.1 mentions `project/<slug>.md` but not how concept dossiers (vs project dossiers vs ad-hoc concept hits from semantic fallback) render.

---

## 4. UI work the design hasn't accounted for

The design specifies surfaces or affordances that have no mockup precedent:

1. **Brain Inbox surface** — needs a dedicated mockup; verbal description in design §9.3 isn't enough for build.
2. **Click-to-edit + provenance badges + lock indicators** — every field on the dossier card needs `:hover` affordance, badge sub-component, lock icon. None mocked.
3. **Per-field history drawer + date-slider time-travel + revert button** — entirely net-new UI.
4. **`active.md` preview surface** — design says it's "always loaded"; user should be able to see what's in it.
5. ~~**Sync UX (Phase-3 stretch)**~~ — **dropped** (Sean direction, post-Rachel review). Backup is handled outside this design layer; brain stays single-device. Tailscale-only-remote guardrail comes off the wall.
6. **Sensitive-meeting marker** — design §10.4 names it; no UI mockup.
7. **End-of-day Geomi nudge format** — design says "summarize what happened today, surface unsettled commitments" but has no visual.
8. **Mid-meeting suggestion bubble** — owned in Geomi-concept but the design's `BrainTriggers::mid_meeting_suggest` returns a `Suggestion` shape that needs a UI binding.

Items (1)-(3) are critical-path UX features the architecture depends on. Items (4)-(8) can lag.

---

## 5. Recommended edits to `docknote-brainmemory-design.md`

Ten specific changes, with section / line citations:

1. **§4.1 (line ~108):** Reconcile file taxonomy with SDD §3. Either rename `~/Docknote/brain/` to match SDD's `~/Docknote/index/`, or add a migration note explicitly migrating from `index/people/` → `brain/person/`. Ensure `person/`, `project/`, `concept/` are all explicit. Recommend keeping `~/Docknote/brain/` and migrating SDD layout — preserves the privacy/portability wedge.

2. **§4.4 (lines 182-220):** Add `body_blurb` field to card frontmatter (2-3 sentence prose). Add `day_quick_list: [{time, name}]` for date cards. Decide `differentiation_bar` (recommend: fold into `role_line` accent, or drop). Document the `role_line.accent` trailing-`·`-segment styling convention.

3. **§5 / Stage 0.5 (line ~258, ~418):** Demote `pivot_route` LLM call to optional/fallback. Make SDD §5's deterministic resolver chain (dateparser → person fuzzy → concept exact → semantic ≥0.55 cosine) the canonical first-pass.

4. **§3 / Layer 3 (line ~94):** Name Tide / Hearth-as-card / Murmur explicitly; cite `ux-mockups/brain-memory/murmur-tide-hifi-v8.html` as the canonical composition.

5. **§6 (line ~298):** Either expand Layer 2.5 trigger list to align with `geomi-concept.md` §5's ten touchpoints, or explicitly scope to the three brain-grounded triggers and note that the other seven are owned by the per-meeting recording pipeline. Recommend the latter — keeps Brain Memory's API surface focused.

6. **§8 Stage 4 (line ~446):** Clarify that the structured `{themes: [{name, description, citations}]}` JSON is internal; user-facing render is a 2-4-sentence prose paragraph with citations anchored in the fragment list below (Granola pattern, not inline `[n]`).

7. **§9.3 (line ~482):** Promote the verbal Brain Inbox sketch to a `tasks/todo` entry to produce a hi-fi mockup. Note that current state is verbal-only and unbuildable.

8. **§12.1 (line ~571):** Note that `differentiation_bar` and `body_blurb` are not currently scored by the persona corpus; either extend the corpus or remove the fields from the design schema.

9. **§13 (line ~592):** Add an eval-corpus extension item: include `body_blurb` and concept-card fields if §4.4 keeps them.

10. **New §3.5 or §11:** Document `active.md` and `persona.md` surface treatment in the UI — settings panel, "What I always know about you" preview — currently invisible to user.

---

## 6. Recommended next steps

### Path 1 — quick reconciliation (1-2 hours)

Apply the 10 edits in place to `docknote-brainmemory-design.md`. This is surgical: add missing schema fields, demote LLM router, name Tide/Hearth/Murmur, scope Geomi triggers, clarify synthesis prose output, drop or merge `differentiation_bar`, reconcile file taxonomy at §4.1, route Brain Inbox / edit UI / time-travel to net-new mockup work.

After path 1, the design doc accurately reflects existing UI/SDD constraints and surfaces unmocked work explicitly.

### Path 2 — broader reconciliation (heavier, schedule for later)

Update both `docknote-brainmemory-design.md` AND propose patches to Monica's `brain-memory-sdd.md` to add `active.md`, `persona.md`, `commitments.md`, `patterns.md`, `project/`, and the Brain Inbox surface, which the SDD doesn't cover at all.

Recommended: do path 1 now; schedule path 2 for after Min/Rachel produce mockups for Brain Inbox + click-to-edit + history. The SDD is Phase-2 work locked at v6 visual reference; updating it is heavier coordination than fixing the design doc inline.

### Path 3 — net-new UI mockups required (parallel track)

Three mockup deliverables for the missing surfaces. Owners: Min (visual) + Rachel (interaction).

- **Brain Inbox** hi-fi: list view of proposed updates with sentence-level diff, per-field accept/reject, conflict resolution UX.
- **Click-to-edit + provenance badges + lock indicators**: `:hover` affordances, badge sub-components, lock icons on every dossier card field.
- **Time-travel / per-field history**: date slider on the dossier card, revert button, history drawer (5-deep visible, "show full history" → side panel).

Without these mockups, the corresponding SDD sections cannot be written and the design's audit/edit story is verbal-only.

---

## 7. Open coordination questions

1. **Who owns reconciling the SDD?** Monica wrote it for Phase 2; v8 in Phase 3 means a new SDD pass. Coordinate with her before path 2.
2. **Does `~/Docknote/brain/` win over `~/Docknote/index/`?** Strategic call. Recommend yes (privacy moat), but the migration touches Chandler's existing per-meeting indexer.
3. **Are the new files (`active.md`, `patterns.md`, `commitments.md`, `persona.md`, `project/`) all v8-ship, or do some defer?** Design currently says all v8; SDD says none. Compromise possible — e.g., `persona.md` + `active.md` v8-ship; `patterns.md` + `commitments.md` ship as auto-generated views from facts table; `project/<slug>.md` defers to v9.
4. **Brain Inbox: v8 or v9?** Design treats as v8-ship. If mockup work isn't ready, descoping to v9 is the honest call. Without it, the audit story degrades to "git log in a terminal" — still better than Granola, but not the consumer-software promise the design implies.
5. **Edit UI: v8 or v9?** Same question. Read-only v8 + edit v9 is a respectable phasing.
