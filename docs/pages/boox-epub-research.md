---
layout: page
title: "Research on optimum settings for epubs for Boox eReaders"
permalink: /boox-epub-research/
---
# Research Catalogue: boox-epub Skill

This document catalogues the research conducted to inform the design of the `boox-epub` skill — a Claude skill for generating EPUB documents optimised for the Boox Go 10.3 e-ink tablet.

## Research Method

The research was conducted across a single session using web search and targeted page fetches. Nine search queries were run, plus three direct page fetches for deeper reading. The findings were synthesised into a comprehensive reading guide before being distilled into the skill's SKILL.md, CSS stylesheet, conversion script, and NeoReader reference document.

![Research flow: 9 searches and 3 deep reads synthesised into four deliverables](/assets/images/boox-epub-research/research-flow.svg)

---

## Search 1: NeoReader Format Support

**Query:** `Boox NeoReader supported formats HTML EPUB PDF`

**Key sources found:**

| Source | URL |
|--------|-----|
| The eBook Reader Blog — NeoReader walkthrough | https://blog.the-ebook-reader.com/2024/12/03/boox-video-tutorial-a-complete-guide-to-onyxs-neoreader-app/ |
| Boox Shop — 6 Tips for Productive eBook Reading | https://shop.boox.com/blogs/news/6-tips-for-productive-ebook-reading-with-neoreader |
| eReaders Forum — NeoReader walkthrough thread | https://www.ereadersforum.com/threads/neoreader-walkthrough-exploring-features-and-settings-on-boox-devices.3346/ |
| Boox Help — Adjust PDF and Similar | https://help.boox.com/hc/en-us/articles/10701486858388-Adjust-PDF-and-Similar |
| MobileRead Forums (multiple threads) | https://www.mobileread.com/forums/showthread.php?t=343059, https://www.mobileread.com/forums/showthread.php?t=350513 |

**What was learned:**
NeoReader supports PDF, EPUB, EPUB3, HTML, MOBI, AZW3, TXT, DOC, DOCX, RTF, CHM, FB2, CBR, CBZ, DJVU, PPT, and PPTX. The app has extensive settings for font, layout, and refresh mode. It does not support DRM-protected content. PDF handling is considered best-in-class among e-ink readers. The MobileRead forums provided community experiences with various formats on Boox hardware.

**Design impact:**
This confirmed EPUB3 as a viable primary target format (natively supported with good rendering). It also confirmed that HTML, while supported, is a secondary format — the forum posts indicated inconsistent rendering compared to EPUB. This led to the skill targeting EPUB3 output exclusively, with PDF mentioned only as a fallback for fixed-layout content.

---

## Search 2: EPUB CSS and Code Block Formatting

**Query:** `NeoReader EPUB HTML CSS support code blocks formatting`

**Key sources found:**

| Source | URL |
|--------|-----|
| Boox Help Community — NeoReader CSS realization | https://help.boox.com/hc/en-us/community/posts/360075854092-Neo-reader-CSS-realization |
| cmichel.io — How to Create Beautiful EPUB Programming eBooks | https://cmichel.io/how-to-create-beautiful-epub-programming-ebooks |
| GitHub — epub-css-starter-kit (Matt Harrison) | https://github.com/mattharrison/epub-css-starter-kit |
| GitHub — epub-styles (Paul Wellner Bou) | https://github.com/paulwellnerbou/epub-styles |
| Friends of EPUB — eBookTricks | https://friendsofepub.github.io/eBookTricks/ |

**What was learned:**
The cmichel.io article was particularly influential. It described a Pandoc-based workflow for creating EPUBs with source code, noting that many epub.css files look poor with code snippets. It recommended decreasing font-size for code blocks, adding a border on the side, and dealing with the fact that syntax highlighting breaks on some e-readers and horizontal scrollbars appear when code overflows. The article also suggested a two-stage pipeline (Markdown → HTML → EPUB via Calibre) for better results, though Pandoc direct conversion was also viable.

