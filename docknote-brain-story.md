# Docknote Brain — the story

**Read this first.** Everything else in the v8 design follows from here.

---

## Two principles

Before the architecture, two principles. Read these first; the rest is what follows from them. (Full articulation: `docknote-brain-philosophy.md`.)

1. **Less structure, more intelligence.** Don't pre-structure data into tables and schemas. The markdown corpus is the only source of truth; SQLite holds only indexes; the LLM reconstructs meaning at recall time. Lets the system be smarter than its schema.

2. **Action catalyzes memory.** The act of confirming a memory is what makes the user remember it. v8 is not a replacement for the user's memory — it's a system that strengthens the user's memory through the ritual of confirming what the system extracts. Geomi-mediated confirmation isn't a compromise on automation; it's the feature.

These two principles explain every design decision in v8 — why there's no `facts` table, why Geomi mediates writes, why we diverge from Karpathy on authority, why the architecture looks the way it looks.

---

## The book

Docknote Brain reads like a **commonplace book**. A commonplace book is the personal volume people kept for centuries — pages of prose accumulated over a life, an index at the front, marginalia in the margins, the spine thickening with every entry. It's the literal antecedent of every "personal knowledge base" idea that came after, including Karpathy's LLM Wiki and v8 itself.

Your brain *is* your commonplace book. There's a page for every person you talk to, every topic you carry, every meeting that happens. The book turns its own page while you sleep — at 03:00 it re-reads what was said and updates the relevant pages. In the morning you sit with it briefly, read what's new, and initial the entries you want to keep. The marginalia are dated.

> **The book remembers because you read it; you remember because you wrote in it.**

That sentence is the whole product. Principle 1 is *what* the book is (prose, not tables). Principle 2 is *how* it works (you reading and initialing is the encoding event). Everything else is plumbing.

| In the book | In Docknote |
|---|---|
| **A page** — one entry, in prose | one file in `cortex/` (a person, topic, or meeting page) |
| **Marginalia** — dated handwriting in the margins | the user's confirmations / edits, stamped with date and authority |
| **The index at the front** | `cortex/index.md` (auto-maintained catalog) |
| **The spine** — order of entry | git history (chronological, navigable) |
| **Loose papers tucked between pages** — yet to be transcribed | `raw/meeting/<id>/transcript.md` (the source captures the page is written from) |
| **The bookplate / colophon** — whose book this is, how to read it | `persona.md` (who you are; how the book should treat you) |
| **The morning re-read** — sit with what was added overnight | the daily Geomi confirmation ritual |

---

## The brain (behind the cover)

The book is the surface; the architecture is a brain. Every layer has a real neuroscience counterpart, and the metaphor is the map:

| Brain | Docknote |
|---|---|
| **Neurons / concept cells** — atomic representations | rows in `entity_index` (one per person, topic, date the user tracks) |
| **Synapses / Hebbian links** — atomic associations between neurons | atomic claims in meeting-page frontmatter (`decisions[]`, `action_items[]`, `claims[]`) — born in the page where they were said |
| **Engram (cell assembly)** — the activation pattern that constitutes one memory | one **page** in `cortex/` (a rendered narrative, derived from the underlying corpus) |
| **Raw sensory substrate** — the experience before it becomes memory | `~/Docknote/brain/raw/` (immutable transcripts, audio, future Slack/email/voice-memo) |
| **Cortex** — the LLM-maintained wiki where engrams live | `~/Docknote/brain/cortex/` (the bound pages, plus the front-of-book index) |
| **Hippocampus** — encodes, indexes, retrieves; reconstructs distributed traces at recall | the index layer (SQLite + sqlite-vec + RAPTOR) + the Brain API |
| **The galaxy** — high-dimensional space where related things sit close, distant things sit far | the sentence-level vector index over every transcript chunk |
| **Constellations** — hierarchical structure imposed over the galaxy | the RAPTOR forest (per-meeting trees + weekly + quarterly rollups) |
| **Working memory** — the small buffer always at hand | always-loaded `persona.md` + the Recent view |
| **Encoding** — laying down a new memory from experience | the per-meeting-close extraction |
| **Consolidation** — strengthening and integrating memories over time | the nightly 03:00 pass |
| **Recall** — hippocampus reconstructs from cortical traces at query time | the 5-stage read path |
| **Schemas** — recurring patterns the brain extracts across memories | the Recurring view (auto-detected themes) |
| **Brain** — the whole | **Docknote Brain** |

