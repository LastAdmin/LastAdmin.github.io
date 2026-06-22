# LastAdmin.github.io

Persönliches Portfolio. Jekyll-Seite, gehostet auf GitHub Pages — kein
Build-Schritt lokal nötig, GitHub kompiliert bei jedem Push.

Live unter: https://lastadmin.github.io

## Ein Projekt hinzufügen

1. Neue Datei in `_projects/` anlegen, z. B. `_projects/mein-ding.md`
2. Front Matter aus einem bestehenden Projekt kopieren und anpassen
3. Inhalt in Markdown schreiben
4. Bilder in `assets/images/projects/` ablegen und referenzieren als
   `/assets/images/projects/dein-bild.jpg`
5. Committen und pushen — die Seite wird in ca. 30 Sekunden neu gebaut

Front-Matter-Felder:

| Feld       | Pflicht | Beispiel                                          |
|------------|---------|---------------------------------------------------|
| `title`    | ja      | `Mein Ding`                                       |
| `category` | ja      | `Code` · `Web` · `Hardware` · `Motorrad` · `Elektronik` · `Texte` |
| `date`     | ja      | `2026-06-16` (steuert die Reihenfolge)            |
| `summary`  | ja      | Einzeiler, der auf der Karte erscheint            |
| `status`   | nein    | `Live`, `Laufend`, `Archiviert`                   |
| `tech`     | nein    | YAML-Liste: `[Python, Postgres]`                  |
| `cover`    | nein    | `/assets/images/projects/ding.jpg`                |
| `repo`     | nein    | `https://github.com/...`                          |
| `url_live` | nein    | `https://...`                                     |

Eine neue `category` einzuführen ist gratis — sie erscheint automatisch
als Filter-Chip auf der Projektseite. Für eine eigene Farbe eine Regel
`.project-category.deine-kat` in `assets/css/style.css` ergänzen.

## Foto von dir einfügen

In `index.html` ist im Hero-Bereich ein Avatar-Platzhalter (die zwei
Initialen „YM" in einem gestrichelten Kreis). Um dein Foto einzufügen:

1. Lege deine Bilddatei unter `assets/images/me.jpg` ab
2. In `index.html` das `<span class="avatar-placeholder">YM</span>`
   ersetzen durch:
   ```html
   <img src="{{ '/assets/images/me.jpg' | relative_url }}" alt="Foto von {{ site.author }}">
   ```
3. Die Zeile `<p class="avatar-caption">Foto folgt</p>` kannst du dann
   löschen

## Restliche Inhalte bearbeiten

- `index.html` — Text der Startseite
- `resume.html` — Lebenslauf-Inhalte
- `_config.yml` — Titel, Tagline, E-Mail, GitHub-Benutzername
- `assets/css/style.css` — komplettes Styling

## Lokale Vorschau (optional)

GitHub Pages baut beim Push, das hier brauchst du nur, wenn du Änderungen
vor dem Push ansehen willst.

```sh
gem install bundler jekyll
bundle install
bundle exec jekyll serve
```

Danach http://localhost:4000 öffnen.
