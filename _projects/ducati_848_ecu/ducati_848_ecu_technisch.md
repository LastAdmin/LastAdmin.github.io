---
title: Technische Dokumentation
parent: ducati_848_ecu_entdeckungen
order: 1
date: 2025-09-15
summary: Alles abgebaut, beschriftet, sortiert.
---

# Ducati 848 — IAW 5AM HW610: Technische Firmware-Dokumentation

**BIN:** `Summer_Map_Backup.bin` · **Variante:** `22ADADPSMA1` · **Prozessor:** ST10F269

---

## 1. Hardware & Architektur

| Eigenschaft | Wert |
|---|---|
| ECU | Magneti Marelli IAW 5AM HW610 |
| Prozessor | STMicroelectronics ST10F269 |
| Architektur | C166/ST10, 16-Bit, Little Endian |
| Flash-Grösse | 327'680 Bytes (0x50000) |
| EEPROM | externer 95160 (Immobilizer, Adaptionswerte) |
| Ghidra-Plugin | keyhana/c166-ghidra-module v1.1.0 |

---

## 2. Speicher-Layout

### 2.1 Flash-Memory-Map

```
0x000000 - 0x04FFFF   Flash ROM (327'680 Bytes = dein BIN)
0xFF0000 - 0xFFFFFF   SFR / Hardware-Register
0xFD0000 - 0xFEFFFF   RAM (zur Laufzeit, im BIN leer/null)
```

### 2.2 DPP-Segmentierung

Der ST10F269 übersetzt 16-Bit-Codeadressen via Data Page Pointer auf 24-Bit physische
Adressen. Die ECU initialisiert beim Reset:

```
DPP0 = 0x01   →   physisch 0x004000 - 0x007FFF   (Code-Zugriff 0x0000-0x3FFF)
DPP1 = 0x12   →   physisch 0x048000 - 0x04BFFF   (Code-Zugriff 0x4000-0x7FFF)
DPP2 = 0x13   →   physisch 0x04C000 - 0x04FFFF   (Code-Zugriff 0x8000-0xBFFF)
DPP3            →   Systembereich / SFR
```

**Adress-Auflösung:**
```
Physisch = (DPP_Wert × 0x4000) + (16-Bit-Offset & 0x3FFF)
```

**Wichtig für Ghidra:** Tabellenreferenzen erscheinen nicht automatisch.
Manuell nach Little-Endian-Offset suchen: Search → Memory → Hex-Sequenz.

---

## 3. Bekannte Kalibrier-Tabellen (Flash)

### 3.1 Verifizierte Tabellen mit Defekten

| Physische Adresse | Name | Grösse | Status | Defekt |
|---|---|---|---|---|
| `0x04C300` | Ignition Map | 16×16 word | ⚠️ defekt | 0° BTDC in idle/low-load Zone |
| `0x04A940` | Idle RPM Target | 16 × word | ✅ korrekt | — |
| `0x04A960` | Idle Temp-Achse | 16 × word | ⚠️ defekt | non-monoton: ...120, 140, 120°C |
| `0x04A93A` | Cold-Start Timer | word | ⚠️ defekt | ~12s statt 90–180s |
| `0x04A990` | IAC Step Counts | word[] | ⚠️ defekt | Eintrag [0] = 10 statt ~500 |

### 3.2 Neu entdeckte Tabellen (noch zu verifizieren)

| Adresse | Name | Grösse | Verwendet von | Status |
|---|---|---|---|---|
| `0x006c8a` | IAC Referenz-Tabelle | 20 × word | `FUN_021866`, `FUN_0218ba` | RAM → null im BIN |
| `0x006dca` | Idle Fuel Trim Basis | 20 × word | `FUN_021978` | RAM → null im BIN |
| `0x006f0a` | Idle Fuel Trim Modus-7 | 20 × word | `FUN_021978` | RAM → null im BIN |

### 3.3 Kalibrierparameter (Flash, `0x73xx`-Bereich, DPP1)

Alle bei physisch `0x04B3xx`. Funktion: Grenzen und Schrittweiten für IAC-Stepper.

