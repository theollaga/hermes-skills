---
name: blog-publish
category: sokuri-korea
description: >-
  Automates the full publishing workflow for a Sokuri Korea blog post, from front‑matter validation and image handling to Hugo build, Git push, CDN purge, and SEO verification.
version: 1.0.0
---

# Overview
This skill orchestrates the end‑to‑end publishing pipeline for a single post. It is designed to be called by a Revi orchestrator or directly via `hermes skill run`. The skill assumes the post slug (e.g., `hanbok-traditional-korean-attire`) and the path to its markdown file are provided via the `context`.

# Steps
1. **Validate front‑matter**
   - Read the markdown file.
   - Ensure required fields exist: `title`, `date`, `categories`, `tags`, `thumbnail`.
   - If missing, abort and send a message via `send_message`.
2. **Image handling**
   - Load `scripts/${slug}_images.json` if present (list of Unsplash/Pixabay URLs).
   - For each URL: download, convert to WebP, upload to R2 bucket under `<category>/<slug>/<slug>-imageN.webp`.
   - Update the markdown `figure-credit` shortcodes with the new R2 URLs and proper `alt` text.
   - Set `thumbnail` front‑matter to the first uploaded WebP image URL.
3. **Patch markdown**
   - Use `patch` to replace old image links and insert `figure-credit` blocks at the correct headings.
4. **Build site**
   - Run `hugo --gc --minify` in background.
   - Poll the process until completion; capture stdout/stderr.
   - If build fails, send failure message and abort.
5. **Git commit & push**
   - `git add . && git commit -m "publish: ${slug}" && git push origin main`.
   - If push fails, report error.
6. **Verify deployment**
   - Use `curl -I https://korea.sk-sokuri.com/posts/${slug}/` and check for HTTP 200.
   - Verify `og:image` meta tag points to the uploaded thumbnail.
   - Run Lighthouse audit via `lighthouse` CLI; capture performance score.
   - Send a summary message containing URL, HTTP status, OG image URL, and Lighthouse score.
7. **Cache purge (optional)**
   - If `PURGE_CACHE=true` env var is set, call `hermes gateway send --method POST /purge` (or Cloudflare API) to purge the CDN for the post URL.

# Error handling
- Any step that returns a non‑zero exit code triggers an immediate `send_message` with the error details and aborts the skill.
- All temporary files are placed under `tmp/` and are removed at the end of the skill.

# Usage example
```bash
hermes skill run sokuri-korea/blog-publish --context "slug=hanbok-traditional-korean-attire"
```

# Dependencies
- `hermes` CLI (installed)
- `git`
- `hugo`
- `curl`
- `lighthouse` (npm package globally installed)
- `image‑handling` skill for download/convert/upload
- Access to R2 bucket `sokuri-korea-images`

# Pitfalls & Tips
- Verify that the post `categories` field matches one of the six allowed slugs.
- If the Hugo build exceeds 5 minutes, increase the `timeout` argument of the `terminal` call.
- To execute this skill after publishing, call the generated API endpoint (e.g., via `curl` or a custom client) or use the provided wrapper script `scripts/run_blog_publish.sh`.
- The wrapper script expects the slug as its argument and will exit with an error if required fields are missing.
- Keep the script executable (`chmod +x`) and ensure the Hermes virtual environment path matches your installation.
```