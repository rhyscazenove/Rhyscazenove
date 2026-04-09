# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A personal blog at blog.rhyscazenove.com. Jekyll site hosted on GitHub Pages, source in the `docs/` directory.

## Build and Serve

```bash
cd docs
bundle install          # first time / after Gemfile changes
bundle exec jekyll serve --livereload
```

Site is at http://localhost:4000. GitHub Pages builds automatically on push to main.

## Site Structure

```
docs/
  _config.yml           # site config (title, plugins, permalink pattern)
  _layouts/             # default > post, page, blog, home
  _includes/            # header.html, footer.html
  _posts/               # blog posts (YYYY-MM-DD-slug.md)
  pages/                # standalone reference pages
  presentations/        # HTML slide decks
  assets/css/style.css  # single stylesheet, CSS custom properties for theming
  index.md              # homepage (layout: default)
  blog.md               # blog listing (layout: blog)
  about.md              # about page (layout: page)
```

## Content Conventions

### Blog posts (`_posts/`)

Filename: `YYYY-MM-DD-title-slug.md`. Required frontmatter:

```yaml
---
layout: post
title: "Post Title"
date: YYYY-MM-DD
categories: [category1, category2]
tags: [tag1, tag2]
description: "One-line summary shown in post listings."
---
```

Add `published: false` to hide a post from the live site while keeping it in the repo.

### Pages (`pages/`)

Standalone reference pages linked from posts. Required frontmatter:

```yaml
---
layout: page
title: "Page Title"
permalink: /url-slug/
---
```

Pages use `layout: page`, not `layout: post`. No `date` or `categories` fields.

## Theming

All colours use CSS custom properties defined in `:root`. Dark mode has two mechanisms:

1. `@media (prefers-color-scheme: dark)` on `html:not([data-theme="light"])` for system default
2. `html[data-theme="dark"]` for user override via localStorage

The theme toggle in the footer cycles: auto > light > dark. The blocking script in `<head>` applies the saved theme before first paint.

Mermaid diagrams (loaded from CDN when `language-mermaid` code blocks are present) re-render on theme change with matching colour variables.

## Writing Style

All prose on this site goes through the `/deslop` skill before publishing. Key rules:

- No bold-first bullet patterns (every list item starting with a bolded keyword)
- No em dashes; use commas, periods, or semicolons
- No self-posed rhetorical questions answered immediately
- No meta-commentary ("This post walks through...", "Let's break this down")
- No "here's the thing" / "it's worth noting" filler
- Cut adverbs. Active voice. Name the actor.
- Trust readers; skip hand-holding and pedagogical framing

Run `/deslop` on any new or edited prose before considering it done.