| Offset (Code) | Physisch | Bedeutung |
|---|---|---|
| `0x730f` | `0x04B30f` | IAC Schrittweite / Increment |
| `0x7317` | `0x04B317` | IAC Start-Position |
| `0x731b` | `0x04B31b` | Obere Grenze Kanal 1 |
| `0x731d` | `0x04B31d` | IAC Referenz-Position |
| `0x7323` | `0x04B323` | Step-Increment für Regelung |
| `0x7d00` | `0x04B??? ` | Maximum IAC-Position |
| `0x8300` | `0x04C300` | Minimum IAC-Position (= Ignition Map Basis-Adresse — Doppelnutzung prüfen!) |

---

## 4. Funktions-Dokumentation

### 4.1 Idle Control Call-Graph

```
FUN_0216ce  [Scheduler / Task-Manager]
    │
    └── FUN_021866  [Idle Control Dispatcher]
            │
            ├── FUN_0218ba  [IAC Initialisierung]
            │       └── FUN_01a7ee  [Interpolation Typ A]
            │       └── FUN_01a4ec  [unbekannt — Skalierung?]
            │
            ├── FUN_021978  [Idle Fuel Trim Controller]
            │       └── FUN_01a6da  [Interpolation Typ B]
            │
            └── FUN_021a0e  [IAC Stepper Motor Controller]
                    └── FUN_031a1e  [Stepper HW-Abstraction]
```

### 4.2 FUN_021866 — Idle Control Dispatcher

**Adresse:** `0x021866`  
**Aufgerufen von:** `FUN_0216ce` bei `0x0217d0`

**Logik:**
1. Prüft `DAT_00d8c1` (Idle-Enable-Flag)
2. Prüft `DAT_00d97f` (sekundäres Enable-Flag, Byte)
3. Vergleicht `DAT_00d8a5` (aktueller Zustand) mit `DAT_00d8a6` (vorheriger Zustand)
4. Bei Zustandsänderung: ruft `FUN_0218ba` (Initialisierung) auf
5. Bei stabilem Zustand: ruft `FUN_021978` + `FUN_021a0e` auf

**Eigener Tabellen-Lookup:**
- Tabelle: `WORD_ARRAY_006c89` via `FUN_01a7ee`
- Achsen: `DAT_00dc6d`, `DAT_00dc75`
- Ergebnis → `DAT_00d8c7`

---

### 4.3 FUN_021a0e — IAC Stepper Motor Controller

**Adresse:** `0x021a0e`  
**Aufgerufen von:** `FUN_021866` bei `0x0218ac`

**Drei parallele IAC-Kanäle:**

| Kanal | RAM-Position | Beschreibung |
|---|---|---|
| 1 | `DAT_00d8e7` | IAC Kanal 1 aktuelle Schrittposition |
| 2 | `DAT_00d8eb` | IAC Kanal 2 aktuelle Schrittposition |
| 3 | `DAT_00d8e9` | IAC Kanal 3 aktuelle Schrittposition |

**Modus-Steuerung:** `DAT_00dc8d`
- Wert `0x5`: Normalbetrieb
- Wert `0x7`: erweiterter Modus (aktiviert zweiten Fuel-Trim-Lookup)

**Grenzwerte:**
- Minimum: `DAT_008300` (`0x04C300`) — Sicherheits-Clamp Untergrenze
- Maximum: `DAT_007d00`

---

### 4.4 FUN_021978 — Idle Fuel Trim Controller

**Adresse:** `0x021978`  
**Aufgerufen von:** `FUN_021866` bei `0x0218a8`

**Datenfluss:**
```
DAT_00d8f1  ←  Trim-Akkumulator (wird bei jedem Aufruf um /2 gedämpft: ashr #1)
DAT_00d88f  ←  aktueller Idle Fuel Trim Ausgabewert
DAT_00d8f3  ←  Minimum-Grenze Fuel Trim
DAT_00d8f5  ←  Maximum-Grenze Fuel Trim
```

**Lookup 1 (immer):**
```
Tabelle:  0x006dca  (20 Einträge)
Funktion: FUN_01a6da
Achse 1:  DAT_00dc6d
Achse 2:  DAT_00dc75
```

**Lookup 2 (nur Modus 7):**
```
Tabelle:  0x006f0a  (20 Einträge)
Funktion: FUN_01a6da
Achse 1:  DAT_00dc6d
Achse 2:  DAT_00dc75
```

