# Publish Workflow Reference

This document records the class‑level publishing workflow for Sokuri Korea blog posts, abstracted from the recent session.

## Overview
1. **Front‑matter validation** – required fields: `title`, `date`, `categories`, `tags`, `thumbnail`.
2. **Image handling** – download Unsplash/Pixabay URLs, convert to WebP, upload to R2 bucket under `<category>/<slug>/<slug>-imageN.webp`. Update `figure-credit` shortcodes and set `thumbnail` to the first image URL.
3. **Markdown patching** – replace old image links, insert `figure-credit` blocks at appropriate headings.
4. **Hugo build** – run `hugo --gc --minify` (adjust `timeout` if build >5 min). Capture stdout/stderr; abort on failure.
5. **Git commit & push** – `git add . && git commit -m "publish: ${slug}" && git push origin main`.
6. **Deployment verification** – `curl -I https://korea.sk-sokuri.com/posts/${slug}/` → HTTP 200; check `og:image` meta tag.
7. **SEO audit** – optional Lighthouse run, report performance score.
8. **Cache purge** – if `PURGE_CACHE=true`, call Cloudflare API or `hermes gateway send` to purge the post URL.

## Common Pitfalls & Fixes
- **Hugo timeout** – increase `timeout` argument in the `terminal` call when the site is large.
- **Wrapper script usage** – the script `scripts/run_blog_publish.sh` expects the slug as its sole argument; it activates the Hermes venv and exits with a clear error if required fields are missing.
- **Git authentication** – HTTPS credentials are stored in `~/.git-credentials`; the token must have `repo` scope.

Use this reference file for any future `blog-publish` executions.
