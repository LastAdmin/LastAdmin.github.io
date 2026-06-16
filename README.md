# LastAdmin.github.io

Personal portfolio. Jekyll site, hosted on GitHub Pages — no build step on your
machine, GitHub compiles it on every push.

Live at: https://lastadmin.github.io

## Adding a project

1. Create a new file in `_projects/`, e.g. `_projects/my-thing.md`
2. Copy the front matter from any existing project and edit it
3. Write the body in markdown
4. Drop any images into `assets/images/projects/` and reference them as
   `/assets/images/projects/your-image.jpg`
5. Commit and push — the site rebuilds in ~30 seconds

Front matter fields:

| field      | required | example                                      |
|------------|----------|----------------------------------------------|
| `title`    | yes      | `My Thing`                                   |
| `category` | yes      | `Code` · `Web` · `Hardware` · `Motorcycle` · `Electronics` · `Writing` |
| `date`     | yes      | `2026-06-16` (controls ordering)             |
| `summary`  | yes      | one-line description shown on the card       |
| `status`   | no       | `Shipped`, `In progress`, `Archived`         |
| `tech`     | no       | YAML list: `[Python, Postgres]`              |
| `cover`    | no       | `/assets/images/projects/thing.jpg`          |
| `repo`     | no       | `https://github.com/...`                     |
| `url_live` | no       | `https://...`                                |

Adding a new `category` value is free — it becomes a filter chip on the
projects page automatically. To give it a custom colour, add a `.project-category.your-cat`
rule in `assets/css/style.css`.

## Editing the rest

- `index.html` — landing page copy
- `resume.html` — resume content
- `_config.yml` — site title, tagline, email, GitHub username
- `assets/css/style.css` — all styling

## Local preview (optional)

GitHub Pages compiles on push, so this is only if you want to preview
changes before pushing.

```sh
gem install bundler jekyll
bundle init && bundle add jekyll
bundle exec jekyll serve
```

Then open http://localhost:4000.