**Ausgangsberechnung:**
```
r8 = Lookup1_Ergebnis + DAT_00d8f1 + Lookup2_Ergebnis_oder_0
DAT_00d88f = clamp(r8, DAT_00d8f3, DAT_00d8f5)
```

---

### 4.5 FUN_0218ba — IAC Initialisierung

**Adresse:** `0x0218ba`  
**Aufgerufen von:** `FUN_021866` bei `0x021882` (nur bei Zustandsänderung)

Führt zwei Tabellen-Lookups durch und initialisiert Arbeits-RAM:
- `WORD_ARRAY_006c89` via `FUN_01a7ee` → `DAT_00d8c7`
- `DAT_0071c2` via `FUN_01a7ee` → `DAT_00d8c9`
- `DAT_007303` via `FUN_01a4ec` → `DAT_00d8cb`

---

### 4.6 Interpolationsfunktionen

| Funktion | Signatur | Verwendung |
|---|---|---|
| `FUN_01a7ee` | `(Tabelle, Achse1, Achse2, Grösse)` | `FUN_021866`, `FUN_0218ba` |
| `FUN_01a6da` | `(Tabelle, Achse1, Achse2, Grösse)` | `FUN_021978` |
| `FUN_01a4ec` | `(Wert, Parameter)` | `FUN_0218ba` — evtl. Skalierung/Normierung |
| `FUN_031a1e` | Hardware-nah | Stepper-Motor Treiber |

**Offene Frage:** Unterschied zwischen `FUN_01a7ee` und `FUN_01a6da` — möglicherweise
lineare vs. bilineare Interpolation, oder 1D vs. 2D Lookup.

---

## 5. RAM-Variablen (Laufzeit)

| Adresse | Name | Beschreibung |
|---|---|---|
| `DAT_00d88f` | idle_fuel_trim | aktueller Idle Fuel Trim Ausgabewert |
| `DAT_00d893` | idle_mode | Betriebsmodus (5 = normal, 7 = erweitert) |
| `DAT_00d8a5` | idle_state_current | aktueller Idle-Zustandsbyte |
| `DAT_00d8a6` | idle_state_previous | vorheriger Zustandsbyte (für Änderungserkennung) |
| `DAT_00d8c1` | idle_enable_flag | Idle-Enable (0x0000 oder 0xFFFF) |
| `DAT_00d8c7` | iac_ref_lookup_result | Ergebnis Referenz-Tabellen-Lookup |
| `DAT_00d8c9` | iac_ref2_result | Ergebnis zweiter Referenz-Lookup |
| `DAT_00d8cb` | iac_ref3_result | Ergebnis dritter Referenz-Lookup |
| `DAT_00d8cd` | iac_init_flag | Initialisierungs-Flag |
| `DAT_00d8d3` | iac_ch1_counter | IAC Kanal 1 Schritt-Zähler |
| `DAT_00d8d9` | iac_ch3_counter | IAC Kanal 3 Schritt-Zähler |
| `DAT_00d8e7` | iac_ch1_position | IAC Kanal 1 aktuelle Position |
| `DAT_00d8e9` | iac_ch3_position | IAC Kanal 3 aktuelle Position |
| `DAT_00d8eb` | iac_ch2_position | IAC Kanal 2 aktuelle Position |
| `DAT_00d8f1` | fuel_trim_accumulator | laufender Trim-Akkumulator (gedämpft) |
| `DAT_00d8f3` | fuel_trim_min | Fuel Trim Minimum-Grenze |
| `DAT_00d8f5` | fuel_trim_max | Fuel Trim Maximum-Grenze |
| `DAT_00dc6d` | sensor_axis1 | Sensor-Achse 1 für Tabellen-Lookups (RPM? TPS?) |
| `DAT_00dc75` | sensor_axis2 | Sensor-Achse 2 für Tabellen-Lookups |
| `DAT_00dc8d` | operating_mode | globaler Betriebsmodus |
| `DAT_00d97f` | idle_enable2 | sekundäres Idle-Enable-Flag |
| `DAT_00da5d` | unknown_da5d | — zu identifizieren |
| `DAT_00da6d` | unknown_da6d | — zu identifizieren |
| `DAT_00da6f` | unknown_da6f | — zu identifizieren |

---

## 6. Noch zu finden