The Friends of EPUB "eBookTricks" resource provided detailed CSS patterns for EPUB readers, including guidance on using `%` for margins (not `em`, which scales inversely with font size), handling legacy RMSDK quirks, and the V2 engine's improved CSS support.

The Boox community post confirmed that NeoReader's CSS support is basic but functional for standard block-level elements, and that the V2 engine provides better CSS fidelity.

**Design impact:**
This directly shaped the CSS stylesheet's approach: `pre-wrap` for code blocks (to prevent horizontal overflow on a device that cannot scroll horizontally), a left border as a visual separator, reduced font size for code, and the use of `em`/`%` units. The cmichel article validated the Pandoc-based pipeline approach and the need for a dedicated CSS file rather than relying on defaults.

---

## Search 3: Monospace Fonts and Custom Font Installation

**Query:** `Boox NeoReader available fonts monospace code custom font install`

**Key sources found:**

| Source | URL |
|--------|-----|
| MobileRead Forums — font threads | https://www.mobileread.com/forums/ (multiple threads) |
| Boox Help — various font-related articles | https://help.boox.com/ |

**What was learned:**
Boox devices allow installing custom fonts by placing `.ttf` or `.otf` files in a `/Fonts/` directory at the root of storage. NeoReader picks these up and offers them in its font selection menu. For EPUB specifically, fonts can also be embedded directly in the EPUB file via `@font-face` declarations, which NeoReader respects. This means the skill could either rely on pre-installed device fonts or embed them in the EPUB itself.

**Design impact:**
The skill embeds JetBrains Mono directly in the EPUB via Pandoc's `--epub-embed-font` flag, making the output self-contained. Users don't need to pre-install anything on their device. The SKILL.md also mentions the `/Fonts/` directory as an alternative for users who prefer to install fonts device-wide.

---

## Search 4: NeoReader HTML Rendering and V2 Engine

**Query:** `Boox NeoReader HTML rendering tables CSS support limitations`

**Key sources found:**

| Source | URL |
|--------|-----|
| Boox V3.1 firmware changelog | https://shop.boox.com/blogs/news/ (firmware post) |
| Onyx International Forums — NeoReader EPUB compatibility request | https://bbs.onyx-international.com/t/feature-request-neoreader-full-epub-compatibility/1994 |

**Direct page fetch:** `https://bbs.onyx-international.com/t/feature-request-neoreader-full-epub-compatibility/1994`

**What was learned:**
The Boox firmware changelog revealed that the V2 engine was introduced to "support original EPUB CSS, including multiple fonts, image and text layout, vertical layout." It requires manual activation in NeoReader's settings. The Onyx forum thread contained user reports of specific CSS features that fail in NeoReader — `visibility: hidden` being ignored, `float` on images behaving unpredictably, and `<hr>` elements sometimes not rendering. Users also reported that the V2 engine occasionally has page navigation bugs, particularly with RTL text.

**Design impact:**
This created the "Reliably Supported" vs "Unreliable/Unsupported" classification in the NeoReader CSS reference document. The skill's CSS was deliberately constrained to only use features in the "reliable" category — no floats, no visibility tricks, no flexbox/grid, no pseudo-elements. The V2 engine recommendation ("try V2 first, fall back to V1") came directly from this research.

![NeoReader CSS support: reliably supported features vs unreliable or unsupported features](/assets/images/boox-epub-research/neoreader-css-support.svg)

---

## Search 5: Best Monospace Fonts for E-Ink Displays

**Query:** `best monospace font e-ink display code reading programming`

**Key sources found:**

| Source | URL |
|--------|-----|
| MobileRead Forums — font readability threads | https://www.mobileread.com/forums/showthread.php?t=366520 |
| Various font comparison and programming font recommendation sites | (multiple results) |

**Direct page fetch:** `https://www.mobileread.com/forums/showthread.php?t=366520`

