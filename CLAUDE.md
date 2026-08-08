# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Yane's personal tech blog — a Hugo static site using the PaperMod theme, hosted on GitHub Pages at `visiongem.github.io/Blog`. Content is primarily Chinese (zh-cn), covering Android/Kotlin development and blockchain/DeFi topics.

## Essential Commands

```bash
# Clone with theme submodule
git clone --recurse-submodules <repo-url>

# Install Hugo (macOS, needs extended version)
brew install hugo

# Local dev server with drafts
hugo server -D
# → http://localhost:1313

# Production build
hugo --minify

# Create a new post
hugo new content posts/<slug>.md
# Then edit the generated file and set draft: false when ready
```

## Architecture

- **Static site generator**: Hugo (extended version). The PaperMod theme is a git submodule at `themes/PaperMod/`.
- **No custom layouts**: The `layouts/` directory and `assets/`, `static/`, `data/`, `i18n/` directories are all empty — everything comes from the PaperMod theme.
- **Config**: Single `hugo.toml` at repo root. PaperMod params (search, social links, edit links, table of contents, code copy buttons) are configured there.

## Content Structure

- **Posts**: `content/posts/<slug>.md` — the main content directory. Each post is a single `.md` file with YAML frontmatter.
- **Special pages**: `content/about.md`, `content/archives.md`, `content/search.md` — these use PaperMod's built-in layouts via the `layout` field in frontmatter.
- **Archetype**: `archetypes/default.md` generates new posts with TOML-format (`+++`) frontmatter, but existing posts use YAML-format (`---`). Both work; be consistent with whatever the surrounding posts use.

## Frontmatter Conventions

Posts use YAML frontmatter with these fields:

```yaml
---
title: "Post Title"
date: 2026-06-02T21:00:00+08:00
draft: false           # true = hidden unless `hugo server -D`
tags: ["tag1", "tag2"]
categories: ["CategoryName"]
---
```

Date format: ISO 8601 with `+08:00` timezone offset. `draft: true` posts only appear locally with the `-D` flag.

## Renaming Posts

Once a post has been published (`draft: false` and pushed), its URL is live — search engines, external links, and RSS all reference it. Renaming the file changes the slug and breaks the old URL (404).

**When you rename or move a post, always add an `aliases` entry to its frontmatter** so the old URL keeps working via a generated redirect page:

```yaml
aliases: ["/posts/<old-slug>/"]   # lowercase, must match the URL as it was actually served
```

Gotchas:
- Hugo lowercases all real page URLs (`ViewModel_and_MVVM.md` → `/posts/viewmodel_and_mvvm/`), but does **not** lowercase `aliases` paths. Write the alias in the same lowercase form the old URL was served under.
- If a rename is unavoidable, add the alias in the **same commit** as the rename.

## Images

Images are hosted externally via jsDelivr CDN + GitHub (not stored in this repo). The URL pattern is:

```
https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/<path>
```

## Deployment

GitHub Actions (`.github/workflows/deploy.yml`) builds and deploys automatically on every push to `master`. No manual steps needed. The workflow uses `peaceiris/actions-hugo` with the latest extended Hugo version and builds with `--minify`.
