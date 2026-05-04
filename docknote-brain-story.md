# Docknote Brain — the story

**Read this first.** Six paragraphs. Everything else in the v8 design follows from here.

---

## The metaphor

Docknote Brain is built like a human brain. Not as marketing — as architecture. Every layer has a real neuroscience counterpart, and the metaphor is the map:

| Brain | Docknote |
|---|---|
| **Neurons / concept cells** — atomic representations | rows in `entity_index` (one per person, topic, date the user tracks) |
| **Synapses / Hebbian links** — atomic associations between neurons | atomic claims in meeting frontmatter (`decisions[]`, `action_items[]`, `claims[]`) — born in the meeting where they were said |
| **Engram (cell assembly)** — the activation pattern that constitutes one memory | one **file** in `cortex/` (a rendered narrative, derived from the underlying corpus) |
| **Raw sensory substrate** — the experience before it becomes memory | `~/Docknote/brain/raw/` (immutable transcripts, audio, future Slack/email/voice-memo) |
| **Cortex** — the LLM-maintained wiki where engrams live | `~/Docknote/brain/cortex/` (markdown files, plus `index.md` / `log.md` / `SCHEMA.md` navigation) |
| **Hippocampus** — encodes, indexes, retrieves; reconstructs distributed traces at recall | the index layer (SQLite + sqlite-vec + RAPTOR) + the Brain API |
| **The galaxy** — high-dimensional space where related things sit close, distant things sit far | the sentence-level vector index over every transcript chunk |
| **Constellations** — hierarchical structure imposed over the galaxy | the RAPTOR forest (per-meeting trees + weekly + quarterly rollups) |
| **Working memory** — the small buffer always at hand | always-loaded `persona.md` + the Recent view |
| **Encoding** — laying down a new memory from experience | the per-meeting-close extraction |
| **Consolidation** — strengthening and integrating memories over time | the nightly 03:00 pass |
| **Recall** — hippocampus reconstructs from cortical traces at query time | the 5-stage read path |
| **Schemas** — recurring patterns the brain extracts across memories | the Recurring view (auto-detected themes) |
| **Brain** — the whole | **Docknote Brain** |

Every row is a real distinction in either neuroscience or our codebase. The metaphor isn't decoration — it's how we navigate the system.

---

## The atomic units: three layers, not one

A close reading of the brain reveals three atomic things, not one. Our system has the equivalent three.

**Concept cells / entities.** In the brain, individual neurons (or small populations) represent specific concepts — "Jack," "delivery," "the date 9/9/2019." In Docknote, these live as rows in `entity_index`: a name, a kind, aliases, an engram path. Tiny. Atomic.

**Synapses / atomic claims.** In the brain, synapses are the connecting tissue: weighted links between concept cells that fire together. In Docknote, atomic claims like *"Jack promised delivery on 2019-09-09"* live as structured frontmatter in the meeting engram where they were born — `cortex/meeting/<date>.md` carries `decisions[]`, `action_items[]`, `claims[]`, each with span and confidence citing back into `raw/meeting/<date>/transcript.md`. **Single source of truth, in the engram of the meeting where it was said.** No parallel `facts` table accumulating over years.

**Engrams / files.** In the brain, an engram is the *cell assembly* that activates together when you remember Jack — distributed, reconstructed at recall. In Docknote, an engram is a markdown file (`person/<slug>.md` or `topic/<slug>.md` or `meeting/<date>.md`): a rendered narrative the consolidator writes by reading across the corpus. The file is the human-readable projection of "what we know about this entity"; it's not the only place that knowledge lives, and it's regenerable from the corpus.

Three kinds of engrams plus a special one:

- **Person engrams** — Mrs. Chen, Alex Chen, opposing counsel. The doctor's patient files, the lawyer's client files, the sales team's account files.
- **Topic engrams** — Henderson v Acme, the Q3 pricing experiment, the timeline-quality tradeoff.
- **Meeting engrams** — discrete events with full transcripts. Episodic memory in Tulving's sense.
- **The user's own engram** (`persona.md`) — a special person file that's always loaded into working memory.

Visually, every engram looks like the one professional artefact every doctor, lawyer, and salesperson already keeps: **a manila folder with a tab and contents.**

---

## Why we're not a database

The brain isn't a database. It's a galaxy of distributed representations connected by associations, with a hippocampal index that reconstructs memories from those traces at recall time.

Earlier drafts of this design tried to mirror that with a SQL `facts` table — atomic `(subject, predicate, object, valid_at, invalid_at)` rows accumulating in SQLite. That was wrong. **The brain isn't tabular** — and a SQL store of years of structured claims about hundreds of people accumulates contradictions, fake-precision artifacts, and rigidity that the brain doesn't have.

Instead: **the markdown corpus is the only structured truth.** Atomic claims live in their birth meeting files. The SQLite layer is *pure indexes* — sentence-level vectors (the galaxy), RAPTOR trees (the constellations), entity name → file resolution. Indexes are pointers into the cortex; they're not parallel data. They're regenerable from the markdown at any time.

