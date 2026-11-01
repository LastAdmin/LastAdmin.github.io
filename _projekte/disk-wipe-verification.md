---
# ---------- Pflichtfelder ----------
title: Disk Wipe Verification Tool
order: 99
summary: Ein Windows-Tool, das nach dem Löschen einer Festplatte Sektor für Sektor prüft, ob wirklich keine Daten mehr wiederherstellbar sind, und das Ergebnis als PDF-Prüfbericht festhält. Entstanden beim Aufbau eines Prozesses, mit dem ausgemusterte Firmen-Notebooks sicher weiterverkauft werden konnten.

# ---------- Optional ----------
category: Berufliches Projekt · AIP Plus · Cyber Security
tags: [PowerShell, C#, Datenlöschung, Windows Forms, NIST 800-88, PDF-Reporting]

facts:
  - label: Zeitraum
    value: 2026
    note: Januar bis April, ca. 3,5 Monate
  - label: Rolle
    value: Konzept und alleinige Umsetzung
  - label: Technologie
    value: PowerShell & C#
  - label: Umfang
    value: 10 Hauptversionen
    note: rund 4000 Zeilen eigener Code

links:
  - label: GitHub-Repository
    url: https://github.com/LastAdmin/CoreProtocol
---

Wenn eine Firma alte Notebooks verkauft, dürfen auf den Festplatten keine Firmendaten mehr zu finden sein. Das Löschen ist dabei nur die halbe Arbeit: Man muss auch belegen können, dass es geklappt hat. Für diesen Nachweis habe ich das **Disk Wipe Verification Tool** entwickelt. Es liest eine gelöschte Festplatte bis auf den letzten Sektor aus, erkennt übrig gebliebene Datenreste und erstellt daraus einen unterschriftsreifen PDF-Bericht. Auf Knopfdruck kommt ein Hardware-Bericht dazu, der alle wichtigen Angaben zum Gerät für den Verkauf zusammenfasst.

## Worum geht es?

Bei AIP Plus kam die Weisung, ausgemusterte Notebooks nicht mehr zu entsorgen, sondern zu verkaufen. Vorher mussten die Festplatten aber nachweislich gelöscht werden. Ein normales «Formatieren» reicht dafür nicht, denn dabei verschwindet nur das Inhaltsverzeichnis der Festplatte. Die eigentlichen Daten bleiben liegen und lassen sich mit frei verfügbaren Programmen oft problemlos wiederherstellen.

Naheliegend wäre gewesen, eine spezialisierte Firma mit der zertifizierten Löschung zu beauftragen. Das Problem: Die Dienstleistung hätte pro Gerät mehr gekostet, als wir beim Verkauf für das Notebook verlangen konnten. Der Verkauf hätte sich so nie gelohnt.

Ich bekam deshalb den Auftrag, einen eigenen Prozess aufzubauen: Festplatten intern sicher löschen, das Ergebnis prüfen und alles so dokumentieren, dass es auch später noch nachvollziehbar ist.

## Ziel

Der Prozess bestand aus zwei Teilen. Für das Löschen selbst habe ich das bewährte Open-Source-Programm **Eraser** eingesetzt, das die Festplatte mehrfach mit Nullen, Einsen und Zufallsdaten überschreibt. Was fehlte, war ein Werkzeug für die Kontrolle danach. Es sollte:

- jede gelöschte Festplatte **vollständig und nur lesend** prüfen, ohne etwas zu verändern
- zuverlässig erkennen, ob noch **Dateien, Text oder andere Datenreste** vorhanden sind
- zwischen echten Datenresten und den Zufallsdaten einer Löschung **unterscheiden** können
- automatisch einen **Prüfbericht als PDF** erstellen, den Techniker und Vorgesetzte unterschreiben
- einen **Hardware-Bericht** mit den Gerätedaten für den Verkauf erzeugen
- mit einer **einfachen Oberfläche** auch von Kollegen ohne Programmierkenntnisse bedienbar sein
- auf ganz normalen Firmen-Notebooks laufen, ohne zusätzliche Software zu installieren

## Umsetzung

Das Tool ist in PowerShell geschrieben, weil es damit auf jedem Windows-Rechner ohne Installation läuft. Der Ablauf für den Techniker sieht so aus:

1. Festplatte mit Eraser löschen
2. Tool als Administrator starten und den eigenen Namen eintragen
3. Die zu prüfende Festplatte aus der Liste wählen und den Scan starten
4. Nach dem Scan öffnet sich der fertige Prüfbericht. Ausdrucken, unterschreiben, ablegen.

Jeder Durchlauf bekommt eine eindeutige Session-ID. Damit lassen sich Prüfbericht und Hardware-Bericht eines Geräts auch Monate später noch eindeutig zuordnen.

### Schritt 1: Die Festplatte direkt auslesen

Damit nichts übersehen wird, liest das Tool die Festplatte nicht über das Dateisystem (also nicht über Ordner und Dateien), sondern direkt auf der untersten Ebene: Sektor für Sektor, so wie die Daten physisch abgelegt sind. Dafür braucht es Administratorrechte und direkten Zugriff auf Windows-Funktionen. Die Festplatte wird dabei ausschliesslich lesend geöffnet, sodass das Tool keine Daten verändern kann.

### Schritt 2: Jeden Sektor bewerten

Das Herzstück ist die Frage: Sieht dieser Sektor aus wie gelöscht, oder steckt da noch etwas drin? Jeder Sektor wird dafür auf mehrere Merkmale untersucht:

| Prüfung | Was sie herausfindet |
|---|---|
| Nur Nullen oder nur Einsen? | Klassisches Löschmuster, der Sektor ist sauber |
| Entropie (Mass für Zufälligkeit) | Echte Zufallsdaten einer Löschung sind maximal «chaotisch», normale Daten haben Struktur |
| Verteilung der Byte-Werte | Bei einer Löschung kommen alle Werte etwa gleich oft vor, bei echten Daten nicht |
| Anteil lesbarer Zeichen | Viel lesbarer Text deutet auf übrig gebliebene Dokumente hin |
| Datei-Signaturen | Erkennt den typischen Anfang von PDF-, Word-, ZIP-, JPEG-, PNG- und weiteren Dateien |

Aus der Kombination ergibt sich pro Sektor eines von drei Urteilen: **gelöscht**, **verdächtig** oder **nicht gelöscht**. Eine Herausforderung war, Zufallsdaten nicht fälschlich als Datei zu erkennen. In grossen Mengen Zufallsdaten taucht rein statistisch auch einmal eine Byte-Folge auf, die wie ein Dateianfang aussieht. Die Signaturprüfung läuft darum nur bei Sektoren, die nicht ohnehin schon zufällig wirken.

Findet das Tool Datenreste, merkt es sich die genaue Position auf der Festplatte und was dort gefunden wurde. So kann man gezielt nachschauen, statt im Dunkeln zu suchen.

### Schritt 3: Schnell genug für ganze Festplatten

Die erste Version war komplett in PowerShell geschrieben und für ganze Festplatten schlicht zu langsam: Eine 500-GB-Festplatte besteht aus rund einer Milliarde Sektoren, und die Prüfung hätte Tage gedauert. Ich habe die Analyse deshalb in eine kleine, kompilierte C#-Komponente ausgelagert, die direkt im Tool mitläuft. Dazu kommt, dass die Festplatte in grossen Blöcken von 1 MB gelesen wird statt Sektor für Sektor. Zusammen brachte das je nach Gerät einen Geschwindigkeitsgewinn um den Faktor 100 bis 500.

Ebenso wichtig war der Speicherverbrauch. Statt die gelesenen Daten zu sammeln, führt das Tool nur noch Strichlisten: wie oft jeder der 256 möglichen Byte-Werte vorkam. Dadurch braucht ein Scan konstant etwa 2 MB Arbeitsspeicher, egal ob die Festplatte 128 GB oder 2 TB gross ist.

```powershell
# Vereinfacht: Die Festplatte wird in 1-MB-Blöcken gelesen,
# die kompilierte C#-Engine bewertet alle Sektoren eines Blocks auf einmal.
while (($bytesRead = $disk.Read($chunkBuffer, 0, $chunkSize)) -gt 0) {
    $stats = [DiskAnalysisEngine]::AnalyzeChunk($chunkBuffer, $bytesRead, $sectorSize, $patternCounts, $sectorIndex)
    $sectorIndex += $bytesRead / $sectorSize
}
```

### Schritt 4: Eine Oberfläche für das ganze Team

Damit das Tool nicht nur von mir bedient werden kann, hat es eine grafische Oberfläche (Windows Forms) bekommen: Festplattenliste, Einstellungen, Fortschrittsbalken, ein Protokollfenster und eine farbige Statusanzeige mit dem Ergebnis. Ein laufender Scan lässt sich jederzeit abbrechen. Weil PowerShell-Oberflächen während langer Berechnungen normalerweise «einfrieren», wird die Anzeige während des Scans regelmässig aktualisiert, damit das Fenster bedienbar bleibt.

### Schritt 5: Berichte als PDF

Am Ende eines Scans entstehen auf Wunsch zwei Berichte, jeweils als HTML und PDF. Für die PDF-Umwandlung nutzt das Tool einen bereits installierten Browser (Edge, Chrome oder Brave) und weicht bei Bedarf auf Alternativen aus.

| Bericht | Inhalt |
|---|---|
| **Prüfbericht** | Festplattendaten, Gesamturteil, Anteil gelöschter Sektoren, erkannte Löschmuster, Liste gefundener Datenreste mit Position, Prüfmethode und ein Unterschriftenfeld für Techniker und Vorgesetzte |
| **Hardware-Bericht** | Hersteller, Modell, Seriennummer, Prozessor, Arbeitsspeicher, BIOS-Angaben, Windows-Lizenz sowie Zustand der Festplatte (Gesundheit, Betriebsstunden, gelesene und geschriebene Datenmenge) |

Für die Festplatten-Gesundheit bindet das Tool das bekannte Programm CrystalDiskInfo ein, für die Windows-Lizenz ShowKeyPlus. Die Ergebnisse beider Programme werden automatisch ausgelesen und in den Bericht übernommen. So muss niemand die Daten mehr von Hand abtippen.

Das Gesamturteil im Prüfbericht ist bewusst streng gehalten:

| Ergebnis | Bedeutung |
|---|---|
| **VERIFIED CLEAN** | Mindestens 99,5 % der Sektoren zeigen ein Löschmuster, die Festplatte ist bereit für den Verkauf |
| **MOSTLY CLEAN** | 95 bis 99,5 %, eine manuelle Kontrolle ist nötig |
| **NOT VERIFIED** | Weniger als 95 %, es sind wiederherstellbare Daten vorhanden |

## Ergebnis

Mit dem Tool wurde aus einer teuren externen Dienstleistung ein interner Prozess, den das IT-Team selbst durchführen kann. Ausgemusterte Notebooks konnten so verkauft werden, statt Kosten zu verursachen, und für jedes Gerät liegt ein unterschriebener Nachweis über die Löschung vor.

- **Vollständige Prüfung** jeder Festplatte, Sektor für Sektor
- **100 bis 500 Mal schneller** als die erste reine PowerShell-Version
- **Rund 2 MB Arbeitsspeicher**, unabhängig von der Festplattengrösse
- **Zwei PDF-Berichte pro Gerät**: Löschnachweis und Hardware-Datenblatt für den Verkauf
- **Bedienbar ohne Fachwissen** dank grafischer Oberfläche und klarer Ampel-Anzeige

Die Prüfmethode orientiert sich an den gängigen Standards für Datenlöschung, DoD 5220.22-M und NIST 800-88. Das Tool ersetzt keine offizielle Zertifizierung, liefert aber einen nachvollziehbaren, dokumentierten Nachweis, der für unseren Zweck ausreichte.

## Technologien

- **PowerShell**: Grundgerüst, Ablaufsteuerung, Berichte und Anbindung der Hilfsprogramme
- **C# (über `Add-Type`)**: kompilierte Analyse-Engine für die schnelle Auswertung von Milliarden Sektoren
- **Windows API (`CreateFile`, `ReadFile`)**: direkter, rein lesender Zugriff auf die physische Festplatte
- **Windows Forms**: grafische Oberfläche
- **HTML/CSS und Edge/Chrome im Headless-Modus**: Gestaltung der Berichte und Umwandlung in PDF
- **CrystalDiskInfo**: Gesundheitswerte der Festplatte (S.M.A.R.T.) für den Hardware-Bericht
- **ShowKeyPlus**: Auslesen der Windows-Lizenz
- **Eraser**: das eigentliche Löschen der Festplatte, vor der Prüfung

## Was ich daraus mitnehme

**Löschen ist nur so gut wie der Nachweis dazu.** Eine gelöschte Festplatte sieht von aussen genauso aus wie eine, bei der das Löschen schiefgegangen ist. Erst die unabhängige Prüfung und der unterschriebene Bericht machen aus «sollte gelöscht sein» ein belastbares Ergebnis.

**Die richtige Technik für den Engpass wählen.** PowerShell war ideal für Oberfläche, Berichte und die Verbindung der einzelnen Programme, für die rechenintensive Analyse aber zu langsam. Statt alles neu zu schreiben, habe ich nur den Engpass in C# ausgelagert. So blieb das Tool einfach zu verteilen und wurde trotzdem schnell genug.

**Eine Stichprobe reicht nicht, wenn es um Sicherheit geht.** Anfangs prüfte das Tool nur ausgewählte Stellen der Festplatte, was schnell und statistisch aussagekräftig war. Für einen Löschnachweis zählt aber jeder einzelne Sektor. Darum ist die vollständige Prüfung heute Standard und die Stichprobe nur noch im Testmodus verfügbar.

**Sauber versionieren zahlt sich aus.** Mehrmals hat sich ein Umbau am Prüfverfahren als Sackgasse erwiesen. Weil jede Version dokumentiert war, konnte ich jeweils schnell auf den letzten funktionierenden Stand zurück und von dort neu ansetzen.
