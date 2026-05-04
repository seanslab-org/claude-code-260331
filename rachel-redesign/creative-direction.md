# Creative Direction — Brain Memory v9

**Director's notes for the team**
Rachel · 2026-05-04

---

## North star

**Brain Memory v9 should feel like a well-kept commonplace book that quietly turned the page while you slept — and is waiting for you to read it before it presumes to know.**

Not a database. Not an inbox. A book that has been written in, by you and by something patient that watches the meetings happen. The book is paper, the ink is real, the marginalia are dated. When the brain proposes an edit, it proposes it the way a careful reader passes you a sticky note — not the way Slack pings you about an unread thread.

That's the whole brief in one image. Everything below is in service of it.

---

## What I'm carrying forward (no relitigation)

The v8 surface is right. Ribbon left, dossier card and synthesis paragraph in conversation on the right, fragment rows full-width below. The teal is right — the strong variant `#0097a7` reads as ink-blue in print and as authority on screen, and the soft `#E0F2F4` glows like a watercolour wash without ever shouting. The serif (Iowan Old Style → Charter → Georgia) is the voice of the artefact; the system font is the voice of the chrome. Citations live in fragment rows, never inline `[n]`. We honour Granola's pattern there because it's correct: the synthesis is one breath, the receipts are a second breath, never interrupting each other.

I am not redesigning any of this. I'm extending it into the seven surfaces the audit identified.

---

## Visual story

The brain has two registers — **present-tense** (warm) and **past-tense** (cool, faded, slightly desaturated). Today's meeting pulses; a 90-day-old `age_band: background` claim sits at 60% opacity in a serif italic, the way a margin note from a year ago looks when you flip back. This isn't decorative. It's diegetic. The user can read the age of a fact off its rendering without parsing a date.

The Brain Inbox is the most important new scene. It must not feel like email. It should feel like the morning desk: a small stack of suggestions on cream paper, each one with a quoted line from a real meeting beneath it, each one with two affordances — keep or set aside — and nothing else. No bulk-actions toolbar. No counters. No "process inbox in 2 minutes" gamification. The user reads them at their own pace and the queue waits.

Time-travel is the second important scene. It is not a "history view" — that framing kills it. It's the dossier card learning what it used to be. The card sits still; the user drags a slider; the fields rewind in place, with the words being added crossfading over the words being removed at field level, never globally. The Tide ribbon (which is meeting-time) recedes during time-travel; a second slim ribbon appears across the top of the card, in a slightly cooler grey, marking *card-time*. Two timelines, never confused, never on screen at the same intensity.

