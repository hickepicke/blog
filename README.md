# hicke.se

[![Deploy](https://github.com/hickepicke/blog/actions/workflows/deploy.yml/badge.svg)](https://github.com/hickepicke/blog/actions/workflows/deploy.yml)
[![Hugo](https://img.shields.io/badge/Hugo-0.147.0-blue?logo=hugo)](https://gohugo.io/)
[![Cloudflare Workers](https://img.shields.io/badge/Cloudflare-Workers-orange?logo=cloudflare)](https://workers.cloudflare.com/)

**Write a Markdown file. Push. It's live.**

A personal blog running on Hugo + GitHub Actions + Cloudflare Workers. Zero servers, zero monthly cost, ~30 second deploys.

→ **[hicke.se](https://hicke.se)**

---

## Why this setup

- **No CMS, no admin panel** — posts are just `.md` files in a Git repo
- **Free** — Hugo is open source, GitHub Actions free tier is generous, Cloudflare Workers has a free plan
- **Fast** — static HTML served from Cloudflare's global edge network
- **Automatic SSL** — Cloudflare handles certificates
- **Full version control** — every post, edit, and draft is in Git history

---

# add the file blog.png here:



## Writing a post

Create a file in `content/posts/`:

```markdown
---
title: "Post title"
date: 2026-05-01
draft: false
tags: ["tag"]
---

Your content here. Push to publish.
```

Images go in `static/images/` and are referenced as `![alt](/images/photo.jpg)`.

Push to `master` — GitHub Actions builds and deploys automatically.

---

## Local preview

```bash
# Requires Hugo extended 0.146.0+
hugo server
```

Open [localhost:1313](http://localhost:1313).

---

## How it works

```
Push to master
    → GitHub Actions checks out repo + PaperMod submodule
    → Hugo builds site into public/
    → Wrangler deploys public/ to Cloudflare Workers
    → Live on hicke.se
```

Two secrets required in the GitHub repo:

| Secret | Where to get it |
|---|---|
| `CF_API_TOKEN` | Cloudflare → Profile → API Tokens → "Edit Cloudflare Workers" template |
| `CF_ACCOUNT_ID` | Cloudflare → Workers & Pages overview page |

---

## Repo structure

```
.github/workflows/deploy.yml   CI/CD pipeline
content/
  archive.md                   archive page (lists all posts by date)
  posts/                       blog posts
themes/PaperMod/               theme (git submodule)
hugo.toml                      Hugo + menu config
wrangler.jsonc                 Cloudflare deployment config
```

---

## License

[MIT](LICENSE)
