---
title: "Hugo and GitHub: A Perfect Cloudflare Integration"
description: ""
date: 2026-05-01T14:29:39.590Z
preview: ""
draft: false
tags: []
categories: []
---

In this post, I will share my experience of integrating Hugo with GitHub and Cloudflare to create a seamless workflow for deploying my static site.
### Setting Up Hugo
First, I installed Hugo on my local machine and created a new site using the command:

```bash
hugo new site my-hugo-site
```
I then added a theme and created some content for my site. After customizing the site to my
liking, I built the site using:

```bash
hugo
```
This generated the static files in the `public` directory, which I was ready to deploy.
### Integrating with GitHub
Next, I initialized a Git repository in my Hugo site directory and pushed the code to GitHub
using the following commands:

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin
git push -u origin main
```
### Deploying with Cloudflare
To deploy my site, I set up a Cloudflare Worker that fetches the static files from
GitHub and serves them to visitors. I created a new Worker in the Cloudflare dashboard and added the following code:

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})
async function handleRequest(request) {
  const url = new URL(request.url)
  const response = await fetch(`https://raw.githubusercontent.com/username/repository/branch/path/to/public${url.pathname}`)
  return response
}
```
I replaced `username`, `repository`, `branch`, and `path/to/public` with the
appropriate values for my GitHub repository. This Worker fetches the static files directly from GitHub and serves them to visitors.
### Conclusion
Integrating Hugo with GitHub and Cloudflare has been a great experience. It allows me to
easily manage my content with Hugo, version control with GitHub, and deploy seamlessly with Cloudflare. This setup has significantly streamlined my workflow and made it easier to maintain my static site.
I highly recommend this integration for anyone looking to create and deploy a static site efficiently.