Geomi is the third scene worth thinking hard about. The audit's brief is right: ghost genius, never begs. My addition: Geomi's bubble has *no border by default*. It is text + a small breathing dot + a single first-action verb, set in the same serif voice. It looks like marginalia, not chrome. The "×" to dismiss is so small it's almost a discovery — and the dismissed state isn't replaced with a "Geomi was here" placeholder. It is gone. (This is the Pixar restraint: the bubble was never demanding your attention; if you didn't want it, it never imposed.)

---

## Color and material

The v8 palette is intact. I'm proposing four state semantics on top of it that the v8 mockups already half-imply but never declared:

- **Warm ink** (`--ink: #1F1F1E`, full saturation) — present, authoritative, the user wrote this or the auto-extractor wrote it within the last 14 days.
- **Faded ink** (`--ink-soft: #5A5854` at 80%) — older than 14 days, still load-bearing.
- **Background ink** (`--ink-faint: #94918B` italic) — `age_band: background`, more than 90 days old, kept because the user might want it but visually quiet.
- **Brand-strong** (`#0097a7`) — entity references, decisions, anything that anchors a citation. Never used as a "click me" cue; only as a *recognition* cue.

Two new material moves for v9:

1. **The lock state has its own paper-finish.** A `human_locked: true` field gets a faint cream-on-cream stripe behind it — not a yellow highlight, not a coloured border — the texture of vellum laid over the underlying paper. It says *this has been read and approved*, not *do not touch*.
2. **The Brain Inbox uses a slightly different paper than the canvas.** The canvas paper is `#F7F5EF`; the Inbox paper is `#F4F1E8` (one notch warmer, a hair more saturated). It's the same brain on the same desk, but the inbox stack is a different sheaf. This is the move *Heptabase* makes between today's note and your weekly review, and *Reflect* makes between the editor and the daily log.

I'm not adding new accent colours. v8 has too many already (six fragment-stripe hues). The audit's instinct to keep the palette tight is correct.

---

## Motion language

Ink takes a moment to dry. Geomi breathes. Cards settle.

Concretely:

- **Idle breathing** on the Geomi orb is the only continuous animation on screen. 3.4s cycle, scale 1.0 → 1.04, opacity 0.55 → 0.30. (The v8 Tide pulse already does this for today's meeting; we extend the same rhythm to Geomi so they breathe together.)
- **Field commit** — when the user accepts a proposed update, the new value crossfades in over 240ms with a tiny vertical settle (4px → 0px translateY). Old value drifts down 4px and fades. No bounce, no spring. Ink dries.
- **Slider scrub** in time-travel — fields update at 60fps with the card's frontmatter; the slider thumb leaves a faint trail (6px, 8% brand) for 300ms after release so the user can see where they were. This is the *only* place I want a trail in the entire product. It earns it because the surface is explicitly about motion through time.
- **Lock toggle** — when a field is locked, the vellum stripe doesn't slide on; it *develops* — a 320ms opacity ramp like a Polaroid coming up. The unlock is faster (160ms) and more declarative.
- **Inbox accept** — the card doesn't disappear with a swoosh. It writes itself into the dossier behind it. The proposed-update card slides 8px right and fades; behind it, the dossier field updates in place with the same crossfade. The user sees the diff *land* in the brain. This is the moment that converts the inbox from "task" to "ritual."

Everything else is still or near-still. Provenance badges fade in on `:hover` in 140ms; that's the v8 pattern and I'm preserving it.

---

## Character moments per surface

**Brain Inbox.** The framing line at the top is *"The brain learned things while you slept."* (Not "5 unread updates." Never that.) Below it: a date — last night's pass, with timestamp. Below that: a small italic line that says how many things the brain wants you to know about, written as prose: *"Three rewrites and a new theme."* Then the cards. At the bottom of the queue, when it's empty: *"Nothing tonight. Sleep well."* — and that's the whole empty state. (The system is allowed to like the user. It's not allowed to flatter them.)

**Hearth edit states.** Hovering a field reveals a single hairline pencil glyph at the right edge of the field — 11px tall, ink-faint, no chrome, no tooltip. Click and the field becomes a borderless contenteditable with the same typography it had a moment ago. The provenance badge below the field — `auto-extracted · 2 days ago · qwen-3-32b` — is always visible at 50% opacity and rises to 100% on hover. The lock icon is a tiny serif glyph (a paragraph mark turned ninety degrees would work, or just a hairline padlock) that sits in the badge row, never on the field itself. The lock isn't decoration *on the field* — it's metadata *about* the field, and it lives where metadata lives.

**Time-travel.** The slider is along the *top* of the card, not below or beside it. Why: the user's eye reads top-to-bottom, and the timeline is the *condition* for the card's content, not a control adjacent to it. The slider has three to five tick marks on it that correspond to commits that actually changed something on this card (not all commits — only meaningful ones). Hovering a tick reveals a small label: *"Apr 12 — you set the differentiator"* / *"Mar 28 — auto-extractor opened a new role line."* These are the chapter marks of the card's life. The "revert this field" button only appears when you've scrubbed somewhere and a field differs from current; otherwise the affordance is invisible. (The card is not asking to be reverted.)

**Brain settings (`active.md` / `persona.md` preview).** The framing here is the most important micro-decision in the brief. Not *"Memory contents"* (database language). Not *"Your profile"* (LinkedIn language). The two sections are titled, in serif, *"What I always know about you"* and *"What's on your mind this fortnight."* These are the sentences that should appear on the wall in the team room. They name the relationship correctly: the brain has read about you and is currently thinking about your last two weeks. Both files are shown as previews — section headers, line counts, last-updated. Both are editable inline. There's a small "open in editor" affordance for power users, set in monospace as a wink: *`vim ~/Docknote/brain/active.md`*. (Only show that if the user has revealed the developer panel in preferences. We don't want to scare anyone with a vim incantation on first run.)

**Geomi end-of-day nudge.** Lives in the bottom-right of the Brain scene. Three lines max, set in serif, italics for the warmest line. Example I drafted: *"Two things didn't settle today — the Q3 pricing call back to Alex, and your Friday with Pat. Want me to put them on tomorrow's brief?"* with a single first-action verb: **Brief tomorrow** (in brand-strong, the only chrome). Dismiss is a hairline ×. If dismissed, gone.

**Geomi mid-meeting suggestion.** Lives in the corner of the live transcript view. Even quieter — two lines max. Example: *"Alex is referring to the pricing decision from March 14."* — first-action verb: **Show the call**. Same dismiss rule. Critical: the bubble does *not* contain the answer. The bubble is a finger pointing; the answer lives on the next surface. This keeps Geomi from doing the user's reading for them, which would be both presumptuous and a privacy footgun in a meeting context.

**Sync — dropped from scope.** Originally a Phase-3 stretch with a Tailscale guardrail dialog. Sean's call (post-review): data backup is handled outside this design — Docknote-managed, not user-managed git remote. The brain stays single-device for v9. If multi-device returns later, it returns as a backup-restore flow, not a real-time sync. The guardrail screen I'd planned has come off the wall.