| Tabelle | Priorität | Hinweis |
|---|---|---|
| AFR Target Idle | hoch | wahrscheinlich in `FUN_0216ce` oder Callees |
| Decel Fuel Cut Schwellen | hoch | separate Funktion erwartet |
| Initialisierungs-Defaults für RAM-Tabellen | mittel | in `FUN_0218ba` oder Init-Code |
| Sensor-Mapping `DAT_00dc6d`/`0x00dc75` | mittel | welche Sensoren sind das? |
| Vollständige `0x73xx` Parameterliste | mittel | IAC-Kalibrierung komplett kartieren |
| Bedeutung `FUN_01a7ee` vs `FUN_01a6da` | niedrig | algorithmischer Unterschied |

---

## 7. Nachtrag: FUN_0216ce — Idle Mode State Machine

### 7.1 Funktion

**Adresse:** `0x0216ce`  
**Aufgerufen von:** `FUN_020f60` bei `0x020f6a`

Oberste Idle-Control-Ebene. Berechnet Idle-Modus aus Sensor-Zuständen, setzt
Flags, und ruft `FUN_021866` auf.

### 7.2 Neu: State-Übergangs-Tabelle

| Adresse | Physisch | Typ | Beschreibung |
|---|---|---|---|
| `0x006be6` | `0x006be6` | byte[5×N] | 2D State-Matrix — bestimmt Idle-Betriebsmodus |

Index-Berechnung:
```c
idx = (axis1_norm * 5 * 2) + axis2_norm
state = table[idx]   // Byte-Wert = Idle-Modus
```

### 7.3 Neu: AFR / Fuel Target Tabelle

**RAM-Variable:** `DAT_00d8ed` = aktueller Idle Fuel Target

Wird aus drei Flash-Einzelwerten geladen je nach Zustand:

| Adresse | Physisch | Bedingung | Bedeutung |
|---|---|---|---|
| `0x00704f` | `0x01704f` | `DAT_00d8bf == 0` | AFR Target Normal-Idle |
| `0x007051` | `0x017051` | Bit 2 in `DAT_00d8ad` | AFR Target Zustand 2 |
| `0x007053` | `0x017053` | Bit 4 + `DAT_00dc91==0` + `DAT_00d8a0!=0` | AFR Target Aufwärm-Idle |

### 7.4 Sensor-Identifikation (teilweise)

| RAM-Adresse | Normierung | Hypothese |
|---|---|---|
| `DAT_00dc6d` | `(x + 0x80) >> 8` | Temperatursensor (Kühlwasser?) |
| `DAT_00dc75` | `(x + 0x80) >> 8` | Temperatursensor (Ansaugluft?) |

### 7.5 Neue Funktionen zur Analyse

| Funktion | Aufgerufen bei | Hypothese |
|---|---|---|
| `FUN_02156c` | `0x02177c` | Idle-Übergangs-Handler |
| `FUN_021620` | `0x0217de` | Idle-Entry/Exit-Logik |
| `FUN_020f60` | — | übergeordneter Task-Scheduler |

### 7.6 State Machine Logik (vereinfacht)

```
DAT_008dea == 0?  →  Idle gesperrt, alles zurücksetzen
DAT_00d895 == 1?  →  Idle aktiv
  ├── Zustandsänderung?  →  FUN_02156c
  └── stabil          →  FUN_021866 → FUN_021978 + FUN_021a0e
DAT_00d895 != 1   →  Ramp-up/down via FUN_021620
  └── Modus 5 oder 7  →  Vollbetrieb aktivieren
```

### 7.7 Neue Flash-Einzelwerte (noch zu lesen)

In Ghidra zu diesen Adressen navigieren und Werte als `word` definieren:

| Adresse | Bedeutung |
|---|---|
| `0x00704f` | AFR Target Normal-Idle |
| `0x007051` | AFR Target Zustand 2 |
| `0x007053` | AFR Target Aufwärm-Idle |
| `0x006c87` | IAC Referenzwert (wird in State-Zweig geladen) |

---

## 8. Nachtrag: RAM-Variablen erweitert & Sensor-Architektur

### 8.1 AFR Target Variablen (RAM, nicht Flash)

