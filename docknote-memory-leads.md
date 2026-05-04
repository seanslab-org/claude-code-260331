# Brain Memory v8 — broad-sweep leads

**Date:** 2026-05-04
**Companion to:** `docknote-memory-learn-from-cc.md` (Claude Code lessons)
**Status:** Broad sweep complete. Next step: curate shortlist → deep-dive 1 by 1.
**Method:** Three parallel sub-sweeps — GitHub repos, Twitter/X discourse, products + blogs + papers. ~155 leads total. No deep reads; one-line per lead with relevance hook.

---

## Shortlist for deep-dive (suggested order)

A first cut across all three sweeps. Picked for one of three reasons: (a) closest architectural match to Brain Memory v8, (b) strongest evidence of failure modes Docknote must avoid, (c) load-bearing prior art the team should read before SDD.

**Tier 1 — read first (architectural prior art)**
1. **Letta `letta-ai/letta`** — OS-style core/recall/archival memory blocks; the persona-dossier-card is essentially a Letta core block. Read their schema before designing yours. *(GitHub §1)*
2. **Graphiti `getzep/graphiti`** + the Zep arXiv paper — temporal KG with validity windows + provenance back to source episodes. Canonical implementation of "did Alex really say that in March?" recall. *(GitHub §2 + Papers)*
3. **Granola — folder query feature + local-encrypted-DB blog** — closest competitive analog; folders + cross-meeting Q&A + citations is the table-stakes UX. *(Products §1)*
4. **Minutes `silverstein/minutes`** — Markdown-on-disk + MCP server, agents read directly. Most architecturally aligned with Brain Memory v8's likely shape. *(GitHub §7)*
5. **Anthropic memory tool docs + Effective context engineering blog** — first-party reference for tool-based persistent memory; sets the wire format if Docknote agent piece talks to Claude. *(Blogs)*

**Tier 2 — read for failure modes**
6. **`mem0ai`** thread on Claude Code's silent-truncation limit + **Mariandipietra**'s recalled-memory-confused-with-user-turn bug — concrete failure modes. *(Twitter §2, §1)*
7. **Simon Willison — "I really don't like ChatGPT's new memory dossier"** — context-bleed cardinal warning. *(Twitter §3 / Blogs)*
8. **Charles Packer (Letta CEO) — "memory in chatgpt has several problems"** thread — admission criteria, user-visible/editable memory. *(Twitter §2)*
9. **Weaviate Engram thread** — Claude refused to use Engram because MEMORY.md is always-loaded; sometimes the file system beats any retrieval engine. *(Twitter §7)*
10. **memU `NevaMind-AI/memU`** — hierarchical filesystem memory; 92.09% on LoCoMo. Closest existing analogue to name/date/concept pivot trees. *(GitHub §1)*

**Tier 3 — eval + theory baseline**
11. **LongMemEval (ICLR 2025)** + **LoCoMo (ACL 2024)** — the two benchmarks Brain Memory should evaluate against. *(Papers)*
12. **CoALA — Cognitive Architectures for Language Agents** — working/long-term/episodic/semantic/procedural taxonomy; conceptual scaffolding before naming components. *(Papers)*
13. **Position: Episodic Memory is the Missing Piece** + **EM-LLM (ICLR 2025)** — defines spec a meeting-memory system must satisfy; event-segmentation maps to chunking long meetings. *(Papers)*
14. **Generative Agents (Park et al.)** — recency × importance × relevance retrieval scoring. Still a great default. *(Papers)*

**Tier 4 — meeting-tool category map (light scan, not deep-dive)**
- Granola, Limitless, Plaud, Otter, Fireflies, Read.ai, Fathom, tl;dv, Krisp, Spinach, Notta, MeetCard — pricing + memory UX inventory only. One pass for category positioning.

The full sweep follows; deep-dive picks can pull from anywhere below.

---

# A. GitHub — open-source landscape

## A.1 Dedicated agent-memory libraries / frameworks

