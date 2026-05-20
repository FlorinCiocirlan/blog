# florin.log

An editorial-style engineering blog built with Astro. Light theme, serif headlines, terminal-style code blocks. Posts authored as Markdown.

## Run

```sh
npm install
npm run dev      # http://localhost:4321
npm run build    # static output in ./dist
npm run preview  # serve the build locally
```

## Write a post

Drop a `.md` (or `.mdx`) file in `src/content/posts/`. Frontmatter:

```yaml
---
title: "Your title"
description: "One-sentence dek that shows up on the index page."
pubDate: 2026-05-20
tags: ["angular", "engineering"]
readingTime: "8 min read"   # optional
draft: false                # optional, hides from index when true
---
```

Then write Markdown. Supported out of the box:

- **Headings, lists, blockquotes, links** — standard Markdown.
- **Images** — drop them in `public/` and reference as `/your-image.png`, or use remote URLs. Use the `<figure>` HTML element if you want a caption.
- **Code blocks** — fenced with the language for syntax highlighting (`ts`, `tsx`, `js`, `bash`, `sql`, ...). Line numbers and a corner label render automatically.
- **Inline code** — backticks. Renders in vermilion.
- **Pull-quotes** — wrap text in `<p class="pull">...</p>` for an oversized centered quote.
- **MDX** — rename to `.mdx` and you can import components.

## Structure

```
src/
  content/
    posts/         ← your .md / .mdx posts
    config.ts      ← frontmatter schema
  layouts/
    BaseLayout    ← masthead, nav, footer
    PostLayout    ← article shell
  pages/
    index.astro   ← home with post list
    archive.astro ← grouped by year
    tags.astro    ← all tags
    tags/[tag].astro
    posts/[...slug].astro
    rss.xml.ts
  styles/global.css
public/
  favicon.svg
```

## Deploy

Static output — drop `dist/` on Netlify, Vercel, Cloudflare Pages, GitHub Pages, anywhere.

Update `site:` in `astro.config.mjs` to your real domain before building (used by sitemap + RSS).