Every row is a real distinction in either neuroscience or our codebase. The metaphor isn't decoration — it's how we navigate the system. The book is what the user touches; the brain is how it's built. They are isomorphic.

---

## The atomic units: three layers, not one

A close reading of the brain reveals three atomic things, not one. Our system has the equivalent three.

**Concept cells / entities.** In the brain, individual neurons (or small populations) represent specific concepts — "Jack," "delivery," "the date 9/9/2019." In Docknote, these live as rows in `entity_index`: a name, a kind, aliases, a page path. Tiny. Atomic.

**Synapses / atomic claims.** In the brain, synapses are the connecting tissue: weighted links between concept cells that fire together. In Docknote, atomic claims like *"Jack promised delivery on 2019-09-09"* live as structured frontmatter in the meeting page where they were born — `cortex/meeting/<date>.md` carries `decisions[]`, `action_items[]`, `claims[]`, each with span and confidence citing back into `raw/meeting/<date>/transcript.md`. **Single source of truth, in the page of the meeting where it was said.** No parallel `facts` table accumulating over years.

**Engrams / pages.** In the brain, an engram is the *cell assembly* that activates together when you remember Jack — distributed, reconstructed at recall. In Docknote, an engram is one markdown page (`person/<slug>.md` or `topic/<slug>.md` or `meeting/<date>.md`): a rendered narrative the consolidator writes by reading across the corpus. The page is the human-readable projection of "what we know about this entity"; it's not the only place that knowledge lives, and it's regenerable from the corpus.

**One name, three audiences.** *Engram* in code and SDD (the brain metaphor earns its keep when distinguishing the index from the trace). *File* when describing the architecture (a page is literally a markdown file on disk). *Page* in product copy and to the user (this is what they see when they open it).

Three kinds of pages plus a special one:

- **Person pages** — Mrs. Chen, Alex Chen, opposing counsel. The doctor's patient pages, the lawyer's client pages, the sales team's account pages.
- **Topic pages** — Henderson v Acme, the Q3 pricing experiment, the timeline-quality tradeoff.
- **Meeting pages** — discrete events with full transcripts. Episodic memory in Tulving's sense.
- **Your own page** (`persona.md`) — the bookplate; always loaded into working memory.

---

## Why we're not a database

The brain isn't a database. It's a galaxy of distributed representations connected by associations, with a hippocampal index that reconstructs memories from those traces at recall time.

Earlier drafts of this design tried to mirror that with a SQL `facts` table — atomic `(subject, predicate, object, valid_at, invalid_at)` rows accumulating in SQLite. That was wrong. **The brain isn't tabular** — and a SQL store of years of structured claims about hundreds of people accumulates contradictions, fake-precision artifacts, and rigidity that the brain doesn't have.

Instead: **the markdown corpus is the only structured truth.** Atomic claims live in their birth meeting page. The SQLite layer is *pure indexes* — sentence-level vectors (the galaxy), RAPTOR trees (the constellations), entity name → page resolution. Indexes are pointers into the cortex; they're not parallel data. They're regenerable from the markdown at any time.

This means cross-meeting reasoning ("what has Jack been saying about pricing?") is **reconstructive at query time**, not pre-computed. The hippocampus retrieves the relevant chunks; the LLM weaves them into an answer. The way you do it.

---

## How the brain works, end to end

A user records a meeting. Docknote transcribes it.

The **raw transcript** lands at `raw/meeting/<date-slug>/transcript.md` — immutable thereafter, the immutable substrate. (`raw/` is a separate namespace alongside `cortex/`; future Slack/email/voice-memo capture lands here too.)

The **hippocampus encodes** what happened: extracts decisions, action items, claims, the people and topics touched, by running a bounded LLM pass over the transcript. A new meeting page is rendered at `cortex/meeting/<date-slug>.md` with structured frontmatter (decisions, action items, attendees) and a Summary body — the page cites back into `raw/` for every claim. Sentence-level embeddings go into the galaxy. RAPTOR builds a per-meeting summary tree.

