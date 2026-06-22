---
title: Dieses Portfolio
category: Web
date: 2026-06-16
status: Live
summary: Die Seite, die du gerade siehst — Jekyll auf GitHub Pages, markdown-getrieben.
tech:
  - Jekyll
  - HTML
  - CSS
  - GitHub Pages
repo: https://github.com/LastAdmin/LastAdmin.github.io
url_live: https://lastadmin.github.io
---

## Warum

Ich wollte einen Ort, an dem ich ein laufendes Protokoll meiner Projekte
führe, ohne ein CMS zu hüten. Jekyll auf GitHub Pages war die naheliegende
Antwort: eine Markdown-Datei nach `_projects/` legen, pushen, fertig.

## Stack

- **Jekyll** — statischer Site-Generator, in GitHub Pages eingebaut
- **Vanilla CSS** — keine Frameworks, ein Stylesheet
- **Kein JavaScript-Build** — das einzige JS auf der Seite sind die
  Filter-Chips auf der Projektseite

## Ein Projekt hinzufügen

Eine neue Datei unter `_projects/` anlegen, das Front Matter aus einem
bestehenden Projekt übernehmen, den Inhalt in Markdown schreiben, committen.
GitHub baut die Seite automatisch neu.

```yaml
---
title: Mein Ding
category: Code        # oder Web, Hardware, Motorrad, Elektronik, Texte
date: 2026-06-16
status: Live
summary: Einzeiler, der auf der Karte angezeigt wird.
tech: [Python, Postgres]
cover: /assets/images/projects/ding.jpg
repo: https://github.com/me/ding
url_live: https://example.com
---

Danach folgt der Inhalt des Projekts in Markdown.
```

Das ist das ganze CMS.
