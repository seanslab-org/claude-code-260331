# Brain Memory v8 — design dialog log

**Date:** 2026-05-04
**Purpose:** A record of the design dialog that turned the research dossiers into the locked v8 architecture. Captures *why* each decision was made, not just *what*. The locked design lives in `docknote-brainmemory-design.md`; this file is the journey.

**Companions:**
- `docknote-memory-learn-from-cc.md` — Claude Code memdir lessons
- `docknote-memory-leads.md` — broad-sweep leads (~155 entries)
- `docknote-memory-deepdive-tier1.md` — Letta, Graphiti, Granola, Minutes, Anthropic memory tool
- `docknote-memory-deepdive-tier2.md` — mem0, MemMachine, HippoRAG 2 + RAPTOR, Letta sleep-time, Cline memory bank
- `docknote-brainmemory-design.md` — locked design (output of this dialog)

---

## 0. Where we entered

After Tier 1 + Tier 2 deep-dives, the architecture was ~95% specified. All five decision axes resolved:

| Axis | Decision |
|---|---|
| Extract-and-store vs. keep-raw-and-project | Hybrid: raw transcripts as truth, typed extraction at meeting close as derived index |
| Filesystem vs. KG vs. vector vs. hybrid | Markdown source-of-truth + SQLite/sqlite-vec + RAPTOR forest. No graph DB. |
| Implicit auto-extract vs. explicit user-edited | Both, with human-wins. Auto-extract + nightly consolidation; per-field human-lock + Brain Inbox for proposed updates |
| Per-session blank slate vs. always-loaded global | Always-loaded persona/active.md + Anthropic memory tool as scratchpad |
| Cloud vs. local-first | Local-first by default. SnapGPU/Qwen for synthesis. Memory contents never auto-leave the tailnet |

**One open piece:** the audit trail. Tier 2 had specified a custom `revisions/<card>/<iso8601>.json` ledger to power the "what changed?" UI. It worked, but it was custom code reinventing version control.

That's the door the dialog walked through.

---

## 1. Use git for audit and version control

### The proposal (user)

> "idea: use git to control docknote memory, every change can be traced. how about this."

### Analysis

The proposal was strictly better than the custom JSON ledger I had specced. Six concrete reasons:

