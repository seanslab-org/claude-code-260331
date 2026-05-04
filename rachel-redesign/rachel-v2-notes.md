# v2 redline notes — Rachel

Sean locked the two principles after v1. I went back through the seven surfaces and cut everything that lied about the architecture or buried the ritual. Tight notes follow, per surface.

## What I cut, and why

**Brain Inbox → Morning chart review.** Was: file paths and `field` chips at the top of every "diff card" (`person/alex-chen.md · role_line`). That's database language pretending to be a UI; it implies a `facts` table that doesn't exist. Cut. Now Geomi speaks in plain sentences ("After today's visit, I'd add 'reports new dizziness' to Mrs. Chen's current concerns. Confirm?") with two replies: *Yes, remember that* / *Not yet*. Also cut: the `lock-collision` proposal with three buttons, the cosine-similarity number on the theme card, the model name (`qwen-3-32b`) on every row. **Drives:** principle 1 (no schema chrome) + principle 2 (the conversation IS the feature, so make it feel like one).

**Hearth — edit states → A folder, opened.** Biggest cut of the round. v1 was a four-state catalog (read-only / hover / locked / editing) with field labels in CAPS, a hover-pencil glyph, vellum field-locks, contenteditable inline editor as the primary path. **All of that violates principle 2** — direct field editing was the front door, Geomi confirmation was buried. It also violated principle 1 — `Role line / Body / Differentiator` field-labels imply tabular fields where none exist. The folder now shows three to five paragraphs of prose (which is what the brain actually wrote) and the only edit affordance is "Ask Geomi to update this." A doctor reading this in three seconds understands: *this is what the brain remembers about Alex*. **Drives:** both principles.

**Time-travel.** Was: a slider with tick marks, "card-time vs meeting-time" double-ribbon, history drawer with commit hashes (`auto-consolidator · q3-pricing`), revert buttons per field. A doctor doesn't think in commits. Replaced with a single named-meeting picker: "Show me what I knew before *the leadership offsite (29 Apr).*" The folder rewinds to that day. No tick marks. No revert affordance. **Drives:** the simpler-test ("3 seconds between calls"), and principle 1 (commit lists imply structured edit history — soften to chapters).

**Brain settings → What the brain knows about you.** Cut the file-path toolbars (`~/Docknote/brain/persona.md`), the token cap meter, the `vim ~/Docknote/brain/active.md` incantation, and the `## Heading` markdown-preview chrome. All of those say "this is a database / a config file." The pages are now manila folders with paragraphs, like every other folder. The vim escape hatch moved to a single sentence at the bottom — *"To change anything here, tell Geomi"* — with the power-user line tucked into preferences (off-screen). **Drives:** principle 1.

**Geomi bubbles.** Cut the four-rule grid at the bottom and the `REQ-075 §7.3` dismiss-spec footnote. Those were designer notes for the team, not user surface. Bubbles speak for themselves now. **Drives:** simpler-test.

**Sensitive meeting.** Tightened pre-meeting copy ("Sensitive meetings stay on your machine. They never leave for any cloud model."), dropped the `snapgpu` / Claude routing footnote. Kept the vellum-and-hatching mask intact — that's correct. **Drives:** simpler-test.

**Mockup index.** Reordered: morning chart review is now the lead card, double-wide, with the heading "The heart · the morning chart review." Inbox-as-confirmation-ritual is the product, not surface 01-of-six. **Drives:** principle 2.

## What I kept

- The Tide / Hearth / Murmur palette and serif voice — still right.
- The vellum stripe for "you wrote this" — quiet, diegetic, earns its place.
- The breathing dot on Geomi — the only continuous animation, calibrated to the Tide pulse.
- The sensitive-meeting hatching — a deliberate cover, not `***`.
- The manila-folder metaphor, now actually visible: tab on top-left, paper texture, paragraphs inside.

## What I'd push for in v3

1. **Voice replies to Geomi.** A doctor between visits should be able to say *"yes, add it"* aloud while washing hands. The morning chart review is built for it — the prompts are already conversational.
2. **One-tap bulk confirm.** When all four prompts are about the same patient/account, offer a single *"Yes to all"* — but only when there are no contradictions. Earned, not default.
3. **The folder thumbnail in lists.** When you scan a list of patients/clients, you should see the manila tab — same shape, smaller. The metaphor should hold all the way down.

— Rachel · 2026-05-04
