---
published: false
layout: post
title: "£25 for 42 Books I'd Never Read — So I Turned Them into AI Podcasts"
date: 2026-04-09
categories: [projects, ai]
tags: [python, tts, elevenlabs, epub, podcasts, eleventy, text-to-speech]
excerpt: "I bought 42 technical ebooks in a Humble Bundle for £25 but had no time to read them. So I built a Python pipeline to turn them into AI-generated two-host podcasts I could listen to anywhere."
---

It started with a Humble Bundle deal. Forty-two DRM-free technical ebooks — security, software architecture, AI/ML — for £25. An absolute bargain. I hit "purchase" without thinking twice.

Then reality set in. I didn't have an e-reader. I didn't have much free time. And these weren't light reads — we're talking *Building Secure and Reliable Systems*, *Threat Modeling*, *Natural Language Processing with Transformers*. Dense, important books that would sit in a folder gathering digital dust.

I started thinking about York Notes — those condensed study guides that got half of Britain through their GCSEs. What if I could create my own version using AI? Not just summaries, but something I could actually absorb passively. Something I could listen to on my commute, while cooking, while walking the dog.

A conversational podcast felt right. Two hosts discussing the key ideas from each book, the way you might chat with a colleague who'd read the same thing. Not a dry audiobook reading — a genuine back-and-forth that makes technical concepts stick.

So I built it. A 7-stage Python pipeline that takes EPUB files in one end and produces podcast-quality audio out the other, served on a static site where you can browse, search, and listen with paragraph-level text synchronization. Fifty-five episodes. 1.4 gigabytes of audio. And the script generator — the part that writes the actual dialogue — doesn't use an LLM at all.

Here's how it works.

## The Pipeline at a Glance

```
EPUB files (42 books)
  │  extract_epubs.py
  v
Markdown + images (extracted/)
  │  optimize_for_llm.py
  v
Clean markdown (llm_optimized/)
  │  create_minimized.py
  v
Chapter summaries (minimized/)         ← 87-98% smaller
  │  generate_podcast_script.py
  v
Two-host dialogue scripts (podcast_scripts/)
  │  tts_converter.py
  v
2,682 audio segments (audio_output/)
  │  combine_podcast_audio.py
  v
55+ podcast MP3s with metadata + cover art
  │  Eleventy static site
  v
Browsable, searchable, listenable website
```

Each stage feeds the next, but every intermediate output is useful on its own. The extracted markdown works for full reading in any text editor. The LLM-optimized version is clean enough for RAG pipelines or embeddings. The minimized summaries are perfect for quick reference before a meeting. You can stop at any stage and get value.

## Progressive Compression: From 1.6MB to 30KB

The first challenge was getting content out of EPUB files. EPUBs are essentially ZIP archives containing XHTML, CSS, and images. I used `ebooklib` to crack them open, `BeautifulSoup` to rewrite image paths, and `html2text` to produce clean markdown. Across 42 books, this extracted over 3,400 images.

The raw markdown is comprehensive but noisy. Publisher boilerplate — copyright pages, O'Reilly contact information, praise sections, table of contents — adds nothing for learning. A regex-based cleanup pass strips this out, preserving every line of technical content while removing the metadata noise. Typically 1-5% reduction.

The real compression happens in the minimization stage. For each chapter, the script extracts:

- **Topics covered**: every section heading, giving you the chapter's skeleton
- **Key concepts**: the 5 most substantial paragraphs, capped at 500 characters each
- **Code indicators**: whether examples exist, with the first short one preserved
- **Author summaries**: the original chapter summary sections, if present

The result is dramatic:

| Stage | Example Size | Reduction | Best For |
|-------|-------------|-----------|----------|
| Extracted | 1.4 MB | baseline | Deep reading |
| LLM-optimized | 1.37 MB | ~3% | RAG, embeddings |
| Minimized | 99 KB | 93% | Quick reference |
| Podcast script | ~30 KB | 98% | Audio production |

A book that started as 1.4 megabytes of dense technical prose becomes a 30-kilobyte podcast script. That's the input for the interesting part.

## The Surprising Decision: No LLM for Script Generation

When I tell people about this project, this is the part that gets a double-take. The podcast script generator — the thing that produces natural-sounding two-host dialogue — is 100% deterministic Python. No API calls. No prompt engineering. No tokens burned.

