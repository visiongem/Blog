---
name: blog-voice
description: Use when writing, editing, reviewing, or proofreading any blog post (content/posts/*.md) for 妮K妮K妮 — including any request to 审核/审/检查/校对/review a post. ALWAYS invoke this skill for those tasks. It enforces the author's learning-in-public voice and blocks AI writing patterns the author consistently rejects.
---

# Blog Voice: 妮K妮K妮

## Core Principle

**The author is documenting their own learning, not teaching others.** Every post is a study note written for themselves first. The act of explaining is the act of learning.

## Positioning Rule

Write as someone who just figured something out and is taking notes so they don't forget — not as someone delivering a lecture. **These are summary notes, not a tutorial: do not address a reader as "你".** There is no student on the other end; the writing talks to no one but the author's future self. When a sentence would use "你", either drop the pronoun (「可以看到…」「需要注意…」) or rephrase impersonally.

## Voice Markers

### Do (from the author's actual writing)

| Pattern | Example |
|---------|---------|
| Learning-journal framing | "今天开始 Kotlin Multiplatform 的学习" |
| Self-directed soft endings | "...吧" at end of opening sentence |
| Metacommentary / self-talk | "咦，前面不是一直在讲 iOS 吗？" |
| Signature diary close | "今日份学习先到这。" |
| Humble positioning | "为了自己学习", "整理一下。" |
| Casual connectives | "因此", "那", "好" |

### Don't (AI patterns the author rejects)

| Pattern | Example | Why rejected |
|---------|---------|--------------|
| Metaphorical flourish | "让客户端开口说话" | Performative, sounds like a TED talk |
| Authoritative declarations | "这篇讲 X" | Author documents learning, doesn't announce |
| "在这篇文章中" / "本文将" | — | Academic AI filler |
| "首先...其次...最后" chains | — | Sounds like a textbook |
| "值得注意的是" / "关键的是" | — | Generic AI emphasis |
| Pseudo-inspirational closings | "让我们一起探索" | Not the author's voice |
| "顾名思义" / "众所周知" | — | Lazy AI transitions |
| "到底" 开头的追问句式 | "到底是什么" / "究竟为什么" / "到底怎么..." | Dramatic suspense the author never uses; plain "什么是" without 到底 is fine |
| 频繁用"你"称呼读者 | "你可以看到" / "你需要注意" / "如果你..." | This is a self-note, NOT teaching someone. Either drop the pronoun entirely (「可以看到...」「需要注意...」) or rephrase impersonally. Occasional 自指的"我" is fine; addressing a reader as "你" is not. |

## Opening Sentence Formula

Bad (AI voice):
```
第一篇把项目骨架跑起来了。这篇做X——用Y，拿到Z，再显示在W上。
```

Good (author's voice):
```
第一篇...建立好了，为了自己学习，...也在。因此今日份学习如何...吧。
```

Pattern: acknowledge previous post → humble framing ("为了自己学习") → soft connective ("因此") → what I'm learning today → end with 吧 or similar soft particle.

## Structural Signatures (Keep)

These are the author's consistent patterns across all posts — preserve them:

1. **Thesis-first distillation** — open with what this post is about in one concrete sentence
2. **Code before explanation** — show the thing, then explain it (never the reverse)
3. **Comparison tables** — whenever contrasting two or more concepts, use a table
4. **Real-world analogy per post** — at least one relatable metaphor
5. **Section numbering** — 一、二、三... with subsections like 3.1, 3.2
6. **Numbered takeaways at the end** — "几个关键点：" followed by numbered list

## Punctuation & Formatting

**Quotation marks: use full-width Chinese quotes 「“…”」 in body text — never straight ASCII quotes `"…"`.**

- All emphasis and quoted phrases inside the article body use `“”` (e.g. 生成的是“Demo”，不是“产品”).
- The ONE exception is YAML frontmatter: `title:` wrapped in single quotes may keep ASCII `"` inside it (e.g. `title: 'Vibe Coding 最大的"骗局"，...'`), and `tags`/`categories` lists use ASCII `"` as required by YAML syntax. Never convert those.
- When editing an existing post, sweep the whole body and normalize any stray `"…"` to `“…”`.

**Markdown rendering conflicts — bold + quotes together often break.** Because the body is Markdown, combining `**加粗**` with full-width quotes (or other adjacent markers) sometimes fails to render — the `**` gets swallowed or the emphasis leaks. Watch for these:

- **Bold wrapping a quoted phrase**: prefer `**“Demo”**` (bold outside the quotes), not `“**Demo**”`. Keep the `**` on the outermost edge so the markers pair cleanly.
- **No stray space between a marker and CJK text**: `** 加粗 **` won't render — write `**加粗**` tight against the text.
- **Punctuation glued to a closing `**`**: a full-width `，。：`right after `**` can swallow the marker. If bold doesn't render, move the punctuation outside the `**`.
- **After writing or editing, re-scan every line that mixes `**` with `“”` or list/heading markers** and confirm the pairing is correct — this is the single most common formatting bug in these posts.

## Frontmatter (Required)

Every post starts with YAML frontmatter — do not begin the file with a `#` H1 title (PaperMod renders the frontmatter `title` as the page heading, so an H1 in the body duplicates it):

```yaml
---
title: 'Post Title'
date: 2026-07-17T21:00:00+08:00
draft: false
tags: ["tag1", "tag2"]
categories: ["分类"]
---
```

## Closing Formula

Two options depending on the post type:

**Learning-journal posts (KMP series, new topics):**
```
今日份学习先到这。
```

**Teaching-leaning posts (established expertise topics):**
Wrap back to the opening thesis with a crisp summary, then sign off.

**All posts end with:**
```
🌈关注我吖~❤️

公众号：**妮K妮K妮**
```

## Red Flags — Rewrite Immediately

Any of these in draft text means the AI voice took over:

- Metaphors that personify code ("开口说话", "学会思考")
- "这篇" used as a subject declaring what will happen
- Sentences that could be a conference talk title
- Three or more "的" in one sentence (too formal)
- Any sentence starting with "值得注意的是" / "需要强调的是" / "可以看到"
- Closings that sound like a motivational speech
- Missing "吧/呢/吖/呀" when the tone calls for softness
- Section headings with "到底" ("到底是什么" / "究竟为什么") — the author never builds suspense through dramatic rhetorical questions