| Adresse | Wert im BIN | Bedeutung |
|---|---|---|
| `0x00704f` / phys. `0x01704f` | `0xFFFF` | AFR Target Normal-Idle — RAM, zur Laufzeit gesetzt |
| `0x007051` / phys. `0x017051` | `0xFFFF` | AFR Target Zustand 2 — RAM |
| `0x007053` / phys. `0x017053` | `0xFFFF` | AFR Target Zustand 3 — RAM |

`0xFFFF` = Initialisierungszustand ("noch nicht gesetzt"). Echte Werte kommen
aus Flash via `FUN_020e42` (schreibt `DAT_00d8ed`).

### 8.2 Sensor-Architektur

**Schreib-Zugriffe (Sensor-Einlese-Layer bei `0x02dxxx`):**

| Funktion | Schreibt | Adresse |
|---|---|---|
| `FUN_02d92a` | `DAT_00dc6d` | `0x02d932` |
| `FUN_02d92a` | `DAT_00dc75` | `0x02d93e` |
| `FUN_02d992` | `DAT_00dc6d` | `0x02d99e` |
| `FUN_02d9dc` | `DAT_00dc75` | `0x02d9e8` |

**Identifikationshypothese:**
- `DAT_00dc75`: Kühlwassertemperatur (17 XREFs, wird in Kalibrierungs-Funktionen `0x01cxxx` verwendet)
- `DAT_00dc6d`: Ansauglufttemperatur oder TPS (11 XREFs)

### 8.3 Idle Fuel Target Anwendung

`DAT_00d8ed` (Idle Fuel Target) wird in `FUN_02107e` an 4 Stellen gelesen —
das ist die Funktion die den Wert tatsächlich auf die Einspritzung anwendet.

### 8.4 Task-Ausführungsreihenfolge (vollständig)

```
FUN_01963e          [Haupt-Task-Loop]
  └── FUN_020f60    [Idle Task — nur wenn DAT_00d893 != 0]
        ├── FUN_0211f6   [Sensor-Vorverarbeitung]
        └── FUN_0216ce   [Idle State Machine]
              └── FUN_021866  [Idle Control Dispatcher]
                    ├── FUN_0218ba  [IAC Init]
                    ├── FUN_021978  [Idle Fuel Trim]
                    └── FUN_021a0e  [IAC Stepper]
```

### 8.5 Noch zu analysierende Prioritäts-Funktionen

| Funktion | Priorität | Grund |
|---|---|---|
| `FUN_02d92a` | hoch | schreibt beide Sensor-Variablen direkt |
| `FUN_0211f6` | hoch | Sensor-Vorverarbeitung vor Idle-Control |
| `FUN_020e42` | hoch | schreibt echten AFR-Target in `DAT_00d8ed` |
| `FUN_02107e` | mittel | wendet Fuel Target auf Einspritzung an |
| `FUN_01963e` | niedrig | Haupt-Task-Loop, gibt Gesamtüberblick |

---

## 9. Nachtrag: Sensor-Block & Idle-Initialisierung

### 9.1 Sensor-RAM-Block

Zusammenhängender Block `0x00dc69`–`0x00dc85`:

| Offset | RAM-Adresse | Kanal | Bekannte Verwendung |
|---|---|---|---|
| 0 | `DAT_00dc69` | Sensor 0 | — |
| 2 | `DAT_00dc6b` | Sensor 1 | — |
| 4 | `DAT_00dc6d` | Sensor 2 | Idle-Lookups Achse 1 |
| 6 | `DAT_00dc6f` | Sensor 3 | — |
| 8 | `DAT_00dc71` | Sensor 4 | — |
| 10 | `DAT_00dc73` | Sensor 5 | — |
| 12 | `DAT_00dc75` | Sensor 6 | Idle-Lookups Achse 2 |
| 14 | `DAT_00dc77` | Sensor 7 | — |
| 16 | `DAT_00dc79` | Sensor 8 | — |
| 18 | `DAT_00dc7b` | Sensor 9 | — |
| 20 | `DAT_00dc7d` | Sensor 10 | — |
| 22 | `DAT_00dc7f` | Sensor 11 | Lookup-Achse in `FUN_020e42` |
| 24 | `DAT_00dc81` | Sensor 12 | — |
| 26 | `DAT_00dc85` | — | Reset-Status-Flag (0x01 = Reset abgeschlossen) |

### 9.2 IAC Stepper Default-Startpositionen (Flash)