**What was learned:**
Community consensus pointed to JetBrains Mono as the top choice for code on e-ink, due to its increased x-height (improving legibility at small sizes), clear character distinction (0 vs O, 1 vs l vs I), and good weight that renders well on e-ink's limited greyscale. Source Code Pro (Adobe), Fira Mono/Fira Code (Mozilla), Inconsolata, and DejaVu Sans Mono were also well-regarded. The MobileRead thread discussed how e-ink's 16 greyscale levels and lack of sub-pixel rendering mean that fonts with fine hairline strokes (like Consolas or some light weights) can disappear, while fonts with consistent stroke width render cleanly.

For body text, the discussion favoured clean sans-serifs (Roboto, Source Sans, Noto Sans) for technical documents, with serif fonts (Charter, Literata) for longer-form reading.

**Design impact:**
JetBrains Mono was selected as the primary embedded font. The CSS declares a fallback chain (`"JetBrains Mono", "Source Code Pro", monospace`) so that if embedding fails, a reasonable fallback is used. The font sizing at `0.82em` for code blocks was calibrated to balance character count per line (~75 chars) against readability at 300 PPI — informed by the MobileRead discussion about minimum readable sizes on e-ink.

![Font comparison matrix: JetBrains Mono selected as the winner across x-height, stroke consistency, character distinction, and greyscale rendering](/assets/images/boox-epub-research/font-comparison.svg)

---

## Search 6: Pandoc EPUB Conversion with Code and Fonts

**Query:** `pandoc markdown EPUB embedded fonts code blocks syntax highlighting e-ink`

**Key sources found:**

| Source | URL |
|--------|-----|
| cmichel.io (revisited) | https://cmichel.io/how-to-create-beautiful-epub-programming-ebooks |
| GitHub — epub-styles | https://github.com/paulwellnerbou/epub-styles |

**Direct page fetch:** `https://github.com/paulwellnerbou/epub-styles`

**What was learned:**
Pandoc's `--syntax-highlighting` flag accepts named themes. The available themes are: pygments, tango, espresso, zenburn, kate, monochrome, breezedark, and haddock. On a 16-level greyscale display, colour-based themes (pygments, tango, etc.) map many distinct colours to the same grey level, making them useless for differentiating tokens. The `monochrome` theme uses bold for keywords/types and italic for comments — distinctions that survive greyscale perfectly.

The epub-styles repository provided a worked example of a Pandoc-to-EPUB pipeline with custom CSS and embedded fonts, validating the approach of using `--epub-embed-font` and `--css` flags together.

**Design impact:**
The skill mandates `--syntax-highlighting=monochrome` and the CSS reinforces this with explicit `code span.kw { font-weight: bold; }` and `code span.co { font-style: italic; }` rules. The Pandoc command in the SKILL.md and the conversion script both use the monochrome theme. The troubleshooting table includes "syntax colours look identical" → "use monochrome" as an explicit entry.

---

## Search 7: Boox Go 10.3 Hardware Specifications

**Query:** `Boox Go 10.3 screen resolution ppi specifications`

**Key sources found:**

| Source | URL |
|--------|-----|
| eWritable — Boox Go 10.3 specs | https://ewritable.net/brands/boox/tablets/boox-go-10-3/ |
| Good eReader — Full Review | https://goodereader.com/blog/reviews/full-review-of-the-onyx-boox-go-10-3-with-300-ppi |
| My eReader Substack — Review | https://myereader.substack.com/p/onyx-boox-go-103-review-the-combination |
| B&H Photo — Product listing | https://www.bhphotovideo.com/c/product/1836430-REG/boox_opc1213r_10_3_go_e_reader.html |

**What was learned:**
Confirmed specs: 10.3" E Ink Carta 1200 display, 2480×1860 pixels, 300 PPI, 16 greyscale levels, no front light, Android 12, 4GB RAM, 64GB storage. The 4:3 aspect ratio and physical screen dimensions (~190×140mm usable area in portrait) were important for calculating how many characters fit per line at various font sizes.

