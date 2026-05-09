# parthgdev — Claude Guidance

Astro-based personal blog at parthg.dev. Source for posts and projects.

## Writing workflow

The blog has a two-stage flow: drafts mature in the Obsidian vault, then ship to this repo as final posts.

### Where things live

- **Drafts and fragments** → Obsidian vault at `~/Vaults/world/Notes/Drafts/`
  - Outline files, fragment piles, in-progress shaping all stay here
  - One file per post-in-progress, named after the working title
- **Final posts** → `src/data/blog/<slug>.md` in this repo
  - Slug is kebab-case from the final title
  - Move here only when the post is shipped (passed the post-draft checklist)

### Obsidian conventions for drafts

- Use `[[wiki-links]]` to reference other vault notes — search the vault before writing a new concept to see if it already has a note
- No frontmatter required on draft files (Obsidian doesn't need it)
- Outline files can carry section briefs ("Job of this section", "What goes here", "Good when") — that structure is load-bearing, preserve it

### Final post format (Astro frontmatter)

When promoting a draft to `src/data/blog/`, wrap in this frontmatter:

```yaml
---
title: "Post title"
description: "One-sentence description for previews and SEO."
pubDatetime: 2026-MM-DD
tags: ["tag1", "tag2"]
draft: false
---
```

Optional fields (only when needed): `modDatetime`, `featured`, `ogImage`, `canonicalURL`, `hideEditPost`, `timezone`. Defaults: author from site config, tags falls back to `["others"]`.

## Editorial standards

These apply to every post drafted in this repo. They're stricter than the stock `edit-article` skill — honor them on the final pass.

### Voice

- Concise, plain-language tech writing
- Argue with evidence, don't preach
- Skeptical of hype framings, but not mean about it
- Concrete vignettes over abstractions
- One quotable sentence minimum per post

### Post-draft checklist

Run before promoting from Obsidian to `src/data/blog/`:

- [ ] One-sentence thesis is unmistakable (at the top, or implied in the opening section)
- [ ] Every paragraph either advances the argument or earns the reader's attention — if neither, cut
- [ ] No proprietary specifics (internal service names, vendor names, deployment shape)
- [ ] No claim you can't back up if challenged — verify or soften
- [ ] At least one quotable sentence; if none, find it and sharpen it
- [ ] Read aloud once — stumbles get rewritten
- [ ] Cut 25–30% from first draft (target final word count: 1200–1800 unless the post needs more)
- [ ] Last sentence is the takeaway, not a wave-off, summary, or "what do you think?"
- [ ] Paragraphs ≤ 240 characters where possible

## Skills available in this repo

Five writing skills are installed at `.claude/skills/`. Use them in this order across a post's lifecycle:

1. **`/grill-me`** — pre-draft thesis stress-test. Run on the riskiest argument before writing prose.
2. **`/writing-fragments`** — mine raw observations into a pile. Saves to Obsidian drafts.
3. **`/writing-shape`** — argument-shape drafting, paragraph by paragraph with format arguments.
4. **`/writing-beats`** — narrative-shape drafting, beat by beat.
5. **`/edit-article`** — final pass. Apply this checklist, not just the stock 240-char rule.