| Flash-Adresse | RAM-Ziel | Bedeutung |
|---|---|---|
| `DAT_00e001` | `DAT_00d8e7` | Kanal 1 Default-Startposition |
| `DAT_00e003` | `DAT_00d8e9` | Kanal 2 Default-Startposition |
| `DAT_00e005` | `DAT_00d8eb` | Kanal 3 Default-Startposition |

→ In Ghidra als `word` definieren, Werte direkt kalibrierbar.

### 9.3 Neue unbekannte Flash-Adressen

| Adresse | Physisch | Typ | Verwendung |
|---|---|---|---|
| `0x006b74` | `0x006b74` | Tabelle? | Lookup mit `FUN_01a67e` + Sensor `DAT_00dc7f` |
| `0x006b95` | `0x006b95` | word | Init-Wert für `DAT_00d891`/`d8b1`/`d8b3` |
| `0x00704d` | `0x00704d` | word | Idle-Konfigurations-Flag `DAT_00d8bf` |

### 9.4 Noch offene Initialisierungsquellen

| RAM-Variable | Beschreibung | Quelle unbekannt |
|---|---|---|
| `DAT_00d8f3` | Fuel Trim Minimum | Write-Zugriffe suchen |
| `DAT_00d8f5` | Fuel Trim Maximum | Write-Zugriffe suchen |
| `DAT_00d8ed` | Idle Fuel Target | Echte Werte kommen nicht aus `FUN_020e42` |

---

## 10. Nachtrag: FUN_0211f6 — Kern-Zustandsautomat

### 10.1 Übersicht

**Adresse:** `0x0211f6`  
**Aufgerufen von:** `FUN_020f60` bei `0x020f66` (vor `FUN_0216ce`)

Implementiert den vollständigen Idle-Modus-Zustandsautomaten mit 7 Zuständen.

### 10.2 Switch-Table

```
Tabellen-Adresse: 0x007ec   (7 × word Sprungadressen)
Modus-Register:   DAT_00d893 (0–6)
Modus-Setter:     FUN_02142e (Parameter = neuer Zielzustand)
```

| Modus | Bedeutung | Übergang via |
|---|---|---|
| 0 | Idle inaktiv | — |
| 1 | Initialisierung | `FUN_02142e(#2)` |
| 2 | Warte-Zustand | `FUN_02142e(#3)` |
| 3 | Übergang | `FUN_02142e(#4)` |
| 4 | Aktiver Idle | `FUN_02142e(#5)` |
| 5 | Kaltstart/Sonder | `FUN_02142e(#6)` |
| 6 | Fehler/Neustart | `FUN_02142e(#7)` |

### 10.3 Neu entdeckte Tabelle: Idle Enable

| Eigenschaft | Wert |
|---|---|
| Adresse | `0x00707e` |
| Grösse | 20 Einträge (word[20]) |
| Interpolation | `FUN_01a7ee` |
| Achse 1 | `DAT_00dc6d` |
| Achse 2 | `DAT_00dc75` |
| Ergebnis | `DAT_00d8c1` (Idle Enable Flag) |

Diese Tabelle ist kritisch — ihr Ausgabewert `DAT_00d8c1` steuert direkt
ob `FUN_021866` (Idle Control Dispatcher) ausgeführt wird.

### 10.4 Neu entdeckte Flash-Einzelwerte

| Adresse | Physisch | Ziel | Bedeutung |
|---|---|---|---|
| `0x006b97` | `0x006b97` | `DAT_00d8b9` | Idle-Schwellen-Init |
| `0x006bbf` | `0x006bbf` | `DAT_00d8b5` | Timeout-Wert |
| `0x006bc1` | `0x006bc1` | `DAT_00d8b7` | Kalibrierparameter |

### 10.5 Bekannte RAM-Variablen (ergänzt)

| Adresse | Name | Beschreibung |
|---|---|---|
| `DAT_00d893` | idle_mode | Aktueller Zustand (0–6) |
| `DAT_00d89d` | idle_mode_prev | Vorheriger Zustand |
| `DAT_00d8b5` | idle_timer | Timeout-Zähler |
| `DAT_00d8b7` | idle_param_b7 | Kalibrierparameter |
| `DAT_00d8b9` | idle_threshold | Idle-Schwellenwert |
| `DAT_00d8c1` | idle_enable | Idle Enable (0=aus, sonst=ein) |
| `DAT_00dc8d` | global_mode | Globaler Betriebsmodus (Modus 8 = Sperrbedingung) |

