# Docknote Brain Memory v8 — design

**Read this first:** `docknote-brain-story.md` — the architecture as story. Everything below assumes the vocabulary it establishes:

- **engram = file = page.** *Engram* in this doc and code (the brain metaphor distinguishes the index from the trace). *File* when describing the on-disk artifact. *Page* in user-facing copy and Rachel's mockups (the commonplace-book metaphor — the unit the user reads).
- **cortex** = the bound pages, the markdown layer (`~/Docknote/brain/cortex/`).
- **hippocampus** = SQLite indexes + RAPTOR + the Brain API (recall reconstructs at query time).
- **consolidation** = the nightly 03:00 pass; **encoding** = the per-meeting close.
- **working memory** = `persona.md` + the Recent view, always-loaded.

**Status:** Design draft for SDD review. Architecture decisions locked; SDD-level interface specs and prompts to follow.
**Date:** 2026-05-04
**Author:** Lead
**Phase:** Phase 3 (Brain Memory v8 was deferred from Phase 2 per REQ-076 §2)

**Companions (research dossiers, derived from):**
- `docknote-brain-story.md` — the 30-second story; read first
- `docknote-memory-learn-from-cc.md` — lessons from Claude Code's `src/memdir/` implementation
- `docknote-memory-leads.md` — broad sweep, ~155 leads (GitHub / Twitter / products+blogs+papers)
- `docknote-memory-deepdive-tier1.md` — Letta, Graphiti, Granola, Minutes, Anthropic memory tool
- `docknote-memory-deepdive-tier2.md` — mem0, MemMachine, HippoRAG 2 + RAPTOR, Letta sleep-time internals, Cline memory bank
- `docknote-memory-visuals.html` — visual reference gallery
- `rachel-redesign/` — the v9 UI/UX mockups

**Source product reqs:**
- REQ-070 — Brain Memory v8 (this doc)
- REQ-075 — Geomi: one brain, five faces (companion model that consumes this layer)
- REQ-031 — simple is better; REQ-032 — trust the LLM at the right seams
- REQ-076 §2 — v8 deferred from Phase 2; persona-corpus harness preserved as artifact
- Acceptance: `tests/eval-brain-memory/persona-corpus.yaml` (24 personas × 3 queries × 3 sub-checks; bar ≥ 80% = ≥ 58/72)

---

## 1. Scope

### In scope (Phase 3 — v8 ship)

- Cross-meeting recall organized by **name / date / concept** pivots
- **Person-, topic-, and meeting-shaped engrams** with structured fields per kind (`role_line`, `day_headline`, `stat_line`, `differentiation_bar`)
- **Multi-meeting theme synthesis** — extract 3-4 themes with cited source meetings
- Per-meeting incremental consolidation + nightly 03:00 cross-engram consolidation
- **Citations to source meeting + timestamp span** on every engram claim and synthesis output
- **Per-field provenance + git-backed audit trail** with one-click revert
- **Memory landing surface — galaxy view of all meeting notes**, clusters held together by edge weight (project / series / attendee / sameday), recency encoded as opacity, with a **floating Geomi ask pill** at the bottom (matches Notes-view pattern). The user-facing unit is **note** (every Docknote-generated note from a meeting); architecture/code calls it engram. See §9.1.
- **Vertical time column (right side)** — always-visible grouped scope selector (recent · months · quarters · years · all time). Click a scope → galaxy spotlights notes in that scope, others fade; cluster geometry never reorganizes. No companion list, no horizontal ribbon, no drag-pin rewind on the landing (rewind moves to per-engram §9.5). See §9.1.
- **People as a first-class lens** on the Memory surface. A person view answers four queries directly, as Geomi-written prose with marginalia awaiting initials: *who said what · whose progress · who needs care · who committed to what*. People are a sibling lens to galaxy, not a tab inside it. See §9.1 and decision log #15.
- **The killer-move combination** — `time-scope × person-pin` collapses cross-meeting attribution to a single click: "show me everything Alex said in February (or Q3, or last week)" lands as the natural gesture for the doctor / lawyer / salesperson user. See §9.1.
- **Brain Inbox** UI for reviewing auto-extracted changes (the morning re-read state of the Memory surface when marginalia is pending)
- Geomi proactive triggers (pre-meeting brief, mid-meeting bubble suggestions, end-of-day nudge)
- Acceptance against the 24-persona corpus at ≥ 80%

### Out of scope (deferred)

- **MCP server.** Filesystem + git is the universal interop layer in v8. MCP earns its keep only when synthesis tools become genuinely useful for non-Docknote agents — re-evaluate post-launch.
- **Anthropic memory-tool wire format.** Same logic. The agent is in-process against SnapGPU/Qwen; no protocol needed.
- **Cross-source recall** (Slack / email / docs / calendar). v8 is meeting-only. Calendar event metadata is the one exception (drives proactive triggers and pre-meeting briefs).
- **Cloud-hosted / shared-team brain.** Single-user only in v8. Multi-user shared brain is post-v8.
- **Multi-device sync.** v9 is single-device. Backup is handled outside this design layer (Docknote-managed, not via user-managed git remote). If multi-device support returns later, it returns as a backup-restore flow, not a real-time sync.
- **Encryption-at-rest** (OI-001 — deferred to Phase 3 per REQ-076 §1). Layered in via FileVault or `git-crypt` when it lands; doesn't conflict with this design.

### Non-goals

- Replicating Granola's chat UI. v8 is a distinct second-top-level scene; chat happens in Geomi.
- Becoming a vector-DB benchmark winner. The architecture aims for >80% on the persona corpus on Qwen-32B, not SOTA on LongMemEval.
- General-purpose agent memory infra. v8 is meeting-recall–shaped.

---

## 2. UVP alignment