1. **Audit trail for free.** `git log -p -- person/alex.md` and `git blame` give everything the custom ledger would, plus tooling the world already knows.
2. **Atomicity built in.** A single commit can touch N files (meeting markdown + multiple dossier cards + facts table) as one transaction. No fsync discipline to invent.
3. **Time travel for free.** `git show HEAD~30 -- person/alex.md` answers "what did this say a month ago?" — a feature not previously scoped.
4. **Rollback is trivial.** `git revert <sha>` or `git checkout <sha> -- person/alex.md`. UI just wraps these.
5. **Multi-device sync solved.** Optional private remote (Tailscale-served Gitea, or private GitHub repo with the user's own keys) — beats Granola's "we encrypt and hold the keys" because the user owns everything.
6. **Cline already validates the pattern** — their published audit story is literally "git diff is the audit trail," and `basic-memory` does the same.

### Counterpoints considered

- **Commit volume.** Naive implementation could mean hundreds of commits/day. → Mitigated by the transactional `start/finish_consolidation` already in the design — one commit per consolidation pass, not per field.
- **Performance at scale.** Git scales to Linux-kernel-size repos; not a real concern for personal scale (hundreds of meetings/year).
- **Conflict resolution.** Merge conflicts when meeting-close + user editing simultaneously. Mitigated by per-card advisory lock during meeting-close (5-30s).
- **Privacy risk.** `git push` to wrong place is a one-keystroke leak. Mitigated by Tailscale-only-remote guardrail.

### Design tweak this forced

Custom JSON ledger gave per-field provenance natively (each ledger entry was scoped to one field). Git tracks file-level history; for field-level you'd parse the diff. To preserve the field-level "why" without a sidecar:

**Move per-field provenance into the markdown frontmatter itself:**

```yaml
role_line:
  value: "VP Eng at Acme · pushes timeline back, holds line on quality"
  source: { meeting_id: "2026-05-04-q3-pricing", span: "00:12:34-00:13:02" }
  model: "qwen-3-32b"
  authored_by: "auto-consolidator"
  authored_at: "2026-05-04T15:30:21Z"
  human_locked: false
```

Now every git diff is parseable to "which field changed, why, when, sourced from where" without a sidecar. The frontmatter *is* the provenance.

### Decision

**Replace the custom revisions ledger with git.** Per-field provenance moves into frontmatter. Three commit identities (`auto-extractor`, `auto-consolidator`, `<user>`). Three commit types (`meeting-close`, `nightly-consolidation`, `user-edit`).

### Design impact

- Drop `revisions/<card>/<iso8601>.json` from the architecture.
- Add per-field provenance schema to frontmatter (later locked into critical-path deliverable #1).
- Add commit-message format spec (touched files, extracted counts, model, prompt hash).
- Add Tailscale-only-remote guardrail to privacy section.
- "Human edits always win" semantics now expressed as `human_locked: true` field on each provenance block.

---

## 2. "Git or something like git"

### The user's nudge

> "use git or sth. like git"

Open door to alternatives. The question is *property*, not specific tool: versioned, atomic, auditable, traceable, ideally distributed.

### Candidates compared

| | git (libgit2 / git2-rs) | jj (Jujutsu) | Fossil |
|---|---|---|---|
| Files-on-disk source of truth | ✓ | ✓ | ✓ (history in SQLite sidecar) |
| Atomic multi-file commits | ✓ | ✓ | ✓ |
| Auto-snapshot every save | ✗ (explicit commit) | ✓ native | ✗ |
| Conflict-resolution UX | manual, sharp edges | better merging | manual |
| Embeddable Rust crate | ✓ `git2-rs` mature | `jj-lib` exists, pre-1.0 | weak |
| User familiarity | universal | unfamiliar | niche |
| Push/pull to remote | trivial | git-compatible | trivial |
| Battle-tested at scale | yes | not yet | yes (small projects) |

Skipped: Pijul (mathematically lovely, small community), DoltDB (loses files-on-disk property), custom event-log (NIH).

### Decision rationale

**git via `git2-rs`.** In order:

1. **Works today on every platform Tauri ships.** No "is jj 1.0 yet" risk on the critical path.
2. **Users who poke at `~/Docknote/brain/`** get a repo they already understand. Power user can `git log` and feel at home.
3. **Sync via private remote** is one-line for multi-device, user owns the keys (counter to Granola's lock-in).
4. **`git2-rs` is mature and embedded** — no subprocess fork/exec.

### The jj-style auto-snapshot, with git underneath

jj's "every working-copy state is automatically a snapshot" property is genuinely nice. Replicated in git with discipline:

- **Auto-extractor** commits at meeting close (one transaction).
- **Auto-consolidator** commits at end of nightly pass (one transaction).
- **User edit** in the dossier UI commits on field-blur or save-click. App handles it; user never types `git commit`.

Same end-state as jj's auto-snapshot, achieved with the more proven backend.

### Future-proofing

Wrap persistence behind a small Rust trait — `MemoryStore::write_transaction(files, author, message)`, `read_at(path, sha)`, `history(path)`, `diff(sha_a, sha_b, path)`. Day-1 impl is `git2-rs`. If jj stabilizes or Pijul gets traction, swap implementations without touching the rest of v8. Keep the abstraction tiny and git-shaped.

### UX rule

**Never expose git vocabulary to users.** Surface is "history of this card," "what changed last night," "revert this auto-update," "sync to other Mac." Underneath: git ops. Same pattern Cline uses — git is the substrate, not the UI. Keeps non-technical users unscared while keeping the audit story bulletproof for technical ones.

---

## 3. "Why MCP?"

### The challenge (user)

> "why mcp?"

I had been carrying MCP forward as if it were obvious. Wasn't.

### Layer separation surfaced

I had collapsed three distinct concerns:

1. **Persistence** — git + Markdown + SQLite indexes. "Where the brain lives."
2. **Brain API** — in-process Rust calls the Docknote app uses to read/write/synthesize. Type-safe, fast, no protocol overhead. **Never MCP.**
3. **External access** — how non-Docknote agents talk to the brain.

I had been wrong to label Layer 2 as `brain.search` MCP in the Tier 2 summary. That's a Layer 3 concern only.

### The realization

Once you split them out, MCP is optional. The strongest interop story Docknote has is **the filesystem itself**. Markdown + git in `~/Docknote/brain/` is readable by:

- Obsidian, Logseq, any markdown editor
- `ripgrep`, `grep`, `awk`
- Cursor `@`-mentions and file context
- Claude Code's built-in file tools
- Anthropic's memory tool (which is literally filesystem CRUD)
- Any future LLM client that knows how to read a file

That's a **broader** interop surface than MCP. MCP is sugar on top, not the foundation.

### When MCP would still earn its keep

Real reasons (none of which are urgent):

- **Synthesis tools that filesystem-read can't do well**: `themes_for_person(alex, since=90d)`, `consistency_check(decision_x)`, `time_window_query("pricing", "March 2026")`.
- **Tool-name alignment with Minutes** so users running both can swap engines.
- **Synthesis-pass exposure to external agents** so a user's Claude Code session can ask "what are the recurring concerns from this quarter's investor meetings."

### When MCP would be over-engineering

- Single-app use (Docknote talking to its own brain) — direct Rust API wins.
- Raw read access — filesystem already provides it.
- Permission model — every MCP client gets the same access; per-client gating is non-trivial.
- Process boundary cost — stdio/HTTP overhead per call.

### Initial recommendation

Phase 1 ship: in-process Rust API only. Filesystem is the universal interop. **No MCP server.** Phase 2 (post-launch, only if there's real demand): thin MCP server exposing the synthesis-pass tools.

---

## 4. "no mcp"

### The lock (user)

> "no mcp"

Decisive. Locked.

### What dropped

- The MCP server (`brain.search`, `get_person_profile`, etc.).
- **Anthropic memory-tool wire format** — same logic. If Docknote's agent runs in-process against SnapGPU/Qwen, it just calls the Brain API directly. No tool-use protocol needed.
- The "two retrieval surfaces" framing from Tier 2 (memory tool scratchpad + MCP canonical) — there's now one surface.
- The `~/Docknote/brain/.agent-memory/<session-id>/` sandbox dir — no longer needed.

### What stayed

- **Persistence:** Markdown source-of-truth + git via `git2-rs` + SQLite/sqlite-vec for derived indexes.
- **Brain API:** in-process Rust trait — read/write/history/time-travel/synthesis. The only programmatic surface.
- **External access for users:** filesystem (markdown is plain text, openable in Obsidian/Logseq/grep/anything) + git CLI/GUI for power users.

### Strategic positioning unchanged

The wedge against Granola's encrypted-DB lock-in was never MCP — it was **openness**. Markdown + git is *more* open than MCP. Anyone can read the files; they don't need an MCP-aware client. Stronger story, less code.

The "MCP-native" line in counter-positioning becomes "Markdown-on-disk you own."

---

## 5. UVP alignment check

### The question (user)

> "is our brain-memory research and design match Docknote unique value proposition"

Strategic alignment check. Does the architecture serve Docknote's positioning, or have we been solving the wrong problem?

### The mapping

Each Phase-2 thesis leg + Geomi REQ-075 + the eval bar mapped against architectural choices:

**Phase-2 leg 1 — "privacy moat is literal" (Slide-4 ground truth):** strongest match.
- Markdown source-of-truth = literal opposite of Granola's encrypted SQLite.
- git audit trail = users can verify what model wrote vs what they wrote.
- SnapGPU + Qwen = no Anthropic/OpenAI egress.
- "No MCP" decision strengthens this — fewer wire protocols = fewer cloud bridges.
- Tailscale-only-remote guardrail makes the claim engineering-verifiable.

**REQ-075 — Geomi "one brain, five faces":** strong match (architecture is the substrate).
- `active.md` always-loaded → live context for every interaction.
- Persona dossier cards with `role_line`/`day_headline`/`stat_line` → Geomi authors specific observations, not platitudes.
- Synthesis pass over RAPTOR forest → pattern-aware bubbles.
- Per-field provenance → Geomi can cite *why* it's saying something.

**Phase-2 leg 3 — production-grade packaging:** match.
- `git2-rs` drops into Tauri without subprocess fork/exec.
- SQLite + sqlite-vec is single-file, fits signed-`.dmg` pipeline.
- No external service deps.

**REQ-070 acceptance corpus:** match in shape, execution risk in synthesis quality.
- Three-pivot routing maps directly to corpus's `pivot_type`.
- Bi-temporal facts handle "in March" temporal queries.
- `expected_synthesis_themes` is exactly what the synthesis pass builds.

**REQ-031 simple / REQ-032 trust the LLM:** match.
- Dropping MCP, dropping custom revisions ledger, dropping LLM-only dedup honors REQ-031.
- "Trust the LLM at well-chosen seams" honors REQ-032 — the boring file-system-shaped pattern is exactly that posture.

### The gap surfaced — Layer 2.5

The retrieval/synthesis API I had designed was **user-query-driven**. Geomi is **event-driven** — calendar events, mentioned entities mid-meeting, theme-completion triggers. The Brain Memory layer needed an explicit "trigger" surface so Geomi's bubble logic doesn't reach into internals.

**Three primitives:**

```rust
trait BrainTriggers {
    fn pre_meeting_brief(&self, ev: &CalendarEvent) -> Result<Brief>;
    fn mid_meeting_suggest(&self, ctx: &MeetingCtx) -> Result<Option<Suggestion>>;
    fn end_of_day(&self) -> Result<EveningNudge>;
}
```

Critical detail: `mid_meeting_suggest` is a **negative-default** surface. Returns `None` aggressively to honor REQ-075's "ghost genius — never begs."

### Where it's a stronger asset than initially realized

- **The git decision aligns unusually well with the honest-engineering ethos.** It's not just an audit feature — it's a thesis: "we trust you with your data because we built it so you can verify ours." Same posture as `memset_s` and X25519: features users can audit, not promises they have to take on faith.
- **Counter-positioning vs Granola is multi-axis, not one-shot.** Open files (markdown) + open history (git) + open inference (SnapGPU) + open transport (Tailscale only). Each independently defensible; combined they're a coherent identity.

### Decision

UVP fit is strong on every leg. Add **Layer 2.5 (BrainTriggers)** to the architecture. The two real risks are execution-side (Qwen synthesis quality on the persona corpus, Geomi proactivity heuristics), not architecture-side.

---

## 6. Design doc lock

### The instruction (user)

> "yes, document docknote-brainmemory-design.md"

### Output

`docknote-brainmemory-design.md` — ~6.5k words, 18 sections. The design-locked architecture document. Includes:

1. Status / scope / companions
2. Scope (in / out / non-goals)
3. UVP alignment table
4. Architecture one-pager
5. Persistence layer (filesystem, git, SQLite, frontmatter schema, human-locked rule)
6. Brain API (Layer 2 — Rust trait sketch + validation contracts)
7. Proactive triggers (Layer 2.5 — Geomi)
8. Write path (per-meeting + nightly)
9. Read path (5 stages)
10. Edit UX
11. Privacy & security
12. Phase boundaries
13. Acceptance criteria
14. Eval design
15. Open questions for SDD
16. Decision log (11 rows with reasoning + reversibility)
17. Risk summary
18. References + what's next

This is the canonical design ref. The Tier 1 / Tier 2 dossiers remain as research artifacts; the design lives in the new doc.

---

## 7. Critical-path explainer

### The follow-up (user)

> "explain critical path 1 and 2"

The design doc had named two SDD-level deliverables as critical path: **frontmatter schema spec** and **prompt library**. The user wanted substance.

### Critical path 1 — Frontmatter schema spec

**What it is:** A complete, versioned YAML contract for every file type in `~/Docknote/brain/`. The data model the rest of v8 reads from and writes to.

**What's in it:**

- Per card type fully specified: `persona.md`, `active.md`, `patterns.md`, `commitments.md`, `person/<slug>.md`, `project/<slug>.md`, `meeting/<date-slug>.md`.
- Per-field provenance schema: `value`, `source: {meeting_id, span}`, `model`, `authored_by`, `authored_at`, `human_locked`, `confidence`, `prior_values[]`.
- Validation rules: ISO 8601 datetimes, meeting_id format, span format, alias requirements, lock invariants.
- Schema versioning: top-level `_schema: v8` field; migration spec for v8 → v9.

**Why it's critical path:** Every other deliverable depends on this. Prompts produce JSON that must conform to it. `BrainStore` Rust impl validates against it. UI renders fields based on it. Eval scores against fields it defines. Migration code can only be written once it's locked.

**Effort + risk:** 1-2 days of spec work. Output: JSON Schema + human-readable spec doc. Primary risk: schema fights — the persona-corpus's `expected_dossier_card` fields must match exactly. Cross-check schema against corpus before locking.

### Critical path 2 — Prompt library

**What it is:** The set of LLM prompts v8 runs across the write and read paths. Quality of these prompts is the dominant variable in passing the 80% acceptance bar.

**What's in it (11 prompts across 3 surfaces):**

Write-path:
- (a) Extraction prompt — per topic-segment, JSON output of decisions/actions/claims/people/themes with sources.
- (b) Update reconciler — mem0 ADD/UPDATE/DELETE/NONE shape with strict ID-preservation.
- (c) Per-meeting consolidation — Letta sleep-time, field-level tool calls.
- (d) Nightly cross-card consolidation — theme rollups + age-banding.

Read-path:
- (e) Pivot router — `name | date | concept` classifier.
- (f) Manifest selector — Sonnet picks ≤5 cards from filtered manifest.
- (g) Synthesis prompt — **the v8-differentiating one.** 3-4 themes with cited meetings.
- (h) RAPTOR cluster summarizer — build-time summaries per tree level.

Geomi:
- (i) Pre-meeting brief composer.
- (j) Mid-meeting suggestion judge — **negative-default**, REQ-075 enforcement.
- (k) End-of-day nudge composer.

Per prompt: full text (versioned, hash-pinned), few-shots encoding the schema, output JSON schema, failure modes, model assignment, token budgets.

**Why it's critical path:**
1. **Quality dominates the eval.** Extraction + synthesis prompts together determine 70%+ of the persona-corpus score. Tuned with eval-in-the-loop, not just authored.
2. **Few-shots encode the schema.** Every prompt's few-shots tied to frontmatter spec — change schema, must update prompts. They co-evolve.
3. **Prompt versioning is part of the audit story.** Every consolidation commit cites the prompt hash. Without a locked prompt library, audit messages are meaningless.

**Effort + risk:** 3-5 days of prompt engineering with eval-driven iteration. Not "write once" — measure, adjust, re-measure on the seeded corpus.

Primary risks:
- **Synthesis quality on Qwen-3-32B vs GPT-4o.** Mem0 / RAPTOR / HippoRAG benchmarks were on GPT-4o. Qwen may need more aggressive few-shotting. Dominant unknown.
- **Mid-meeting suggestion tuning.** Too eager → REQ-075 violated. Too quiet → useless. Hand-calibrate on real meeting recordings.
- **Prompt drift between releases.** One-line change can swing 3-5pp. CI must pin hashes and gate score regressions.

### Why the two are co-dependent

- The extraction prompt's output JSON IS the frontmatter shape it produces.
- Sleep-time consolidation tool calls (`update_card_field`) match `BrainStore` API signatures, derived from the schema.
- Eval harness scores extraction + synthesis output against the corpus's expected frontmatter values.
- Audit commit messages reference both schema (which fields) and prompt hash (what produced the change).

They evolve in tandem. Spec the schema → write prompts whose few-shots use it → run on seeded corpus → discover schema gaps → revise schema → revise prompts. Two or three iterations before lock.

Once locked, the rest of v8 — `BrainStore` impl, UI, eval harness, sync, migration — is straight implementation work.

---

## 8. Net architectural changes from this dialog

| Change | Source |
|---|---|
| Custom revisions ledger → git via `git2-rs` | §1 |
| Per-field provenance moved into frontmatter | §1 |
| Three commit identities + three commit types defined | §1 |
| Tailscale-only-remote guardrail added | §1 |
| `MemoryStore` trait abstraction added | §2 |
| MCP server dropped from v8 | §3, §4 |
| Anthropic memory-tool wire format dropped from v8 | §4 |
| `.agent-memory/` sandbox dir dropped | §4 |
| "Two retrieval surfaces" framing collapsed to one | §4 |
| Layer 2.5 BrainTriggers added | §5 |
| `mid_meeting_suggest` specified as negative-default | §5 |
| Critical path of two SDD deliverables identified and scoped | §6, §7 |

The locked design (in `docknote-brainmemory-design.md`) reflects all of these.

---

## 9. What's next

Per the design doc §18:

**Critical path (must precede everything else):**
1. Frontmatter schema spec
2. Prompt library

**Then in priority order:**
3. `BrainStore` trait module spec — full Rust signatures, error types, contracts.
4. Eval harness spec — 24-persona corpus run, scoring rubric, regression CI.
5. Migration plan — v8 → v9 frontmatter shape changes.
6. Geomi trigger heuristics — concrete rules for `mid_meeting_suggest`, calibrated against real meeting samples.
7. Sync UX spec — conflict resolution flow, remote setup wizard with Tailscale guardrail.

(3) and (4) can run in parallel after (1) and (2). (5), (6), (7) can run later.

Open questions routed to SDD review (design doc §14):
1. Sentence boundary detector (punkt vs spaCy vs language-aware)
2. Embedding model (`bge-m3` quantized vs alternatives)
3. Topic-segmentation algorithm (fixed window vs LLM-decided vs Bayesian-surprise)
4. Conflict resolution UX for sync
5. Brain Inbox cadence (daily vs weekly)
6. `active.md` size cap and rotation policy
7. Calendar integration scope
8. Sensitive-meeting marking UX
