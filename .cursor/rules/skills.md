---
description: Skill routing — single source of truth is .claude/skills/, do not duplicate
globs: []
alwaysApply: false
---

# Skill Routing

Skills live in `.claude/skills/` as the single source of truth. Never copy skill content into Cursor rules. Instead, read the relevant skill file on demand.

## Available Skills

When a task matches one of the descriptions below, use the Read tool to load the full skill from its path, then follow it.

| Trigger | Skill Path |
|---------|------------|
| Drafting blog posts, essays, guides, tutorials, long-form content | `.claude/skills/article-writing/SKILL.md` |
| Building or reusing a writing style profile, voice consistency across content | `.claude/skills/brand-voice/SKILL.md` |
| Creating social posts (X/LinkedIn), multi-platform content, repurposing articles | `.claude/skills/content-engine/SKILL.md` |
| Researching markets, competitors, technology trends with sourced claims | `.claude/skills/market-research/SKILL.md` |

## How to Use

1. Identify which skill matches the task.
2. Read `SKILL.md` from the path above — do not guess the content.
3. If the skill references other files (e.g. `brand-voice` → `references/voice-profile-schema.md`), read those too from the same base path.
4. Follow the skill's workflow exactly.

## Important

- `.claude/skills/` is the canonical location. Cursor rules are only a router.
- Run `brand-voice` before `content-engine` or `article-writing` when voice consistency matters.
