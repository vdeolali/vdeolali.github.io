---
permalink: null
---

# Project Context for Grok (and other agents)

This is a minimal Jekyll blog using the `minima` theme.

## Site Structure
- Posts: `_posts/YYYY-MM-DD-slug.md` (date **must** be in filename for Jekyll to pick it up)
- Front matter (keep it minimal, 3 lines typical):
  ```yaml
  ---
  title: "Short, descriptive title"
  date: YYYY-MM-DD
  ---
  ```
- Layouts: Use default for most; `layout: home` on `index.md`, `layout: page` on `archive.md`
- Content tone: Practical, technical, honest, first-person experience reports. Focus on cloud infrastructure, OCI, OpenShift, networking, tooling, and real-world lessons learned (including pain points). Conversational but precise. Use **bold** for emphasis. Include code blocks with proper language tags.
- No extra plugins beyond what's in `_config.yml` (jekyll-feed, jekyll-sitemap). Do not invent new `_config.yml` values or layouts.

## Agent Instructions (to avoid context waste and tool flailing)
- **Always** read an existing post first (`read_file`) to match tone, style, and exact front-matter before creating or editing.
- Use `read_file` and `grep` before broad exploration.
- For new posts: filename **must** start with today's date in `YYYY-MM-DD-` format + kebab-case slug. Create with `write` tool or precise edits.
- Prefer dedicated tools (`read_file`, `search_replace`, `list_dir`, `grep`) over shell commands when possible.
- Keep changes minimal and reversible. Confirm Jekyll builds cleanly.
- This file is the canonical project note — read it first in every session.

This addresses the "12% context window on orientation" problem from the Kimi K3 posts. Front-load this.

Last updated: 2026-07-17
