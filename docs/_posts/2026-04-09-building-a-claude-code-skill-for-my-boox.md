---
published: false
layout: post
title: "Building a Claude Code Skill to Make EPUBs for My Boox E-Reader"
date: 2026-04-09
categories: [ai, tooling, e-ink]
tags: [claude-code, epub, boox, e-ink, ai-skills, pandoc]
description: "How I built a custom Claude Code skill that converts Markdown into EPUBs optimised for my Boox Go 10.3 e-ink tablet, and what Claude Code skills are."
---

I read a lot of technical documentation on my Boox Go 10.3. It's a 10.3-inch e-ink tablet with a 300 PPI display, and it's the best way I've found to absorb a long technical document without the distractions of a backlit screen. One problem: most documents look terrible on it.

PDFs don't reflow. Web pages need Wi-Fi. Generic EPUBs treat code blocks as an afterthought. Syntax highlighting that relies on colour is meaningless on a 16-level greyscale display, and code that runs off the right edge of the screen is gone. E-ink doesn't scroll horizontally.

So I built a **Claude Code skill** for it: a reusable capability that Claude can invoke whenever I ask it to create a document for my tablet.

## What's a Claude Code Skill?

[Claude Code](https://claude.ai/claude-code) is Anthropic's CLI tool for working with Claude. It reads files, writes code, runs commands. Skills extend what it can do.

A skill is a specialised instruction set: a Markdown file with YAML front matter that tells Claude *when* to activate and *how* to carry out the task. You write the expertise once, and Claude applies it whenever the situation matches, without you spelling out the details each time.

The front matter defines the trigger conditions:

```yaml
---
name: boox-epub
description: "Create EPUB documents optimised for reading on a Boox Go 10.3
  e-ink tablet (300 PPI, greyscale, 2480x1860). Use this skill whenever the
  user wants to produce a document, report, guide, or technical writeup
  intended for reading on their Boox e-ink device..."
---
```

The body of the file provides the full instructions: what tools to use, what flags to pass, what CSS to apply. Claude reads this at invocation time and follows the recipe, adapting it to whatever content is at hand.

Skills encode *domain expertise*. In my case: e-ink display constraints, NeoReader's CSS rendering engine, and the pipeline needed to produce a good-looking EPUB. I figure out the hard stuff once, write it down in a format Claude can follow, and stop thinking about it.

## Why Not Just Run Pandoc?

The Boox Go 10.3 has a 2480x1860 pixel display at 300 PPI, sharp but limited to 16 levels of grey. No colour. NeoReader, the built-in reader app, has a CSS rendering engine that supports no flexbox, no grid, no `calc()`, no media queries, no floats. (I documented the full research into NeoReader's rendering capabilities, font choices, and device specs in a [separate research catalogue](/output/boox-epub-research) if you want the sources.)

The problems I kept hitting with vanilla `pandoc input.md -o output.epub`:

- Without `white-space: pre-wrap`, long code lines vanish past the right edge. E-ink doesn't scroll horizontally.
- Pandoc's default syntax themes use colour to distinguish keywords, strings, and comments. On greyscale, half these colours collapse to the same shade of grey.
- EPUBs don't execute JavaScript, so Mermaid fenced blocks show up as raw text.
- A six-column table at default font sizes doesn't fit on a 10.3-inch screen.
- Without an embedded monospace font, NeoReader picks whatever it has, and code alignment breaks.

## The Pipeline

The skill defines a three-stage pipeline that handles all of this:

```
Markdown source
    |
    +-- Mermaid blocks? --> Render to PNG (mmdc, 2000px wide, white bg)
    |                       Replace ```mermaid blocks with ![](image.png)
    |
    +-- Download JetBrains Mono if not cached (~/.cache/boox-epub-fonts/)
    |
    +-- Pandoc conversion:
         -f gfm -> -t epub3
         --css epub-eink.css
         --epub-embed-font JetBrainsMono-Regular.ttf
         --epub-embed-font JetBrainsMono-Bold.ttf
         --syntax-highlighting=monochrome
         --toc --toc-depth=3
```

A Python script (`boox-epub-convert.py`) orchestrates the pipeline. Claude invokes it as part of the skill.

### Stage 1: Mermaid Diagram Pre-rendering

Mermaid diagrams are extracted from the Markdown, rendered to PNG using `mmdc` (the Mermaid CLI), and the fenced blocks are replaced with image references. Two critical flags:

- `-b white`, because transparent backgrounds render as solid black on e-ink
- `-w 2000`, because 2000 pixels maps well to the 2480px screen at 300 PPI

The script processes blocks in reverse order to preserve string offsets during replacement. If you iterate forward and replace a block with a shorter image reference, every subsequent block's offset is wrong.

### Stage 2: Font Embedding

JetBrains Mono gets downloaded once and cached at `~/.cache/boox-epub-fonts/`. If the download fails (network restrictions in sandboxed environments), the script falls back to NeoReader's built-in monospace font. Acceptable, not as crisp. JetBrains Mono won out over Source Code Pro and Fira Mono because of its increased x-height and consistent stroke width, both of which matter on e-ink's limited greyscale (more on the font selection in the [research catalogue](/output/boox-epub-research)).

### Stage 3: Pandoc with E-Ink CSS

The `--syntax-highlighting=monochrome` flag tells Pandoc to use bold and italic for token distinction instead of colour. The custom CSS stylesheet handles everything else.

## The CSS

The stylesheet (`epub-eink.css`) took the most iteration. Every decision traces back to the 16-level greyscale display.

Code blocks use `pre-wrap` to soft-wrap long lines instead of clipping them. Font size is set to `0.82em`, small enough to fit roughly 75 characters per line at readable size. A `border-left: 3px solid #666` provides a visual separator that renders as a clear dark grey.

Syntax highlighting is monochrome. Keywords and data types get `font-weight: bold`. Comments get `font-style: italic` with `color: #555`. Everything else is unstyled. On e-ink, this is more readable than any colour theme, because bold and italic remain distinct in greyscale where different colours collapse to the same shade.

All font sizes use `em` or `%` units, never `px`. Everything scales proportionally when you adjust NeoReader's font size slider. People adjust text size constantly on e-ink; one `px` value and the slider stops working.

Tables are set to `0.9em` and `width: 100%` to help wide tables fit. Table headers get a `#e0e0e0` background, visible enough on greyscale to distinguish them from data rows.

Page breaks use `page-break-after: avoid` on headings to prevent orphaned headers at the bottom of a page, and `orphans: 2; widows: 2` to avoid single-line paragraph fragments.

The greyscale visibility guide I worked out through trial and error:

| Hex Range | Appearance on E-Ink |
|-----------|-------------------|
| `#000` -- `#333` | Solid/near-black (text, borders) |
| `#444` -- `#999` | Clear mid-greys (secondary elements) |
| `#aaa` -- `#ccc` | Light but visible (subtle borders) |
| `#d0d` -- `#e0e` | Barely visible (faint backgrounds) |
| `#e8e` -- `#fff` | May be invisible -- use cautiously |

## Writing the Skill Definition

The skill definition file (`SKILL.md`) contains YAML front matter with trigger conditions, device specs, step-by-step instructions with flags, content authoring guidelines, and a troubleshooting table.

The trigger description matters. Mine includes phrases like "make an epub", "create a document for my boox", "format this for e-ink", and "make this into something I can read on my tablet". Claude pattern-matches against these when deciding which skills to activate.

The body reads like instructions for a knowledgeable colleague. It explains *what* to do and *why*, because Claude can adapt instructions to new situations if it understands the reasoning. For example:

> Use `-b white` (essential; transparent backgrounds render as black on e-ink)

That parenthetical tells Claude *why* the flag matters. If it later needs to render a PlantUML diagram, it can apply the same logic without me adding a new instruction.

## Using It

After setting this up, my workflow for creating a Boox-readable document is:

1. Write content in Markdown (or ask Claude to help me write it)
2. Say "make an epub of this for my Boox"
3. Transfer the EPUB to my tablet

Step 2 is one sentence. Claude recognises the intent, activates the skill, processes Mermaid diagrams, downloads fonts if needed, runs Pandoc with the right flags, and hands me an EPUB. I don't think about CSS selectors or NeoReader's rendering quirks.

## Lessons Learned

I went through several iterations of the CSS before landing on values that worked on the device. Calibre's EPUB viewer lies. The greyscale rendering is not a simple linear mapping, and NeoReader's CSS support has gaps you only find by testing on hardware.

Monochrome syntax highlighting turned out better than I expected. Bold keywords and italic comments provide enough visual structure on a 300 PPI display, and your brain adapts to the absence of colour within a page or two. Pandoc ships eight highlight themes; on a 16-level greyscale display, the colour-based ones map multiple distinct tokens to the same grey. `monochrome` is the only one that survives greyscale intact.

Relative units (`em`, `%`) are non-negotiable. One `px` value breaks the font size slider, and e-ink readers depend on that slider.

Skills that explain their reasoning age better than skills that list steps. "Use `-b white` because transparent backgrounds render as black on e-ink" lets Claude generalise to new diagram tools. "Use `-b white`" alone is brittle.

## Try It Yourself

The full skill is [on GitHub](https://github.com/rhys/boox-epub-maker-skill). If you have a Boox (or any e-ink reader), fork it and adjust the CSS for your device's specs. The pipeline works with any EPUB3-compatible reader. The CSS is the device-specific part.

If you don't have an e-ink tablet but want to try Claude Code skills, the structure is the same: a `SKILL.md` file with YAML front matter defining triggers and a Markdown body containing the instructions. Drop it into your project or your Claude Code config, and Claude picks it up automatically.

I use this a few times a week now. Technical docs on e-ink with proper code formatting and working diagrams, with no manual Pandoc flags to remember.

If you want to see the full research behind the design decisions (nine searches, three deep-dives, sources for every CSS value), I published the [research catalogue](/output/boox-epub-research) alongside this post.
