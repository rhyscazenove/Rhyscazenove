---
name: decision-journal
description: Structures implementation blog posts around the decisions made during a build, not just the end result. Use this skill whenever the user asks to write up how they built something, document an implementation as a blog post, explain a tool or script they created, or write a "here's how I built X" post. Triggers include "write up this project", "blog about how I built", "explain this implementation", "write a post about my tool/script/site", "turn this build into a post", or any request for a technical write-up that focuses on why decisions were made rather than just what was built. Do NOT use for argument-driven posts (use narrative-cohesion), pure API docs or READMEs (those are documentation, not blog posts), or opinion pieces without an implementation.
---

# Decision Journal — Implementation Blog Post Skill

This skill structures blog posts about things you built. Not documentation, not tutorials, not argument-driven essays. The specific kind of post where the interesting part is *why you made the decisions you made*, not just what the end result looks like.

The core problem it solves: implementation write-ups tend to become either flat lists of technical steps ("then I did X, then I did Y") or context-free code dumps. Neither is interesting to read. A good implementation post makes the reader feel the weight of each decision by showing the constraint space it came from.

## The structural principle

A decision journal isn't organised by argument (that's narrative-cohesion). It's organised by *decision sequence*. Each section earns its place by presenting a fork in the road: here's what I needed, here are the options I considered, here's what I chose, and here's what that choice meant for everything downstream.

The reader's experience should be: "I understand why this ended up the way it did, and I could make my own informed choices if my constraints were different."

---

## When to use this skill

Use for any post about something the user built where the primary goal is explaining *why* the decisions were made. The content will typically be 50/50 code and reasoning. This includes build write-ups, tool/script explainers, architecture decision posts, "how I made this site" posts, and project retrospectives that walk through the implementation.

Do NOT use for argument-driven posts (use narrative-cohesion), pure tutorials where the reader copies your steps exactly, or reference documentation.

---

## Phase 1: Pre-Writing

### 1.1 Define the entry point

Why should the reader care about this build? Not a thesis or a claim, but a *situation* the reader recognises. The entry point answers: "What problem were you facing that led to building this?"

The entry point should be concrete enough that the reader either has the same problem or finds the constraint space interesting.

Good entry points:
- "Most documents look terrible on an e-ink display, and the tools to fix that don't exist."
- "Our CI pipeline was running Claude Code skills locally, but we needed them to run unattended."
- "The museum website was slow. Not a little slow. First-contentful-paint-over-four-seconds slow."

Bad entry points:
- "I decided to build a tool." (no constraint, no problem)
- "I wanted to learn Rust." (the reader learns nothing applicable)
- "This post describes my implementation of X." (meta-description, not an entry point)

### 1.2 Map the decision chain

List the 3-7 key decisions that shaped the build. These become the sections of the post. Order them by dependency, not chronology. If decision C only makes sense after understanding decisions A and B, it comes after them, even if you made C first in real life.

For each decision, note:
- **The constraint** that forced a choice
- **The options** you considered (minimum two, including the one you rejected)
- **What you chose** and why
- **What that choice meant** for downstream decisions

Example:
```
1. Output format: EPUB vs PDF vs HTML
   Constraint: e-ink needs reflowable text
   Options: PDF (no reflow), HTML (needs Wi-Fi), EPUB (reflowable, offline)
   Chose: EPUB
   Downstream: now constrained by NeoReader's CSS support

2. Syntax highlighting: colour vs monochrome
   Constraint: 16-level greyscale display
   Options: Colour themes (collapse to same grey), monochrome (bold/italic)
   Chose: monochrome
   Downstream: need Pandoc's --syntax-highlighting flag, limits theme options
```

### 1.3 Identify the interesting decisions

Not all decisions are equally interesting. Rank your decision chain:

- **Surprising decisions** (where the obvious choice was wrong) deserve the most space
- **Constrained decisions** (where only one option was viable) can be covered quickly
- **Trade-off decisions** (where you gave something up) are always interesting

Allocate word count accordingly. A decision with one viable option gets a paragraph. A decision where you tried the obvious approach and it failed gets a full section.

---

## Phase 2: Outline

Structure the post as follows. Not every element is required; use what the post needs.

### Opening (required)

The entry point from Phase 1, expanded to 2-3 paragraphs. Establishes the problem, makes the constraint space concrete, and tells the reader what was built (one sentence, not a full explanation). The reader should know within the opening what the post will walk through.

### Decision sections (required, 3-7 sections)

Each section follows this internal structure:

1. **The constraint** (1-2 sentences): What you needed, what was limiting you, or what went wrong with the previous approach. This is the section's hook.
2. **The options** (1-3 paragraphs): What you considered. Include code/config for the approaches you evaluated, not just the winner. Showing what you rejected is what makes the post a decision journal rather than a tutorial.
3. **The choice and its reasoning** (1-2 paragraphs): What you picked and why. Be specific. "It was faster" is weak. "It reduced build time from 14 minutes to 3 because it avoided recompiling the dependency tree" is useful.
4. **Code/config** (as needed): The actual implementation. Enough for the reader to understand the approach, not necessarily enough to copy-paste and run. Annotate the non-obvious parts.
5. **The downstream effect** (1-2 sentences): How this choice shaped what came next. This is the connective tissue between sections.

The downstream effect is critical. It's what makes the sections progressive rather than parallel. Without it, the post reads as a list of independent decisions. With it, each section flows into the next because the reader understands how constraints cascaded.

### Lessons / reflection (optional)

