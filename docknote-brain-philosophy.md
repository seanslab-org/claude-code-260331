# Brain Memory — philosophy

**Two principles drive every design decision in v8.** The architecture, the UI, the consolidation pass, the Geomi confirmation flow, the choice not to ship a SQL store of facts — all of it follows from these. Read them once, and the rest of the design reads as inevitable.

---

## 1. Less structure, more intelligence

The temptation in any memory system is to **pre-structure**: extract entities, normalize relations, populate tables, enforce schemas. Make the data legible to code. The instinct is right — structure makes things queryable. But for memory specifically, it backfires. Years of meeting transcripts produce thousands of structured rows that contradict each other, miss the nuance of how things were actually said, and accumulate fake precision the source material never had. The brain doesn't work this way. It stores raw experience as distributed traces and reconstructs meaning at recall time.

**Brain Memory v8 follows the brain.** The markdown corpus is the only source of truth. SQLite holds *only* indexes — sentence-level vectors (the galaxy), RAPTOR clusters (the constellations), entity name resolution. There is no `facts` table, no structured claims store, no parallel database accumulating over years. When you ask *"what did Alex say about pricing in March?"*, the system retrieves the relevant transcript chunks and lets a 32-billion-parameter model reconstruct the answer. The intelligence is in the model, not in the schema.

This principle, applied honestly, kills a lot of plausible-looking architecture. It's why v8 dropped Graphiti's bi-temporal facts table after three weeks of designing with it. It's why we chose Markdown over an encrypted SQLite (the Granola-shape we counter-position against). It's why each meeting's `decisions[]` and `action_items[]` live in the meeting file's frontmatter — born where they were said, never indexed into a parallel store. **Less structure. More intelligence.**

---

## 2. Action catalyzes memory

The deepest reason Brain Memory v8 keeps the human in the loop isn't accuracy — it's cognitive science. **The act of confirming a memory is what makes the human remember it.**

A doctor who reads *"after today's visit, I'd add 'reports new dizziness' to Mrs. Chen's page — confirm?"* and types yes is not just approving a system note. She is encoding the observation in her own memory by acting on it. Tomorrow morning, when Mrs. Chen calls to describe a fall, the doctor will *remember* the dizziness, not because the page will tell her, but because she confirmed it yesterday. The page is the safety net; the confirmation is the memory. The marginalia she initialled this morning is what she'll recall this afternoon.

This is the principled divergence from Karpathy's "LLM owns the wiki" gist. Karpathy optimizes for LLM productivity — the human is freed from maintaining the wiki. That's the right shape for a coding agent, where the LLM is the primary user of its own memory. But v8 serves a different user: a doctor, a lawyer, a salesperson, whose **own memory** is the professional asset. The system's job is not to replace the user's memory but to strengthen it.

So Geomi mediates every consequential update through a conversational confirmation prompt. Confirmed memories outrank unconfirmed ones in future recall. Direct file editing is available as a power-user escape hatch but is not the primary workflow. The chart-review queue is the ritual that turns auto-extracted text into the user's own remembered knowledge.

**Action catalyzes memory.** The confirmation flow isn't a compromise on automation — it's the feature. Every yes the user types is a small encoding event in their own brain.

---

## What this means in practice

These two principles, taken together, explain why v8 looks the way it looks:

- **Markdown corpus + indexes only** (principle 1)
- **No `facts` table, no structured claims store** (principle 1)
- **Reconstructive recall at query time** (principle 1)
- **Geomi-mediated confirmation flow as the primary write path** (principle 2)
- **Confirmed memories ranked above unconfirmed ones** (principle 2)
- **Brain Inbox as a chart-review ritual** (principle 2)
- **Human edits remain authoritative** (principle 2)

Where these principles conflict with received wisdom, they win. The `facts` table looked right on paper — Graphiti shipped it, mem0 shipped it, BoscoV4 ships it. We dropped it because principle 1 doesn't make exceptions. The "LLM owns the wiki" stance looked right when Karpathy published it — every implementation we surveyed adopted it. We diverged because principle 2 is non-negotiable for our user.

When the SDD work begins and a hundred small decisions need to be made — what gets a column, what gets an index, what surfaces a confirmation prompt, what doesn't — these two principles are the appeal court.

---

## Where these came from

**"Less structure, more intelligence"** was already the working thesis of `~/seanslab/autoresearch/askserver/`, the Jetson-Orin research server that taught us a single 27B model + a JSONL catalog beats a vector pipeline at the corpus sizes that matter. v8 is its Mac-app cousin, sharing the philosophy and adding the engram layer for cross-meeting recall.

**"Action catalyzes memory"** is the cognitive-science principle that closes the loop between the system's job and the user's job. It's why this product is different from Granola, from Limitless, from Mem0, from every other meeting-recall tool that treats the user as a passive consumer of recalled facts. v8 treats the user as the active participant whose own remembering the system exists to strengthen.

Both principles are Sean's. They were not written down anywhere until this point in the design dialog. They are the doc.

---

**Now read** `docknote-brain-story.md` for the architecture as story, or `docknote-brainmemory-design.md` for the locked design.
