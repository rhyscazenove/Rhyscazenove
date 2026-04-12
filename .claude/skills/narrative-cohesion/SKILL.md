---
name: narrative-cohesion
description: Enforces structural narrative cohesion in blog posts and long-form writing. Use this skill whenever the user asks to write a blog post, article, thought-leadership piece, technical essay, or any long-form content (500+ words). Also trigger when the user says "write a post about", "draft an article on", "help me blog about", "turn this into a post", or asks for content that needs to argue a point rather than just list information. This skill focuses on the skeleton — throughline, argument, narrative arc, and structural logic — not sentence-level style. It pairs with the tone-rewriter skill for voice.
---

# Narrative Cohesion — Structural Writing Skill

This skill produces blog posts with a coherent narrative arc and a clear argument. It operates on the skeleton of a piece — the throughline, the logical progression, the tension and resolution — not on word choice or sentence rhythm.

The core problem it solves: AI-generated posts tend to produce information without argument, structure without progression, and conclusions without payoff. The result reads like a Wikipedia article with an opinion stapled to the end. This skill prevents that by enforcing architectural discipline before and during drafting.

**This skill does not handle voice or style.** For that, use the tone-rewriter skill after this skill has produced a structurally sound draft.

---

## When to use this skill

Use for any writing request that will be 500+ words and needs to make a point. This includes blog posts, articles, thought-leadership pieces, technical essays, conference talk scripts, and newsletter content. Do NOT use for factual summaries, documentation, changelogs, or reference material where argument isn't the goal.

---

## Phase 1: Pre-Writing (mandatory before any prose)

Before writing a single paragraph, complete these four steps. Present them to the user for confirmation before proceeding to the outline.

### 1.1 Define the throughline

State the single core idea of the piece in one sentence. This is the thesis — the claim the post exists to make. Every section must serve this throughline or be cut.

**Test:** If someone reads only this sentence, do they know what the post argues? If it sounds like a topic ("This post is about observability") rather than a claim ("Most teams instrument everything and observe nothing"), rewrite it.

Bad throughlines:
- "This post explores the benefits of platform engineering." (topic, not claim)
- "AI is transforming software development." (too broad, no tension)
- "There are several approaches to handling technical debt." (listicle setup)

Good throughlines:
- "Platform engineering fails when it optimises for developer experience without measuring developer outcome."
- "The strongest argument against rewriting your system is usually the one your team is afraid to make."
- "Evals are the boring part of AI adoption — and the part that determines whether anything else matters."

### 1.2 Construct the ABT

Reduce the piece to an And/But/Therefore structure. This is the narrative engine.

- **And** — Establish the shared context. What does the reader already know or agree with?
- **But** — Introduce the tension. What's the problem, contradiction, or gap?
- **Therefore** — State the consequence. What follows from this tension?

**Diagnostic:** If the piece reads as And/And/And (fact, fact, fact), it's a list, not an argument. If it reads as But/But/But, it's unfocused anxiety. The ABT forces a single narrative movement.

Example:
> "Teams are investing heavily in AI tooling for developers [AND], and productivity metrics look promising on paper. [BUT] But most teams can't tell you whether the code their developers ship with AI assistance is actually better — they're measuring speed without measuring quality. [THEREFORE] Therefore, AI adoption without an evaluation framework is a bet with no feedback loop."

### 1.3 Identify the knowledge gap

What does the reader not know (or not realise) that this post will reveal? Curiosity arises from a gap between what someone knows and what they want to know. The post must open this gap early and close it by the end.

State the gap as a question the reader should be asking by the end of the introduction.

### 1.4 Choose the structural model

Select one based on the post's purpose. Read `references/structural-models.md` for detailed guidance on each.

| Model | Best for | Shape |
|-------|----------|-------|
| **Narrative Arc** | Thought leadership, experience-based posts | Hook → Rising tension → Turn → Resolution → Implication |
| **Pyramid** | Analytical posts, recommendations | Conclusion first → Supporting arguments → Evidence |
| **Gap-and-Fill** | Explainers, tutorials with a point | Open gap → Deepen gap → Fill gap → Expand implications |
| **Contrarian** | Challenge conventional wisdom | Orthodoxy → Evidence against → Reframe → New position |

---

## Phase 2: Outline (mandatory before drafting)

Build a section-level outline. Each section gets one line with three elements:

```
[Section title] — [What this section does for the argument] — [What the reader knows/feels after reading it]
```

Example:
```
1. Opening — Presents the orthodoxy that platform teams should optimise for DX — Reader nods along, agrees with common view
2. The metric trap — Shows that DX metrics don't correlate with delivery outcomes — Reader's certainty wavers
3. What outcome-oriented looks like — Concrete example of a team that measured differently — Reader sees an alternative
4. The uncomfortable conversation — Why teams resist outcome measurement (it reveals uncomfortable truths) — Reader recognises their own avoidance
5. Close — Restate the throughline with the earned authority of the preceding evidence — Reader has a new mental model
```

### The chain test

Between each consecutive section, insert "so", "but", or "therefore". If none of these words create a logical connection, there is a structural gap. Either add a bridging section or reorder.

```
Opening [BUT] The metric trap [SO] What outcome-oriented looks like [BUT] The uncomfortable conversation [THEREFORE] Close
```

If you can only connect sections with "and" or "also" or "additionally", the sections are parallel (a list), not progressive (an argument). Restructure so each section changes the reader's understanding.

