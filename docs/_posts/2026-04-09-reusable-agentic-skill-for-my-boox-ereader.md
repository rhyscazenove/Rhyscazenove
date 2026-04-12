---
published: true
layout: post
title: "Generating optimised ePubs for My E-Ink Tablet"
date: 2026-04-06
categories: [ai, tooling, e-ink]
tags: [claude-code, github-copilot, codex, epub, boox, e-ink, ai-skills, pandoc]
description: "How I built a reusable agentic skill that converts Markdown into EPUBs optimised for my Boox Go 10.3 e-ink tablet."
---

I read a lot of technical documentation on my Boox Go 10.3. It's a 10.3-inch e-ink tablet with a 300 PPI display, and it's the best way I've found to absorb a long technical document without the distractions of a backlit screen. One problem: most documents look terrible on it.

PDFs don't reflow. Web pages need Wi-Fi. Generic EPUBs treat code blocks as an afterthought, and syntax highlighting that relies on colour is meaningless on a 16-level greyscale display. Code that runs off the right edge of the screen is gone; e-ink doesn't scroll horizontally.

So I built a **skill** for it: a reusable capability that a coding agent can invoke whenever I ask it to create a document for my tablet. I built it in Claude Code, but the same pattern works with GitHub Copilot, OpenAI Codex, and other LLM-powered coding tools.

## What's an Agentic Skill?

An agentic skill is a Markdown file that tells a coding agent *when* to activate and *how* to carry out a task. You write the expertise once; the agent applies it whenever the situation matches. Claude Code calls them skills. GitHub Copilot and Codex have their own conventions for custom instructions, but the content is the same: a file with trigger conditions and a body of instructions that the agent reads at invocation time.

The front matter defines when it fires:

```yaml
---
name: boox-epub
description: "Create EPUB documents optimised for reading on a Boox Go 10.3
  e-ink tablet (300 PPI, greyscale, 2480x1860). Use this skill whenever the
  user wants to produce a document, report, guide, or technical writeup
  intended for reading on their Boox e-ink device..."
---
```

The body contains the full instructions: what tools to use, what flags to pass, what CSS to apply. The agent follows the recipe and adapts it to whatever content is at hand.

In my case, the skill encodes e-ink display constraints, NeoReader's CSS rendering engine, and the pipeline needed to produce a good-looking EPUB. 

## Challenges and Learnings

![Boox Go 10.3 device constraints: 2480x1860 at 300 PPI, 16 greyscale levels, NeoReader with no flexbox, grid, calc(), media queries, or floats](/assets/images/boox-epub-skill/device-constraints.svg)

The display is 2480x1860 at 300 PPI, limited to 16 levels of grey. No colour. NeoReader, the built-in reader app, supports no flexbox, no grid, no `calc()`, no media queries, no floats.

Full research into NeoReader's rendering capabilities, font choices, and device specs is in the [research catalogue](/boox-epub-research).

The problems I kept hitting with vanilla `pandoc input.md -o output.epub`:

- Without `white-space: pre-wrap`, long code lines vanish past the right edge. E-ink doesn't scroll horizontally.
- Pandoc's default syntax themes use colour to distinguish keywords, strings, and comments. On greyscale, half these colours collapse to the same shade of grey.
- EPUBs don't execute JavaScript, so Mermaid fenced blocks show up as raw text.
- A six-column table at default font sizes doesn't fit on a 10.3-inch screen.
- Without an embedded monospace font, NeoReader picks whatever it has, and code alignment breaks.

## The Pipeline

The skill defines a three-stage pipeline that handles all of this:

![Three-stage conversion pipeline: Mermaid pre-rendering, font embedding, and Pandoc conversion to EPUB3](/assets/images/boox-epub-skill/conversion-pipeline.svg)

A Python script (`boox-epub-convert.py`) orchestrates the pipeline. The agent invokes it as part of the skill.

### Stage 1: Mermaid Diagram Pre-rendering

Mermaid diagrams are extracted from the Markdown, rendered to PNG using `mmdc` (the Mermaid CLI), and the fenced blocks are replaced with image references. Two critical flags:

- `-b white`, because transparent backgrounds render as solid black on e-ink
- `-w 2000`, because 2000 pixels maps well to the 2480px screen at 300 PPI

The script processes blocks in reverse order to preserve string offsets during replacement. If you iterate forward and replace a block with a shorter image reference, every subsequent block's offset is wrong.

### Stage 2: Font Embedding

JetBrains Mono gets downloaded once and cached at `~/.cache/boox-epub-fonts/`. If the download fails (network restrictions in sandboxed environments), the script falls back to NeoReader's built-in monospace font. Acceptable, not as crisp. JetBrains Mono won out over Source Code Pro and Fira Mono because of its increased x-height and consistent stroke width, both of which matter on e-ink's limited greyscale (more on the font selection in the [research catalogue](/boox-epub-research)).

