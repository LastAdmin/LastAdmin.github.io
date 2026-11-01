# Portfolio – Yannick Morgenthaler

Persönliche Portfolio-Webseite mit Lebenslauf und Projekt-Detailseiten.
Läuft auf **GitHub Pages** mit Jekyll – alle Inhalte werden in Markdown-Dateien gepflegt.

## Wo steht was?

| Datei / Ordner | Inhalt |
| --- | --- |
| `index.md` | Kopfbereich (Name, Rolle, Status, Kontakt), Kennzahl-Kacheln und Profiltext |
| `_lebenslauf/*.md` | Je ein Abschnitt des Lebenslaufs (Berufserfahrung, Ausbildung, Skills …) |
| `_projekte/*.md` | Je ein Projekt mit eigener Detailseite unter `/projekte/<dateiname>/` |
| `assets/img/` | Bilder; das Profilfoto heisst `foto.jpg` |
| `_config.yml` | Titel, Beschreibung, URL der Seite |
| `_layouts/`, `_includes/`, `assets/css/main.css` | Design und Vorlagen (für Inhalte nicht nötig) |
| `_referenz/` | Ursprünglicher HTML-Lebenslauf (wird nicht veröffentlicht) |

## Lebenslauf-Abschnitte (`_lebenslauf/`)

Jede Datei hat oben einen Kopf zwischen `---`:

```yaml
---
title: Weiterbildungen   # Überschrift der Karte
order: 30                # Reihenfolge (klein = weiter oben)
width: half              # full | half | third  (Breite der Karte)
type: timeline           # optional: timeline | skills | languages | projects
---
```

- **Ohne `type`**: Der Markdown-Text unter dem Kopf wird angezeigt (Listen, Absätze, **fett** …).
  In Listen erscheint `*kursiver Text*` als graue Zusatzinfo, z.B. `*2020 · Basel*`.
- **`timeline`**: Einträge unter `items:` mit `role`, `date`, `org`, `columns` und `points`
  (siehe `10-berufserfahrung.md`). Mit `project: <dateiname>` erscheint ein Link zum Projekt.
- **`skills`**: Kacheln mit `name`, `detail`, `core: true` für Kernkompetenzen.
- **`languages`**: Sprachen mit `name`, `level`, `percent`, `note`.
- **`projects`**: Zeigt automatisch alle Projekte aus `_projekte/`.

Neuer Abschnitt = neue `.md`-Datei im Ordner anlegen. Abschnitt entfernen = Datei löschen.
Zwei `half`- oder drei `third`-Karten hintereinander stehen nebeneinander.

## Projekte (`_projekte/`)

**Neues Projekt:** `_projekte/vorlage.md` kopieren, umbenennen (der Dateiname wird zur Adresse)
und die Zeile `published: false` löschen. Die Vorlage selbst ist dank dieser Zeile unsichtbar.
Alle Felder und Formatierungen sind darin erklärt.

```yaml
---
title: Eigene VMware-ESXi-Testumgebung
order: 3                      # Nummer und Reihenfolge
category: Privates Projekt · Infrastruktur
summary: Ein bis zwei Sätze für die Übersicht.
tags: [VMware ESXi, Backup]
facts:                        # Kennzahl-Kacheln (optional)
  - label: Hypervisor
    value: VMware ESXi
links:                        # optional
  - label: GitHub
    url: https://github.com/...
---

## Ausgangslage
Text …

## Umsetzung
Text, Listen, Tabellen, Bilder: ![Beschreibung](/assets/img/projekte/bild.jpg)
```

Jede `## Überschrift` wird zu einem eigenen Panel. Abschnitte, die nur einen
`<!-- Kommentar -->` enthalten, werden ausgeblendet – so können Notizen stehen
bleiben, bis der Text fertig ist.

## Veröffentlichen

1. Repository `LastAdmin.github.io` auf GitHub anlegen und den Inhalt pushen.
2. Unter **Settings → Pages** als Quelle „Deploy from a branch“, Branch `main`, Ordner `/ (root)` wählen.
3. Nach 1–2 Minuten ist die Seite unter <https://lastadmin.github.io> erreichbar.

Jede Änderung, die auf `main` gepusht wird, ist kurz danach online.
Markdown-Dateien lassen sich auch direkt im Browser auf github.com bearbeiten.

## Lokal ansehen (optional)

Benötigt Ruby ≥ 3.0 (z.B. via `brew install ruby`):

```sh
bundle install
bundle exec jekyll serve
```

Danach <http://localhost:4000> öffnen.

## Als PDF

Der Knopf **PDF** oben rechts öffnet den Druckdialog. Dort „Als PDF sichern“ wählen
und „Hintergrundgrafiken“ aktivieren, damit die farbigen Flächen mitgedruckt werden.
