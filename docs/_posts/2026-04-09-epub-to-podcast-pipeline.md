---
draft: true
layout: post
title: "£25 for 42 Books I'd Never Read, So I Turned Them into AI Podcasts"
date: 2026-04-09
categories: [projects, ai]
tags: [python, tts, elevenlabs, epub, podcasts, eleventy, text-to-speech, claude-code]
excerpt: "I bought 42 technical ebooks in a Humble Bundle for £25 but had no time to read them. I built a pipeline (Python scripts, Claude Code skills, ElevenLabs TTS) that turns them into two-host podcasts I can listen to anywhere."
---

It started with a Humble Bundle deal. Forty-two DRM-free technical ebooks (security, software architecture, AI/ML) for £25. I hit "purchase" without thinking twice.

Then reality set in. I had no e-reader and no free time. And these weren't light reads: *Building Secure and Reliable Systems*, *Threat Modeling*, *Natural Language Processing with Transformers*. Dense books that would sit in a folder gathering digital dust.

I started thinking about York Notes, those condensed study guides that got half of Britain through their GCSEs. I wanted something I could absorb passively. Listen to on my commute or while cooking.

A conversational podcast felt right. Two hosts discussing each book's key ideas, the way you'd chat with a colleague who'd read the same thing. A back-and-forth that makes technical concepts stick.

I built a multi-stage pipeline: Python scripts, Claude Code skills, and AI text-to-speech. EPUBs go in one end, podcast audio comes out the other, served on a static site with paragraph-level text synchronisation. Fifty-five episodes, 1.4 gigabytes of audio.

## The Pipeline at a Glance

![Full pipeline: EPUB processing, AI script generation, audio production, and Eleventy website](/assets/images/epub-to-podcast/pipeline-overview.svg)

Each stage feeds the next, but every intermediate output is useful on its own. The extracted markdown works for reading in any text editor. The LLM-optimised version is clean enough for RAG pipelines or embeddings. The minimised summaries work for quick reference before a meeting.

## Progressive Compression: From 1.6MB to 30KB

Getting content out of EPUB files was the first step. EPUBs are ZIP archives containing XHTML, CSS, and images. I used `ebooklib` to crack them open, `BeautifulSoup` to rewrite image paths, and `html2text` to produce clean markdown. Across 42 books, this extracted over 3,400 images.