### Stage 3: Pandoc with E-Ink CSS

The `--syntax-highlighting=monochrome` flag tells Pandoc to use bold and italic for token distinction instead of colour. The custom CSS stylesheet handles everything else.

## The CSS

The stylesheet (`epub-eink.css`) took the most iteration. Every decision traces back to the 16-level greyscale display.

> Code blocks use `pre-wrap` to soft-wrap long lines instead of clipping them. Font size is set to `0.82em`, roughly 75 characters per line at readable size. A `border-left: 3px solid #666` provides a visual separator that renders as a clear dark grey.
>
> Syntax highlighting is monochrome. Keywords and data types get `font-weight: bold`. Comments get `font-style: italic` with `color: #555`. Everything else is unstyled.
>
> All font sizes use `em` or `%` units, never `px`. Everything scales proportionally when you adjust NeoReader's font size slider.
>
> Tables are set to `0.9em` and `width: 100%` to help wide tables fit. Table headers get a `#e0e0e0` background, visible enough on greyscale to distinguish them from data rows.
>
> Page breaks use `page-break-after: avoid` on headings to prevent orphaned headers at the bottom of a page, and `orphans: 2; widows: 2` to avoid single-line paragraph fragments.

On e-ink, monochrome highlighting is more readable than any colour theme. Bold and italic remain distinct in greyscale where different colours collapse to the same shade.

![Side-by-side comparison: colour syntax theme collapses to indistinguishable greys on e-ink, while monochrome theme uses bold and italic for clear contrast](/assets/images/boox-epub-skill/syntax-comparison.svg)

People adjust text size constantly on e-ink; one `px` value and the slider stops working.

The greyscale visibility guide I worked out through trial and error:

![Greyscale visibility guide showing actual hex colour ranges from solid black to potentially invisible on e-ink](/assets/images/boox-epub-skill/greyscale-visibility.svg)

## Writing the Skill Definition

> The skill definition file (`SKILL.md`) contains YAML front matter with trigger conditions, device specs, step-by-step instructions with flags, content authoring guidelines, and a troubleshooting table. The agent pattern-matches against the trigger description when deciding which skills to activate.

The trigger description matters. Mine includes phrases like "make an epub", "create a document for my boox", "format this for e-ink", and "make this into something I can read on my tablet".

The body reads like instructions for a knowledgeable colleague. It explains *what* to do and *why*, because any capable coding agent can adapt instructions to new situations if it understands the reasoning. For example:

> Use `-b white` (essential; transparent backgrounds render as black on e-ink)

That parenthetical tells the agent *why* the flag matters. If it later needs to render a PlantUML diagram, it can apply the same logic without me adding a new instruction.

## Using It

After setting this up, my workflow for creating a Boox-readable document is:

1. Write content in Markdown (or ask the agent to help me write it)
2. Say "make an epub of this for my Boox"
3. Transfer the EPUB to my tablet

Step 2 is one sentence. The agent handles the rest: diagrams, fonts, Pandoc flags, CSS. I don't think about NeoReader's rendering quirks.

## What I Got Wrong (and Right)

I went through several iterations of the CSS before landing on values that worked on the device. Calibre's EPUB viewer lies. The greyscale rendering is not a simple linear mapping, and NeoReader's CSS support has gaps you only find by testing on hardware.

Monochrome syntax highlighting turned out better than I expected. Bold keywords and italic comments provide enough visual structure on a 300 PPI display, and your brain adapts to the absence of colour within a page or two. Pandoc ships eight highlight themes; on a 16-level greyscale display, the colour-based ones map multiple distinct tokens to the same grey. `monochrome` is the only one that survives greyscale intact.

Relative units (`em`, `%`) are non-negotiable. One `px` value breaks the font size slider, and e-ink readers depend on that slider.

Skills that explain their reasoning age better than skills that list steps. "Use `-b white` because transparent backgrounds render as black on e-ink" lets the agent generalise to new diagram tools. "Use `-b white`" alone is brittle.

## The Skill and the Source

The full skill is [on GitHub](https://github.com/rhyscazenove). If you have a Boox (or any e-ink reader), fork it and adjust the CSS for your device's specs. The pipeline works with any EPUB3-compatible reader; the CSS is the device-specific part.

If you don't have an e-ink tablet but want to try building skills, the structure is the same regardless of agent: a Markdown file with front matter defining triggers and a body containing instructions. In Claude Code, drop a `SKILL.md` into your project. Copilot and Codex have their own conventions, but the content transfers directly.

I use this a few times a week now. Technical docs on e-ink with proper code formatting and working diagrams, no manual Pandoc flags to remember.

The full research behind the design decisions is in the [research catalogue](/boox-epub-research).
