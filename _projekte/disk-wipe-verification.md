---
# ---------- Pflichtfelder ----------
title: Disk Wipe Verification Tool
order: 99
summary: Ein Windows-Tool, das nach dem Löschen einer Festplatte Sektor für Sektor prüft, ob wirklich keine Daten mehr wiederherstellbar sind, und das Ergebnis als PDF-Prüfbericht festhält. Entstanden aus eigener Initiative, um zu zeigen, dass ausgemusterte Firmen-Notebooks auch intern sicher gelöscht und weiterverkauft werden können.

# ---------- Optional ----------
category: Berufliches Projekt · AIP Plus · Cyber Security
tags: [PowerShell, C#, Datenlöschung, DoD 5220.22-M, Windows Forms, PDF-Reporting]

facts:
  - label: Zeitraum
    value: 2026
    note: Januar bis April, ca. 3,5 Monate
  - label: Rolle
    value: Initiant, Konzept & Entwicklung
    note: KI-gestützt umgesetzt
  - label: Technologie
    value: PowerShell & C#
  - label: Umfang
    value: 10 Hauptversionen
    note: rund 4000 Zeilen Code

links:
  - label: GitHub-Repository
    url: https://github.com/LastAdmin/CoreProtocol
---

Wenn eine Firma alte Notebooks verkauft, dürfen auf den Festplatten keine Firmendaten mehr zu finden sein. Das Löschen ist dabei nur die halbe Arbeit: Man muss auch belegen können, dass es geklappt hat. Für diesen Nachweis habe ich das **Disk Wipe Verification Tool** entwickelt. Es liest eine gelöschte Festplatte bis auf den letzten Sektor aus, erkennt übrig gebliebene Datenreste und erstellt daraus einen unterschriftsreifen PDF-Bericht. Auf Knopfdruck kommt ein Hardware-Bericht dazu, der alle wichtigen Angaben zum Gerät für den Verkauf zusammenfasst.

## Worum geht es?

Bei AIP Plus kam die Weisung, ausgemusterte Notebooks nicht mehr zu entsorgen, sondern zu verkaufen. Vorher mussten die Festplatten aber nachweislich gelöscht werden. Ein normales «Formatieren» reicht dafür nicht, denn dabei verschwindet nur das Inhaltsverzeichnis der Festplatte. Die eigentlichen Daten bleiben liegen und lassen sich mit frei verfügbaren Programmen oft problemlos wiederherstellen.

In einem Teammeeting wurde das weitere Vorgehen besprochen. Die IT-Leitung ging davon aus, dass sich eine sichere Löschung intern nicht umsetzen lässt, und wollte dafür eine spezialisierte Firma beauftragen. Das Problem: Die Dienstleistung hätte pro Gerät mehr gekostet, als wir beim Verkauf für das Notebook verlangen konnten. Ich habe im Meeting offen angesprochen, dass sich das so nicht rechnet und der Verkauf auf diese Weise keinen Sinn ergibt.

Statt es bei der Kritik zu belassen, habe ich noch während des Meetings ein kurzes Konzept für einen internen Prozess skizziert. Gegen Ende habe ich angeboten, das Konzept bis zum nächsten Teammeeting auszuarbeiten, zu testen und die Ergebnisse dort vorzustellen. Den Auftrag habe ich mir also selbst gegeben, mit einem klaren Ziel: zu zeigen, dass wir das intern sehr wohl können.

## Ziel

Der Prozess bestand aus zwei Teilen: dem Löschen selbst und der Kontrolle danach. Für das Löschen gab es mit **Eraser** bereits ein bewährtes Open-Source-Programm. Was fehlte, war ein Werkzeug für die Kontrolle. Es sollte:

- jede gelöschte Festplatte **vollständig und nur lesend** prüfen, ohne etwas zu verändern
- zuverlässig erkennen, ob noch **Dateien, Text oder andere Datenreste** vorhanden sind
- zwischen echten Datenresten und den Zufallsdaten einer Löschung **unterscheiden** können
- automatisch einen **Prüfbericht als PDF** erstellen, den Techniker und Vorgesetzte unterschreiben
- einen **Hardware-Bericht** mit den Gerätedaten für den Verkauf erzeugen
- mit einer **einfachen Oberfläche** auch von Kollegen ohne Programmierkenntnisse bedienbar sein
- auf ganz normalen Firmen-Notebooks laufen, ohne zusätzliche Software zu installieren

## Umsetzung

Das Tool ist in PowerShell geschrieben, weil es damit auf jedem Windows-Rechner ohne Installation läuft. Der Ablauf für den Techniker sieht so aus:

1. Festplatte mit Eraser nach DoD 5220.22-M löschen
2. Tool als Administrator starten und den eigenen Namen eintragen
3. Die zu prüfende Festplatte aus der Liste wählen und den Scan starten
4. Nach dem Scan öffnet sich der fertige Prüfbericht. Ausdrucken, unterschreiben, ablegen.

Jeder Durchlauf bekommt eine eindeutige Session-ID. Damit lassen sich Prüfbericht und Hardware-Bericht eines Geräts auch Monate später noch eindeutig zuordnen.

Konzept, Aufbau, Oberfläche, Berichte und das Zusammenspiel aller Teile stammen von mir. Für die technisch anspruchsvollsten Bausteine habe ich gezielt mit einer KI gearbeitet: Den direkten Festplattenzugriff, die Entropie-Berechnung und die kompilierte C#-Analyse-Engine hat die KI nach meinen Vorgaben geschrieben. Ich habe die Teile integriert, getestet und so lange nachgebessert, bis sie im Gesamtablauf zuverlässig liefen.

### Schritt 1: Löschen nach Militärstandard

Für das Löschen habe ich in Eraser das Verfahren **DoD 5220.22-M** gewählt, einen Standard des US-Verteidigungsministeriums. Dabei wird die gesamte Festplatte dreimal überschrieben:

| Durchgang | Was geschrieben wird |
|---|---|
| 1 | Nur Nullen |
| 2 | Nur Einsen, also das genaue Gegenteil des ersten Durchgangs |
| 3 | Zufallsdaten |

Ehrlich gesagt war die Wahl anfangs auch ein Statement: Ich wollte der IT-Leitung zeigen, dass wir intern sogar einen militärischen Standard umsetzen können. Das hat überzeugt. DoD 5220.22-M wurde danach zum festen Standard-Löschverfahren im Prozess.

Für die Prüfung hat das eine wichtige Folge: Nach dem letzten Durchgang ist die Festplatte nicht leer, sondern voller Zufallsdaten. Das Tool muss also erkennen können, ob ein Sektor «zufälligen Müll» aus der Löschung enthält oder echte Daten. Genau dafür ist die Entropie in Schritt 3 da.

### Schritt 2: Die Festplatte direkt auslesen

Damit nichts übersehen wird, liest das Tool die Festplatte nicht über das Dateisystem (also nicht über Ordner und Dateien), sondern direkt auf der untersten Ebene: Sektor für Sektor, so wie die Daten physisch abgelegt sind. Ein Sektor ist der kleinste Block einer Festplatte, meist 512 Byte gross. Für diesen Zugriff braucht es Administratorrechte und Funktionen direkt aus Windows. Die Festplatte wird dabei ausschliesslich lesend geöffnet, sodass das Tool keine Daten verändern kann.

### Schritt 3: Zufall messen mit der Entropie

Die zentrale Kennzahl der Prüfung ist die **Shannon-Entropie**. Sie misst, wie unvorhersehbar die Bytes in einem Sektor sind. Jedes Byte kann einen von 256 Werten annehmen. Das Tool zählt, wie oft jeder Wert vorkommt, und berechnet daraus:

**H = − Σ pᵢ · log₂(pᵢ)**

Dabei ist *pᵢ* der Anteil, den der Byte-Wert *i* im Sektor ausmacht. Das Ergebnis liegt zwischen 0 und 8:

| Entropie | Was sie bedeutet |
|---|---|
| 0 | Alle Bytes sind gleich, z. B. nur Nullen. Klassisches Löschmuster. |
| unter 5 | Deutliche Struktur, typisch für Text, Programme oder Tabellen. Sehr wahrscheinlich echte Daten. |
| 5 bis 7 | Teilweise zufällig, z. B. komprimierte oder verschlüsselte Dateien. Verdächtig. |
| über 7,5 | Nahezu perfekter Zufall, so wie ihn der letzte DoD-Durchgang hinterlässt. |

Allein reicht die Entropie aber nicht: Auch eine ZIP-Datei oder ein JPEG-Bild wirken ziemlich zufällig. Darum prüft das Tool zusätzlich, ob alle 256 Byte-Werte wirklich gleichmässig verteilt sind. Erst wenn beides stimmt, also sehr hohe Entropie und gleichmässige Verteilung, gilt ein Sektor als sauber überschrieben.

### Schritt 4: Jeden Sektor bewerten

Neben der Entropie fliessen weitere Merkmale in das Urteil ein:

| Prüfung | Was sie herausfindet |
|---|---|
| Nur Nullen oder nur Einsen? | Klassisches Löschmuster, der Sektor ist sauber |
| Entropie und Verteilung | Unterscheidet Zufallsdaten der Löschung von echten Daten (siehe Schritt 3) |
| Anteil lesbarer Zeichen | Viel lesbarer Text deutet auf übrig gebliebene Dokumente hin |
| Datei-Signaturen | Erkennt den typischen Anfang von PDF-, Word-, ZIP-, JPEG-, PNG- und weiteren Dateien |

Aus der Kombination ergibt sich pro Sektor eines von drei Urteilen: **gelöscht**, **verdächtig** oder **nicht gelöscht**. Eine Herausforderung war, Zufallsdaten nicht fälschlich als Datei zu erkennen. In grossen Mengen Zufallsdaten taucht rein statistisch auch einmal eine Byte-Folge auf, die wie ein Dateianfang aussieht. Die Signaturprüfung läuft darum nur bei Sektoren, die nicht ohnehin schon zufällig wirken.

Findet das Tool Datenreste, merkt es sich die genaue Position auf der Festplatte und was dort gefunden wurde. So kann man gezielt nachschauen, statt im Dunkeln zu suchen.

### Schritt 5: Schnell genug für ganze Festplatten

Die erste Version war komplett in PowerShell geschrieben. Bei den Tests zeigte sich schnell, dass die Analyse der Engpass war: Eine 500-GB-Festplatte besteht aus rund einer Milliarde Sektoren, und die Prüfung hätte Tage gedauert. Auf meine Problembeschreibung hin schlug die KI vor, die Analyse in eine kompilierte C#-Komponente auszulagern, die direkt im Tool mitläuft, und setzte sie auch um. Zusätzlich wird die Festplatte in grossen Blöcken von 1 MB gelesen statt Sektor für Sektor. Zusammen brachte das je nach Gerät einen Geschwindigkeitsgewinn um den Faktor 100 bis 500.

Ebenso wichtig war der Speicherverbrauch. Die frühen Versionen sammelten die gelesenen Daten im Arbeitsspeicher und kamen damit bei grossen Festplatten an ihre Grenzen. Meine Idee war, die Daten gar nicht aufzubewahren, sondern nur eine **Strichliste** zu führen: wie oft jeder der 256 möglichen Byte-Werte vorkam. Mehr braucht die Entropie-Formel nämlich nicht. Die KI hat diese Idee in die Entropie-Berechnung eingebaut. Seither braucht ein Scan konstant etwa 2 MB Arbeitsspeicher, egal ob die Festplatte 128 GB oder 2 TB gross ist.

```powershell
# Vereinfacht: Die Festplatte wird in 1-MB-Blöcken gelesen,
# die kompilierte C#-Engine bewertet alle Sektoren eines Blocks auf einmal
# und führt dabei nur die Strichliste der 256 Byte-Werte nach.
while (($bytesRead = $disk.Read($chunkBuffer, 0, $chunkSize)) -gt 0) {
    $stats = [DiskAnalysisEngine]::AnalyzeChunk($chunkBuffer, $bytesRead, $sectorSize, $patternCounts, $sectorIndex)
    $sectorIndex += $bytesRead / $sectorSize
}
```

### Schritt 6: Eine Oberfläche für das ganze Team

Damit das Tool nicht nur von mir bedient werden kann, hat es eine grafische Oberfläche (Windows Forms) bekommen: Festplattenliste, Einstellungen, Fortschrittsbalken, ein Protokollfenster und eine farbige Statusanzeige mit dem Ergebnis. Ein laufender Scan lässt sich jederzeit abbrechen. Weil PowerShell-Oberflächen während langer Berechnungen normalerweise «einfrieren», wird die Anzeige während des Scans regelmässig aktualisiert, damit das Fenster bedienbar bleibt.

<!-- PLATZHALTER: Screenshot der Benutzeroberfläche unter assets/img/projekte/ ablegen und Dateinamen hier anpassen. -->
![Benutzeroberfläche des Disk Wipe Verification Tools mit Festplattenliste, Einstellungen und Statusanzeige](/assets/img/projekte/disk-wipe-verification-gui.png)

### Schritt 7: Berichte als PDF

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

Am nächsten Teammeeting konnte ich einen funktionierenden Prozess vorstellen statt nur ein Konzept. Aus einer teuren externen Dienstleistung wurde ein interner Ablauf, den das IT-Team selbst durchführen kann. Ausgemusterte Notebooks konnten so verkauft werden, statt Kosten zu verursachen, und für jedes Gerät liegt ein unterschriebener Nachweis über die Löschung vor.

- **Löschen nach DoD 5220.22-M**, inzwischen das Standardverfahren im Prozess
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
- **Eraser**: das eigentliche Löschen der Festplatte nach DoD 5220.22-M
- **CrystalDiskInfo**: Gesundheitswerte der Festplatte (S.M.A.R.T.) für den Hardware-Bericht
- **ShowKeyPlus**: Auslesen der Windows-Lizenz
- **KI-Assistenz**: Umsetzung von Festplattenzugriff, Entropie-Berechnung und C#-Engine nach meinen Vorgaben

## Was ich daraus mitnehme

**Einwände wirken stärker mit einem Gegenvorschlag.** Zu sagen, dass sich ein Vorhaben nicht rechnet, ist einfach. Überzeugt hat die IT-Leitung erst ein ausgearbeitetes und getestetes Konzept. Mit einem konkreten Termin, dem nächsten Teammeeting, habe ich mich selbst in die Pflicht genommen.

**Löschen ist nur so gut wie der Nachweis dazu.** Eine gelöschte Festplatte sieht von aussen genauso aus wie eine, bei der das Löschen schiefgegangen ist. Erst die unabhängige Prüfung und der unterschriebene Bericht machen aus «sollte gelöscht sein» ein belastbares Ergebnis.

**Den Engpass zu finden ist die halbe Lösung, und KI ist dafür ein starker Partner.** Durch meine Tests wusste ich genau, dass die Analyse das Tool ausbremst und der Speicher bei grossen Festplatten nicht reicht. Mit dieser klaren Problembeschreibung hat die KI eine passende Lösung vorgeschlagen und umgesetzt, die Strichlisten-Idee kam von mir. Meine Aufgabe war, die Probleme zu erkennen, die Vorschläge zu beurteilen und das Ergebnis zu prüfen.

**Eine Stichprobe reicht nicht, wenn es um Sicherheit geht.** Anfangs prüfte das Tool nur ausgewählte Stellen der Festplatte, was schnell und statistisch aussagekräftig war. Für einen Löschnachweis zählt aber jeder einzelne Sektor. Darum ist die vollständige Prüfung heute Standard und die Stichprobe nur noch im Testmodus verfügbar.

**Sauber versionieren zahlt sich aus.** Mehrmals hat sich ein Umbau am Prüfverfahren als Sackgasse erwiesen. Weil jede Version dokumentiert war, konnte ich jeweils schnell auf den letzten funktionierenden Stand zurück und von dort neu ansetzen.
