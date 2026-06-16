---
title: This Portfolio
category: Web
date: 2026-06-16
status: Live
summary: The site you're looking at — Jekyll on GitHub Pages, markdown-driven.
tech:
  - Jekyll
  - HTML
  - CSS
  - GitHub Pages
repo: https://github.com/LastAdmin/LastAdmin.github.io
url_live: https://lastadmin.github.io
---

## Why

I wanted one place to keep a running log of the things I work on, without a
CMS to babysit. Jekyll on GitHub Pages was the obvious answer: drop a
markdown file into `_projects/`, push, done.

## Stack

- **Jekyll** — static site generator, built into GitHub Pages
- **Vanilla CSS** — no frameworks, one stylesheet
- **No JavaScript build step** — the only JS on the site is the filter chips
  on the projects page

## Adding a project

Create a new file under `_projects/`, copy the front matter from any
existing project, write the body in markdown, commit. GitHub rebuilds the
site automatically.

```yaml
---
title: My Thing
category: Code        # or Web, Hardware, Motorcycle, Electronics, Writing
date: 2026-06-16
status: Live
summary: One-line description that shows on the card.
tech: [Python, Postgres]
cover: /assets/images/projects/thing.jpg
repo: https://github.com/me/thing
url_live: https://example.com
---

Then the body of the project in markdown.
```

That's the whole CMS.
