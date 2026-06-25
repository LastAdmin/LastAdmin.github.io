# Ducati 848 — IAW 5AM HW610 ECU: Entdeckungshistorie

**BIN-Datei:** `Summer_Map_Backup.bin` (327'680 Bytes)  
**MAP-Variante:** `22ADADPSMA1`  
**Prozessor:** STMicroelectronics ST10F269 (C166/ST10-Architektur, 16-Bit)  
**Reverse-Engineering-Tool:** Ghidra 12.1 + keyhana/c166-ghidra-module v1.1.0

---

## Chronologie

### Phase 1 — Grundlagen & bekannte Defekte (vor Ghidra)

Ausgangspunkt war ein persistentes Idle-Backfire- und Rauhlaufproblem. Durch direkten
Hex-Analyse des BIN und Abgleich mit bekannten XDF-Dateien wurden folgende Defekte
im Calibration-Bereich identifiziert:

| Adresse | Tabelle | Befund |
|---|---|---|
| `0x04C300` | Ignition Map | 0° BTDC advance über gesamte idle/low-load Zone — primäre Ursache des Backfires |
| `0x04A940` | Idle RPM Target | 16 Einträge (1600 RPM kalt → 1450 RPM warm) — strukturell korrekt |
| `0x04A960` | Idle Temp-Achse | Non-monoton: Werte steigen dann fallen (...120, 140, 120°C) — verhindert korrekte ECU-Interpolation |
| `0x04A93A` | Cold-Start Timer | ~12 Sekunden statt erwarteter 90–180 Sekunden |
| `0x04A990` | IAC Step Counts | Erster Eintrag: 10 Steps statt erwartetem ~500 — anomal |

Physikalische Massnahme: Throttle-Body Bypass-Screw-Differenz (~60% Imbalance zwischen
Zylindern) wurde angepasst — verbesserte Idle-Glätte, löste Backfire nicht.

---

### Phase 2 — Ghidra Setup & Import-Problem

**Problem:** Import des BIN mit Plugin v1.2.0 (inoffizieller Build) schlug fehl:
```
Unknown pentry register: CP
```

**Ursache:** `CP` (Context Pointer, physisches ST10-Register) war in der `c166.cspec`
als Calling-Convention-Parameter-Register deklariert, aber nicht korrekt im SLEIGH
Register-Space definiert. Ghidra 12.x verschärfte die cspec-Validierung gegenüber 11.x.

**Lösung:** Downgrade auf offizielles Release v1.1.0 (November 2025).

---

### Phase 3 — DPP-Adressschema aufgelöst

Der ST10F269 verwendet ein segmentiertes 16-Bit → 24-Bit Speichermodell via
Data Page Pointer (DPP). Die Init-Sequenz im Reset-Vektor-nahen Code setzt:

```
DPP0 = 0x01   →   physisch 0x004000–0x007FFF
DPP1 = 0x12   →   physisch 0x048000–0x04BFFF
DPP2 = 0x13   →   physisch 0x04C000–0x04FFFF
```

**Formel:** `physisch = (DPP × 0x4000) + (16-Bit-Offset & 0x3FFF)`

**Konsequenz für Ghidra:** Tabellenreferenzen erscheinen nicht automatisch, weil
Ghidra nur den 16-Bit-Offset sieht, nicht die aufgelöste physische Adresse.
Manuelles Memory-Search nach Little-Endian-Offset-Bytes nötig.

**Umrechnungstabelle für bekannte Adressen:**

| Physische Adresse | DPP | 16-Bit Offset im Code | Suche (LE-Bytes) |
|---|---|---|---|
| `0x04A93A` | DPP1 | `0x693A` | `3A 69` |
| `0x04A940` | DPP1 | `0x6940` | `40 69` |
| `0x04A960` | DPP1 | `0x6960` | `60 69` |
| `0x04A990` | DPP1 | `0x6990` | `90 69` |
| `0x04C300` | DPP2 | `0x8300` | `00 83` |

---

### Phase 4 — Erste Code-Referenzen via Memory-Search

Search nach `00 83` (Offset `0x8300` = Ignition Map) ergab zwei Treffer:

```
021ad8   cmp r12, #DAT_008300
021c3c   cmp r12, #DAT_008300
```

Beide in `FUN_021a0e`. Analyse ergab: `0x8300` wird dort **nicht** als
Tabellen-Lookup verwendet, sondern als **Minimum-IAC-Schrittgrenze** (Clamp).

---

### Phase 5 — Idle Control Call-Graph aufgedeckt

Durch XREF-Analyse von `FUN_021a0e` wurde der übergeordnete Aufrufbaum sichtbar:

```
FUN_0216ce          ← übergeordneter Scheduler/Task-Manager
  └── FUN_021866    ← Idle Control Dispatcher
        ├── FUN_0218ba   ← IAC Initialisierung / Referenz-Tabellen-Lookup
        ├── FUN_021978   ← Idle Fuel Trim Controller  ← NEU
        └── FUN_021a0e   ← IAC Stepper Motor Controller
```

**FUN_021866 (Idle Control Dispatcher):**
- Prüft Idle-Flags (`DAT_00d8c1`, `DAT_00d97f`)
- Vergleicht Zustandsbytes (`DAT_00d8a5` vs. `DAT_00d8a6`)
- Ruft bei Zustandsänderung `FUN_0218ba` (Initialisierung) auf
- Ruft bei normalem Idle `FUN_021978` + `FUN_021a0e` auf
- Enthält eigenen Tabellen-Lookup: `WORD_ARRAY_006c89` via `FUN_01a7ee`

**FUN_021a0e (IAC Stepper Motor Controller):**
- Verwaltet drei unabhängige IAC-Kanäle parallel
- RAM-Zustandsvariablen: `DAT_00d8e7`, `DAT_00d8eb`, `DAT_00d8e9`
- Ruft `FUN_031a1e` (Schrittmotor-Treiber Hardware-Abstraction) auf
- Prüft Betriebsmodus via `DAT_00dc8d` (Wert 5 oder 7)
- Kalibrierparameter aus Flash bei `0x73xx` (DPP1, physisch `0x04Bxxx`)

---

### Phase 6 — Idle Fuel Trim Tabellen entdeckt (NEU)

**FUN_021978 (Idle Fuel Trim Controller)** analysiert:

Funktion führt zwei separate Tabellen-Lookups durch, beide über `FUN_01a6da`
(generische Interpolationsfunktion):

**Lookup 1 — immer aktiv:**
```
Tabelle:  WORD_ARRAY_006dca   (20 Einträge)
Achse 1:  DAT_00dc6d          (Sensor-Wert 1)
Achse 2:  DAT_00dc75          (Sensor-Wert 2)
```

**Lookup 2 — nur wenn Modus = 7** (`DAT_00d893 == 7`):
```
Tabelle:  WORD_ARRAY_006f0a   (20 Einträge)
Achse 1:  DAT_00dc6d          (gleiche Achsen wie Lookup 1)
Achse 2:  DAT_00dc75
```

Ergebnis beider Lookups wird summiert, gegen `DAT_00d8f1` (laufender Trim-Akkumulator)
addiert und in `DAT_00d88f` (aktueller Idle Fuel Trim Wert) geschrieben.

Grenzen: `DAT_00d8f3` (Minimum), `DAT_00d8f5` (Maximum).

**Status der Tabellen:**
- `WORD_ARRAY_006dca`: alle Einträge = `0x0000` → RAM-Tabelle, zur Laufzeit befüllt
- `WORD_ARRAY_006f0a`: alle Einträge = `0x0000` → RAM-Tabelle, zur Laufzeit befüllt
- `WORD_ARRAY_006c89`: alle Einträge = `0x0000` → RAM-Tabelle, zur Laufzeit befüllt

**Hypothese:** Diese Tabellen sind adaptiv — die ECU lernt Korrekturen und schreibt
sie ins RAM. Initialisierungswerte kommen aus einer noch zu findenden Flash-Tabelle
(wahrscheinlich in `FUN_0218ba` oder dessen Callees).

---

## Offene Fragen / Nächste Schritte

- [ ] `FUN_0216ce` analysieren — übergeordneter Task-Scheduler, gibt Kontext für alle Idle-Funktionen
- [ ] `FUN_01a7ee` und `FUN_01a6da` vergleichen — sind das zwei verschiedene Interpolationsalgorithmen?
- [ ] Initialisierungsquelle der RAM-Tabellen finden (Flash-Default-Werte)
- [ ] Sensoren `DAT_00dc6d` und `DAT_00dc75` identifizieren (RPM? TPS? MAP?)
- [ ] AFR Target Idle Tabelle lokalisieren
- [ ] Decel Fuel Cut Parameter lokalisieren
- [ ] Kalibrierparameter bei `0x73xx` vollständig kartieren

---

### Phase 7 — State Machine & AFR Targets entdeckt (FUN_0216ce)

**FUN_0216ce (Idle Mode State Machine Controller)** analysiert:

Übergeordnete Funktion die den Idle-Modus steuert und `FUN_021866` aufruft.

**Neu entdeckt: State-Übergangs-Matrix bei `0x006be6`**

```
021708   movb RL1, [r12 + #DAT_006be5+1]
```
2D-Array-Zugriff mit berechneten Indizes aus `DAT_00dc6d` und `DAT_00dc75`.
Matrixgrösse: 5×N (Multiplikation mit 5 im Index-Berechnung sichtbar).
Enthält Byte-Werte die den Idle-Betriebsmodus bestimmen.

**Neu entdeckt: AFR/Fuel Target Werte**

`DAT_00d8ed` ist der Idle Fuel Target Ausgabewert. Er wird aus drei Flash-Adressen
geladen je nach Betriebszustand:

| Adresse | Physisch | Bedingung |
|---|---|---|
| `0x00704f` | `0x01704f` | Normal-Idle (`DAT_00d8bf == 0`) |
| `0x007051` | `0x017051` | Zustand Bit 2 gesetzt |
| `0x007053` | `0x017053` | Zustand Bit 4 gesetzt + Nebenbedingungen |

**Sensoren `DAT_00dc6d` / `DAT_00dc75` teilweise identifiziert:**

Beide werden mit `(Wert + 0x80) >> 8` auf ein Byte normiert und als Matrix-Indizes
verwendet. Die Normierungsoperation und Verwendung als Lookup-Index deutet auf
**Temperatursensoren** hin (Kühlwasser + Ansaugluft wahrscheinlich).

**Zwei neue Funktionen zur Analyse:**
- `FUN_02156c` — Idle-Übergangs-Handler (aufgerufen bei Zustandswechsel)
- `FUN_021620` — Idle-Entry/Exit-Logik

---

### Phase 8 — Sensor-Schreiber und Task-Scheduler identifiziert

**AFR Target Adressen sind RAM, nicht Flash:**
`0x00704f`, `0x007051`, `0x007053` zeigen `0x0000` (DPP0-Sicht) / `0xFFFF` (physisch).
`0xFFFF` = "noch nicht initialisiert"-Marker. Die echten AFR-Werte kommen aus einer
noch unbekannten Flash-Tabelle, wahrscheinlich via `FUN_020e42` (hat Write auf `DAT_00d8ed`).

**Sensor-Schreiber lokalisiert:**

`DAT_00dc6d` wird beschrieben von:
- `FUN_02d92a` bei `0x02d932`
- `FUN_02d992` bei `0x02d99e`

`DAT_00dc75` wird beschrieben von:
- `FUN_02d92a` bei `0x02d93e`
- `FUN_02d9dc` bei `0x02d9e8`

`FUN_02d92a` schreibt beide Sensoren → das ist der primäre Sensor-Einlese-Code.
Funktionen im `0x02dxxx`-Bereich sind der Sensor-Verarbeitungs-Layer.

**`DAT_00dc75` zusätzlich verwendet von:**
- `FUN_01c35e`, `FUN_01ca66` → Sensor-Kalibrierung/Konvertierung (`0x01cxxx`-Bereich)
- `FUN_02c9a4`, `FUN_02f57c` → weitere Steuerungsfunktionen

17 XREFs für `DAT_00dc75` vs. 11 für `DAT_00dc6d` → `DAT_00dc75` ist der wichtigere
Sensor, wahrscheinlich **Kühlwassertemperatur**.

**`DAT_00d8ed` (Idle Fuel Target) wird gelesen von `FUN_02107e`** (4 Read-Zugriffe) —
das ist die Funktion die den Fuel Target tatsächlich anwendet/umsetzt.

**FUN_020f60 — Task-Scheduler (sehr einfach):**
```
if DAT_00d893 == 0: return  ← globales Idle-Enable, alles gesperrt wenn 0
calls FUN_0211f6             ← Sensor-Update VOR Idle-Control
calls FUN_0216ce             ← Idle State Machine
```
`FUN_0211f6` aktualisiert wahrscheinlich `DAT_00dc6d`/`DAT_00dc75` bevor sie
in den Tabellen-Lookups verwendet werden.

**Offene Fragen nach dieser Phase:**
- `FUN_02d92a` analysieren → Sensor-Identifikation definitiv klären
- `FUN_0211f6` analysieren → Sensor-Vorverarbeitung verstehen
- `FUN_020e42` analysieren → woher kommen die echten AFR-Target-Werte?
- `FUN_02107e` analysieren → wie wird `DAT_00d8ed` (Fuel Target) angewendet?

---

### Phase 9 — Idle-Initialisierung vollständig kartiert (FUN_020e42)

**FUN_02d92a ist kein Sensor-Leser, sondern Reset-Funktion:**
Setzt Sensor-Datenblock `0x00dc69`–`0x00dc81` (13 Wörter = 13 Sensor-Kanäle) auf null.
Setzt Abschluss-Flag `DAT_00dc85 = 0x01`. Echte ADC-Einlese-Funktionen sind
`FUN_02d992` und `FUN_02d9dc` (noch zu analysieren).

**FUN_020e42 ist die Idle-System-Initialisierung:**
Aufgerufen beim Motorstart von drei verschiedenen Stellen (`FUN_019660`, `FUN_0196b6`,
`FUN_0196e0`). Setzt alle Idle-RAM-Variablen auf Startwerte und kopiert Flash-Defaults ins RAM.

**Neu entdeckte Flash-Adressen:**

| Adresse | Verwendung |
|---|---|
| `0x006b74` | Tabellen-Lookup via `FUN_01a67e` mit Sensor `DAT_00dc7f` → `DAT_00d8b9` |
| `0x006b95` | Initialisierungswert für `DAT_00d891`, `DAT_00d8b1`, `DAT_00d8b3` |
| `0x00704d` | Direkt in `DAT_00d8bf` (Idle-Konfigurations-Flag) |

**IAC Stepper Startpositionen aus Flash:**

| Flash-Adresse | Ziel RAM | Bedeutung |
|---|---|---|
| `DAT_00e001` | `DAT_00d8e7` | IAC Kanal 1 Default-Startposition |
| `DAT_00e003` | `DAT_00d8e9` | IAC Kanal 2 Default-Startposition |
| `DAT_00e005` | `DAT_00d8eb` | IAC Kanal 3 Default-Startposition |

Diese Werte sind direkt kalibrierbar — in Ghidra als `word` definieren.

**Fuel Trim Grenzen werden auf 0 initialisiert** — echte Werte kommen später
aus noch unbekannter Quelle (Write-Zugriffe auf `DAT_00d8f3`/`DAT_00d8f5` suchen).

**Sensor-Datenblock identifiziert:**
RAM `0x00dc69`–`0x00dc85` = zusammenhängender Block mit 13 Sensor-Kanälen + Status-Flag.
`DAT_00dc6d` (Sensor 1) und `DAT_00dc75` (Sensor 2) sind Kanäle 3 und 5 in diesem Block.

---

### Phase 10 — Kern-Zustandsautomat & Idle-Enable-Tabelle entdeckt (FUN_0211f6)

**FUN_0211f6 ist der Kern-Zustandsautomat** für den Idle-Modus — nicht nur
Sensor-Vorverarbeitung. Implementiert eine Switch-Table mit 7 Zuständen (0–6)
via indirektem Sprung:

```
Switch-Tabelle: 0x007ec  (7 × word = 14 Bytes)
Modus-Variable: DAT_00d893
Modus-Setter:   FUN_02142e (Parameter = neuer Zustand)
```

**Neu entdeckte Tabelle `DAT_00707e` — Idle Enable Table:**
```
Tabelle:  0x00707e  (20 Einträge, interpoliert via FUN_01a7ee)
Achsen:   DAT_00dc6d, DAT_00dc75
Ergebnis: DAT_00d8c1  ← das ist das Idle-Enable-Flag!
```
Diese Tabelle bestimmt ob Idle überhaupt aktiv ist. Direkt relevant für
das Backfire-Problem — wenn die Tabellenwerte falsch sind, schaltet die
ECU im falschen Moment in/aus dem Idle-Modus.

**Neu entdeckte Flash-Einzelwerte:**

| Adresse | Ziel RAM | Verwendung |
|---|---|---|
| `DAT_006b97` | `DAT_00d8b9` | Idle-Schwellen-Initialisierungswert |
| `DAT_006bbf` | `DAT_00d8b5` | Timeout/Zähler-Wert (2× verwendet) |
| `DAT_006bc1` | `DAT_00d8b7` | Kalibrierparameter am Modus-Ende |

**Ghidra-Problem:** Bytes `0x021248`–`0x0212df` sind nicht disassembliert
(erscheinen als `??`). Das sind Zustands-Handler 2–4 der Switch-Table.
Lösung: In Ghidra zu `0x021248` → Rechtsklick → Disassemble.

---

### Phase 11 — Switch-Table, K-Line Handler & Zustandsübergänge präzisiert

**Switch-Table bei `0x007ec`:**
Liegt eingebettet in `FUN_0007e2`. Ghidra markiert sie als `switchdataD_0007ec`.
In Ghidra zu `0x0007ec` → `word[7]` definieren → gibt 7 Handler-Adressen für
die 7 Idle-Zustände.

**FUN_0007e2 ist der K-Line Diagnose-Handler:**
Enthält ISO 9141 / KWP2000 Fast-Init Sequenz:
- Hardware-Timer T5 für Timing
- Port P3 Bit 11 für K-Line Signal
- Sync-Bytes: `0x55` (KWP2000 Sync), `0x4A` (Komplementbyte), `0x56` (Antwort)
- TuneECU / IAW-Reader kommunizieren über diese Funktion

**FUN_0211f6 Zustandsübergänge vollständig:**

| Modus | Bedingung für Übergang |
|---|---|
| 1→2 | `DAT_00db4b==1` AND `DAT_00db0a==0` AND `DAT_00d8b9==0` |
| 2→7 | RPM-Schwelle `DAT_00d891` erreicht |
| 3→4 | Timer `DAT_00d8b5` abgelaufen |
| 4→5 | `DAT_00d8c1` aktiv AND Bit 3 in `DAT_00d8ad` |
| 5→6 | Fehler in `FUN_021536` |
| →3/4 | Timer-basierter Rückfall |

**Tabelle `0x00707e` Aktivierungsbedingung:**
Wird nur berechnet wenn `DAT_00d8c1 == 0xFFFF` AND Bit 3 in `DAT_00d8ad` gesetzt.
Das ist eine Hysterese — die Tabelle aktualisiert sich nur bei vollem Idle-Enable.

**DAT_00707e noch zu lesen:**
In Ghidra `word[20]` definieren um echte Tabellenwerte zu sehen.