**Sensitive meeting.** Two states. Pre-meeting: a small toggle in the meeting header — when on, the meeting card in the Tide carries a hairline diagonal hatching (think the Federal Reserve's "watermark" pattern at 5% opacity). Post-meeting: the Hearth and Murmur views still render the card and synthesis, but the *body* prose is replaced with a vellum overlay carrying the line: *"This meeting is marked sensitive. The synthesis is hidden by default."* with a single **Reveal for this session** affordance. The masking pattern matters: it shouldn't look broken, it should look *deliberately covered*, like a redacted page in a briefing book. Not `***`. Hatching, vellum, paper.

---

## Restraint principles (the things I'm explicitly *not* doing)

1. **No "AI" iconography.** No glowing orbs (the v8 mem-glyph radial gradient is the *single* exception, and it's earned because it's the typing cursor, not a brand mark). No constellations, no neural-network line art, no "thinking..." spinners. The brain is a book.
2. **No badges with numbers.** The Brain Inbox does not have a red dot with a count. There is a faint ink-faint count next to the Brain icon in the sidebar — set in tabular numerals — and that's all. Discoverable, never demanding.
3. **No emoji in load-bearing UI.** The serif doesn't render emoji well, and emoji is a different register than this product wants to live in. Icons OK; emoji out.
4. **No success animations.** Accepting an update is not a "" moment. The card lands, the queue advances, the brain knows one more thing. We don't celebrate work that the user is doing because it's their job, not a achievement.
5. **No tooltips on chrome.** Tooltips appear *only* on the provenance badges, where they reveal source-meeting + span. Everywhere else, if a thing needs explanation, the explanation is the surface itself.
6. **No empty-state cleverness.** The Brain Inbox empty state is one line. The negative-search state is one line. We don't write paragraphs to fill space.

---

## Signature flourishes I'm proposing

Three. They're the things I want on the wall:

**1. The vellum lock.** A `human_locked: true` field doesn't get a yellow highlight or a padlock badge. It gets a hairline cream-on-cream stripe behind the field — barely visible until you look. When you hover the field, the stripe brightens 6%. It says: *the user signed this off; it persists.* It is the most invisible piece of authority in the entire UI, and that's the point. Privacy and authorship are quiet here.

**2. "Got it." moment in the Brain Inbox.** When the user accepts the last proposed update in the queue, the queue closes with a single line in serif italic, centred, in ink-soft: *"Got it. The brain is up to date as of \<timestamp\>."* No checkmark. No confetti. The brain heard the user. That's the entire reward.

**3. The card's age band has its own typography.** Fields with `age_band: background` are rendered in 95% size, in italic, with letter-spacing tightened by 0.005em. Fields with `age_band: archived` aren't rendered on the card at all — they're available in time-travel. The user can *see* the card breathing across time, with the recent breath warm and the old breath italic. This is the closest the UI comes to having a heart, and it's the move I'd defend hardest in a review.

---

## On the Hearth ↔ Inbox feedback loop

A note for SDD-2: the Hearth and the Brain Inbox are not separate surfaces conceptually. The Inbox is a *staging area* for things that want to enter the Hearth. The pattern I want preserved across both: a proposed update in the Inbox shows the *Hearth field as it currently is*, with the *proposed change inline*, in the same typography the Hearth uses. The user is not reading a diff in some other UI; they are reading the Hearth with a pencil hovering over a word. This means the Inbox card and the Hearth card share 80% of their CSS. Treat them as one component with two states. (The mockup demonstrates this. Don't re-implement them as two unrelated views.)

---

## What I'd want to discuss before we ship

Two things, in order of importance:

The Brain Inbox **cadence and discovery** is unresolved. Daily? Weekly? Real-time? My instinct is *weekly digest by default with a faint count badge that updates daily,* but I would defer to whoever is closest to the user research. The trap is making it daily-by-default and creating an inbox-zero compulsion that the design just spent two pages refusing.

The **time-travel slider** in a multi-edit-per-day card might end up dense. If a card has 40 commits in the last week, the tick marks become unreadable. My current sketch handles up to ~12 ticks gracefully. Beyond that, we cluster — but I haven't designed cluster expansion, and I don't want to. We should probably cap visible ticks at 8 and offer a "Show full history" drawer when there are more.

---

## Closing

The brain memory product is the soul of Docknote. It's the thing that, if it works, the user will remember a year from now as the moment the tool stopped feeling like an app and started feeling like a relationship with their own past work. Every surface in v9 should be in service of that — and that means erring on the side of less, quieter, slower, more legible, more honest.

Ship it.

— Rachel