| UVP leg | How v8 serves it |
|---|---|
| **Privacy moat is literal (Slide-4 ground truth)** | Markdown source-of-truth (vs Granola's encrypted SQLite); git audit you can verify line-by-line; SnapGPU/Qwen for synthesis = no Anthropic/OpenAI egress; brain stays single-device; audio not retained. |
| **Geomi "one brain, five faces" (REQ-075)** | working memory (`persona.md` + Recent view) always-loaded; structured engram fields enable specific observations rather than platitudes; synthesis pass powers pattern-aware bubbles; per-field provenance lets Geomi cite *why* it's surfacing something. |
| **Production-grade packaging** | `git2-rs` + SQLite + sqlite-vec drop into Tauri without subprocess fork/exec; no external service deps; signed `.dmg` pipeline unaffected. |
| **REQ-070 acceptance corpus (≥80%)** | Three-pivot routing matches `pivot_type`; bi-temporal facts handle "in March"; synthesis pass produces the `expected_synthesis_themes` shape directly. |
| **REQ-031 simple / REQ-032 trust the LLM** | Boring file-system-shaped stack (Claude Code's playbook); LLM-driven judgment at well-chosen seams; no NIH'd vector DB or graph DB. |
| **Honest engineering** | git history + per-field provenance = users can audit every claim. The same posture as `memset_s` and X25519: verifiable, not promised. |

---

## 3. Architecture — one-page summary

```
┌─────────────────────────────────────────────────────────────────────┐
│  CORTEX — durable engram storage                                    │
│  ~/Docknote/brain/*.md   (one file per engram, git-versioned)       │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ git2-rs · sqlx · sqlite-vec
                              │
┌─────────────────────────────────────────────────────────────────────┐
│  HIPPOCAMPUS — indexes (pointers into the cortex, never truth)      │
│  ~/Library/Application Support/Docknote/brain.db                    │
│  · sentence_index (sqlite-vec) — the galaxy                         │
│  · raptor_nodes — hierarchical constellations over the galaxy       │
│  · entity_index — name → engram resolution                          │
│  · meeting_fts — keyword fallback                                   │
│                                                                     │
│  Brain API (in-process Rust trait, the only programmatic surface):  │
│    read · write · history · time-travel · synthesis                 │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────────────┐
│  TRIGGERS — proactive surface for Geomi                             │
│                                                                     │
│  pre-meeting brief · mid-meeting suggestion · end-of-day nudge      │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────────────┐
│  UX — the surfaces the user touches                                 │
│                                                                     │
│  Brain scene (Tide · Hearth · Murmur) · Brain Inbox · Geomi bubbles │
└─────────────────────────────────────────────────────────────────────┘
```

**Vocabulary (per `docknote-brain-story.md`):**

| Brain (architecture) | Book (user-facing) | Docknote |
|---|---|---|
| Neurons (substrate) | — | bits / bytes |
| Engram (one memory) | a **page** | one file in the cortex |
| Cortex (durable storage) | the bound pages | `~/Docknote/brain/cortex/*.md` |
| Hippocampus (index + recall) | the index at the front + the spine | SQLite + sqlite-vec + RAPTOR + Brain API |
| Working memory (always-loaded buffer) | bookmarked pages | `persona.md` + Recent view |
| Encoding | a new page added | per-meeting-close extraction |
| Consolidation | the page turning while you sleep | nightly 03:00 pass |
| Recall | re-reading | the 5-stage read path |
| Schemas | recurring themes across pages | the Recurring view (auto-detected themes) |
| (LLM-proposed update) | **marginalia awaiting your initial** | a row in `proposed_updates`, surfaced via Geomi |

The cortex is the universal interop layer. Plain markdown files on disk; no MCP, no HTTP API, no wire protocol. Power users who poke under the hood get exactly what's there: a markdown directory and a git repo.

---

## 4. Cortex — the engram store

The cortex is the durable layer where engrams live. One file = one engram. Markdown source-of-truth, versioned by git, edited through the Brain API.

### 4.1 Filesystem layout — two layers, the Karpathy three-layer model

Following Karpathy's [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) (April 2026), the brain repo splits into two physical namespaces plus an editable schema layer. v8 independently arrived at the same shape via Tier 1 + 2 deep-dives; we adopt Karpathy's vocabulary for clarity:

```
~/Docknote/brain/                    # git repo (the whole brain)
│
├── raw/                             # immutable source captures
│   └── meeting/
│       └── <YYYY-MM-DD>-<slug>/
│           ├── transcript.md        # full transcript with [SPEAKER_n M:SS] markers
│           ├── audio.opus           # source audio (deletable per-meeting)
│           └── metadata.json        # duration, attendees, calendar event id
│
├── cortex/                          # the LLM-maintained wiki (engrams live here)
│   ├── index.md                     # navigation catalog (auto-generated by lint)
│   ├── log.md                       # chronological event log (auto-appended)
│   ├── SCHEMA.md                    # engram frontmatter contract (user-editable)
│   ├── persona.md                   # the user's own engram (always-loaded)
│   ├── person/
│   │   └── <slug>.md                # one engram per person
│   ├── topic/
│   │   └── <slug>.md                # one engram per case / project / concept
│   └── meeting/
│       └── <YYYY-MM-DD>-<slug>.md   # one rendered meeting engram (summary +
│                                    #   structured frontmatter), cites raw/
│
└── .gitattributes / .gitignore
```

**Why two namespaces.** `raw/` is the immutable substrate — every transcript, every future Slack/email/voice-memo capture lands here, never modified after creation, only ever read. `cortex/` is the LLM-owned wiki layer — engrams are rendered narratives that cite back into `raw/`. The split future-proofs for non-meeting input types (`raw/email/`, `raw/slack/`, `raw/voice-memo/`) without touching the engram layer.

**Why three engram kinds in cortex/.**

| Kind | What it tracks | Examples |
|---|---|---|
| **person** | someone you talk to | Mrs. Chen · Alex Chen · opposing counsel |
| **topic** | a thing tracked over time | Henderson v Acme · Q3 pricing · diabetes management |
| **meeting** | one discrete event (rendered summary + frontmatter; cites raw/meeting/<id>/) | 2026-05-04 Q3 pricing call · Mrs. Chen's March 14 visit |

The meeting *transcript* lives in `raw/`, never in `cortex/`. The meeting *engram* in `cortex/meeting/<id>.md` is a structured summary with `decisions[]`, `action_items[]`, `claims[]`, `attendees[]` in frontmatter and a Summary body section. It's a rendered view; if it's lost, regenerate from `raw/meeting/<id>/transcript.md`.

**No system files** (`active.md`, `patterns.md`, `commitments.md` from earlier drafts are gone). They were not engrams; they were *views* the user wanted. The hippocampus computes them on demand:

| View | What it is | Computed from |
|---|---|---|
| **Recent** (working memory) | engrams touched in last 14 days | filter `cortex/` by mtime |
| **Pending** | open action items / unsettled decisions | aggregate from `action_items[]` across meeting engrams |
| **Recurring** (schemas) | themes the consolidation pass detects | RAPTOR rollups + theme entries the lint pass surfaces |

**No `.agent-memory/` sandbox dir.** That existed in earlier iterations to host the Anthropic memory-tool format; with no MCP and no memory tool, the agent reads/writes through the Brain API directly.

### 4.1.1 The three navigation files (Karpathy convention)

**`cortex/index.md`** — auto-generated by `lint()`. The catalog of all engrams: name, kind, one-line `description` from frontmatter, last-seen, age-band. The LLM reads this on every recall to orient itself ("what does this user have engrams about?"). The user can scan it as the table-of-contents view of the brain.

**`cortex/log.md`** — chronological, append-only. One line per consolidation event: `2026-05-04 14:42 · meeting/2026-05-04-q3-pricing.md created · person/alex.md updated (role_line) · 2 proposed updates queued`. The "what happened to my brain" timeline. Geomi reads recent entries to compose the end-of-day nudge.

**`cortex/SCHEMA.md`** — user-editable markdown contract describing engram frontmatter. The LLM reads it before every encoding/consolidation pass to know what fields are valid, what their semantics are, what's required vs optional. Pulled out of Rust code into a markdown document so it's self-describing and the user can extend it (add a custom field, document a new engram kind) without touching code. Schema changes propagate via a `migration` commit that re-validates the corpus.

### 4.2 git as the audit substrate

- **Backend:** `git2-rs` (libgit2 bindings). Embedded, no subprocess fork/exec, native to Rust/Tauri.
- **Init:** `git init` on first run; configure local user (default `Docknote <user@local>`); **no remote, ever** — backup is handled outside this design layer.
- **Commit identities:**
  - `auto-extractor` — meeting-close commits
  - `auto-consolidator` — nightly 03:00 commits
  - `<user>` — human edits (the user's own git identity)
- **Commit shape (transactional):** one commit per consolidation pass, touching N files atomically. Never per-field. Message format below.
- **No `.git` exposure in UI.** All "history of this engram", "what changed last night", "revert" affordances wrap git ops; user never sees git CLI vocabulary.

#### Commit message format

```
<type>: <one-line summary>

meeting:    <meeting_id> · <YYYY-MM-DD HH:MM> · <participants>
touched:    person/alex.md person/bob.md topic/q3-pricing.md meeting/2026-05-04-q3-pricing.md
extracted:  decisions(2) action_items(3) themes(0)
model:      qwen-3-32b @ snapgpu/spark1
prompt:     <prompt-hash>
```

Types: `meeting-close`, `nightly-consolidation`, `user-edit`, `migration`. Machine-parseable header lines below the summary so the UI can render rich diffs without re-running the analysis.

#### Sync (out of scope)

Multi-device sync via user-managed git remote was previously scoped as a Phase-3 stretch. **Dropped from v9.** Backup is handled outside this design layer (Docknote-managed). The brain stays single-device; git remains useful as the local audit substrate independent of any remote.

### 4.3 Hippocampus indexes — pointers into the cortex, not parallel data

Stored at `~/Library/Application Support/Docknote/brain.db`. **All indexes are derived views over the markdown corpus, regenerable from disk at any time, never the source of truth.** The cortex (markdown) is the only durable store.

```
sentence_index            # sentence-level embeddings — the galaxy
                          # (sqlite-vec; high-dimensional semantic space)
                          # cols: meeting_id, span_start, span_end, text, vector

meeting_fts               # FTS5 over meeting markdown bodies (keyword fallback)

entity_index              # name → engram path resolution
                          # cols: id, kind, primary_name, aliases[], email,
                          #       calendar_id, zoom_id, engram_path

raptor_nodes              # RAPTOR forest — hierarchical constellations over the galaxy
                          # cols: id, parent_id, level, text, vector,
                          #       source_meeting_ids[], scope (meeting/week/quarter)

proposed_updates          # auto-extracted changes pending user review (Brain Inbox)
                          # cols: engram_path, field, before, after, sources[],
                          #       model, created_at

# Optional / lazy: derived indexes over meeting frontmatter (materialized
# views, not a separate truth store). Add only if specific query types
# prove slow at scale:
#   action_items_index    cols: meeting_id, assignee, task, due, status
#   decisions_index       cols: meeting_id, text, topic
```

**No `facts` table.** An earlier draft borrowed Graphiti's `(subject, predicate, object, valid_at, invalid_at, episode_id)` schema as a structured-claims store. **Dropped** — see §15 decision log for full reasoning. The brain isn't tabular; structured claims accumulating in a SQL table over years of meetings is the wrong shape. Atomic claims live where they were born — in the meeting frontmatter (`decisions[]`, `action_items[]`) — and the LLM reconstructs from raw context at recall time.

### 4.4 Engram frontmatter (with per-field provenance)

Per-field provenance lives in the markdown frontmatter so git diffs are field-meaningful without a sidecar.

```yaml
---
kind: person                # person | topic | meeting | self
slug: alex-chen
canonical_name: "Alex Chen"
aliases: ["Alex", "alex@acme.com", "Alex C."]
calendar_id: "alex.chen@acme.com"

role_line:
  value: "VP Eng at Acme · pushes timeline back, holds line on quality"
  source: { meeting_id: "2026-05-04-q3-pricing", span: "00:12:34-00:13:02" }
  model: "qwen-3-32b"
  authored_by: "auto-consolidator"
  authored_at: "2026-05-04T15:30:21Z"
  human_locked: false

day_headline:
  value: "Q3 Pricing Discussion"
  source: { meeting_id: "2026-05-04-q3-pricing", span: "—" }
  authored_by: "auto-extractor"
  authored_at: "2026-05-04T14:42:00Z"
  human_locked: false

stat_line:
  value: "1 decision · 2 open questions"
  source: { meeting_id: "2026-05-04-q3-pricing" }
  authored_by: "auto-extractor"
  authored_at: "2026-05-04T14:42:00Z"
  human_locked: false

differentiation_bar:
  value: "the Eng counterweight on the leadership team"
  authored_by: "user"
  authored_at: "2026-04-12T09:15:00Z"
  human_locked: true            # <— consolidation pass cannot rewrite

age_band: current   # current | recent | background | archived
last_seen: "2026-05-04"
---

## Background
... (free prose, body sections)

## Recent threads
- 2026-05-04 — Q3 pricing: argued for monthly billing experiment with 10 advisors.
- 2026-04-29 — All-hands: announced Q4 hiring freeze.

## Open questions
- Does Alex sign off on the timeline-vs-quality tradeoff in OKR planning?
```

### 4.5 The "human edits always win" rule

- Any field with `human_locked: true` is **immutable to auto-consolidation**. The pass may queue a `proposed_updates` row but cannot rewrite the field.
- Human edits set `human_locked: true` automatically and bump `authored_by: user`.
- **Unlock via the Brain Inbox** — accepting a proposed update for a locked field unlocks it (auto-pass authoritative again) until the next human edit.
- The consolidation pass MUST query frontmatter `human_locked` before any field write. Empty `sources` or violated lock = commit rejected at validation.

---

## 5. Hippocampus — encoding · indexing · recall (the Brain API)

The hippocampus is everything between the cortex (engrams on disk) and the rest of Docknote: the SQLite indexes, the bi-temporal facts table, the RAPTOR forest, and the Rust trait that exposes them to Docknote app code. The **only** programmatic surface — no MCP, no HTTP. All UI surfaces and Geomi triggers go through this.

```rust
trait Hippocampus {
    // ---- Recall (reads) ----
    fn read_engram(&self, path: &EngramPath) -> Result<Engram>;
    fn read_meeting(&self, id: &MeetingId) -> Result<Meeting>;
    fn search_sentences(&self, q: &Query) -> Result<Vec<SentenceHit>>;
    fn search_meetings(&self, q: &MeetingQuery) -> Result<Vec<MeetingMatch>>;
    fn entity_resolve(&self, name_or_alias: &str) -> Result<Vec<Entity>>;
    fn facts_about(&self, subject: &EntityId, at: Option<DateTime>) -> Result<Vec<Fact>>;

    // ---- Synthesis (the v8-novel work) ----
    fn synthesize_themes(&self, q: &SynthesisQuery) -> Result<ThemeSet>;
    fn pivot_route(&self, q: &str) -> Result<PivotKind>;       // name | date | concept

    // ---- Encoding & consolidation (writes, transactional, git-committed) ----
    fn open_encoding(&self) -> ConsolidationTxn;               // per-meeting close
    fn open_consolidation(&self) -> ConsolidationTxn;          // nightly pass
    fn user_edit(&self, edit: &FieldEdit) -> Result<CommitId>;
    fn accept_proposed_update(&self, p: &ProposedUpdateId) -> Result<CommitId>;
    fn reject_proposed_update(&self, p: &ProposedUpdateId) -> Result<()>;

    // ---- History / time-travel ----
    fn history(&self, path: &EngramPath) -> Result<Vec<CommitRef>>;
    fn diff(&self, a: &CommitId, b: &CommitId, path: &EngramPath) -> Result<FieldDiff>;
    fn read_engram_at(&self, path: &EngramPath, at: &CommitId) -> Result<Engram>;
    fn revert_field(&self, path: &EngramPath, field: &str) -> Result<CommitId>;

    // ---- Lint: health, orphans, contradictions, gaps ----
    /// Karpathy's third operation. Runs as part of nightly consolidation
    /// AND on demand. Outputs a structured health report consumed by:
    /// (a) the lint pass that regenerates cortex/index.md and cortex/log.md,
    /// (b) Geomi's end-of-day nudge ("3 contradictions need your attention"),
    /// (c) the Brain Inbox surface (orphan citations, stale claims).
    fn lint(&self, scope: LintScope) -> Result<LintReport>;
    fn rebuild_index_md(&self) -> Result<CommitId>;
    fn append_log_md(&self, entry: LogEntry) -> Result<()>;
}

enum LintScope { Full, SinceLastLint, Engram(EngramPath) }

struct LintReport {
    orphan_engrams:        Vec<EngramPath>,    // referenced but missing
    dangling_engrams:      Vec<EngramPath>,    // present but never referenced
    broken_citations:      Vec<BrokenCitation>, // sources point to missing meeting
    stale_claims:          Vec<StaleClaim>,    // age_band: current with no recent corroboration
    contradictions:        Vec<Contradiction>, // same field, conflicting authoritative claims
    schema_violations:     Vec<SchemaViolation>, // frontmatter doesn't match cortex/SCHEMA.md
    gaps:                  Vec<Gap>,            // entity in attendees but no person engram
    age_band_transitions:  Vec<AgeBandChange>,  // suggested current → recent → background
}

trait ConsolidationTxn {
    fn update_engram_field(&mut self, path: &EngramPath, field: &str,
                           new_value: Value, sources: Vec<Source>) -> Result<()>;
    fn demote_field(&mut self, path: &EngramPath, field: &str,
                    to_section: &str) -> Result<()>;
    fn reconcile_flag(&mut self, path: &EngramPath, field: &str,
                      reason: &str) -> Result<()>;
    fn add_proposed_update(&mut self, p: ProposedUpdate) -> Result<()>;
    fn commit(self, message: CommitMessage) -> Result<CommitId>;       // atomic
    fn abort(self) -> Result<()>;
}
```

### Validation contracts (enforced by the impl, not the caller)

- Every `update_engram_field` MUST have `sources.len() ≥ 1`. Empty sources → commit rejected.
- Every `update_engram_field` MUST check `human_locked` on the target field. Locked → divert to `add_proposed_update`.
- A `ConsolidationTxn` must `commit` or `abort` — drop = abort. Partial state never escapes.
- `user_edit` always succeeds (the user authored it; lock semantics don't apply to the user).

---

## 6. Triggers — the proactive surface for Geomi (and the confirmation mediator)

Geomi has two roles in the brain layer: the proactive "ghost genius" bubbles (pre-meeting / mid-meeting / end-of-day) AND **the confirmation mediator** for LLM-proposed engram updates. The second role is the v8 answer to Karpathy's "the LLM owns the wiki" stance — for our user (doctor / lawyer / salesperson), wiki authority lives with the human, surfaced through Geomi as confirmation prompts.

```rust
trait Triggers {
    /// Pre-meeting brief: 30 minutes before a calendar event,
    /// surface the relevant person/topic engrams + last-time-with-them digest.
    fn pre_meeting_brief(&self, ev: &CalendarEvent) -> Result<Brief>;

    /// Mid-meeting nudge: given a streaming transcript chunk and the meeting
    /// context, decide whether a bubble is warranted.
    /// Returns Suggestion::None most of the time (REQ-075's "ghost genius —
    /// never begs"); rate-limited to ≤ 1 bubble per stage of the meeting.
    fn mid_meeting_suggest(&self, ctx: &MeetingCtx) -> Result<Option<Suggestion>>;

    /// End-of-day nudge: summarize what happened today, surface unsettled
    /// commitments, propose tomorrow-prep items, surface lint contradictions.
    fn end_of_day(&self) -> Result<EveningNudge>;

    /// Confirmation surface for LLM-proposed engram updates.
    /// Pulls from the proposed_updates queue, formats as a conversational
    /// prompt, and returns the user's verdict.
    /// Example bubble:
    ///   "After today's meeting with Alex, I'd update his role line:
    ///      was: VP Eng at Acme · pushes timeline back, holds line on quality
    ///      now: VP Eng at Acme · willing to negotiate timeline if quality holds
    ///    From the 12:34 mark of today's call. Confirm?"
    /// User says yes/no/modify; Geomi commits with the user as authority.
    fn next_confirmation(&self) -> Result<Option<ConfirmationPrompt>>;
    fn record_verdict(&self, p: &ProposedUpdateId, v: Verdict) -> Result<CommitId>;
}

enum Verdict {
    Confirm,                    // accept the proposed value as-is, mark human-confirmed
    Reject,                     // drop the proposed update; field stays as it was
    Modify(Value),              // user-rewritten value, mark human-edited
    Defer,                      // ask again later (stays in queue)
}
```

**Implementation notes:**
- `pre_meeting_brief` is largely recall: pull person engrams for attendees, last 2-3 meeting engrams, open commitments. One LLM call to compose the natural-language brief.
- `mid_meeting_suggest` is the trickiest — it's a *negative-default* surface (silence is the right answer most of the time). Heuristics: speaker mentions an entity with an active commitment, contradiction with a prior decision, vocabulary unknown. Rate-limited to one suggestion per "stage" (intro / discussion / wrap), suppressed for the rest of the meeting once dismissed.
- `end_of_day` is a nightly-pass byproduct — runs after consolidation + lint so themes and contradictions are fresh.
- `next_confirmation` / `record_verdict` are the **Karpathy-aligned authority loop**: the LLM proposes (consolidation pass writes to `proposed_updates`), Geomi mediates (formats one proposal as a conversational prompt), the user decides (verdict committed with their authority). Confirmed memories carry weight in future synthesis — a `human_confirmed: true` field outranks an unconfirmed auto-extracted claim during recall ranking.

REQ-075's "ghost genius — never begs" rule applies to all three proactive triggers AND to the confirmation surface: Geomi never spams confirmations. Rate-limit ≤ 3 confirmation prompts per session; defer the rest into the Brain Inbox for batch review.

**The shift from earlier drafts.** Earlier v8 said "human edits always win" — which assumed the user types into engram fields. Karpathy's gist instead has the LLM owning the wiki entirely. v8's revised stance: **the user rarely types into engrams; they confirm or modify Geomi's proposals.** Direct file editing is still possible (it's markdown on disk) but it's the power-user escape hatch, not the primary workflow. The audit trail is now mostly a record of *human verdicts* on LLM proposals, not raw human keystrokes — closer to a chart-review queue than a text editor.

---

## 7. Encoding & consolidation (the write path)

Two cadences: **encoding** at meeting close (creating new engrams from raw experience), and **consolidation** nightly (strengthening engrams, integrating across the week).

### 7.1 Encoding — per-meeting close

Trigger: meeting recording completes + transcription finishes.

```
1. Save raw transcript → raw/meeting/<YYYY-MM-DD>-<slug>/transcript.md
   (Verbatim, with [SPEAKER_n M:SS] markers; immutable thereafter.
    Audio.opus + metadata.json land in the same dir.)

2. Sentence-level embeddings → sentence_index
   (Local embedding model; sentence boundaries via punkt or equivalent;
    each row carries raw_meeting_id + span_start/end)

3. Topic-segment extraction (one bounded LLM call per ~3-min segment):
   Input:  segment transcript + canonical entity list (substring-resolved)
   Output: structured frontmatter + Summary body for the meeting engram:
             { decisions[], action_items[], claims[], attendees[],
               people_touched[], themes[] }
           each with { span, confidence } citing the raw transcript.
   Written into:  cortex/meeting/<date-slug>.md
                  (the ONE source of truth for atomic claims born in this
                  meeting; cites raw/meeting/<id>/transcript.md for the
                  underlying utterances).
   Model:  Qwen-3-32B via SnapGPU (default) | Claude (opt-in)
   Cost:   ~20-30 LLM calls per 1-hour meeting. Bounded.

   No separate reconciliation against a facts table — there is no facts
   table. Every claim lives in its birth meeting engram. Reconciliation
   across meetings happens later, at consolidation time, where the
   consolidator reads many meeting engrams and writes coherent narratives
   into person/topic engrams.

4. Per-engram incremental consolidation (Letta sleep-time pattern,
   field-level + Geomi-mediated approval):
   For each touched person/topic engram:
     - txn = hippocampus.open_encoding()
     - sleep-time prompt reads: engram_current + this meeting's
       frontmatter + raw transcript chunks + recent meeting engrams
       involving this entity (retrieved via sentence_index)
     - For each field change:
         · If field is auto-derivable (no prior human verdict): emit
           `update_engram_field` directly. Audit-logged.
         · If field has a prior human-confirmed value OR the change is
           consequential (role_line, differentiation_bar, etc.): emit
           `add_proposed_update` and queue for Geomi confirmation.
           See §6 — Geomi formats it as a conversational prompt; user
           verdict commits the change.
     - txn.commit("meeting-close: ...")

   The engram body is a *rendered narrative*, derived from the underlying
   markdown corpus on demand. Re-runnable. Not a parallel data store.

5. RAPTOR build for the meeting:
   Chunks(100tok) → UMAP+GMM soft cluster → GPT-4o-mini-class summary at
   each level → 3-level tree per meeting.
   Append to raptor_nodes; weekly rollup cluster updated incrementally.
```

Wall-clock target: < 60s for a 30-minute meeting on M3 Pro + spark1 SnapGPU.

### 7.2 Consolidation — nightly 03:00 batch

Trigger: cron-style at 03:00 local. No-op if any encoding pass is in flight (advisory lock).

```
1. Cross-engram reconciliation:
   For each engram touched today:
     - Read this week's meeting files involving this entity
       (via sentence_index + entity_index).
     - Re-derive the engram body and dossier-card frontmatter
       from the actual corpus, not from a separate facts store.
     - Detect contradictions across meetings (LLM reads in date order);
       reconcile_flag for fields with unresolvable conflict.

2. Age-banding:
   Any claim in an engram body older than 90 days with no corroborating
   reference in last 30 days demoted to body section "Background."
   age_band: current → recent → background → archived (transitions).
   The full text never leaves the engram; it just moves to a less-prominent
   section. Brain-style decay, not deletion.

3. Theme rollup (schemas):
   RAPTOR weekly rollup: re-cluster meeting summaries from this week.
   Quarterly rollup: re-cluster weekly rollups (only if quarter boundary).
   The Recurring view reads these.

4. Theme dedup:
   For each new theme: cosine ≥ 0.92 vs existing → LLM reconcile → merge.

5. Lint pass (Karpathy's third operation):
   hippocampus.lint(LintScope::SinceLastLint) → LintReport
     - orphan / dangling engrams flagged
     - broken citations (sources pointing to missing raw/)
     - stale claims (age_band: current with no corroboration in 30d)
     - contradictions (queue as proposed_updates for Geomi)
     - schema violations against cortex/SCHEMA.md
     - gaps (entity in attendees but no person engram → suggest creation)
   Then:
     hippocampus.rebuild_index_md()      # regenerates cortex/index.md
     hippocampus.append_log_md(today)    # one line per consolidation event

6. End-of-day nudge generation (consumed by Geomi):
   Surface unsettled commitments (from open action_items in meeting
   frontmatter), theme deltas, contradictions from lint(), prep suggestions.

7. Single git commit: "nightly-consolidation: ..."
```

Wall-clock target: < 5 min on Qwen-3-32B via SnapGPU; runs while user sleeps so latency is permissive.

---

## 8. Recall (the read path)

Five stages. Each can short-circuit if the previous gives a confident answer. Recall reconstructs from cortical engrams via the hippocampus indexes — same shape as biological recall. The substrate is the sentence-level vector space (the galaxy); RAPTOR imposes hierarchical structure (constellations); the LLM weaves retrieved chunks into a coherent answer at query time.

### Stage 0 — working memory (always-loaded, zero retrieval cost)

`persona.md` + the Recent view (engrams touched in last 14 days, top-N) injected into every assistant turn's system prompt. This is working memory in Baddeley's sense — the small buffer always at hand. Combined cap: 4K tokens. Truncated by section, not silently — warn if exceeded.

### Stage 0.5 — pivot routing (optional, often skippable)

`pivot_route(q)` classifies the query into `name | date | concept`. Tiny LLM call (~50 tokens), or skip for queries with obvious entity matches.

### Stage 1 — deterministic coarse filter (sub-ms)

- **Entity scoping:** substring + normalized-name match against `entity_index.aliases` (incl. email, calendar_id, zoom_id) → candidate engram path(s).
- **Date scoping:** parse "March 2026" / "last week" / "yesterday" → date range, filter meeting files by frontmatter `date`.
- **Scope filter** (workspace / project) if specified.
- **Output:** candidate set of ≤ 50 engrams (mix of person/topic engrams + meeting files).

No facts-table query — temporal scoping uses meeting frontmatter, entity scoping uses aliases. If the query is unambiguous (single entity match, recent date), stage 1 may go directly to stage 4.

### Stage 2 — LLM-over-manifest (≤ 1 LLM call)

- Format manifest from candidate set: filename + description (one-line) + last-modified.
- Sonnet-class model picks ≤ 5 most-relevant engrams.
- This is the "Claude Code memdir pattern" applied to v8.

### Stage 3 — RAPTOR collapsed-tree retrieval (no LLM call)

- ANN over flattened forest restricted to the chosen engrams.
- Top-20 nodes within a 2K-token budget.
- Returns mix of leaf chunks + meeting summaries + week summaries based on what cosine prefers — that's the point of collapsed-tree.

### Stage 3.5 — recognition-memory filter (deferable to Phase 4)

HippoRAG 2's pattern: an LLM call that prunes irrelevant retrieved nodes. Adds ~300ms + a tiny model call for precision. **Defer to Phase 4 unless Phase-3 eval shows precision is the binding constraint.**

### Stage 4 — synthesis prompt (1 LLM call)

For non-factoid queries, a final prompt over the retrieved nodes:

```
Given these retrieved engram fragments (each with {meeting_id, span, text}),
extract the 3-4 most important themes relevant to the user's question.
For each theme:
  - name: short label
  - description: 1-2 sentences
  - citations: list of meeting_ids that evidence this theme

Output strict JSON.
```

This is the v8-differentiating step; none of the prior art does it well. Quality of this prompt + RAPTOR tree quality dominate the persona-corpus score.

---

## 9. Memory UX

The Memory surface in Docknote has two layers: the **landing** (how the user sees and navigates the corpus of meeting notes) and the **edit path** (how a single engram is read, edited, and audited). The landing is the v8 face most users touch every day; the edit path is the precision instrument behind it.

### 9.1 Memory landing — surface and lenses

When the user clicks **Memory** in Docknote's left sidebar, the main pane is a **galaxy view** of every meeting note in the corpus. The user-facing unit is `note` (architecture/code retains `engram`). A reference visual exists at `~/seanslab/HiDockSkill/src/galaxyHtml.ts` (D3 force layout, cosmic palette); Docknote uses the same shape in its own warm-paper palette (no purple). **Locked landing mockup:** `~/seanslab/Research/cc-learn/rachel-redesign/v4.3-memory-galaxy-floating-ask.html`. Earlier iterations (v4 galaxy, v4.1 horizontal year ribbon, v4.2 dedicated ask tray) are kept for design lineage but superseded.

The pane has three regions: **galaxy** (center, fills), **time column** (right, ~170px, always visible), and the **floating ask** (pill, centered, absolute-positioned over the galaxy). No bottom ribbon. No companion strip. No dedicated ask tray.

**The galaxy.** Each note is a dot. Notes that share a cluster (by series / project / attendee / sameday) are positioned close to each other; cluster names appear as small-caps italic-serif labels at constellation edges, with a faint cluster-color halo behind each group. **Opacity encodes recency** (older notes dim, recent notes bright); a faint vellum ring around a dot signals unconfirmed marginalia (Geomi waiting for the user to confirm a proposed update). Hover a dot → small note card (date, meeting title, two-line summary, attendees). **No file paths, no field labels, no cosine numbers, no model names** anywhere on the surface (principle 1).

**The time column (right side).** Always visible, no flyout, no dropdown. A grouped list of time scopes the user can think in directly:

- **recent** — *this week · last week · this month · last month*
- **months · {current year}** — a 4-column grid of month abbreviations (jan ... dec); months past "today" render at lower opacity since they haven't happened.
- **quarters** — Q1 · Q2 · Q3 · Q4
- **years** — descending, 3–5 visible (only years with notes)
- **all time** — bottom row, the release affordance

Each row is small italic-serif. Active row gets a brand-teal pill (`--brand-soft` fill, `--brand-strong` text); others stay quiet (`--ink-soft`). Tiny sparklines (1px-tall density bars) appear beside *recent* and *years* rows showing meeting density for that period; the months grid and quarters omit sparklines (too compact to render without becoming chrome). The column replaces the v4.1 horizontal year ribbon entirely; the column scales to deep history (years, quarters) without nesting and uses natural human time vocabulary throughout.

**Spotlight — the only landing gesture on time.** Click any time-column row → that scope becomes active. The galaxy spotlights notes in that scope (full opacity); other notes fade to ~0.15. **Cluster halos and labels stay put — the galaxy does not reorganize.** A breadcrumb appears at the top of the galaxy column: "*spotlight · february · 23 meetings*" with a small × close pill. **No companion meeting strip.** The galaxy is the answer; a list under it would be redundant chrome (principle 1). Click the same row again, click *all time*, click the breadcrumb's ×, or press ESC → release. Active scope is mutually exclusive (clicking Q3 deactivates whatever else was active).

**Drag-pin rewind is dropped from the landing.** Time-travel ("show me what I knew before the leadership offsite") moves to per-engram view as a reading-only history slider — see §9.5. The Memory landing carries one temporal gesture, not two. This simplifies the surface and matches the v8 user's actual landing-time intent (recall *what I have*, not *what I had*).

**The floating ask.** A pill-shaped bar (`border-radius: 999px`, blur backdrop, soft shadow) floats over the galaxy at the bottom — `position: absolute; bottom: 18px; left: 50%; transform: translateX(-50%);` — centered within the galaxy column (does not extend under the time column). Pattern verbatim from `~/seanslab/Docknote/ux-mockups/bottom-bar.html` so Memory feels of-a-piece with Notes. Translucent panel (`rgba(255,255,255,0.88)` light / `rgba(35,37,36,0.78)` dark) lets the warm-paper backdrop and active spotlights glow through. The bar carries: a small breathing Geomi dot, an italic-serif placeholder ("*who, when, what do you want to remember?*"), a blinking caret, and a small send affordance. **This is the only ask field on the surface** — there is no second ask field elsewhere.

**The people lens — first-class to Phase 3.** People are a sibling lens to the galaxy, not a tab nested inside it. The user opens a person by name (via the floating ask, or via a `people` rail/toggle — composition TBD by Rachel in v5). A person view answers **four queries directly, as Geomi-written prose with marginalia awaiting initials**:

- **Who said what** — quoted lines attributed to this person across meetings, with citations to source meeting + timestamp span.
- **Whose progress** — trajectory: how this person's situation, role, project status, or condition has moved over time.
- **Who needs care** — outstanding follow-ups, pending commitments, things this person flagged that haven't been resolved. The doctor's recall list; the salesperson's chase list.
- **Who committed to what** — action items and decisions where this person is the owner / authority, with the originating meeting and quote.

These are **not** a profile with tabs. They are four short prose sections on one page, generated from the engram store via the existing recall pipeline (§8 Stage 0–4). Marginalia (Geomi-proposed updates not yet confirmed) appear inline with a *Yes, remember that* / *Not yet* affordance — the same vocabulary as the Brain Inbox (§9.4), but contextualized to the person.

**The killer-move combination — `time-scope × person-pin`.** With a person pinned (active in the lens), clicking any scope row in the time column collapses to a single-click attribution query: *"show me everything Alex said in February"* (click `feb`), *"...in Q3"* (click `Q3`), *"...this week"* (click *this week*). The galaxy spotlights only this person's notes within that scope; the four-query view filters to that scope's slice. This is the load-bearing gesture for the v8 user — the doctor recalling Mrs. Chen's March visits, the salesperson recalling Alex Chen's Q3 commitments, the lawyer recalling opposing counsel's December positions. **Single click resolves a query that prior tools force into multi-step search.** Decision log #15.

**The morning re-read sub-state.** When `proposed_updates` are pending (typically right after the nightly consolidation pass at 03:00), a quiet one-line banner above the galaxy reads "*N marginalia awaiting your initials → review now.*" Clicking it routes into the Brain Inbox (§9.4). The banner is the only chrome the morning ritual gets on the landing; the galaxy itself stays untouched.

**Light + dark.** The surface honors Docknote's existing `--bg / --panel / --rule / --brand` token set with a dark-mode toggle. Cluster colors are color-blind-safe (not yet verified for tritanopia — open question for Rachel v5).

**Open ergonomic question (Rachel v4.3).** When a spotlit cluster lights up dots directly under the floating ask, the pill can occlude what the user just summoned. Two options on the table: (a) bump pill opacity another notch when a scope is active, (b) drift the pill upward 30–40px on spotlight. Currently static at 18px; resolve in v5 or earlier user testing.

### 9.2 Click-to-edit per field

- Engram UI (the Hearth file view) shows each field as click-to-edit text. Frontmatter `value:` is the edited content.
- Save → `user_edit` API call → immediate git commit + frontmatter `human_locked: true`, `authored_by: user`.
- No "open in markdown editor" required; users who want it can still `cd ~/Docknote/brain && vim person/alex.md`.

### 9.3 Provenance badges

Every field shows a small badge:
- `auto-extracted · 2 days ago · qwen-3-32b` (with hover → source meeting + span)
- `you edited · 22 Apr` (with hover → revision history)

Click badge → side drawer with full revision history (5-deep visible, expand for full git log).

### 9.4 Brain Inbox

A dedicated UI surface listing all `proposed_updates` since the last review.

```
Brain Inbox · 7 proposed updates from last night

[ person/alex.md · role_line ]
  Was: VP Eng at Acme · pushes timeline back, holds line on quality
  Now: VP Eng at Acme · willing to negotiate timeline if quality holds
  Why: 2026-05-04 Q3 Pricing meeting — Alex agreed to monthly experiment
       (12:34-13:02)
  [ Accept ] [ Reject ] [ Open engram ]

[ project/q3-pricing.md · status ]
  ...
```

Per-field accept/reject. Accept → applies the update + unlocks the field. Reject → drops the proposed_update; no commit.

### 9.5 Diff & history

- Per-field history rendered as side-by-side or sentence-level highlight (strikethrough removed, underline added).
- "Time travel": each engram has a date slider; drag back to see the engram's frontmatter at that moment (`read_engram_at`).
- One-click revert: `revert_field` writes a new commit setting the field to a prior value.

---

## 10. Privacy & security

### 10.1 Defaults

- **No remote.** git repo is local-only. Backup is handled outside this design layer (Docknote-managed). The Brain API does not expose `git remote add` to the rest of the app.
- **No cloud LLM** for synthesis by default. SnapGPU on spark1+spark2 over Tailscale.
- **Audio not retained** (matches Granola's bar). Transcripts kept; user can request per-meeting deletion.
- **No telemetry on memory contents.** App-level usage stats may exist (per Phase-2 OI-120 decision); brain content is never reported.

### 10.2 Path traversal

- All `EngramPath` values validated: must start with one of `{persona.md, person/, topic/, meeting/}`; canonicalized; no `..`, no UNC, no symlink-out-of-repo.
- Implementation pattern from Claude Code's `src/memdir/paths.ts:109` — port to Rust.

### 10.3 Sensitive-content filter

- Per-meeting, the user can mark a meeting `sensitive: true` in its frontmatter. Sensitive meetings:
  - Excluded from synthesis prompts that route to a non-local LLM (i.e. Claude opt-in mode).
  - Excluded from the Brain Inbox preview if they would expose the sensitive content in plaintext (badge replaces text with "[sensitive]"; user clicks through to view).
- Default: not sensitive. Marking is the user's call.

### 10.4 Encryption-at-rest (Phase 4)

- v8 ships unencrypted on disk; relies on FileVault for full-disk encryption.
- Phase-4 plan: `git-crypt` over the brain repo, or filesystem-level (separate APFS volume with its own key).
- Per-field encryption (subset of frontmatter fields) is rejected as overcomplicated; whole-repo encryption is the right grain.

---

## 11. Phase boundaries

### In v8 (Phase 3 ship)

- **Cortex** (markdown engrams + git) — full
- **Hippocampus** (SQLite indexes + Brain API trait) — full
- **Triggers** (Geomi proactive surface) — full
- **UX** (Brain scene with Tide / Hearth / Murmur · Brain Inbox · Geomi bubbles) — full
- Acceptance against 24-persona corpus at ≥80%

### Deferred

- **MCP server.** Reconsider post-launch if a real demand emerges from external-LLM users.
- **Anthropic memory-tool wire format.** Same gate.
- **Encryption-at-rest** (OI-001) — Phase 4.
- **Multi-user shared brain** — post-v8.
- **Cross-source ingestion** (Slack / email / docs) — post-v8.
- **HippoRAG 2 recognition-filter as re-ranker** — graft in only if Phase-3 eval shows precision is the binding constraint.
- **MIRIX six-type taxonomy** — current four-pivot taxonomy (name/date/concept + facts) is sufficient; expand only if eval surfaces a query class we can't serve.

---

## 12. Acceptance criteria

### 12.1 Persona corpus (REQ-070)

- **Bar:** ≥ 58 / 72 queries pass all three sub-checks (≥ 80%).
- **Sub-checks per query:**
  1. `pivot_type` correctly classified (name / date / concept).
  2. `expected_dossier_card` fields produced (`role_line` / `day_headline` / `stat_line`) — string-match against the corpus's expected values, semantic-equivalent OK.
  3. `expected_synthesis_themes` (3-4 themes) all appear in the synthesis output.
- **Eval harness:** runs against seeded `tests/eval-brain-memory/seed-data/` corpus, scores each query, produces a `tasks/eval-results-YYYY-MM-DD.md`.

### 12.2 Performance

- Per-meeting consolidation wall-clock: < 60s for a 30-minute meeting.
- Nightly batch: < 5 min on Qwen-3-32B via SnapGPU.
- Read path P95 (cold cache): < 2s for "what did Alex say about pricing in March."

### 12.3 Privacy verification

- Outbound network audit: per-meeting close + read path generates **zero outbound packets** to non-tailnet hosts in default config (verified via `pcap` capture during eval run).
- Git audit completeness: every engram field's current value is reachable via `read_engram_at(path, head)` AND every prior value is reachable via the field's history.

---

## 13. Eval design

### 13.1 Persona corpus run

- Stand up the 24-persona seeded corpus (`tests/eval-brain-memory/seed-data/`) in a clean brain repo.
- Run all 72 queries through the read path.
- Score per the three sub-checks above.
- Hand-review failed queries to categorize: stage-1 miss / stage-2 miss / stage-3 miss / synthesis miss / spurious extraction.

### 13.2 Synthesis-quality probe (separate from acceptance)

- Sample 20 queries from real (anonymized, opt-in) user meetings.
- Hand-author "ground truth" themes per query.
- Score synthesis output against ground truth via LLM-as-judge + human spot-check.
- This is the pre-launch confidence check — acceptance corpus is necessary but synthetic.

### 13.3 Regression suite

- Every commit to v8 code re-runs persona corpus + synthesis-quality probe.
- Score deltas trigger a CI gate (block PR if persona corpus drops > 2pp).

### 13.4 Eval risk

The dominant variable is **Qwen-3-32B's synthesis quality** vs the GPT-4o baseline most papers benchmark on. If the persona corpus comes in below 80% on Qwen, options:
1. Tune the synthesis prompt (cheapest).
2. Add the HippoRAG 2 recognition-filter as a stage-3.5 re-rank (medium cost).
3. Allow opt-in Claude-Sonnet routing for the synthesis pass (privacy-tradeoff for users who want it).
4. Defer v8 ship and step up to Qwen-3-72B (last resort; not a single-spark workload).

---

## 14. Open questions (route to SDD review)

1. **Sentence boundary detector** — punkt vs spaCy vs language-aware (CJK queries flagged in REQ-068 deferred to Phase 3). Decision needed.
2. **Embedding model** — local-only requirement narrows it. Candidates: `bge-m3` quantized via Metal, or whisper.cpp's audio embeddings repurposed. Bench needed.
3. **Topic-segmentation algorithm** — fixed 3-min windows vs LLM-decided breaks vs Bayesian-surprise (EM-LLM style). Default fixed-window for v8 simplicity; revisit.
4. **Brain Inbox cadence.** Notify daily or weekly? Rachel's draft is *weekly digest with a daily-updating faint count badge* — needs one user-research conversation to confirm.
5. **Working memory (Recent view) size cap and rotation policy.** Currently "rolling 14 days." May need hard token cap (4K?) with priority weighting (recent meetings + open commitments + active people).
6. **Calendar integration scope.** Phase 3 needs minimum: pre-meeting brief trigger via calendar event lookup. Scope of "calendar integration" beyond that is out.
7. **Sensitive-meeting marking UX.** Pre-meeting toggle, post-meeting flag, per-line redaction? Default: pre-meeting toggle with persistent annotation; per-line redaction is a later enhancement.
8. **Backup mechanism.** Out of scope here — handled by Docknote's broader backup story (separate design). The brain layer assumes whatever-it-is can read and restore `~/Docknote/brain/` plus the SQLite indexes (or rebuild indexes from markdown post-restore).

---

## 15. Decision log (key calls + reasoning)

| # | Decision | Reasoning | Reversible? |
|---|---|---|---|
| 1 | Markdown source-of-truth, not a DB | Granola's encrypted-DB lockdown is the strategic wedge; openness is the v8 wedge | Yes (with migration) |
| 2 | git as audit substrate, not custom JSON ledger | Stronger primitives (atomicity, history, time-travel) for free; users understand it; Cline validates the pattern | Yes |
| 3 | SQLite/sqlite-vec, not graph DB | Graphiti's Kuzu archived, FalkorDB Lite pre-GA, Neo4j JVM. Single-user Mac doesn't need graph engine. Steal Graphiti's *schema*, skip its engine | Yes |
| 4 | Hybrid extract+raw (mem0 shape over MemMachine substrate) | Pure-extract loses original phrasing; pure-raw lacks typed surfaces. Layered hybrid keeps raw as ground truth, extraction as derived index | Yes |
| 5 | RAPTOR for synthesis pass; skip HippoRAG 2 in v8 | RAPTOR is sense-making-shaped (matches v8 query type); ~20× cheaper to build; tree of `(parent_id, child_id)` rows fits SQLite cleanly | Yes; HippoRAG can graft as Phase-4 re-rank |
| 6 | No MCP server in v8 | Filesystem + git is broader interop than MCP; single-app use case doesn't warrant the protocol overhead. Reconsider post-launch | Yes |
| 7 | No Anthropic memory-tool wire format | Agent runs in-process against SnapGPU/Qwen; no tool-use protocol needed. Same logic as MCP | Yes |
| 8 | Per-field provenance in frontmatter | Makes git diffs field-meaningful without sidecar; supports human-edit lock; supports time-travel UI | Yes (frontmatter migration) |
| 9 | Two consolidator cadences (per-meeting encoding + nightly consolidation), field-level tools, transactional commit | Letta's blob-rewrite pattern + zero-audit-trail is unacceptable for user-facing engrams; ours adds field-level tools, mandatory sources, atomic commits, persistent history | Yes (refine) |
| 10 | Stage 0-4 read path with Stage 3.5 deferred | Five-stage layering keeps cost bounded. Stage 3.5 (recognition-filter) earned only if eval shows precision is binding | Yes; add stages |
| 11 | No git remote, ever (sync dropped) | Backup is handled outside this design layer (Docknote-managed). Brain stays single-device for v9. Removes the entire class of "leak via misconfigured remote" failure modes | Yes (re-add as backup-restore later if needed) |
| 12 | **No `facts` table; reverse the Graphiti bi-temporal borrow** | The brain isn't tabular; structured-claims rows accumulate over years and create fake-precision contradictions where the actual structure is associative and reconstructive. Atomic claims live in their birth meeting frontmatter (`decisions[]`, `action_items[]`, `claims[]`). Cross-meeting truth is reconstructed at recall time by the LLM over retrieved chunks, not pre-computed in a SQL store. The galaxy (sentence-vector space) is primary; RAPTOR is its hierarchy; SQLite holds *indexes only*, never parallel structured truth. This reverses decision #3's "steal Graphiti's *schema*" — we keep `valid_at` discipline only inside meeting frontmatter where it's local and natural. **Independently arrived at the same conclusion as Karpathy's [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) (April 2026)**: markdown SOT, indexes are derived, structured DBs add fake precision. Sibling projects: Memio (`~/seanslab/Memio/`) chose graph memory for its 10-year horizon — defensible at that scale; v8 chose reconstructive recall at single-Mac scale. Both are right at their respective shapes. | Yes (re-add a derived `facts_index` materialized view if specific queries justify it) |
| 13 | **Karpathy three-layer model adopted** (`raw/` + `cortex/` + `cortex/SCHEMA.md`) | `raw/` is the immutable substrate (transcripts, future Slack/email/voice-memo go here). `cortex/` is the LLM-maintained wiki layer (engrams). `cortex/SCHEMA.md` is the user-editable schema contract the LLM reads before every encoding pass. `cortex/index.md` (auto-generated catalog) and `cortex/log.md` (chronological event log) are added as Karpathy's recommended navigation files. v8 had effectively converged on this architecture; we adopt his vocabulary for clarity and cite the gist | Yes (collapse into single namespace if `raw/` proves overkill — but it future-proofs for non-meeting capture) |
| 14 | **Geomi as confirmation mediator** (the v8-Karpathy reconciliation) | Karpathy's gist says "the LLM owns the wiki entirely; you rarely write it yourself." For doctor/lawyer/salesperson users, wiki authority must live with the human — but typing into engram fields is the wrong UX for a meeting-recall tool. Geomi mediates: LLM proposes (consolidation pass writes `proposed_updates`), Geomi formats one as a conversational prompt ("After today's meeting with Alex, I'd update his role line — confirm?"), user gives a verdict (Confirm/Reject/Modify/Defer), commit lands with the user as authority. **Confirmed memories outrank unconfirmed ones in recall ranking.** Direct file editing remains as a power-user escape hatch but is not the primary workflow. | Yes (fall back to direct-edit-only if Geomi confirmation flow proves too noisy in user testing) |
| 15 | **People as a first-class lens on the Memory surface; `time-scope × person-pin` is the load-bearing recall gesture** | The v8 user — doctor, lawyer, salesperson — does not browse memory; they *recall a person*. Person-shaped engrams already exist in the cortex (§4); what was missing was a UI lens that makes them the dominant pivot. The people lens answers four queries directly as Geomi-written prose: *who said what · whose progress · who needs care · who committed to what.* Combined with the time column's spotlight gesture (§9.1), pinning a person and clicking any scope (a month, a quarter, "this week", "last year") resolves cross-meeting attribution to a single click — *"show me everything Alex said in February / in Q3 / this week"* — the gesture prior tools force into multi-step search. This locks people as a sibling lens to galaxy, not a tab nested in it; locks spotlight as the only temporal gesture on the landing (drag-pin rewind moves to per-engram view, §9.5); and drops the companion meeting strip (the galaxy is the answer). Locked landing mockup: `rachel-redesign/v4.3-memory-galaxy-floating-ask.html`. Earlier iterations (`v4`, `v4.1`, `v4.2`) preserved for design lineage. Person-lens mockup is the next deliverable (Rachel v5). | Yes (people-lens composition can iterate; the scope×person combination is the design intent and survives composition changes) |

---

## 16. Risk summary

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Qwen-32B synthesis quality below 80% on persona corpus | Medium | High (delays ship) | Prompt tuning; add Stage 3.5; opt-in Claude routing as escape valve; eval early |
| `human_locked` field bypassed by a bug in consolidation pass | Low | High (silent overwrite of user edits) | Mandatory pre-commit validation; fuzz-test the lock contract; git history is recovery path even if it happens |
| Geomi bubbles too noisy / "begging" — REQ-075 violation | Medium | Medium | Negative-default `mid_meeting_suggest`; rate limiting; suppress-after-dismiss; user-configurable threshold |
| RAPTOR re-cluster cost grows non-linearly with corpus | Low | Medium | Append-only model: new meeting → its own subtree, weekly re-cluster only at rollup level. Bound to cost-of-week, not cost-of-history |
| Embedding model quality low on meeting transcripts (informal speech, CJK names) | Medium | Medium | Bench multiple candidates pre-launch; allow per-language model swap (REQ-068 defer notwithstanding) |
| `auto-extractor` writes spurious facts that pollute downstream | Medium | Medium | Embedding-similarity gate (≥0.92) before LLM update; confidence threshold on facts (≥0.7); reconcile_flag for review |

---

## 17. References

- **Research dossiers in this repo** (see Companions at top)
- **REQ-070 — Brain Memory v8** (`dialog-requirements.md` / Phase-3 inheritance from REQ-076 §2)
- **REQ-075 — Geomi companion model** (`docs/geomi-concept.md`)
- **Acceptance corpus:** `tests/eval-brain-memory/persona-corpus.yaml`
- **Mockups:** `ux-mockups/brain-memory/murmur-tide-hifi-v8.html`, `personas-gallery-v1.html`, `scan-and-synthesis.html`
- **Phase-2 dev plan:** `dev-plan-phase2.md` (v8 deferred per REQ-076 §2)
- **Engineering standard:** seanslab `Engineering Guide.md`

### Prior art derivation (full reasoning in deep-dive docs)

- **Karpathy LLM Wiki gist** ([gist.github.com/karpathy/442a6bf555914893e9891c11519de94f](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f), April 2026) — three-layer model (`raw/` + `wiki/` + schema), three operations (ingest / query / lint), `index.md` + `log.md` navigation, "the wiki is a persistent compounding artifact." v8 independently converged on this; we adopt his vocabulary. Full comparison in `docknote-karpathy-wiki-memory.md`. **Conflict kept:** Karpathy says LLM owns the wiki entirely; v8 uses Geomi-mediated confirmation so the user remains authoritative for professional truth (decision #14).
- **Letta (sleep-time pattern + memory blocks)** — Tier 1, Tier 2 §4
- **Graphiti (bi-temporal schema; not the graph DB)** — Tier 1
- **Granola (citation UX; counter-positioning on encrypted DB)** — Tier 1
- **Minutes (frontmatter shape; tool naming)** — Tier 1
- **Anthropic memory tool (rejected as external surface)** — Tier 1
- **mem0 (two-phase extract + reconcile shape)** — Tier 2 §1
- **MemMachine (keep-raw-as-ground-truth)** — Tier 2 §2
- **RAPTOR (synthesis substrate)** — Tier 2 §3
- **Cline / cursor memory bank (file taxonomy + read-on-every-turn discipline)** — Tier 2 §5
- **Claude Code memdir (lessons doc)** — `docknote-memory-learn-from-cc.md`

### Sibling projects in `~/seanslab/`

- **Memio** (`~/seanslab/Memio/`) — wearable hardware-recorder substrate; chose entity-graph memory for 10-year horizon. Reusable: `MemioUXPhilosophy.md` (sharper "ghost genius" voice than v8's mockups), `memio-memory-research.md` (covers mem0 / MemMachine with concrete numbers).
- **askserver** (`~/seanslab/autoresearch/askserver/`) — research-server substrate on Jetson Orin; chose "Less Structure, More Intelligence" (most aligned with v8 row-12). Reusable: auto-research harness (one-mutable-artifact + dev/tune/holdout + Opus-4.6-judge ratchet) for v8's persona-corpus eval; empirical Qwen-vs-Claude indexer benchmarks de-risk v8 risk #1.
- **BoscoV4** (`~/seanslab/BoscoV4/`) — household-NAS appliance substrate; chose typed SQL tables for decisions/action_items (the Granola-shaped architecture v8 rejects). Reusable: citation-anchor schema (`citation_id / chunk_id / start_ms / end_ms / speaker_label / transcript_preview`) is more rigorous than v8's `{ meeting_id, span }` — adopt.
- Cross-comparison: `docknote-karpathy-wiki-memory.md` §E shows all four projects against Karpathy's gist.

---

## 18. What's next

This doc is design-locked. The next deliverables are SDD-level:

1. **Frontmatter schema spec** — exhaustive YAML schema for each engram kind with validation rules.
2. **Prompt library** — extraction prompt, update reconciler, sleep-time consolidation, synthesis prompt, pivot router. Each with few-shot examples and failure-mode notes.
3. **`Hippocampus` trait module spec** — full Rust signatures, error types, contract documentation.
4. **Eval harness spec** — how the 24-persona corpus run is wired, scoring rubric, regression CI.
5. **Migration plan** — when v8 frontmatter changes in v9, what's the migration commit shape?
6. **Geomi trigger heuristics** — concrete rules for `mid_meeting_suggest`, calibrated against real meeting samples.

Recommend dispatching SDD work in parallel where possible. Critical path runs through (1) frontmatter schema and (2) prompt library — those gate everything else.