### The two-job test

Every section must do at least two things from this list:
- Advance the argument (move the throughline forward)
- Provide evidence (data, example, case study, analogy)
- Create or sustain tension (introduce a complication, counterargument, or stake)
- Shift the reader's emotional state (surprise, recognition, discomfort, relief)

Sections doing only one job need revision. A section that only provides evidence without advancing the argument is a tangent. A section that only advances the argument without evidence is an assertion.

---

## Phase 3: Drafting

Write section by section, following the outline. During drafting, apply these structural constraints:

### 3.1 The "So What?" gate

After writing each paragraph, ask: "So what?" If the paragraph states a fact without explaining why it matters to the throughline, add the implication or cut the paragraph.

Bad:
> "Platform engineering has grown significantly in adoption over the past three years, with many organisations establishing dedicated platform teams."

This is a fact with no implication. So what?

Better:
> "Platform engineering has gone from fringe idea to org-chart fixture in three years. The speed of adoption is itself part of the problem — teams are building platforms before they've agreed on what success looks like."

The second version connects the fact to the throughline (measuring the wrong things).

### 3.2 No false progress

Watch for sections that feel like they're moving forward but actually restate the same point in different words. Each section must add new information, a new angle, or a new consequence. If a section could be deleted without the reader missing a step in the argument, it should be deleted.

### 3.3 Tension before resolution

Never resolve a point before the reader has felt the tension. If you're about to write "The solution is...", check that the preceding section has made the reader feel the weight of the problem. Premature resolution kills engagement.

### 3.4 Transitions carry argument, not just topic

Never use transitions that merely announce the next topic ("Now let's look at...", "Another important consideration is...", "Moving on to..."). Transitions should carry logical force:

- "But this assumption breaks down when..." (introduces counterevidence)
- "This matters because..." (escalates stakes)
- "The same pattern shows up in..." (extends the argument to new territory)

If a transition can be replaced with "Also," the sections are parallel, not progressive.

### 3.5 The opening earns attention

The first 2-3 sentences must do one of:
- Make a claim the reader disagrees with (and will agree with by the end)
- Present a familiar situation and reframe it unexpectedly
- State a specific, concrete, surprising fact
- Voice what the reader is already thinking but hasn't articulated

Never open with a definition, a history lesson, or a broad contextual sweep ("In today's rapidly evolving landscape..."). The reader should feel a gap open within the first paragraph.

### 3.6 The close earns its landing

The conclusion must connect back to the opening, but the reader's understanding should have shifted. If the conclusion could have been written without the body of the post, it's too generic.

The strongest closes do one of:
- Return to the opening image/scenario with new meaning
- State the throughline in sharper terms than the opening, earned by the evidence
- Pose the next question that follows from the argument
- Give the reader a specific, concrete next action

Never close with a vague call to "think about" or "consider" something. The reader should leave with something they didn't have before.

---

## Phase 4: Structural Review (mandatory before presenting the draft)

Before showing the draft to the user, run this self-review. Do not skip it.

### 4.1 Reverse outline

Reduce each section to a single sentence summarising its function. Read these sentences as a standalone sequence. Does the sequence build an argument, or does it read like a table of contents?

### 4.2 Promise-payoff audit

List every claim, question, or gap opened in the first two sections. Verify that each one is addressed or resolved by the end. Unfulfilled promises are the most common structural failure.

### 4.3 Progression check

For each section, answer: "What does the reader know or believe after this section that they didn't before?" If the answer is "the same thing, but with more examples," the section is redundant.

### 4.4 Pacing check

No section should go more than 3-4 paragraphs without a turn — a complication, a counterargument, a new piece of evidence, or a shift in perspective. Long sections without turns read as lectures.

### 4.5 Connective tissue check

Read only the first sentence of each section in sequence. Do they form a logical chain? If reading just these sentences produces a coherent (if compressed) argument, the structure is sound.

---

## What this skill does NOT do

- **Sentence-level style or voice.** Use the tone-rewriter skill for that.
- **SEO optimisation.** This skill is about the reader, not the algorithm.
- **Anti-slop word policing.** Vocabulary tells are a symptom of structural failure, not the cause. Fix the skeleton and most surface-level AI tells disappear.
- **Formatting decisions.** Whether to use headers, bullets, or pure prose is a presentation choice, not a structural one.

---

## Interplay with other skills

This skill produces a structurally sound draft. The recommended pipeline:

1. **narrative-cohesion** (this skill) — builds the skeleton and produces a draft with a clear throughline
2. **tone-rewriter** — rewrites the draft in the user's authentic voice
3. User review and iteration

If the user asks to "write a blog post", trigger this skill first. If the user asks to "rewrite this in my voice", that's the tone-rewriter only.

---

## Quick reference: the structural tests

| Test | What it catches | When to apply |
|------|----------------|---------------|
| **Throughline test** | Topic masquerading as thesis | Phase 1 |
| **ABT test** | Missing narrative engine | Phase 1 |
| **Chain test** | Parallel sections instead of progressive | Phase 2 |
| **Two-job test** | Thin sections doing only one thing | Phase 2 |
| **So What? gate** | Facts without implications | Phase 3 |
| **Promise-payoff audit** | Unfulfilled claims from the opening | Phase 4 |
| **Progression check** | Redundant sections | Phase 4 |
| **Connective tissue check** | Broken logical chain between sections | Phase 4 |
