# hicke.se

Personal blog. Live at [hicke.se](https://hicke.se).

## Stack

- [Hugo](https://gohugo.io/) 0.147.0 — static site generator
- [PaperMod](https://github.com/adityatelange/hugo-PaperMod) — theme (git submodule)
- GitHub Actions — builds and deploys on push to `master`
- Cloudflare Workers — hosts and serves the site globally

## Writing a post

Create a Markdown file in `content/posts/`:

```markdown
---
title: "Post title"
date: 2026-05-01
draft: false
tags: ["tag"]
---

Content here.
```

Set `draft: false` when ready to publish. Push to `master` — the site is live in ~30 seconds.

Images go in `static/images/` and are referenced as `![alt](/images/photo.jpg)`.

## Local preview

```bash
hugo server
```

Requires [Hugo extended](https://gohugo.io/installation/) 0.146.0 or later.

## Repo structure

```
.github/workflows/deploy.yml   # CI/CD pipeline
content/
  archive.md                   # archive page
  posts/                       # blog posts
themes/PaperMod/               # theme (git submodule)
hugo.toml                      # Hugo config
wrangler.jsonc                 # Cloudflare deployment config
```

## CI/CD

Every push to `master` triggers the GitHub Actions workflow which:
1. Checks out the repo including the PaperMod submodule
2. Builds the site with Hugo
3. Deploys to Cloudflare Workers via Wrangler

Required repository secrets: `CF_API_TOKEN`, `CF_ACCOUNT_ID`.