This means cross-meeting reasoning ("what has Jack been saying about pricing?") is **reconstructive at query time**, not pre-computed. The hippocampus retrieves the relevant chunks; the LLM weaves them into an answer. The way you do it.

---

## How the brain works, end to end

A user records a meeting. Docknote transcribes it.

The **raw transcript** lands at `raw/meeting/<date-slug>/transcript.md` — immutable thereafter, the immutable substrate. (`raw/` is a separate namespace alongside `cortex/`; future Slack/email/voice-memo capture lands here too.)

The **hippocampus encodes** what happened: extracts decisions, action items, claims, the people and topics touched, by running a bounded LLM pass over the transcript. A new meeting engram is rendered at `cortex/meeting/<date-slug>.md` with structured frontmatter (decisions, action items, attendees) and a Summary body — the engram cites back into `raw/` for every claim. Sentence-level embeddings go into the galaxy. RAPTOR builds a per-meeting summary tree.

Then the consolidator re-reads relevant person and topic engrams in context of the new meeting and proposes updates to their bodies — a new role line, a new commitment, a new theme. Some changes commit directly (low-stakes auto-derivation); others are queued for **Geomi-mediated confirmation**. Tomorrow morning, Geomi formats one as a conversational prompt: *"After today's meeting with Alex, I'd update his role line — was X, now Y, from the 12:34 mark. Confirm?"* The user says yes/no/modify. The verdict commits with the user as authority. **Confirmed memories outrank unconfirmed ones in future recall.**

Overnight, the **consolidation** pass runs. It re-derives engram bodies for entities touched this week, detects recurring patterns (schemas) by clustering across meeting summaries, runs **lint** (Karpathy's third operation): orphan engrams, broken citations, stale claims, contradictions, schema violations. The lint report regenerates `cortex/index.md` (the navigation catalog) and appends to `cortex/log.md` (the chronological event log). Geomi composes the morning nudge from the day's events.

The next morning, the user asks *"what did I discuss with Alex about pricing in March?"* The **hippocampus recalls**: an entity-resolver matches "Alex" → `cortex/person/alex-chen.md`; meeting engrams with Alex in attendees and date in March are filtered; sentence-level vector search over the galaxy finds relevant transcript chunks (in `raw/`); RAPTOR climbs the hierarchy if more breadth is needed; a synthesis prompt produces a 2-4 sentence answer with citations clickable to the source meeting and timestamp.

---

## What this gives the three target users

- **The doctor** opens Docknote between visits. Each patient is a person engram. Each visit is a meeting engram (rendered) citing the raw transcript. The patient engram body updates after each visit — proposed by the consolidator, approved by the doctor through Geomi's morning chart-review queue. *"After today's visit, I'd add 'reports new dizziness' to Mrs. Chen's current concerns — confirm?"*

- **The lawyer** opens Docknote between calls. Each case is a topic engram; each opposing counsel is a person engram; each call is a meeting engram. Opening a case engram shows recent activity, open commitments, and the recurring positions of the other side — all reconstructed from the underlying meeting files. Time-travel on a person engram shows how opposing counsel's stance has evolved.

- **The salesperson** opens Docknote between calls. Each account is a topic engram; each champion is a person engram; each call is a meeting engram. The Recent view is "what's hot this week"; the Pending view is "what I owe people"; the Recurring view is "what objections keep coming up across my deals."

Three professions, one architecture, one mental model: **claims are born in meetings, the cortex holds engrams that summarize them, the hippocampus retrieves on demand, and everything stays on your machine.**

---

## What's not here

- **No `facts` table.** Earlier drafts borrowed Graphiti's bi-temporal SQL schema; we dropped it. The galaxy isn't tabular; structured claims belong in their birth meeting file.
- **No cloud sync.** The brain is single-device. Backup is handled separately at the Docknote layer.
- **No external protocol surface** (no MCP server, no Anthropic memory-tool wire format) — the brain talks to Docknote's own agent in-process. Filesystem + git is the universal interop layer for any external tool that wants in.
- **No silent overwrites.** Every consequential change is mediated by Geomi: LLM proposes, user confirms, commit lands with human as authority. Confirmed memories outrank unconfirmed ones in recall.
- **No surveillance product anti-patterns.** The Brain Inbox is a working surface, not a "0 unread!" compulsion. The Geomi companion never begs.

---

## The 30-second version

Docknote Brain is a markdown library shaped like a brain. `raw/` holds immutable transcripts (and one day Slack/email/voice memos). `cortex/` is the LLM-maintained wiki — person, topic, and meeting engrams plus three navigation files (`index.md`, `log.md`, `SCHEMA.md`). The SQLite layer holds *only* indexes: a sentence-level vector galaxy, a RAPTOR forest, an entity resolver. Recall reconstructs at query time. The LLM proposes; Geomi mediates; the human confirms; everything stays on your machine.

We independently arrived at the same architecture as Karpathy's [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) (April 2026), with one principled divergence: for doctor / lawyer / salesperson users, the human stays authoritative — Geomi's confirmation flow keeps it that way without making the user type into engram fields.

---

**Now read** `docknote-brainmemory-design.md` for the full architecture, or `rachel-redesign/mockup-index.html` to see the surfaces.
