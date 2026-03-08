# 0xbadcaffe.dev

Personal blog. Kernel // Networking // Reverse Engineering.

Built with [Astro](https://astro.build) and deployed on [Cloudflare Pages](https://pages.cloudflare.com).

## Stack

- **Framework**: Astro 4
- **Styling**: Vanilla CSS (Blade Runner theme)
- **Content**: Markdown / MDX via Astro Content Collections
- **Hosting**: Cloudflare Pages
- **Domain**: 0xbadcaffe.dev

## Local dev

```bash
npm install
npm run dev
```

Then open http://localhost:4321

## New post

Create a file in `src/content/blog/your-slug.md`:

```markdown
---
title: "Your Post Title"
description: "A short description"
pubDate: 2025-03-01
tags: ["kernel", "rust"]
---

Your content here.
```

## Deploy

Push to `main` → Cloudflare Pages auto-deploys.

## Structure

```
src/
  components/     Nav, Footer, Rain
  content/blog/   Markdown posts
  layouts/        Base, Post
  pages/          index, blog/index, blog/[slug], projects
  styles/         global.css
public/           favicon.svg
```