### 10.6 Ghidra-Hinweis

Bytes `0x021248`–`0x0212df` sind nicht disassembliert.
→ In Ghidra zu `0x021248` → Rechtsklick → **Disassemble**
Das deckt Zustands-Handler 2–4 der Switch-Table auf.

### 10.7 Vollständige Task-Ausführungsreihenfolge (aktualisiert)

```
FUN_01963e              [Haupt-Task-Loop]
  └── FUN_020f60        [Idle Task — nur wenn DAT_00d893 != 0]
        ├── FUN_0211f6  [Zustandsautomat — setzt DAT_00d893]
        │     ├── FUN_02142e  [Modus-Setter]
        │     ├── FUN_02156c  [Übergangs-Handler]
        │     ├── FUN_021536  [unbekannt — prüft Bedingungen]
        │     └── Tabelle 0x007ec  [Switch-Table 7 Zustände]
        └── FUN_0216ce  [Idle State Machine — liest DAT_00d893]
              └── FUN_021866  [Idle Control Dispatcher]
                    ├── FUN_0218ba  [IAC Init]
                    ├── FUN_021978  [Idle Fuel Trim]
                    └── FUN_021a0e  [IAC Stepper]
```

---

## 11. Nachtrag: K-Line Handler & präzise Zustandsübergänge

### 11.1 FUN_0007e2 — K-Line Diagnose-Handler

**Nicht Teil der Idle-Steuerung.** Implementiert KWP2000 Fast-Init:

| Element | Wert | Bedeutung |
|---|---|---|
| Timer | `T5` (SFR) | Hardware-Timing |
| Port-Pin | `P3.0xb` | K-Line physisches Signal |
| Sync-Byte | `0x55` | KWP2000 Standard |
| Komplement | `0x4A` | Antwort-Byte 1 |
| Bestätigung | `0x56` | Antwort-Byte 2 |

TuneECU, IAW-Reader, MAP-Reader kommunizieren über diese Funktion.

### 11.2 Idle-Zustandsübergänge (vollständig)

| Von | Nach | Bedingung |
|---|---|---|
| 1 | 2 | `DAT_00db4b==1` AND `DAT_00db0a==0` AND `DAT_00d8b9==0` |
| 2 | 7 | `r8 > DAT_00d891` (RPM-Schwelle) AND `FUN_021536` OK |
| 2 | 6 | `FUN_021536` meldet Fehler |
| 3 | 4 | `DAT_00d8b5==0` (Timer abgelaufen) |
| 3 | Exit | `DAT_00d8b5 > 0` (Timer läuft noch) |
| 4 | 5 | `DAT_00d8c1` aktiv AND Bit 3 in `DAT_00d8ad` |
| 4 | 7 | RPM-Schwelle `DAT_00d891` erreicht |
| 5 | Tabelle | `DAT_00d8c1==0xFFFF` AND Bit 3 in `DAT_00d8ad` → `0x00707e` interpolieren |
| 5 | 6 | `FUN_021536` meldet Fehler |
| 6 | 3 | Standard-Rückfall |

### 11.3 Switch-Table

| Adresse | Inhalt | Definieren als |
|---|---|---|
| `0x0007ec` | 7 × word Sprungadressen | `word[7]` in Ghidra |

Eingebettet in `FUN_0007e2` direkt nach dem `srvwdt` bei `0x0007e6`.

### 11.4 Neue RAM-Variablen (ergänzt)

| Adresse | Name | Beschreibung |
|---|---|---|
| `DAT_00db4b` | start_condition | Startbedingung (muss ==1 für Modus 1→2) |
| `DAT_00db0a` | inhibit_flag | Sperr-Flag (muss ==0 für Modus 1→2) |
| `DAT_00d8ad` | state_bits | Zustandsbit-Register (Bit 3 = Idle-Enable-Hysterese) |
| `DAT_00da7b` | idle_mode_backup | R8-Initialisierungswert am Funktionseingang |
| `DAT_00da7d` | transition_flag | Übergangs-Flag bei Modus-7-Aktivierung |