If the build taught you something you didn't expect, include it. If not, don't force it. A genuine "this surprised me" is valuable. A perfunctory "key takeaways" list is not.

When you do include reflection, distinguish between:
- **Things that would change your approach next time** (useful to the reader)
- **Things that confirmed what you already knew** (less useful, keep brief)

### Links and source (optional)

If the code is public, link to it. If there's supporting research or documentation, link to that. Keep this functional, not promotional.

---

## Phase 3: Drafting

### 3.1 Lead with the constraint, not the solution

Every decision section should open with what forced the choice, not what you chose. The reader needs to feel the constraint before the solution makes sense.

Bad: "I used JetBrains Mono for the embedded font."
Good: "Without an embedded monospace font, NeoReader picks whatever it has, and code alignment breaks. I needed a font with a high x-height and consistent stroke width for greyscale readability."

The second version makes the reader understand why font choice matters before learning which font was chosen. The first version is trivia.

### 3.2 Show what you rejected

The decisions you *didn't* make are as informative as the ones you did. When you evaluated something and rejected it, say why. This is what separates a decision journal from a tutorial.

"I considered Source Code Pro and Fira Mono. Both rendered well on screen but Source Code Pro's lower x-height made it harder to read at small sizes on e-ink, and Fira Mono's variable stroke width lost clarity in greyscale."

This tells the reader: if your constraints are different (e.g. you're not on e-ink), these alternatives might be better for you.

### 3.3 Annotate code, don't just display it

Code blocks should have enough context that the reader knows *why* each non-obvious line exists. Inline comments are fine for short annotations. For longer explanations, put the reasoning in prose before or after the block.

Avoid showing raw config/code without any framing. A code block that appears without a preceding sentence explaining what it does or why it's needed is a decision journal failure.

### 3.4 Use the downstream effect as a transition

Don't use topic-announcing transitions ("Next, let's look at the CSS"). Use the downstream effect of the previous decision to set up the next one.

Bad: "Now let's look at how I handled syntax highlighting."
Good: "Choosing EPUB as the output format meant I was now constrained by NeoReader's CSS rendering engine. That engine's biggest limitation for code: no colour."

The second version connects the sections. The first just announces the next topic.

### 3.5 Balance code and reasoning

For a 50/50 code-to-reasoning post, a good rule of thumb is: no more than two consecutive code blocks without intervening prose, and no more than four consecutive paragraphs of reasoning without a code example or concrete technical detail.

If the post is drifting toward pure code, add the reasoning for the non-obvious parts. If it's drifting toward pure reasoning, ground it in specifics.

### 3.6 The opening should not be a history lesson

Don't open with the origin story of the project. Open with the problem the reader can recognise. "I wanted to build a thing" is not a hook. "Here's a problem you probably have too" is.

If the project's origin is genuinely interesting (an incident, a frustration, a surprising discovery), use that as the opening. If it's just "I decided to build X", skip it and start with the first constraint.

---

## Phase 4: Self-Review

### 4.1 Decision chain check

For each decision section, verify it contains all four elements: constraint, options, choice with reasoning, and downstream effect. Missing elements mean the section is either a tutorial step (no constraint or options) or an assertion (no reasoning).

### 4.2 Constraint-first check

Read the first sentence of each decision section. Does it state the constraint or the solution? If more than one section opens with the solution, restructure to lead with the constraint.

### 4.3 Connective tissue check

Read only the downstream-effect sentences in sequence. Do they form a chain? Each one should set up the constraint of the next section. If the chain breaks, either add the missing connection or reorder the sections.

### 4.4 Code-reasoning balance

Scan the post visually. Are there long stretches of code without reasoning? Long stretches of reasoning without code? Flag any run of three or more code blocks without intervening prose, or five or more paragraphs without a concrete technical detail.

### 4.5 Rejected-options check

Does the post show at least two decisions where you considered and rejected an alternative? If every section reads as "I needed X, so I used Y", the post is a tutorial, not a decision journal. The reader needs to see forks in the road.

---

## What this skill does NOT do

- **Build arguments.** If the post has a thesis or claim it exists to prove, use narrative-cohesion instead.
- **Handle voice or style.** Use the tone-rewriter skill for that.
- **Produce tutorials.** Tutorials assume the reader will follow the same steps. Decision journals assume the reader's constraints may differ and give them enough reasoning to adapt.
- **Write documentation.** Docs are reference material. Decision journals are narratives about constraint spaces.

---

## Interplay with other skills

This skill sits alongside narrative-cohesion, not beneath it. Both produce structurally sound drafts; they handle different post types.

| Post type | Skill |
|-----------|-------|
| "Here's what I think about X" | narrative-cohesion |
| "Here's how I built X and why" | decision-journal |
| Any post, after structural draft | tone-rewriter |

If you're unsure which skill fits, ask: "Does this post exist to make a claim, or to walk through decisions?" If the post has a throughline the reader should be persuaded of, use narrative-cohesion. If the post's value is in the decision-by-decision reasoning, use this skill.

---

## Quick reference: the structural tests

| Test | What it catches | When to apply |
|------|----------------|---------------|
| **Decision chain check** | Sections missing constraint, options, reasoning, or downstream effect | Phase 4 |
| **Constraint-first check** | Sections that lead with solutions instead of problems | Phase 4 |
| **Connective tissue check** | Disconnected sections (parallel, not progressive) | Phase 4 |
| **Code-reasoning balance** | Drift toward pure code dump or pure essay | Phase 4 |
| **Rejected-options check** | Post reads as tutorial rather than decision journal | Phase 4 |
