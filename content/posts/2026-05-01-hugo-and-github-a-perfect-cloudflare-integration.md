---
title: "How I Set Up My Blog with Hugo, GitHub Actions, and Cloudflare Workers"
description: "A step-by-step guide to building a fast, free static blog using Hugo and PaperMod, deployed automatically to Cloudflare via GitHub Actions — with a custom domain and SSL included."
date: 2026-05-01T14:29:39.590Z
draft: false
tags: ["hugo", "cloudflare", "github-actions", "static-site", "devops"]
categories: ["guides"]
---

I wanted a simple blog: write a Markdown file, push to GitHub, and have it live on my domain within a minute. No CMS, no server to maintain, no monthly bill. Here's exactly how I built it.

## The Stack

- **[Hugo](https://gohugo.io/)** — static site generator. Builds the entire site in under 100ms.
- **[PaperMod](https://github.com/adityatelange/hugo-PaperMod)** — clean, fast Hugo theme with dark mode, tags, and [RSS](https://hicke.se/index.xml) out of the box.
- **GitHub** — source of truth. Every push triggers a deploy.
- **GitHub Actions** — builds the site and deploys it on every push to `master`.
- **Cloudflare Workers + Assets** — hosts and serves the static files globally. Free tier is more than enough for a personal blog.
- **Cloudflare DNS** — manages the custom domain with automatic SSL.

Total cost: zero.

## Repository Structure

```
blog/
├── .github/
│   └── workflows/
│       └── deploy.yml       # CI/CD pipeline
├── content/
│   ├── archive.md           # archive page
│   └── posts/               # blog posts go here
├── themes/
│   └── PaperMod/            # git submodule
├── wrangler.jsonc            # Cloudflare deployment config
└── hugo.toml                # Hugo config
```

## Hugo Configuration

The `hugo.toml` at the root of the repo configures the site. Key settings:

```toml
baseURL = "https://hicke.se/"
title = "hicke.se"
theme = "PaperMod"
enableRobotsTXT = true

[[menu.main]]
  name = "Archive"
  url = "/archive/"
  weight = 10
[[menu.main]]
  name = "Tags"
  url = "/tags/"
  weight = 20

[params]
  defaultTheme = "auto"
  ShowReadingTime = true
  ShowPostNavLinks = true
```

PaperMod is added as a git submodule so it tracks upstream fixes:

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

## Cloudflare Workers Configuration

Cloudflare uses a `wrangler.jsonc` file to understand what to deploy. For a static site, it's minimal:

```json
{
  "name": "blog",
  "compatibility_date": "2026-05-01",
  "assets": {
    "directory": "public"
  }
}
```

The `assets.directory` tells Wrangler to upload the contents of `public/` — the folder Hugo outputs to — as static files.

## GitHub Actions: The CI/CD Pipeline

This is the workflow that ties everything together. On every push to `master`, it checks out the repo (including the PaperMod submodule), builds the site with Hugo, and deploys to Cloudflare.

```yaml
name: Deploy to Cloudflare

on:
  push:
    branches: [master]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.147.0"
          extended: true

      - name: Build
        run: hugo

      - name: Deploy
        run: npx wrangler deploy
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CF_ACCOUNT_ID }}
```

Two secrets need to be added to the GitHub repository under **Settings → Secrets and variables → Actions**:

- `CF_API_TOKEN` — a Cloudflare API token created from the "Edit Cloudflare Workers" template
- `CF_ACCOUNT_ID` — your Cloudflare account ID, found on the Workers & Pages overview page

## Custom Domain and SSL

Since `hicke.se` is already managed by Cloudflare DNS, connecting it takes seconds. In the Cloudflare dashboard under **Workers & Pages → blog → Custom domains**, I added `hicke.se` and `blog.hicke.se`. Cloudflare automatically creates the DNS records and provisions SSL certificates — no configuration needed.

## Writing a New Post

New posts live in `content/posts/`. Create a file, write Markdown, set `draft: false`, commit, push:

```markdown
---
title: "My post title"
date: 2026-05-01
draft: false
tags: ["tag"]
---

Content here. Images go in `static/images/` and are referenced as `![alt](/images/photo.jpg)`.
```

From push to live takes about 30 seconds.

## Enabling the Archive Page

PaperMod has a built-in archive layout that lists all posts grouped by year and month. It doesn't activate automatically — you need to create a content file to mount it at a URL.

Create `content/archive.md`:

```markdown
---
title: "Archive"
layout: "archives"
url: "/archive/"
summary: archives
---
```

The `layout: "archives"` key is what activates PaperMod's template. The menu entry in `hugo.toml` pointing to `/archive/` then resolves correctly.

## Things I Ran Into

A few issues worth knowing about before you attempt the same setup:

**Hugo version pinning matters.** PaperMod requires Hugo 0.146.0 or later. The `HUGO_VERSION` environment variable in your workflow (or CF Pages settings) must be set explicitly — do not rely on the default.

**Git submodules need explicit checkout.** The GitHub Actions `checkout` step does not fetch submodules by default. Without `submodules: true`, the `themes/PaperMod` directory is empty and the build fails.

**Use the "Edit Cloudflare Workers" API token template.** Other token types or custom scopes will fail with authentication errors when Wrangler tries to deploy.

**`npx wrangler deploy` auto-detects your config.** As long as `wrangler.jsonc` is present at the root, Wrangler reads it and deploys without any interactive prompts — which is exactly what you want in CI.
