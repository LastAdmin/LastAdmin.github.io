---
title: AD-Benutzerkonten automatisiert aus SharePoint erstellen
order: 1
category: IPA · AIP Plus
summary: Individuelle praktische Arbeit (Note 4.8) – automatisiertes Erstellen von Active-Directory-Benutzerkonten mit PowerShell auf Basis einer bestehenden SharePoint-Liste.
tags: [PowerShell, Active Directory, SharePoint, Automatisierung]

# Kennzahl-Kacheln oben auf der Projektseite (die erste ist farbig)
facts:
  - label: Bewertung
    value: Note 4.8
    note: IPA, Lehrabschluss
  - label: Betrieb
    value: AIP Plus
    note: Pratteln
  - label: Technologie
    value: PowerShell
    note: AD + SharePoint
  - label: Danach
    value: Weiterentwickelt
    note: über die IPA hinaus

# Optionale Links (z.B. GitHub-Repo ohne vertrauliche Daten)
# links:
#   - label: GitHub
#     url: https://github.com/LastAdmin/...
---

## Ausgangslage

Neue Benutzerkonten im Active Directory wurden bisher manuell angelegt, obwohl die benötigten Angaben bereits in einer SharePoint-Liste erfasst waren.

<!-- Ergänzen: Wie viele Konten pro Monat? Welche Probleme gab es beim manuellen Vorgehen (Zeitaufwand, Tippfehler, vergessene Gruppen)? -->

## Ziel

Benutzerkonten sollen automatisiert und einheitlich aus den Daten der SharePoint-Liste erstellt werden.

<!-- Ergänzen: Konkrete Anforderungen, z.B. Namenskonvention, OU, Gruppenmitgliedschaften, Postfach, Benachrichtigung. -->

## Umsetzung

- PowerShell-Skript, das die Einträge der SharePoint-Liste ausliest
- Erstellen der Active-Directory-Benutzerkonten anhand dieser Daten

<!-- Ergänzen: Ablauf (z.B. Graph API / PnP, Validierung, Logging, Fehlerbehandlung), Testkonzept, Projektmethode. Ein Ablaufdiagramm oder Screenshot (ohne Firmendaten) wirkt stark: ![Beschreibung](/assets/img/projekte/ipa-ablauf.png) -->

## Ergebnis

Die IPA wurde mit der **Note 4.8** bewertet. Nach dem Abschluss habe ich die Automatisierung weiterentwickelt.

<!-- Ergänzen: Zeitersparnis, was nach der IPA dazugekommen ist, was ich dabei gelernt habe. -->
