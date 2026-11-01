---
# ==========================================================
#  VORLAGE FÜR EIN NEUES PROJEKT
#  1. Datei kopieren und umbenennen, z.B. "home-assistant.md".
#     Der Dateiname wird zur Adresse: /projekte/home-assistant/
#     (nur Kleinbuchstaben, Zahlen und Bindestriche verwenden)
#  2. Die Zeile "published: false" löschen, sonst bleibt das
#     Projekt unsichtbar.
#  3. Felder ausfüllen. Felder mit "# optional" dürfen weg.
# ==========================================================
published: false

# ---------- Pflichtfelder ----------
title: Projekttitel
order: 99                    # Nummer und Reihenfolge (1 = zuerst, Badge zeigt "01")
summary: Ein bis zwei Sätze, worum es geht. Erscheint auf der Startseite und in der Projektübersicht.

# ---------- Optional ----------
category: Privates Projekt · Infrastruktur   # optional – kleine Zeile über dem Titel
tags: [Technologie 1, Technologie 2, Technologie 3]   # optional – Chips; in der Übersicht die ersten 4

# Kennzahl-Kacheln oben auf der Projektseite (optional, ideal sind 4).
# Die erste Kachel wird hell hervorgehoben.
facts:
  - label: Zeitraum
    value: 2026
    note: ca. 3 Monate          # optional – kleine Zeile darunter
  - label: Rolle
    value: Alleinige Umsetzung
  - label: Technologie
    value: PowerShell
  - label: Status
    value: In Betrieb

# Links am Ende der Seite (optional)
links:
  - label: GitHub-Repository
    url: https://github.com/LastAdmin/
  # - label: Dokumentation
  #   url: https://...

# Tipp: In der Berufserfahrung (_lebenslauf/10-berufserfahrung.md) kann ein
# Eintrag mit "project: <dateiname-ohne-.md>" auf diese Seite verlinken.
---

<!--
  AUFBAU DES TEXTS
  - Jede "## Überschrift" wird zu einem eigenen Panel.
    Trennlinien (---) zwischen den Abschnitten sind darum nicht nötig.
  - Titel und Kurzbeschreibung kommen aus dem Kopf oben, eine "# Überschrift"
    braucht es nicht.
  - Abschnitte, die nur einen Kommentar wie diesen enthalten, werden
    ausgeblendet. Nicht benötigte Abschnitte also leer lassen oder löschen.
  - Text vor der ersten "##" erscheint als eigenes Panel ohne Titel (Einleitung).
-->

Ein kurzer Einleitungsabsatz für Leser, die nur kurz reinschauen: Was ist das Projekt, für wen ist es und was ist das Besondere daran? Drei bis vier Sätze reichen.

## Worum geht es?

<!-- Das Problem oder der Auslöser. Was hat gestört, was fehlte? Für Leser ohne Fachwissen verständlich schreiben. -->

Beschreibung der Ausgangslage.

## Ziel

<!-- Was sollte am Ende funktionieren? Gerne als Liste. -->

- Anforderung 1
- Anforderung 2
- Anforderung 3

## Umsetzung

<!-- Wie bist du vorgegangen? Mit ### lassen sich Unterabschnitte im selben Panel bilden. -->

Kurzer Überblick über den Ablauf.

### Schritt 1: Titel

Beschreibung. Begriffe wie `Get-ADUser` oder Dateinamen in `Backticks` setzen.

### Schritt 2: Titel

Eine Tabelle eignet sich für Vergleiche oder Übersichten:

| Komponente | Aufgabe |
|---|---|
| Komponente A | Was sie macht |
| Komponente B | Was sie macht |

Code-Beispiele erscheinen als Terminal-Fenster. Keine Passwörter, internen Servernamen oder Firmendaten einfügen.

```powershell
# Beispiel
Get-ADUser -Filter * -SearchBase "OU=Benutzer,DC=firma,DC=local"
```

Bilder in `assets/img/projekte/` ablegen und so einbinden:

![Kurze Bildbeschreibung](/assets/img/projekte/beispiel.jpg)

## Ergebnis

<!-- Was ist herausgekommen? Zahlen wirken stark: Zeitersparnis, Anzahl Systeme, Bewertung. -->

Beschreibung des Ergebnisses.

## Technologien

- **Technologie 1**: wofür sie eingesetzt wurde
- **Technologie 2**: wofür sie eingesetzt wurde

## Was ich daraus mitnehme

<!-- Zwei bis vier Erkenntnisse. Ein fett gesetzter Satz pro Erkenntnis plus Erklärung liest sich gut. -->

**Erkenntnis in einem Satz.** Kurze Erklärung, was dahintersteckt und was ich beim nächsten Mal anders oder gleich machen würde.