The raw markdown is comprehensive but noisy. Publisher boilerplate (copyright pages, O'Reilly contact information, praise sections, table of contents) adds nothing for learning. A regex-based cleanup pass strips this out, preserving every line of technical content while removing the metadata noise. Typically 1-5% reduction.

The real compression happens in the minimisation stage. For each chapter, the script extracts:

- Topics covered: every section heading, giving you the chapter's skeleton
- Key concepts: the 5 most substantial paragraphs, capped at 500 characters each
- Code indicators: whether examples exist, with the first short one preserved
- Author summaries: the original chapter summary sections, if present

Across stages, a typical book compresses like this:

![Compression funnel: 1.4 MB extracted to 1.37 MB optimised to 99 KB minimised to 30 KB podcast script](/assets/images/epub-to-podcast/compression-funnel.svg)

A book that started as 1.4 megabytes of dense technical prose becomes a 30-kilobyte podcast script.

## Analysing Books Before Scripting

Before generating podcast scripts, I wanted to understand each book's thematic structure. I built a **book-analyzer** skill for Claude Code, a reusable workflow that extracts text from any EPUB or markdown file and generates five SVG visualisations:

- Word cloud: frequency-based term sizing in a spiral layout, showing what the book talks about most
- Mind map: radial hierarchy with 8 theme branches and sub-nodes, showing how concepts cluster
- Concept network: nodes and edges showing which themes co-occur, sized by frequency
- Theme timeline: line chart tracking how 8 themes evolve across 10 segments of the book
- Topic heatmap: matrix of 12 themes across 8 sections, colour intensity showing density

The skill chains together seven Python scripts (`extract_text.py`, `analyze_themes.py`, and five generators). Running `/book-analyzer` on an EPUB produces all five SVGs in seconds. The timeline and heatmap were the most useful for planning podcast scripts, showing which chapters deserve extended discussion.

## Two Approaches to Script Generation

![Comparison: V1 deterministic Python (fast, free, patterned) vs V2 Claude Code Skill (slower, costs tokens, natural dialogue)](/assets/images/epub-to-podcast/script-generation-comparison.svg)

I ended up with two approaches.

### V1: Deterministic Python

The first script generator (`generate_podcast_script.py`) is 100% deterministic Python. No API calls. No tokens. It takes the minimised chapter data and weaves it into two-host conversation using pattern cycling:

```python
# Vary the opening style
opening_styles = [
    f"**{HOST_B}**: So let's talk about {chapter_data['title']}.",
    f"**{HOST_B}**: Alright, {chapter_data['title']}.",
    f"**{HOST_B}**: Moving into {chapter_data['title']} now.",
    f"**{HOST_A}**: What's next?",
]

if chapter_num == 1:
    script.append(f"**{HOST_B}**: Let's start with {chapter_data['title']}. "
                  "This is where the author lays the groundwork.")
elif chapter_num == total_chapters:
    script.append(f"**{HOST_B}**: And we've reached the final chapter: "
                  f"{chapter_data['title']}.")
else:
    script.append(opening_styles[chapter_num % len(opening_styles)])
```

Alex's clarifying questions rotate through five patterns. Reactions cycle through four templates. Sam's explanations draw directly from the extracted key concepts, truncated and woven into dialogue frames.

It's fast (milliseconds per book), free (zero API cost), and consistent (no hallucinations, no off-topic tangents). The trade-off: if you binge-listen, you'll notice the patterns. The `chapter_num % len(opening_styles)` cycling becomes audible.

### V2: Claude Code Skill

The deterministic approach was a good start, but it produced scripts that sounded like scripts. I wanted something that sounded like two knowledgeable people in conversation. I built a **podcast-script-generator** skill for Claude Code.

A Claude Code skill is a reusable prompt package: a SKILL.md file with workflow instructions, reference documents, and supporting scripts. When I run `/podcast-script-generator`, Claude reads the book content and generates dialogue guided by three reference documents:

Book type profiles define how the conversation adapts to the source material. I created nine profiles (technical, fiction, children's, memoir, self-help, history, science, philosophy, and business), each specifying host personas, tone, structural approach, and question types. For a technical book, Sam is a "practitioner with 10-15 years of experience" who speaks to trade-offs. For fiction, Sam becomes an "engaged reader" who analyses themes and craft. For hybrid books (say, a technical memoir), you blend profiles.

Dialogue quality standards enforce the rhythm that makes conversations feel real. Response lengths must vary: 30% short (1 paragraph), 50% medium (2-3 paragraphs), 20% long (4-5 paragraphs). Alex asks questions roughly 40% of the time. There should be 5-10 challenge moments per script where Alex pushes back constructively, and 2-4 discovery moments where the hosts connect ideas across chapters. Each chapter needs 12-16 speaking turns minimum, not 4-6 long monologues.

Script structure defines the shape of an episode: intro, chapter discussions, midpoint reflection, closing. But the critical instruction is to vary patterns chapter to chapter. No formulaic transitions. No "Let's move on to the next chapter."

The skill includes bad-vs-good dialogue examples:

```text
❌ BAD:
Sam: This chapter talks about dependency injection.
Alex: That's interesting. What does that mean?

✅ GOOD:
Sam: Chapter 13 tackles dependency injection, and what struck me
is how Percival and Gregory frame it as this natural evolution.
You start with a simple Flask app, and suddenly you're passing
four or five dependencies into every function.

Alex: That's the moment where DI stops being theoretical, right?
When your test setup becomes this nightmare of manual wiring.
```

V1 scripts are functional: they cover the content and sound passable. V2 scripts sound like two people who read the book and have opinions about it. The hosts challenge each other, notice connections, and build ideas through dialogue.

The trade-off is cost and speed. V2 scripts take minutes to generate instead of milliseconds, and they consume API tokens. For a project where the output becomes permanent audio, it's worth it.

## Text-to-Speech: Dialogue Batching and Contextual Prosody

I tested three TTS providers: OpenAI TTS, Google Cloud TTS, and ElevenLabs. ElevenLabs Flash v2.5 won on both quality and efficiency.

Dialogue batching made the biggest difference. Most TTS integrations send one segment at a time, one API call per line of dialogue. With a typical podcast script containing 100+ dialogue segments, that's 100+ API calls with network overhead, rate limiting, and cost accumulation.

Instead, I batch consecutive same-speaker segments together, up to 90% of the model's character limit:

```python
# Check if we should start new batch
should_start_new = (
    current_batch['speaker'] is not None and
    (current_batch['speaker'] != speaker or
     len(current_batch['text']) + len(text) + 1 > max_chars)
)
```

With ElevenLabs Flash v2.5's 40K character limit, I use batches of up to 36K characters. A typical podcast script (20-40K characters) fits in 1-2 API calls instead of 100+. That's a 90% reduction in API calls.

The other optimisation is contextual prosody. ElevenLabs accepts `previous_text` and `next_text` parameters that give the model surrounding context:

```python
params = {
    "text": text,
    "voice_id": voice,
    "model_id": model_config["model_id"],
    "output_format": "mp3_44100_128"
}

# Add context parameters if provided (improves prosody)
if previous_text:
    params["previous_text"] = previous_text
if next_text:
    params["next_text"] = next_text
```

Without context, each batch sounds like it's starting a new conversation. With it, the intonation at the end of one batch flows into the beginning of the next.

The gap-filling system handles partial failures. TTS calls can timeout or fail mid-conversion. Rather than re-generating an entire podcast (expensive and slow), `fill_audio_gaps.py` identifies which segments are missing and regenerates only those, with surrounding context for prosody continuity. Cost to fill 11 gaps after a parser bug fix: less than one penny.

Total TTS cost across the entire project: roughly £50.

## The Listening Experience

The final piece is the website. I chose Eleventy 3.1.2 with Nunjucks templates, vanilla JavaScript, and zero framework dependencies. It builds 55+ podcast pages in 0.63 seconds.

The site has a sidebar with cover art thumbnails organised by category: Security, Architecture, AI/ML, Children's Literature, Fiction, Natural History. A live search filters podcasts as you type. Speaker dialogue is colour-coded: Sam in blue, Alex in red.

The audio player is built from scratch. Timeline scrubber, volume control, keyboard shortcuts (space for play/pause, arrow keys to seek). It remembers your playback position per podcast in localStorage, so you can pick up where you left off.

Paragraph-level audio-text synchronisation ties the audio and transcript together. Timing data, extracted per paragraph with start and end timestamps, gets injected into the HTML at build time as `data-start` and `data-end` attributes. At runtime, a small JavaScript class watches the audio's `timeupdate` event and highlights the current paragraph. Click any paragraph to seek the audio to that point. It's bidirectional: the audio drives the text, and the text drives the audio.

There are five colour themes (Light, Dark, Sepia, Ocean, Forest) and eight font options, all persisted in localStorage. The whole thing deploys as a Docker container with Nginx on Fly.io, a 256MB VM that scales to zero when nobody's listening.

No React, no bundler, no npm packages beyond Eleventy itself.

## What I Learned

Start simple, then layer quality. The deterministic Python script generator got me from zero to working podcasts in a day. The Claude Code skill took weeks to refine but produced much better output. Both still exist in the repo. V1 meant I could iterate on the TTS pipeline, the website, and the audio processing without waiting for the "perfect" script generator.

Claude Code skills compress a lot of knowledge into a small format. A skill is a markdown file with workflow instructions, reference documents, and supporting scripts. The podcast-script-generator skill's three reference documents (book type profiles, dialogue quality standards, script structure) encode everything I learned about what makes podcast dialogue work. Anyone with Claude Code can run `/podcast-script-generator` on a book and get the same quality.

Build pipelines with useful intermediate artifacts. Every stage of this pipeline produces something independently useful. The extracted markdown, the LLM-optimised text, the minimised summaries, the thematic visualisations all serve different purposes. When I wanted to add RAG support later, the `llm_optimised/` output was already sitting there.

TTS quality has crossed the useful threshold. ElevenLabs Flash v2.5 with custom voices sounds natural enough that people have asked me who the podcast hosts are. The contextual prosody parameters are the difference between "obviously AI" and "wait, is this AI?"

Design for partial failure. The gap-filling pattern (diff what you expected against what you have, regenerate only the delta) applies to any pipeline where regeneration is expensive. It saved me hours and pounds during development.

## By the Numbers

![Stats: 42 books, 55+ episodes, 2,682 segments, 1.4 GB audio, ~£50 TTS cost, £25 purchase, 0.63s build time](/assets/images/epub-to-podcast/stats-infographic.svg)

That Humble Bundle paid for itself.