**Design impact:**
These specs were embedded directly in the SKILL.md's "Target Device Specs" table and used to calculate the code block font sizing. At 0.82em (roughly 9.5pt at default NeoReader settings), JetBrains Mono fits approximately 75 characters per line on the 10.3" screen — enough for most code without excessive wrapping. The Mermaid diagram rendering width (2000px) was chosen to map well to the 2480px screen width. The greyscale visibility guide in the SKILL.md was derived from the 16-level constraint.

---

## Search 8: Font Readability on E-Ink

**Query:** `fonts readability e-ink e-reader`

**Direct page fetch:** `https://www.mobileread.com/forums/showthread.php?t=366520`

**What was learned:**
This MobileRead thread contained detailed user experiences comparing fonts on e-ink screens. Key points: e-ink's binary pixel rendering (no sub-pixel antialiasing) means fonts need consistent stroke widths to remain legible. Thin/light font weights can vanish. Bold weights render more reliably. Users recommended bumping NeoReader's "bolding" slider up 1-2 notches for small text, particularly code. Serif fonts at small sizes can have strokes that fall between greyscale levels, causing a "shimmer" effect; sans-serif tends to be cleaner for technical content.

**Design impact:**
The SKILL.md includes the NeoReader settings recommendation to bump the bolding slider for code-heavy documents. The CSS uses `font-weight: bold` rather than `font-weight: 600` or `700` to ensure maximum compatibility. The body text is set in serif (inheriting NeoReader's default) but with the note that sans-serif alternatives work well for technical documents.

---

## Search 9: EPUB vs PDF for Technical Documents

**Query:** `EPUB vs PDF technical documents code e-ink reader Boox reflowable`

**Key sources found:**

| Source | URL |
|--------|-----|
| eWritable — Introduction to E-Reader File Formats | https://ewritable.net/guides/an-introduction-to-e-readers-file-formats/ |
| Wikipedia — Comparison of e-book formats | https://en.wikipedia.org/wiki/Comparison_of_e-book_formats |
| Map Systems — EPUB vs PDF Format | https://mapsystemsindia.com/resources/epub-vs-pdf-format.html |
| Boox Shop — Best PDF Reader | https://shop.boox.com/blogs/news/best-pdf-reader |

**What was learned:**
EPUB's reflowable layout is critical on the 10.3" screen — PDF's fixed layout requires zooming and panning to read standard A4/Letter-sized documents, which is slow on e-ink's refresh rate. EPUB allows font size adjustment, reflow to screen width, and CSS customisation. PDF's advantage is preserving complex layouts (multi-column, precise diagram positioning) exactly. For text-heavy technical documents with inline code, EPUB wins decisively. For documents that are primarily diagrams or have complex spatial layouts, PDF is better.

**Design impact:**
This confirmed the decision to make EPUB the primary output format. The SKILL.md mentions PDF as a fallback for "complex fixed layouts only" and includes a PDF Pandoc command using custom paper dimensions matching the Boox screen for when PDF is genuinely needed. The `--top-level-division=chapter` flag ensures H1s create page breaks, giving the EPUB a book-like structure that works well with NeoReader's chapter navigation.

![EPUB vs PDF comparison: EPUB as primary format for reflowable text, PDF as fallback for complex layouts](/assets/images/boox-epub-research/epub-vs-pdf.svg)

---

## Sources Not From Web Search

In addition to the web research, several design decisions drew on existing knowledge:

- **Pandoc's `--syntax-highlighting` options** were verified by running `pandoc --list-highlight-styles` locally, confirming `monochrome` was available
- **mmdc (Mermaid CLI)** was confirmed available in the environment, validating the Mermaid pre-rendering approach
- **`white-space: pre-wrap`** as the code wrapping strategy came from general CSS knowledge, but its NeoReader compatibility was confirmed via the Boox community posts
- **The `orphans`/`widows` CSS properties** are standard pagination controls included based on general EPUB best practices, not specific to any single source

---

## Summary: Research → Design Mapping

![Bipartite graph connecting 9 research sources to 11 skill components, showing which research informed each design decision](/assets/images/boox-epub-research/research-design-mapping.svg)