The two hosts are Sam (big-picture thinker, leads the explanations) and Alex (detail-oriented, asks the clarifying questions you'd want answered). The script generator takes the minimized chapter data and weaves it into conversation using pattern cycling:

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

The same approach applies everywhere. Alex's clarifying questions rotate through five patterns. Reactions cycle through four templates. Sam's explanations draw directly from the extracted key concepts, truncated and woven into natural-sounding dialogue frames.

Why not use an LLM? Three reasons.

**Consistency.** Every episode hits the same quality floor. No hallucinations. No off-topic tangents. No one episode where the AI decided to get creative and spend 500 words on a metaphor about gardening.

**Speed and cost.** A script generates in milliseconds. Across 42 books, that's still under a second total. Zero API spend.

**Editability.** Want to change how Alex asks questions? Edit five lines. Want Sam to be less verbose? Adjust one truncation parameter. The templates are plain Python — no prompt archaeology required.

The honest trade-off: if you binge-listen to five episodes back-to-back, you'll notice the patterns. The `chapter_num % len(opening_styles)` cycling is audible. But for the intended use case — one episode at a time during a commute — it works remarkably well. And there's always the option of an LLM editing pass on top for additional variety.

## Text-to-Speech: Dialogue Batching and Contextual Prosody

I tested three TTS providers: OpenAI TTS, Google Cloud TTS, and ElevenLabs. ElevenLabs Flash v2.5 won on both quality and efficiency.

The key insight was **dialogue batching**. Most TTS integrations send one segment at a time — one API call per line of dialogue. With a typical podcast script containing 100+ dialogue segments, that's 100+ API calls with network overhead, rate limiting, and cost accumulation.

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

The second trick is **contextual prosody**. ElevenLabs accepts `previous_text` and `next_text` parameters that give the model surrounding context, so it knows what came before and what comes after:

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

This makes a noticeable difference. Without context, each batch sounds like it's starting a new conversation. With context, the voice carries natural continuity — the intonation at the end of one batch flows into the beginning of the next.

One production-grade detail I'm proud of: the gap-filling system. TTS calls can timeout or fail mid-conversion. Rather than re-generating an entire podcast (expensive and slow), `fill_audio_gaps.py` identifies which segments are missing and regenerates only those, with surrounding context for prosody continuity. Cost to fill 11 gaps after a parser bug fix: less than one penny.

Total TTS cost across the entire project: roughly £50.

## The Listening Experience

The final piece is the website. I chose Eleventy 3.1.2 with Nunjucks templates, vanilla JavaScript, and zero framework dependencies. It builds 55+ podcast pages in 0.63 seconds.

The site has a sidebar with cover art thumbnails organized by category — Security, Architecture, AI/ML, Children's Literature, Fiction, Natural History. A live search filters podcasts as you type. Speaker dialogue is colour-coded: Sam in blue, Alex in red.

The audio player is built from scratch. Timeline scrubber, volume control, keyboard shortcuts (space for play/pause, arrow keys to seek). It remembers your playback position per podcast in localStorage, so you can pick up where you left off.

The feature I'm most pleased with is **paragraph-level audio-text synchronization**. Timing data — extracted per paragraph with start and end timestamps — gets injected into the HTML at build time as `data-start` and `data-end` attributes. At runtime, a small JavaScript class watches the audio's `timeupdate` event and highlights the current paragraph. Click any paragraph to seek the audio to that point. It's bidirectional: the audio drives the text, and the text drives the audio.

There are five colour themes (Light, Dark, Sepia, Ocean, Forest) and eight font options, all persisted in localStorage. The whole thing deploys as a Docker container with Nginx on Fly.io — a 256MB VM that scales to zero when nobody's listening.

No React. No build-time JavaScript bundling. No npm packages beyond Eleventy itself. It's fast, it's accessible, and it does exactly what it needs to do.

## What I Learned

**Deterministic beats generative for structured content.** LLMs are brilliant at creative, open-ended tasks. But for formulaic content that needs to be consistent across 42 books, plain Python with pattern cycling is faster, cheaper, and more predictable. Save the LLM budget for where it actually matters.

**Build pipelines with useful intermediate artifacts.** Every stage of this pipeline produces something independently valuable. This wasn't deliberate at first — I just needed intermediate steps for debugging. But it turned out to be one of the best architectural decisions in the project. When I wanted to add RAG support later, the `llm_optimized/` output was already sitting there.

**TTS quality has crossed the "good enough" threshold.** ElevenLabs Flash v2.5 with custom voices sounds natural enough that people have asked me who the podcast hosts are. The contextual prosody parameters are the difference between "obviously AI" and "wait, is this AI?"

**Design for partial failure.** The gap-filling pattern — diff what you expected against what you have, regenerate only the delta — is applicable to any pipeline where regeneration is expensive. It saved me hours and pounds during development.

**Static sites remain underrated.** Vanilla JS and Eleventy built something polished in days that a framework-heavy approach would have taken weeks. Sometimes the boring technology is the right technology.

## By the Numbers

- **42** books processed (22 security, 18 architecture, 7 AI/ML, 3 children's)
- **55+** podcast episodes generated
- **2,682** individual audio segments
- **1.4 GB** total audio output
- **0** LLM API calls for script generation
- **~£50** total TTS cost (ElevenLabs Flash v2.5)
- **0.63s** Eleventy build time for the entire site
- **£25** original Humble Bundle purchase price

That Humble Bundle turned out to be quite the bargain after all.