- **[mem0](https://github.com/mem0ai/mem0)** — 54.7k stars · last commit 2026-04 · Universal memory layer for AI agents with multi-level memory (user/session/agent), hybrid retrieval (semantic + BM25 + entity), and bolt-on integrations for LangGraph/CrewAI · *Why relevant:* Drop-in API for "remember user across sessions" with auto fact extraction — exactly the bolt-on Docknote needs to give every meeting recall pivot a working backend on day one.
- **[Letta (formerly MemGPT)](https://github.com/letta-ai/letta)** — 22.4k stars · last commit 2026-03 · Stateful-agent platform where the agent manages its own memory via OS-style core/archival blocks · *Why relevant:* Pioneered the "self-editing memory blocks" pattern; the persona-shaped dossier card is essentially a Letta core block — read their schema before designing yours.
- **[Memori](https://github.com/MemoriLabs/Memori)** — 14.1k stars · last commit 2026-04 · LLM/datastore-agnostic agent memory infrastructure that turns conversations into structured persistent state; 81.95% on LoCoMo at ~1.3k tokens/query · *Why relevant:* Strong reference for token-efficient retrieval at meeting scale where you don't want to pay for 10k-token context every recall.
- **[memU](https://github.com/NevaMind-AI/memU)** — 13.5k stars · last commit 2026-03 · Hierarchical-file-system memory framework for 24/7 proactive agents; 92.09% on LoCoMo · *Why relevant:* Treats memory as a writable, prioritizable filesystem — closest existing analogue to Docknote's name/date/concept pivot trees.
- **[Memary](https://github.com/kingjulio8238/Memary)** — 2.6k stars · last commit 2024-10 · Open-source memory layer with knowledge graph + memory streams + entity tracking · *Why relevant:* Early reference for entity-tracking memory streams; the entity-pivot-page pattern in Docknote dossiers maps cleanly onto theirs even if the repo is now stale.
- **[A-MEM (agiresearch)](https://github.com/agiresearch/A-mem)** — 995 stars · last commit 2025+ (NeurIPS 2025) · Zettelkasten-inspired agentic memory that links and evolves notes dynamically · *Why relevant:* The "memory evolves and re-links itself" pattern is exactly what Docknote needs when the same person/topic keeps recurring across meetings.
- **[ReMe (formerly MemoryScope)](https://github.com/agentscope-ai/ReMe)** — 2.9k stars · last commit 2026-04 · Procedural memory framework focused on agent self-evolution via experience replay · *Why relevant:* If Docknote ever wants the assistant to learn meeting-prep heuristics from past recall sessions, this is the reference.
- **[agentmemory (rohitg00)](https://github.com/rohitg00/agentmemory)** — 2.2k stars · last commit 2026-04 · Persistent SQLite-backed memory engine; 95.2% R@5 on LongMemEval, 92% token reduction vs. paste-context · *Why relevant:* Dead-simple SQLite + 51 MCP tools — a useful template for shipping Brain Memory inside a single Mac app without external infra.

## A.2 Knowledge-graph based memory

- **[Graphiti (getzep)](https://github.com/getzep/graphiti)** — 25.7k stars · last commit 2026-04 · Real-time temporal knowledge graph engine with validity-window facts and full provenance to source episodes · *Why relevant:* Temporal facts with provenance to "the meeting where this was said" is the canonical implementation of Docknote's "did Alex really say that in March?" recall.
- **[Cognee](https://github.com/topoteretes/cognee)** — 17k stars · last commit 2026-05 · 6-line graph+vector memory for agents, 30+ data source connectors · *Why relevant:* Already ships a Claude Code plugin with hook-based session capture — directly cribbable as a meeting-end ingestion pattern.
- **[MemOS (MemTensor)](https://github.com/MemTensor/MemOS)** — 8.9k stars · last commit 2026-04 · Memory operating system for LLMs/agents with hybrid (FTS5+vector) retrieval and tiered skill evolution · *Why relevant:* Hybrid FTS5+vector retrieval is the right cheap-and-correct stack for Mac-local meeting search.
- **[OpenMemory (CaviraOSS)](https://github.com/CaviraOSS/OpenMemory)** — 4.1k stars · last commit 2025-12 · Local Python+Node memory engine with multi-sector memory (episodic/semantic/procedural/emotional/reflective) and a temporal graph · *Why relevant:* Explicit episodic+semantic split mirrors the "what was said" vs "what we learned about Alex" distinction Docknote needs.
- **[Memento MCP](https://github.com/gannonh/memento-mcp)** — 418 stars · last commit 2025+ · Neo4j-backed knowledge graph with point-in-time retrieval, confidence decay, vector embeddings · *Why relevant:* Confidence-decay is a clean way to age out stale facts ("Alex used to lead Project X") without deleting them.
- **[MemoryOS (BAI-LAB)](https://github.com/BAI-LAB/MemoryOS)** — 1.4k stars · last commit 2025-07 · EMNLP 2025 oral; hierarchical short/mid/long-term memory with +49% F1 on LoCoMo · *Why relevant:* Three-tier hierarchy (this meeting / this quarter / forever) is a sensible shape for Brain Memory's recall layers.

## A.3 Local-first / privacy-focused personal memory

- **[Khoj](https://github.com/khoj-ai/khoj)** — 34.4k stars · last commit 2026-03 · Self-hostable AI second brain over local docs, PDFs, Markdown, Notion · *Why relevant:* Reference for shipping a local-first second-brain UI — Obsidian/desktop/mobile surfaces with one knowledge core.
- **[Reor](https://github.com/reorproject/reor)** — 8.6k stars · archived 2026-03 (last release 2025-04) · Local desktop note-taker with LanceDB + Transformers.js auto-linking related notes · *Why relevant:* Auto-linking-related-notes pattern from a fully local app — copy the LanceDB+Transformers.js stack.
- **[Smart Connections (Obsidian)](https://github.com/brianpetro/obsidian-smart-connections)** — 4.9k stars · last commit 2026-04 · Local-by-default semantic backlink plugin for Obsidian with bundled embedding model · *Why relevant:* Best reference for "no-API-key, local embeddings just work" UX.
- **[Obsidian Copilot (logancyang)](https://github.com/logancyang/obsidian-copilot)** — 6.9k stars · last commit 2026-04 · Chat-with-your-vault with agent-callable long-term memory · *Why relevant:* "Agent decides when to write to memory" tool design is a useful inversion of always-on capture.
- **[Quivr](https://github.com/QuivrHQ/quivr)** — 39.1k stars · last commit 2025-02 · Opinionated RAG framework for "second brain" over any files / vector store · *Why relevant:* Big-name reference; mostly RAG (not memory) — cite as baseline of what NOT to ship if you want true cross-meeting synthesis.
- **[basic-memory](https://github.com/basicmachines-co/basic-memory)** — 3.0k stars · last commit 2026-03 · Local-first knowledge management where humans + LLMs co-edit a Markdown knowledge graph, exposed via MCP · *Why relevant:* Plain-Markdown-on-disk as the source of truth (with MCP on top) is the most user-trustable storage shape; Docknote could ship `~/Docknote/brain/` the same way.
- **[txtai](https://github.com/neuml/txtai)** — 12.5k stars · actively maintained 2026 · All-in-one embeddings + agent + memory framework, fully local-capable · *Why relevant:* Single-binary, batteries-included alternative to wiring Chroma+LangChain by hand for an indie Mac app.
- **[OpenRecall](https://github.com/openrecall/openrecall)** — 2.8k stars · 2025-2026 · Local Windows-Recall alternative; OCR over screenshots, fully local · *Why relevant:* Passive context capture between meetings — complementary signal source for dossier cards.
- **[Screenpipe](https://github.com/screenpipe/screenpipe)** — 18.5k stars · last commit 2026-05 · 24/7 local screen + audio recording with event-driven capture, MCP server, Pipes (scheduled agents) · *Why relevant:* Reference architecture for resource-frugal always-on capture (5-10% CPU).

## A.4 MCP memory servers

- **[Official MCP memory server](https://github.com/modelcontextprotocol/servers/tree/main/src/memory)** — 60k+ stars (umbrella) · last commit 2026 · Reference knowledge-graph memory server (entities/relations/observations in JSON) · *Why relevant:* Schema baseline every other memory MCP imitates; matching gets free interop.
- **[supermemory](https://github.com/supermemoryai/supermemory)** — 22.4k stars · 2026 · Memory engine + API; #1 on LongMemEval/LoCoMo/ConvoMem with auto-sync from Drive/Gmail/Notion/OneDrive/GitHub · *Why relevant:* Best-in-class benchmarks plus working multi-source ingestion — target to beat or to plug Docknote into.
- **[supermemory-mcp](https://github.com/supermemoryai/supermemory-mcp)** — 1.7k stars · 2026 · Cloudflare-hosted universal Memory MCP, free, one-command install · *Why relevant:* Showcases the "one MCP, every LLM" UX so users can recall meetings from Claude/ChatGPT/Cursor.
- **[mcp-memory-service (doobidoo)](https://github.com/doobidoo/mcp-memory-service)** — 1.8k stars · last commit 2026-05 · Self-hosted persistent memory MCP with SQLite/Cloudflare/Milvus backends, ONNX local embeddings, ~5ms retrieval, autonomous consolidation · *Why relevant:* Closest production-grade reference for "single self-hosted service speaks every transport"; consolidation/decay logic is what Docknote needs at years-of-meetings scale.
- **[Memora (agentic-mcp-tools)](https://github.com/agentic-mcp-tools/memora)** — small/recent · 2025-2026 · Lightweight MCP server with semantic memory + KG + RAG-with-tool-calling + inter-agent event notifications · *Why relevant:* Fragment-tree document model is interesting for storing transcripts as searchable chunks while keeping pivot-level summaries.
- **[memory-bank-mcp (alioshr)](https://github.com/alioshr/memory-bank-mcp)** — 899 stars · 2025+ · Remote multi-project memory bank MCP, descended from Cline's pattern · *Why relevant:* Multi-tenant MCP pattern if Docknote treats each "workspace/team" as a project.

## A.5 Coding-assistant memory implementations

- **[claude-mem (thedotmack)](https://github.com/thedotmack/claude-mem)** — 2026 · Claude Code plugin: auto-captures sessions, AI-compresses, re-injects context · *Why relevant:* Cleanest existing example of "capture → compress → inject" loop a meeting app needs at meeting end.
- **[ClawMem (yoloshii)](https://github.com/yoloshii/ClawMem)** — 2026 · On-device memory for Claude Code/Hermes/OpenClaw via hooks + MCP + hybrid RAG; runtime-shared memory across coding agents · *Why relevant:* Cross-runtime shared-memory model is interesting if Docknote wants meeting context available in user's coding agents.
- **[memsearch (zilliztech)](https://github.com/zilliztech/memsearch)** — 2026 · Markdown + Milvus unified memory across Claude Code/Codex/OpenClaw · *Why relevant:* Vendor-backed proof that "Markdown source of truth + vector index over it" scales.
- **[Cline Memory Bank](https://github.com/cline/cline)** — ~50k stars · 2026 · Original "memory bank" pattern: structured per-project files Cline maintains and re-reads · *Why relevant:* "AI maintains its own structured project files" is the philosophical ancestor of Docknote's persona-shaped dossier card.
- **[cursor-memory-bank (vanzan01)](https://github.com/vanzan01/cursor-memory-bank)** — 3.0k stars · 2025-12 · Multi-mode (VAN/PLAN/CREATIVE/IMPLEMENT) Cursor workflow with persistent memory across modes, complexity-aware loading · *Why relevant:* Mode-driven memory loading (load only what's relevant for the current task) is exactly what Docknote should do at recall time.
- **[claude-memory (itsjwill)](https://github.com/itsjwill/claude-memory)** — small · 2026 · Persistent memory for Claude Code with semantic SQLite-vec search and Supabase cloud backup · *Why relevant:* Indie-scale architecture mirroring what Docknote can ship — local-first SQLite with optional cloud backup.
- **[open-mem (clopca)](https://github.com/clopca/open-mem)** — small · 2026-02 · OpenCode memory plugin with FTS5+vector+graph hybrid search, Reciprocal Rank Fusion, AGENTS.md generation, automatic API-key redaction · *Why relevant:* RRF over FTS5+vector+graph is a practical retrieval recipe; redaction-by-default is a privacy detail Docknote must implement.

## A.6 Meeting / notes-specific memory

- **[Meetily (Zackriya-Solutions)](https://github.com/Zackriya-Solutions/meetily)** — 11.5k stars · last commit 2026-03 · Privacy-first Mac/Windows AI meeting assistant; Parakeet/Whisper local transcription, Ollama summarization, Rust+Tauri · *Why relevant:* Direct reference stack — same Mac, same privacy stance; study summary formats before designing Brain Memory cards.
- **[Recap (RecapAI)](https://github.com/RecapAI/Recap)** — 702 stars · last commit 2025-08 · Open-source, privacy-first, macOS-native AI meeting summary, pure Swift with WhisperKit + Ollama · *Why relevant:* Closest to Docknote's exact form factor (Swift+macOS); useful baseline for native shipping.
- **[Minutes (silverstein)](https://github.com/silverstein/minutes)** — 1.2k stars · last commit 2026-04 · "Conversation memory layer" that stores every meeting as Markdown in `~/meetings/` with MCP server; agents read directly · *Why relevant:* Most architecturally aligned with Docknote — Markdown-on-disk + MCP for cross-app recall is the shape Brain Memory needs.
- **[transcript-seeker (Meeting-BaaS)](https://github.com/meeting-baas/transcript-seeker)** — 2025+ · Browser-based transcript viewer/manager with bot integration, SQLite for calendar, chat with recordings · *Why relevant:* Reference for the UI of "browse + chat over a library of past meetings."
- **[OpenNotes (alexkroman)](https://github.com/alexkroman/opennotes)** — small/recent · 2025+ · Multi-ASR (Whisper Python/whisper.cpp/Assembly) open-source AI notetaker · *Why relevant:* Useful inventory of ASR backends; their cross-conversation pipeline is the cross-meeting layer Docknote is building.
- **[Pensieve (lukasbach)](https://github.com/lukasbach/pensieve)** — 109 stars · last commit 2025-09 · Local-only Mac desktop app records app audio, Whisper transcription, optional Ollama/ChatGPT summarization · *Why relevant:* Small but well-shaped reference for indie-scale Mac architecture.

## A.7 Persona / character / user-modeling

- **[Memobase](https://github.com/memodb-io/memobase)** — 2.7k stars · actively maintained · User-profile-based long-term memory for chatbots, profile-shaped not agent-shaped · *Why relevant:* The persona-dossier-card data model is essentially a Memobase user profile — borrow their schema for "what we know about Alex" cards.
- **[SillyTavern](https://github.com/SillyTavern/SillyTavern)** — 26.9k stars · last commit 2026-05 · Power-user LLM frontend; the de-facto playground for character/persona memory experiments · *Why relevant:* Most battle-tested ecosystem for "character that remembers years of interaction"; lorebook / character-card formalism informs persona dossier shape.
- **[SillyTavern Smart-Memory (senjinthedragon)](https://github.com/senjinthedragon/Smart-Memory)** — small but recent · 2026-04 · Multi-tier memory: long-term facts + within-session details + scene history + story arcs + away-recap · *Why relevant:* "Scene + arc + away-recap" decomposition maps onto "this meeting + this project + welcome-back briefing."
- **[SillyTavern-MemoryBooks (aikohanasaki)](https://github.com/aikohanasaki/SillyTavern-MemoryBooks)** — small · actively maintained · Auto-extracts JSON-structured scene summaries into lorebooks; multi-tier consolidation · *Why relevant:* JSON-structured scene-level memory entries — same shape as a per-meeting summary record that rolls up into entity dossiers.
- **[sillytavern-character-memory (bal-spec)](https://github.com/bal-spec/sillytavern-character-memory)** — small · recent · Auto-extract structured character memories into ST Data Bank for vector retrieval at gen time · *Why relevant:* "Extract structured facts at write time, vector-retrieve at read time" — the exact loop Brain Memory needs around each meeting.

## A.8 Recent (2025-2026) experiments / academic-adjacent

- **[Awesome-Memory-for-Agents (TsinghuaC3I)](https://github.com/TsinghuaC3I/Awesome-Memory-for-Agents)** — 2025-2026 · Curated paper list on agent memory (episodic/semantic/procedural taxonomies) · *Why relevant:* One-stop literature index — skim before locking the Docknote memory taxonomy.
- **[Awesome-Agent-Memory (TeleAI-UAGI)](https://github.com/TeleAI-UAGI/Awesome-Agent-Memory)** — 2026 · Systems + benchmarks + papers · *Why relevant:* Companion list with a benchmarks lens.
- **[Agent-Memory-Paper-List (Shichun-Liu)](https://github.com/Shichun-Liu/Agent-Memory-Paper-List)** — 2026 · Survey-paper-aligned bibliography "Memory in the Age of AI Agents" · *Why relevant:* Newest survey reading list; quickest scan of 2025-2026 frontier.

---

# B. Twitter / X — discourse 2025-2026

## B.1 Builder threads — "I built X / lessons from shipping memory"

- **[@DhravyaShah, 2025-10-06](https://x.com/DhravyaShah/status/1975244767199138216)** — Raised $3M for Supermemory; reflects on memory as "one of the hardest challenges in AI." · *Why relevant:* Founder narrative on what's hard about cross-app memory.
- **[@DhravyaShah, 2025-08-25](https://x.com/DhravyaShah/status/1948794857969123497)** — New Supermemory with universal memory + graph view + projects to "kill AI memory lock-in." · *Why relevant:* Validates the project/folder pivot; lock-in concern users will have about meeting data.
- **[@DhravyaShah, 2026-04](https://x.com/DhravyaShah/status/2045225774098403474)** — "your memory probably sucks — and it's not a recall problem, it's not an architecture problem, it's a fundamental problem with memory itself." · *Why relevant:* Frames the core builder lesson — what to *call* a memory matters as much as how you store it.
- **[@RHLSTHRM, 2025-12](https://x.com/RHLSTHRM/status/2036029170912809249)** — Built personal Claude Code agent with custom Telegram bridge + memory + cron, then ripped it out. · *Why relevant:* Bespoke memory layers are expensive to maintain; opt for batteries-included.
- **[@Mariandipietra, 2026-04](https://x.com/Mariandipietra/status/2045883043357839572)** — Hermes bug: recalled memory injected into the same user turn caused model to drift and answer the memory instead of the latest message. · *Why relevant:* Never inject recalled context where it can be confused with current user intent.
- **[@chrysb, 2026-03](https://x.com/chrysb/status/2043020014035570784)** — "Why long-term memory for LLMs remains unsolved" — Granola/Reflect operator. · *Why relevant:* Operator at adjacent meeting/notes startup naming the unsolved gap.
- **[@richmondalake, 2026](https://x.com/richmondalake/status/1998143569337491935)** — Day 66/100 of Agent series, end-to-end implementation showing prompt/context/memory engineering. · *Why relevant:* MongoDB DevRel building memory in the open.

## B.2 Critiques of mem0 / Letta / Zep / Cognee

- **[@mem0ai, 2026-01](https://x.com/mem0ai/status/2039041449854124229)** — "I read Claude Code's memory source code. This one limit silently deletes your agent's memory." · *Why relevant:* Direct callout of silent-truncation failure mode Brain Memory must avoid.
- **[@deshrajdry, 2025-12](https://x.com/deshrajdry/status/2020046706684096854)** — Mem0 founder: lost context after resets, truncated history, rising memory usage, flaky cross-session behavior. · *Why relevant:* Even mem0's CEO admits these failure modes.
- **[@courtlandleer, 2025-12](https://x.com/courtlandleer/status/2035750131262177553)** — Plastic Labs cofounder: "dishonest to present a LongMem score as a breakthrough — it's a 3-year-old benchmark." · *Why relevant:* Don't trust vendor benchmarks; build your own from real meetings.
- **[@charlespacker, 2024-12-26](https://x.com/charlespacker/status/1872254532115152968)** — Letta CEO: "memory in chatgpt has several problems: too noisy, saves stupid memories, pollutes the context window." · *Why relevant:* Memory needs strict admission criteria.
- **[@charlespacker, 2025-04-10](https://x.com/charlespacker/status/1910538831864230295)** — "biggest problem with openai's chatgpt memory system is that it's black box… raises the ceiling but lowers the floor." · *Why relevant:* Argues for *user-visible, editable* memory — informs dossier-card UX.
- **[@TDataScience, 2026-04](https://x.com/TDataScience/status/2046200208745271668)** — "None of these tools were designed for agent memory. They were designed for document retrieval at scale." (memweave) · *Why relevant:* Vector DBs ≠ memory.
- **[@dair_ai, 2025-12](https://x.com/dair_ai/status/2018765444702982395)** — "Beyond RAG for Agent Memory — RAG wasn't designed for agent memory. And it shows." · *Why relevant:* Why naive top-k breaks for memory.
- **[@taranjeetio, 2025-04-21](https://x.com/taranjeetio/status/1914343430450499710)** — Mem0 co-founder: "RAG is not the same as Memory." · *Why relevant:* Brain Memory isn't a document index — it's a stateful entity store.

## B.3 Anthropic / OpenAI / Google memory rollouts

- **[@simonw, 2025-08-22](https://x.com/simonw/status/1959019273634107872)** — "ChatGPT just shipped the exact memory feature I've always wanted — automatic memory scoped to a specific project." · *Why relevant:* Project-scoped memory as a UX win.
- **[Simon Willison blog, 2025-05-21](https://simonwillison.net/2025/May/21/chatgpt-new-memory/)** — "I really don't like ChatGPT's new memory dossier" — pelican costume contaminated by Half Moon Bay context. · *Why relevant:* Cardinal warning — global memory bleeds into prompts where users want isolation.
- **[@OpenAI, 2025-10](https://x.com/OpenAI/status/1978608684088643709)** — "ChatGPT can now automatically manage your saved memories — no more 'memory full.'" · *Why relevant:* Even OpenAI hit memory-size explosion in production.
- **[@mynamebedan, 2025-04-24](https://x.com/mynamebedan/status/1915583416176631826)** — "chatgpt persistent memory has one flaw: your system prompt doesn't matter now. stuck in brainrot zoomer mode." · *Why relevant:* Persistent style baked into memory overrides explicit instructions — Brain Memory needs system-prompt isolation.
- **[@NickADobos, 2025-04-12](https://x.com/NickADobos/status/1911294986064408868)** — Disable ChatGPT memory via temporary chat button. · *Why relevant:* Power users want one-click "no memory" mode.
- **[@trq212, 2025-12](https://x.com/trq212/status/2027109375765356723)** — "Auto-memory feature… Claude now remembers what it learns across sessions." · *Why relevant:* Auto-extraction vs. explicit save — an axis Brain Memory must choose on.
- **[@rohanpaul_ai, 2025-12](https://x.com/rohanpaul_ai/status/2028747394683437358)** — "Anthropic made Claude Memory free… launched a tool to move preferences from ChatGPT or Gemini." · *Why relevant:* Memory-portability is becoming table-stakes.
- **[Shlok Khemani blog, 2025-09-12](https://www.shloked.com/writing/claude-memory)** — Reverse-engineers Claude memory: blank-slate per chat, only invoked on request, raw conversation search rather than summaries. · *Why relevant:* Two opposite memory philosophies (Claude explicit-pull vs ChatGPT automatic-push) — Brain Memory must pick a stance.

## B.4 Notable AI engineer accounts on memory

- **[@hwchase17, 2026](https://x.com/hwchase17/status/2042978845347745871)** — "memory is just context… memory isn't a plugin (it's a harness)." (quoting Sarah Wooders) · *Why relevant:* Architectural reframe — co-design with the agent harness, not bolted on.
- **[@charlespacker, 2024-09-23](https://x.com/charlespacker/status/1838272900929065150)** — Announces Letta: "next frontier in AI is the stateful layer above base models — the memory layer / LLM OS." · *Why relevant:* Establishes "memory layer" as a category.
- **[@chipro, 2025-01-07](https://x.com/chipro/status/1876681640505901266)** — 8000-word agents note covering tools/planning/memory; promises memory deep-dive. · *Why relevant:* Canonical reference for memory in agent architecture.
- **[@karpathy, 2023-12-08](https://x.com/karpathy/status/1733299213503787018)** — "Hallucination is all LLMs do. They are dream machines." · *Why relevant:* Reminder that memory recall is reconstructive — Brain Memory must surface provenance.
- **[@omarsar0, 2026](https://x.com/omarsar0/status/2031426008285421933)** — "context engineering → harness engineering — build your own agent harness." · *Why relevant:* Memory as part of harness — informs build-vs-buy.
- **[@svpino, 2026](https://x.com/svpino/status/1998785305038712841)** — "Most agents are duct-taped: messages here, memory there, S3 artifacts, DB journal." · *Why relevant:* Validates that fragmented memory storage is a known pain — consolidate.
- **[@svpino, 2025-04-29](https://x.com/svpino/status/1917248279835972028)** — "Mem0 beats OpenAI's memory by 26%!" · *Why relevant:* Showcases LongMemEval as a marketing axis — and the hype cycle that follows.

## B.5 Meeting-tool memory discourse

- **[@cjpedregal, 2025-09-24](https://x.com/cjpedregal/status/1970927370061349196)** — Granola CEO: shared notes show up in others' Granola apps so you can chat with full project context. · *Why relevant:* Cross-meeting context as collaboration.
- **[@cjpedregal, 2025-05-08](https://x.com/cjpedregal/status/1920521724090515837)** — "Granola has become my second brain." · *Why relevant:* Direct social proof for "meetings as memory substrate."
- **[@meetgranola, 2025-06-05](https://x.com/meetgranola/status/1930667426582028471)** — File Uploads so Granola AI analyzes files alongside meetings. · *Why relevant:* Users want non-meeting context inside the meeting tool.
- **[@colecallinan, 2025-04-24](https://x.com/colecallinan/status/1915485161182974009)** — "Want every Granola call automatically synced to ChatGPT… search, summarize, ask across time, across projects." · *Why relevant:* Power user spelling out the exact cross-meeting recall pivot Brain Memory targets.
- **[@matthewcarano, 2025-06-30](https://x.com/matthewcarano/status/1939480723745812683)** — Manual "follow-up hack" wiring Granola → Claude → Linear. · *Why relevant:* Users duct-taping cross-meeting synthesis — Brain Memory can absorb as first-class.
- **[@crystalwidjaja, 2025](https://x.com/crystalwidjaja/status/2008849251426836512)** — Auto-exports Granola transcripts via Claude-coded agent into weekly reflection journals. · *Why relevant:* Power-user multi-meeting synthesis — dossier card pattern in disguise.
- **[@ku1deep, 2026](https://x.com/ku1deep/status/2036759082556850217)** — "Make Granola regret locking you out of your own meeting transcripts" (OpenOats). · *Why relevant:* Data ownership as competitive wedge.
- **[@aakashg0, 2026](https://x.com/aakashg0/status/1997114841216217441)** — "Meta paid to kill a $99 AI pendant — Limitless raised $33M for always-on memory at $19/mo." · *Why relevant:* Economics of personal-memory subscriptions, and the privacy aftermath.
- **[@PCMag, 2025](https://x.com/PCMag/status/2008142717499432988)** — Plaud Mac/PC application records call audio. · *Why relevant:* Lateral competition.
- **[@MattHartman, 2026](https://x.com/MattHartman/status/2042591174838583559)** — "Ghost Pepper… meeting transcription, 100% private local AI. Nothing leaves your computer." · *Why relevant:* Local-first meeting memory positioning.

## B.6 "Memory is the missing piece" / "context is all you need"

- **[@mem0ai, 2025-10](https://x.com/mem0ai/status/1975283152357560430)** — "Context Engineering is powerful but limited without Memory." · *Why relevant:* Viral pitch line for memory's necessity.
- **[@IntuitMachine, 2025-09-26](https://x.com/IntuitMachine/status/1973327811700998402)** — Anthropic context-engineering report top-10: "Treat Context as a Finite Resource." · *Why relevant:* Frames the engineering constraint Brain Memory operates under.
- **[@MaryamMiradi, 2025-11](https://x.com/MaryamMiradi/status/1989377220381720873)** — "Context Engineering: The #1 Skill for Building AI Agents in 2025." · *Why relevant:* Useful primer for team alignment.
- **[@Hesamation, 2025-11](https://x.com/Hesamation/status/1988750893957730396)** — Google whitepaper masterclass: "memory-as-a-tool pattern, context engineering, memory vs RAG, A2A." · *Why relevant:* "Memory-as-a-tool" pattern is exactly how Brain Memory should expose itself.
- **[@DataScienceDojo, 2025-11](https://x.com/DataScienceDojo/status/1989800577115525266)** — "True intelligence comes from how you assemble context." · *Why relevant:* Justifies investing in retrieval/synthesis quality.
- **[@teej_m, 2025-12](https://x.com/teej_m/status/1996270442374668465)** — "Is there a good top-down explanation of what great 'memory' looks like for agents? The problem feels under-specified." · *Why relevant:* Honest under-specification — opens a door for Brain Memory to publish its own taxonomy.
- **[@nicbstme, 2025-10-01](https://x.com/nicbstme/status/1973551051145154702)** — "RAG Obituary: Killed by Agents, Buried by Context Windows." · *Why relevant:* Question whether vector DB is the right substrate at all.

## B.7 Local-first / privacy-AI memory

- **[@weaviate_io, 2026](https://x.com/weaviate_io/status/2039727522498093471)** — Built Engram memory tool; Claude refused to use it because "MEMORY.md is always loaded — zero latency, zero tool calls, guaranteed in context." · *Why relevant:* Sometimes the file system + a single markdown beats any retrieval engine.
- **[@charliejhills, 2026](https://x.com/charliejhills/status/2035999601954865229)** — "Letta open-sourced the memory layer AI coding agents have always been missing — claude-subconscious watches every Claude Code session." · *Why relevant:* Background-observer pattern; meeting recorder is similarly a passive memory-builder.
- **[@LiorOnAI, 2025-11](https://x.com/LiorOnAI/status/2008161724902355118)** — Claude-Mem free OSS plugin: "saves context so Claude resumes work without re-explaining everything." · *Why relevant:* Resumption-without-re-explanation is the #1 user value Brain Memory must deliver.

## B.8 Failure modes — hallucination, stale, poisoning, explosion

- **[@kalyan_kpl, 2025-11](https://x.com/kalyan_kpl/status/1988997157572231537)** — "Halumem benchmark to evaluate hallucinations in memory systems." · *Why relevant:* Memory-hallucination is now its own benchmark category.
- **[@rohanpaul_ai, 2025](https://x.com/rohanpaul_ai/status/2006305580734947499)** — "MemR3 — most agent memory work fails because the agent does 1 retrieval, gets a messy pile, then guesses anyway." · *Why relevant:* Single-shot retrieval is anti-pattern; Brain Memory needs iterative/agentic retrieval.
- **[@ksg93rd, 2026](https://x.com/ksg93rd/status/2003065588516593733)** — MemoryGraft: persistent compromise via poisoned experience retrieval. · *Why relevant:* Memory-poisoning research — important for any always-on recorder.
- **[@rohit4verse, 2026](https://x.com/rohit4verse/status/2049199654626263281)** — "Harrison Chase walked through 4 ways to give an agent memory. All four assume the model is still holding the right tokens. At token 4,096 the cache ran a silent eviction." · *Why relevant:* Critique of every existing memory pattern — silent-eviction is the lurking bug.
- **[@Indy_triguy, 2025-06-15](https://x.com/Indy_triguy/status/1933289906484453704)** — "Mem0 also reporting problems due to AWS outage." · *Why relevant:* Cloud-memory dependency = downtime risk.
- **[@senthilnayagam, 2026-01](https://x.com/senthilnayagam/status/1991204282222493890)** — "Antigravity possibly has memory leak… eventually restarted my MacBook Air." · *Why relevant:* On-device memory implementations have system-resource failure modes.

## B.9 Notable ancillary

- **[@AndrewYNg, 2025-03-19](https://x.com/AndrewYNg/status/1902395485601853941)** — DeepLearning.AI short course: Long-Term Agentic Memory with LangGraph (Harrison Chase). · *Why relevant:* Curriculum-level overview for team onboarding.
- **[@AndrewYNg, 2024-11-07](https://x.com/AndrewYNg/status/1854587401018261962)** — "LLMs as Operating Systems: Agent Memory" course w/ Letta. · *Why relevant:* Set the "LLM OS" framing.
- **[@GoogleResearch, 2026](https://x.com/GoogleResearch/status/2046631948437921801)** — "ReasoningBank — agent memory framework that learns from successful AND failed experiences." · *Why relevant:* Failed experiences is the missing piece for meeting-followup learning loops.
- **[@cameron_pfiffer, 2026](https://x.com/cameron_pfiffer/status/2022365625637904683)** — Letta memfs: hierarchical memory + progressive disclosure + parallel sleep-time memory agents. · *Why relevant:* "Sleep-time memory agents" — consolidation-while-idle Brain Memory should adopt for nightly dossier synthesis.
- **[@the_smart_ape, 2026](https://x.com/the_smart_ape/status/2041847540455502199)** — "mempalace solved agent memory: store everything, then make it searchable." · *Why relevant:* Maximalist storage — pairs with always-on meeting recording.
- **[@Letta_AI, 2026](https://x.com/Letta_AI/status/2041250583030694278)** — "Letta Code: built for memory-first agents." · *Why relevant:* "Memory-first" as product framing.

X searches that surfaced leads but no clean URLs (rerun via X native search):
- `"shipped memory" lessons agent`
- `"memory layer broke" production agent`
- `"second brain" Granola transcript chat`
- `Limitless Meta acquisition memory privacy`

---

# C. Products + engineering blogs + papers

## C.1 Meeting-recording / notetaking products

- **[Granola — folder query feature](https://www.granola.ai/)** — 2026-01 · product · Mac AI notepad, folders by people/orgs, Q&A across meetings with citations · *Why relevant:* Closest competitive analog; folders + cross-meeting Q&A + citations is table-stakes UX.
- **[Granola — local encrypted DB on Mac](https://www.shadow.do/blog/granola-encrypted-its-local-database-heres-why-that-matters----and-what-to-use-instead)** — 2026 · blog · Granola encrypts SQLite at `~/Library/Application Support/Granola`, breaking 3rd-party MCPs · *Why relevant:* Sets privacy bar; encrypted-at-rest from day 1.
- **[Granola Security & Privacy page](https://www.granola.ai/security)** — 2025 · product · transcripts in US AWS VPC, audio not retained · *Why relevant:* Reference for audio-deletion / transcript-only retention policy.
- **[Limitless Pendant](https://www.limitless.ai/new)** — 2025-12 · product · wearable conversation recorder with "perfect recall" + speaker-attributed cross-conversation search; Meta acquired 2025 · *Why relevant:* How far always-on memory can go and why Meta-acquisition risks make a Mac-native alternative attractive.
- **[Otter AI Chat 2.0 + MCP server](https://help.otter.ai/hc/en-us/articles/19682180167575-Otter-AI-Chat-Overview)** — 2026 · product · Q&A across full library + MCP server · *Why relevant:* MCP exposure is strong pattern — Docknote should ship one.
- **[Otter Sales Agent](https://otter.ai/sales-agent)** — 2026 · product · pulls past CRM context, updates Salesforce/HubSpot from meeting memory, MEDDPICC/BANT in real time · *Why relevant:* Cross-meeting memory plus structured outputs creates value beyond search.
- **[Fireflies AskFred](https://guide.fireflies.ai/articles/1102961402-how-to-use-askfred-to-search-and-get-answers-from-all-past-meetings)** — 2024 · product · ChatGPT-style Q&A across full transcript library · *Why relevant:* Reference UX; critics call out lack of org-wide memory.
- **[Plaud "Ask Plaud" + Multidimensional Summaries](https://www.plaud.ai/)** — 2025 · product · hardware recorder; role-specific summaries (sales/manager/leadership) from one conversation · *Why relevant:* Same recording reused for different downstream views — valuable pattern for dossier cards.
- **[Plaud App 3.0 / Plaud Intelligence](https://www.plaud.ai/blogs/news/plaud-app-3-design-intelligence)** — 2025 · blog · multimodal-input → multidimensional-summary → Ask Plaud · *Why relevant:* Public engineering description of a hardware-meeting-app's memory pipeline.
- **[Read.ai Search Copilot](https://www.read.ai/post/read-ai-launches-search-copilot-an-industry-first-ai-tool-for-cross-platform-search-and-discovery--at-no-cost)** — 2025 · product · cross-platform search across meetings + email + chat + CRM, free tier · *Why relevant:* Sets pricing pressure; Docknote's local-first edge needs a clear value-add.
- **[Fathom 3.0 — Ask Fathom + Claude/ChatGPT](https://chatgate.ai/post/fathom-3-0)** — 2025 · product · bot-free capture, native ChatGPT/Claude bridge · *Why relevant:* "Bot-free" capture and external LLM bridges are baseline expectations.
- **[tl;dv "organizational memory"](https://tldv.io/blog/best-ai-meeting-assistants/)** — 2026 · blog · meetings as "company memory" you can query like a single brain · *Why relevant:* Framing Docknote should adopt: not "transcripts" but "organizational memory."
- **[Spinach AI institutional memory](https://www.spinach.ai/)** — 2025 · product · auto-creates Jira/Asana tickets from action items · *Why relevant:* Action-item extraction wired into external task systems.
- **[Krisp AI Note Taker](https://krisp.ai/ai-note-taker/)** — 2025 · product · audio-layer capture (no bot), pre-meeting briefs · *Why relevant:* "Pre-meeting brief" UX is direct pattern for dossier cards.
- **[Notta](https://www.notta.ai/en/)** — 2025 · product · search by keyword/speaker/date plus AI chat · *Why relevant:* Confirms speaker- and date-scoped search filters as expected pivots.
- **[MeetCard AI networking app](https://meetcard.ai/)** — 2025 · product · "recall the CEO you met last week" — person-centric meeting memory · *Why relevant:* Validates the "dossier card per person" UX directly.

## C.2 Personal AI memory products

- **[ChatGPT Memory & controls](https://openai.com/index/memory-and-new-controls-for-chatgpt/)** — 2024-02 · product · saved memories + chat-history reference; user can view/edit/delete · *Why relevant:* User-control baseline.
- **[OpenAI Memory FAQ](https://help.openai.com/en/articles/8590148-memory-faq)** — 2025 · doc · official UX/permissions doc · *Why relevant:* Reference for "what do you remember about me?" affordance.
- **["Reference saved memories"](https://help.openai.com/en/articles/11146739-how-does-reference-saved-memories-work)** — 2025 · doc · explicit vs implicit memory · *Why relevant:* Two-tier model is proven pattern Docknote can mirror.
- **[ChatGPT Pulse](https://openai.com/index/introducing-chatgpt-pulse/)** — 2025-09 · product · proactive overnight research using memory + chats + Gmail/Calendar · *Why relevant:* Memory unlocks proactive briefings.
- **[Claude chat search & memory](https://support.claude.com/en/articles/11817273-use-claude-s-chat-search-and-memory-to-build-on-previous-context)** — 2025-08 · doc · daily-summarizing chat history · *Why relevant:* Different from OpenAI — server-side rolling summary rather than extracted facts.
- **[Anthropic context management](https://www.anthropic.com/news/context-management)** — 2025-09 · blog · context editing + memory tool for Claude API · *Why relevant:* First-party reference for tool-based persistent memory; 84% token reduction.
- **[Gemini "memories" rollout](https://ppc.land/gemini-brings-memories-and-chat-import-to-uk-users/)** — 2026-04 · product · auto-memory + chat-history import · *Why relevant:* Chat-history import useful when ingesting legacy meeting archives.
- **[Pi by Inflection](https://inflection.ai/)** — 2024 · product · companionship-oriented memory · *Why relevant:* Affective long-term memory UX — dossier cards may benefit.
- **[Mem 2.0 launch](https://get.mem.ai/blog/introducing-mem-2-0)** — 2024 · blog · ground-up rebuild as "AI thought partner" · *Why relevant:* AI-first PKM trajectory; cautionary tale on focus vs. scope.
- **[Reflect Notes AI](https://reflect.app/)** — 2025 · product · AI uses backlink graph for cross-note synthesis; Whisper transcription · *Why relevant:* "Use the graph, not just chunks" — memory-aware retrieval respects existing structure.
- **[Heptabase chat-with-whiteboards](https://wiki.heptabase.com/newsletters/2025-07-23)** — 2025-07 · product · self-hosted embedding model, AI suggestions for related cards · *Why relevant:* Self-hosted embeddings keep local-first promise while offering semantic recall.
- **[Tana supertags + AI meeting agent](https://outliner.tana.inc/docs/tana-ai)** — 2025 · product · structured supertags, meeting supertag auto-detects Persons/Orgs/Tasks/Locations · *Why relevant:* Strong model for typed entity extraction reusable in dossier cards.
- **[Tana voice chat for iOS](https://tana.inc/outliner/articles/talk-through-your-ideas-with-tana-ai-voice-chat-for-ios)** — 2025 · product · voice-first interaction with custom workflows · *Why relevant:* Voice into structured memory — natural fit for meeting-recording.
- **[Roam Live AI Assistant](https://github.com/fbgallet/roam-extension-live-ai-assistant)** — 2025 · product · agentic chat over Roam graph including DNPs, sidebar, queries, PDFs · *Why relevant:* Users want to compose their own context windows from selective subsets.
- **[Personal.ai memory](https://www.personal.ai/memory)** — 2024 · product · fine-tunes on user data · *Why relevant:* Different model than retrieval; useful contrast for local-RAG vs train approach.
- **[Supermemory.ai](https://supermemory.ai/)** — 2025 · product · four-layer memory (working/episodic/semantic/procedural) as API · *Why relevant:* Directly shippable backend if Docknote chooses not to build in-house.

## C.3 Engineering blog posts

- **[Anthropic — Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)** — 2025 · blog · official engineering writeup on memory + compaction + tool clearing · *Why relevant:* Canonical Anthropic-side reference for budgeting context across long agent runs.
- **[Anthropic Memory tool docs](https://docs.claude.com/en/docs/agents-and-tools/tool-use/memory-tool)** — 2025 · doc · spec for `/memories` filesystem tool · *Why relevant:* Wire format if Docknote agent talks to Claude.
- **[Mem0 — Memory in Agents: What, Why and How](https://mem0.ai/blog/memory-in-agents-what-why-and-how)** — 2024 · blog · taxonomy + lifecycle · *Why relevant:* Good framing doc for team vocabulary.
- **[AWS Database Blog — Mem0 + Valkey + Neptune](https://aws.amazon.com/blogs/database/build-persistent-memory-for-agentic-ai-applications-with-mem0-open-source-amazon-elasticache-for-valkey-and-amazon-neptune-analytics/)** — 2025 · blog · production architecture: vector + graph + KV hybrid · *Why relevant:* Deployable reference for hybrid-store memory backend.
- **[Letta — Rearchitecting the Agent Loop](https://www.letta.com/blog/letta-v1-agent)** — 2025 · blog · ReAct, MemGPT, Claude Code lessons applied · *Why relevant:* Lessons-learned from the team that pioneered tiered memory.
- **[Letta — Agent Memory: Build Agents that Learn and Remember](https://www.letta.com/blog/agent-memory)** — 2024 · blog · core/recall/archival blocks · *Why relevant:* Cleanest articulation of OS-style memory hierarchy.
- **[Letta — Benchmarking AI Agent Memory: Is a Filesystem All You Need?](https://www.letta.com/blog/benchmarking-ai-agent-memory)** — 2025 · blog · empirical comparison of memory backends · *Why relevant:* Direct evidence on whether Docknote can ship "just files" vs. building a full vector store.
- **[Zep — Temporal Knowledge Graph Architecture](https://blog.getzep.com/zep-a-temporal-knowledge-graph-architecture-for-agent-memory/)** — 2025-01 · blog · bi-temporal model, conflict resolution · *Why relevant:* If Docknote represents people/projects as graph entities, this solves "who said what when."
- **[Zep — Graphiti for Agentic Apps](https://blog.getzep.com/graphiti-knowledge-graphs-for-agents/)** — 2024 · blog · OSS framework, Neo4j-backed · *Why relevant:* Off-the-shelf option for entity-relationship memory.
- **[Zep — State of the Art in Agent Memory](https://blog.getzep.com/state-of-the-art-agent-memory/)** — 2025 · blog · LongMemEval/DMR benchmark results · *Why relevant:* Headline numbers calibrate expectations.
- **[Neo4j — Graphiti: KG Memory for an Agentic World](https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/)** — 2025 · blog · DB-vendor view · *Why relevant:* Concrete schema patterns for entity-edge memory.
- **[Cognee — How Cognee Builds AI Memory](https://www.cognee.ai/blog/fundamentals/how-cognee-builds-ai-memory)** — 2025 · blog · ECL pipeline; production at 70+ companies · *Why relevant:* Simpler 6-line-of-code SDK; useful as quick prototype path.
- **[Cognee — Memory Fragment Projection](https://www.cognee.ai/blog/deep-dives/memory-fragment-projection-from-graph-databases)** — 2025 · blog · personalized projections of a shared KG · *Why relevant:* Pattern for per-user views over a team-wide meeting graph.
- **[Notion — Speed, Structure, Smarts](https://www.notion.com/blog/speed-structure-and-smarts-the-notion-ai-way)** — 2025 · blog · official engineering on AI architecture · *Why relevant:* Block-structured retrieval is exactly the model Docknote should use over speaker-segmented transcripts.
- **[OneHouse — Notion's data-scale journey](https://www.onehouse.ai/blog/notions-journey-through-different-stages-of-data-scale)** — 2024 · blog · data lake + embeddings infra evolution · *Why relevant:* Realistic infra-evolution path solo → team → enterprise.
- **[Zilliz — Notion's vector search](https://zilliz.com/blog/notion-vector-search-next-problem)** — 2025 · blog · forward-looking on Notion's vector-search limits · *Why relevant:* Highlights "next problem" (memory + offline context) emerging after basic semantic search.
- **[Supermemory — Memory engine inspired by the human brain](https://supermemory.ai/blog/memory-engine/)** — 2025 · blog · custom vector+graph engine, ontology-aware edges, decay · *Why relevant:* Brain-inspired decay/consolidation translates well to "I forgot what we discussed three months ago" UX.
- **[Supermemory — How Perplexity Memory Works](https://blog.supermemory.ai/how-perplexity-memory-works/)** — 2025 · blog · third-party reverse-engineering · *Why relevant:* Counter-reference: search-first product handles memory differently from chat-first.
- **[Embrace The Red — How ChatGPT Remembers You](https://embracethered.com/blog/posts/2025/chatgpt-how-does-chat-history-memory-preferences-work/)** — 2025 · blog · independent reverse-engineering · *Why relevant:* Most accurate non-OpenAI description of actual context-window injection format.
- **[Simon Willison — I really don't like ChatGPT's new memory dossier](https://simonwillison.net/2025/May/21/chatgpt-new-memory/)** — 2025-05 · blog · critical review · *Why relevant:* Failure modes Docknote must avoid (creepy implicit profiles, unwanted context bleed).
- **[Shlok Khemani — ChatGPT Memory and the Bitter Lesson](https://www.shloked.com/writing/chatgpt-memory-bitter-lesson)** — 2025 · blog · OpenAI dumps everything in context vs. clever retrieval · *Why relevant:* Forces a real architectural choice for Docknote.
- **[NocoBase — How Plaud Built Internal Systems](https://www.nocobase.com/en/blog/plaud)** — 2025 · blog · Plaud's no-code internal stack · *Why relevant:* Tangential — non-AI infra choices a meeting-hardware company makes.

## C.4 Papers (2024-2026)

- **[MemGPT (Letta)](https://arxiv.org/abs/2310.08560)** — 2023-10 · paper · LLMs as OS managing tiered memory · *Why relevant:* Foundational; cleanest mental model for cross-meeting recall.
- **[Mem0 — Production-Ready AI Agents with Scalable Long-Term Memory](https://arxiv.org/abs/2504.19413)** — 2025-04 · paper · two-phase extract+update; beats baselines on LoCoMo · *Why relevant:* Concrete reference architecture with benchmark numbers.
- **[Zep — Temporal KG Architecture for Agent Memory](https://arxiv.org/abs/2501.13956)** — 2025-01 · paper · bi-temporal graph; beats MemGPT on DMR · *Why relevant:* Canonical citation if Docknote chooses graph-based memory.
- **[A-MEM: Agentic Memory for LLM Agents](https://arxiv.org/abs/2502.12110)** — 2025-02 · paper · NeurIPS 2025; Zettelkasten-inspired self-organizing memory · *Why relevant:* Auto-link/cross-reference — useful for surfacing "did we discuss this with someone else?"
- **[Generative Agents (Park et al.)](https://arxiv.org/abs/2304.03442)** — 2023-04 · paper · memory stream + reflection + retrieval (recency, importance, relevance) · *Why relevant:* Retrieval scoring formula is a great default.
- **[HippoRAG (NeurIPS 2024)](https://arxiv.org/abs/2405.14831)** — 2024-05 · paper · hippocampal-indexing-inspired RAG with Personalized PageRank · *Why relevant:* Cheap and effective for cross-meeting multi-hop questions.
- **[HippoRAG 2 / From RAG to Memory](https://arxiv.org/abs/2502.14802)** — 2025-02 · paper · improved associative + sense-making memory · *Why relevant:* Practical successor with better latency/cost.
- **[MemoryBank (AAAI 2024)](https://arxiv.org/abs/2305.10250)** — 2023-05 · paper · Ebbinghaus-curve forgetting · *Why relevant:* Concrete decay model Docknote can adopt.
- **[Survey on Memory Mechanism of LLM-Based Agents](https://arxiv.org/abs/2404.13501)** — 2024-04 · paper · ACM TOIS comprehensive survey · *Why relevant:* Best single starting-point read.
- **[From Human Memory to AI Memory: A Survey](https://arxiv.org/abs/2504.15965)** — 2025-04 · paper · maps human memory psychology to LLM systems · *Why relevant:* Useful framing for "Brain Memory" branding.
- **[Memory for Autonomous LLM Agents — Mechanisms, Eval, Frontiers](https://arxiv.org/html/2603.07670v1)** — 2026-03 · paper · most recent survey · *Why relevant:* Latest taxonomy + open problems for roadmap.
- **[Position: Episodic Memory is the Missing Piece](https://arxiv.org/abs/2502.06975)** — 2025-02 · paper · 5 properties of episodic memory · *Why relevant:* Spec a meeting-memory system must satisfy.
- **[EM-LLM: Human-inspired Episodic Memory (ICLR 2025)](https://arxiv.org/abs/2407.09450)** — 2024-07 · paper · Bayesian-surprise event boundaries; beats RAG on LongBench · *Why relevant:* Event-segmentation maps directly to chunking long meetings.
- **[Larimar (ICML 2024)](https://arxiv.org/abs/2403.11901)** — 2024-03 · paper · IBM/Princeton; one-shot editable distributed episodic memory · *Why relevant:* Path to user-editable AI memory at parameter level.
- **[MIRIX: Multi-Agent Memory System](https://arxiv.org/abs/2507.07957)** — 2025-07 · paper · 6 memory types (Core, Episodic, Semantic, Procedural, Resource, Knowledge Vault) coordinated by Meta Memory Manager · *Why relevant:* Most modular memory taxonomy in literature; maps onto dossier-card structure.
- **[MemMachine: Ground-Truth-Preserving Memory](https://arxiv.org/abs/2604.04853)** — 2026-04 · paper · stores entire conversational episodes; 80% fewer tokens than Mem0 · *Why relevant:* Counterargument to extraction-based — keep raw episodes, project on demand.
- **[LongMemEval (ICLR 2025)](https://arxiv.org/abs/2410.10813)** — 2024-10 · paper · 500 questions over multi-session chat; 30-60% accuracy drop on long-context LLMs · *Why relevant:* Benchmark Brain Memory should evaluate against.
- **[LoCoMo: Very Long-Term Conversational Memory (ACL 2024)](https://arxiv.org/abs/2402.17753)** — 2024-02 · paper · 35-session, 9K-token conversations · *Why relevant:* Companion benchmark; tests temporal reasoning across sessions.
- **[CoALA: Cognitive Architectures for Language Agents](https://arxiv.org/abs/2309.02427)** — 2023-09 · paper · working/long-term/episodic/semantic/procedural taxonomy · *Why relevant:* Conceptual scaffolding before naming components.
- **[Reflexion: Language Agents with Verbal RL](https://arxiv.org/abs/2303.11366)** — 2023-03 · paper · self-reflection stored as memory · *Why relevant:* Useful for "lessons learned" notes attached to recurring meeting series.
- **[RAPTOR (ICLR 2024)](https://arxiv.org/abs/2401.18059)** — 2024-01 · paper · recursive cluster-and-summarize tree retrieval · *Why relevant:* Strong fit for hierarchical meeting summaries (meeting → week → quarter rollups).
- **[GraphRAG: From Local to Global](https://arxiv.org/abs/2404.16130)** — 2024-04 · paper · Microsoft KG + community-summary RAG · *Why relevant:* Necessary for global questions ("what themes have we discussed this quarter?").
- **[Microsoft Research — GraphRAG project](https://www.microsoft.com/en-us/research/project/graphrag/)** — 2024 · blog · official MSR home · *Why relevant:* Open-source code path if Docknote builds graph-based memory in-house.
- **[Learning to Forget: Sleep-Inspired Memory Consolidation (SleepGate)](https://arxiv.org/abs/2603.14517)** — 2026-03 · paper · KV-cache forgetting gate · *Why relevant:* Addresses "outdated info clutters retrieval" — crucial for long-running personal histories.
- **[SCM: Sleep-Consolidated Memory](https://arxiv.org/html/2604.20943)** — 2026-04 · paper · multi-dimensional importance + intentional forgetting · *Why relevant:* Practical forgetting policy alternative to MemoryBank.
- **[Language Models Need Sleep (OpenReview)](https://openreview.net/forum?id=iiZy6xyVVE)** — 2025-10 · paper · self-modify-during-dreaming · *Why relevant:* Speculative; offline "consolidation pass" Docknote could run nightly.
- **[GAM: Hierarchical Graph-based Agentic Memory](https://arxiv.org/html/2604.12285)** — 2026 · paper · semantic-event-triggered consolidation, episodic buffering · *Why relevant:* Two-phase encode→consolidate is a clean pipeline architecture.
- **[AriGraph (IJCAI 2025)](https://www.ijcai.org/proceedings/2025/0002.pdf)** — 2025 · paper · KG world model with episodic memory · *Why relevant:* Combines world-model + episodic.
- **[Anatomy of Agentic Memory](https://arxiv.org/html/2602.19320v1)** — 2026-02 · paper · taxonomy + system limitations under evaluation · *Why relevant:* Identifies eval pitfalls Docknote should avoid.

---

## Notes on the sweep

- **Coverage gaps:** Twitter URLs are fragile (some 202x.x format IDs may not resolve); rerun via X native search if a specific quote needs verification.
- **Cross-references:** Letta / Mem0 / Granola / Plaud / Limitless appear in multiple sweeps (repo + blog + tweet) — different vantage points on same target. When deep-diving, pull from all three.
- **Deliberately excluded:** generic LangChain/LlamaIndex memory wrappers (well-documented elsewhere), pure RAG frameworks without a memory-layer claim, dead repos with no commits in 18+ months unless historically significant (e.g., MemGPT pre-Letta, Generative Agents 2023).
- **Decision points the literature forces:**
  1. Extract-and-store (Mem0/MemoryBank) vs. keep-raw-and-project (MemMachine/Claude rolling-summary).
  2. Filesystem (Letta benchmarks, Claude Code) vs. KG (Graphiti/Zep) vs. vector (Chroma/LanceDB) vs. hybrid (MemOS/open-mem).
  3. Implicit auto-extraction (ChatGPT, Mem0) vs. explicit user-edited (Cline memory bank, Claude memory tool).
  4. Per-session blank slate (Claude) vs. always-loaded global (ChatGPT).
  5. Cloud vs. local-first (privacy moat) vs. hybrid (Granola: local-only DB but cloud LLM calls).

Each of these maps to a Brain Memory v8 SDD decision. Pick deep-dive targets that span the axes you're least sure on.
