# parthg.dev

Personal site for Parth Gupta. Hosts the blog and a short list of projects.

Built on Astro with the [AstroPaper](https://github.com/satnaing/astro-paper)
theme as a starting point. Mostly static, no client JS beyond the theme
toggle, the post-page progress bar, and view-transition glue.

## Stack

- **Astro** — static site generator
- **AstroPaper** — theme foundation (heavily trimmed)
- **Tailwind CSS v4** — styling
- **TypeScript** — config and components
- **Markdown** — blog posts and project entries (content collections)

## Local development

Requires Node 20+ (Node 22+ recommended).

```bash
npm install
npm run dev        # http://localhost:4321
npm run build      # type-check + build to ./dist
npm run preview    # preview the built site
```

## Project layout

```
src/
  config.ts                # site identity (title, URL, author, timezone, ...)
  constants.ts             # social links shown in header/footer
  content.config.ts        # Astro content-collection schemas (blog + projects)
  data/
    blog/                  # one .md file per post
    projects/              # one .md file per project (4 entries)
  pages/
    index.astro            # homepage (name, tagline, recent posts)
    about.md               # /about
    projects.astro         # /projects
    posts/[...page].astro  # /posts (paginated index)
    posts/[...slug]/       # /posts/<slug> detail pages
    rss.xml.ts             # /rss.xml
  components/              # Header, Footer, Card, Datetime, ...
  layouts/                 # Layout, Main, AboutLayout, PostDetails
public/                    # static assets served at /
```

## Adding a blog post

Create a new file in `src/data/blog/<slug>.md`:

```markdown
---
title: "Post title"
description: "One-line summary used for OG tags and the post list."
pubDatetime: 2026-05-01            # required
modDatetime: 2026-05-02            # optional, set when you edit
tags: ["ai-agents", "defi"]        # optional, currently no tag UI
draft: false                       # optional, true to hide
featured: false                    # optional, currently unused
---

Post body in markdown. Use code fences, lists, etc. as normal.
```

Frontmatter schema lives in [`src/content.config.ts`](src/content.config.ts).
The post will appear automatically on `/`, `/posts`, and `/rss.xml`,
sorted by `pubDatetime` descending.

## Editing a project

Project entries live in `src/data/projects/<slug>.md`. Each file is YAML
frontmatter only — no body. Schema:

```markdown
---
name: "Project name"
description: "One-line description."
tech: ["TypeScript", "Solidity"]
hackathon: "Some Hackathon 2025"
recognition: "Winner"
team: "Solo build."                # or "Team of 2 — I built X, ..."
order: 1                           # render order on /projects
links:
  - label: "GitHub"
    href: "https://github.com/..."
  - label: "DoraHacks"
    href: "https://dorahacks.io/buidl/..."
draft: false                       # optional
---
```

The `/projects` page reads them sorted by `order` ascending.

## Deploying

The site is a fully static build (`./dist` after `npm run build`). Configs
for both Cloudflare Pages and Vercel are checked in.

### Cloudflare Pages (recommended)

1. Connect this repo to a new Pages project at
   [dash.cloudflare.com](https://dash.cloudflare.com/?to=/:account/pages).
2. Use these settings (also encoded in `wrangler.toml`):
   - Framework preset: **Astro**
   - Build command: `npm run build`
   - Build output directory: `dist`
   - Node version: `20` (or newer)
3. Add the custom domain `parthg.dev`:
   - In the Pages project → **Custom domains** → add `parthg.dev` and `www.parthg.dev`.
   - Cloudflare will create the CNAME / ALIAS records for you if the domain
     is on Cloudflare DNS. If it's elsewhere, point the apex (`parthg.dev`)
     at `<project>.pages.dev` via ALIAS / ANAME, and `www` via CNAME.

`public/_headers` adds basic security headers on the Pages CDN.

### Vercel (alternative)

1. Import this repo at [vercel.com/new](https://vercel.com/new). Vercel
   will detect Astro automatically (settings are also pinned in
   `vercel.json`).
2. Custom domain: in **Project → Settings → Domains**, add `parthg.dev` and
   `www.parthg.dev`. Vercel will give you the records to set on your DNS
   provider (an A record for the apex and a CNAME for `www`).

### Custom domain notes for `parthg.dev`

- Pick **one** host. Don't run both — DNS can only point at one origin.
- For Cloudflare Pages with the domain on Cloudflare DNS, the setup is
  fully automatic via the **Custom domains** tab.
- For the apex `parthg.dev`, prefer ALIAS/ANAME (Cloudflare proxies the
  CNAME for you). Some registrars don't support ALIAS at the apex — in
  that case use the A records the host gives you.
- Add `www.parthg.dev` as a CNAME to your project hostname and let the
  host issue its 301 redirect to the apex.
- HTTPS is provisioned automatically on both hosts; nothing to configure.

## What was deliberately left out

No analytics, no cookie banners, no newsletter, no tag/search/archive UI,
no comments, no share buttons, no related-posts widget. Anything you'd
expect from a "blog template" that's not on the live site was removed on
purpose. The schema still has a `tags` field so future tag UI is a small
addition, not a re-architecture.

## License

Theme foundation © Sat Naing under the
[MIT license](https://github.com/satnaing/astro-paper/blob/main/LICENSE).
Site content © Parth Gupta.