Then the consolidator re-reads relevant person and topic pages in context of the new meeting and proposes updates to their bodies — a new role line, a new commitment, a new theme. Some changes commit directly (low-stakes auto-derivation); others are queued for **Geomi-mediated confirmation** — the marginalia waiting for your initial. Tomorrow morning, Geomi formats one as a conversational prompt: *"After today's meeting with Alex, I'd update his role line — was X, now Y, from the 12:34 mark. Confirm?"* The user says yes/no/modify. The verdict commits with the user as authority. **Confirmed memories outrank unconfirmed ones in future recall.**

Overnight, the **consolidation** pass runs. It re-derives page bodies for entities touched this week, detects recurring patterns (schemas) by clustering across meeting summaries, runs **lint** (Karpathy's third operation): orphan pages, broken citations, stale claims, contradictions, schema violations. The lint report regenerates `cortex/index.md` (the index at the front of the book). Geomi composes the morning nudge from the day's events.

The next morning, the user asks *"what did I discuss with Alex about pricing in March?"* The **hippocampus recalls**: an entity-resolver matches "Alex" → `cortex/person/alex-chen.md`; meeting pages with Alex in attendees and date in March are filtered; sentence-level vector search over the galaxy finds relevant transcript chunks (in `raw/`); RAPTOR climbs the hierarchy if more breadth is needed; a synthesis prompt produces a 2-4 sentence answer with citations clickable to the source meeting and timestamp.

---

## What this gives the three target users

- **The doctor** opens Docknote between visits. Each patient has a page. Each visit is a meeting page citing the raw transcript. The patient page updates after each visit — Geomi proposes the marginalia, the doctor initials them in the morning re-read. *"After today's visit, I'd add 'reports new dizziness' to Mrs. Chen's current concerns — confirm?"*

- **The lawyer** opens Docknote between calls. Each case is a topic page; each opposing counsel is a person page; each call is a meeting page. Opening a case page shows recent activity, open commitments, and the recurring positions of the other side — all reconstructed from the underlying meeting pages. Time-travel on a person page shows how opposing counsel's stance has evolved.

- **The salesperson** opens Docknote between calls. Each account is a topic page; each champion is a person page; each call is a meeting page. The Recent view is "what's hot this week"; the Pending view is "what I owe people"; the Recurring view is "what objections keep coming up across my deals."

Three professions, one book, one mental model: **claims are born in meetings, the cortex holds the pages that summarize them, the hippocampus retrieves on demand, and everything stays on your machine.**

---

## What's not here

- **No `facts` table.** Earlier drafts borrowed Graphiti's bi-temporal SQL schema; we dropped it. The galaxy isn't tabular; structured claims belong in their birth meeting page.
- **No cloud sync.** The brain is single-device. Backup is handled separately at the Docknote layer.
- **No external protocol surface** (no MCP server, no Anthropic memory-tool wire format) — the brain talks to Docknote's own agent in-process. Filesystem + git is the universal interop layer for any external tool that wants in.
- **No silent overwrites.** Every consequential change is mediated by Geomi: LLM proposes, user confirms, commit lands with human as authority. Confirmed memories outrank unconfirmed ones in recall.
- **No surveillance product anti-patterns.** The morning re-read is a working ritual, not a "0 unread!" compulsion. The Geomi companion never begs.
- **No manila folders.** A folder is a *container of papers* — the wrong primitive for a thing that's actually one rendered narrative. The page in the book is the unit; folders were a v2 misdirection we corrected.

---

## The 30-second version

Docknote Brain is your **commonplace book**, shaped like a brain inside. `raw/` holds the immutable transcripts (and one day Slack/email/voice memos). `cortex/` is the bound pages — one per person, topic, and meeting — plus the index at the front. The SQLite layer holds *only* indexes: a sentence-level vector galaxy, a RAPTOR forest, an entity resolver. Recall reconstructs at query time. The LLM proposes the marginalia; Geomi reads them aloud each morning; you initial what you want to keep. Everything stays on your machine.

We independently arrived at the same architecture as Karpathy's [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) (April 2026), with one principled divergence: for doctor / lawyer / salesperson users, the human stays authoritative — Geomi's confirmation flow keeps it that way without making the user type into the pages.

---

**Now read** `docknote-brainmemory-design.md` for the full architecture, or `rachel-redesign/mockup-index.html` to see the surfaces.
