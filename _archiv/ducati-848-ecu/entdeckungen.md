---
title: Entdeckungen - chronologisch
parent: ecu_reverse_engineering_blog
order: 1
date: 2026-06-01
summary: der weg durch Ghidra, Fund für Fund
---

# Ducati 848 — IAW 5AM HW610 ECU: Entdeckungshistorie

**BIN-Datei:** `Summer_Map_Backup.bin` (327'680 Bytes)  
**MAP-Variante:** `22ADADPSMA1`  
**Prozessor:** STMicroelectronics ST10F269 (C166/ST10-Architektur, 16-Bit)  
**Reverse-Engineering-Tool:** Ghidra 12.1 + keyhana/c166-ghidra-module v1.1.0

---

# Ducati 848 — IAW 5AM HW610: Entdeckungshistorie

**BIN:** `Summer_Map_Backup.bin` · **Variante:** `22ADADPSMA1`  
**Prozessor:** STMicroelectronics ST10F269 (C166/ST10, 16-Bit, Little Endian)  
**Tool:** Ghidra 12.1 + keyhana/c166-ghidra-module v1.1.0

---

## Methodischer Ansatz

Nach einer ersten explorativen Phase (XREF-getriebene Suche nach bekannten Tabellen)
wurde der Ansatz auf **Top-Down vom Reset-Vektor** umgestellt. Grund: ohne Verständnis
des Software-Aufbaus wurden Funktionen falsch eingestuft und RAM-Variablen als
Flash-Kalibrierdata interpretiert. Die neue Methode folgt dem Ausführungspfad der
ECU von Anfang an, Funktion für Funktion.

---

## Phase 1 — Ausgangslage: Bekannte Defekte (vor Ghidra)

Durch Hex-Analyse des BIN und Abgleich mit XDF-Dateien identifizierte Defekte:

| Adresse (physisch) | Tabelle | Befund |
|---|---|---|
| `0x04C300` | Ignition Map | Bereich idle/low-load = 10° BTDC — zu tief, primäre Backfire-Ursache |
| `0x04A940` | Idle RPM Target | Strukturell korrekt (1600→1450 RPM) |
| `0x04A960` | Idle Temp-Achse | Non-monoton (…120, 140, 120°C) — bricht Interpolation |
| `0x04A93A` | Cold-Start Timer | ~12 Sekunden statt erwarteter 90–180 Sekunden |

Praktischer Test: 98 RON Kraftstoff eliminierte Backfires im Standgas — bestätigt
Zündtiming als primäre Ursache, nicht Gemisch.

---

## Phase 2 — Ghidra Setup

**Problem:** Plugin v1.2.0 (inoffizieller Build) schlug fehl mit:
```
Unknown pentry register: CP
```
`CP` (Context Pointer) war in `c166.cspec` als Calling-Convention-Register deklariert
aber nicht korrekt im SLEIGH-Register-Space definiert. Ghidra 12.x validiert cspec strenger.

**Lösung:** Downgrade auf offizielles Release v1.1.0.

---

## Phase 3 — Reset-Vektor & Interrupt-Vektor-Tabelle (Top-Down Start)

**`0x000000` — Interrupt-Vektor-Tabelle (IVT)**

Die ersten ~0x200 Bytes sind die IVT des ST10F269. Jeder Eintrag = 4 Bytes (`jmps`).

Aktive ISRs (alle anderen zeigen auf Default-Handler `thunk_FUN_02e8cc`):

| IVT-Adresse | Vektor # | Handler | Status |
|---|---|---|---|
| `0x000000` | Reset | `FUN_000222` | Init-Funktion → analysiert |
| `0x0000e8` | 58 | `thunk_FUN_022b5c` | unbekannt — zu analysieren |
| `0x0000ec` | 59 | `thunk_FUN_022d24` | unbekannt — zu analysieren |
| `0x0000f0` | 60 | `FUN_0040f0` | unbekannt — zu analysieren |
| `0x0000f8` | 62 | `thunk_FUN_02afe4` | unbekannt — zu analysieren |
| `0x000100` | 64 | `thunk_FUN_02a0e4` | unbekannt — zu analysieren |
| `0x000110` | 68 | `thunk_FUN_027a5c` | unbekannt — zu analysieren |
| `0x00011c` | 71 | `thunk_FUN_02cd70` | unbekannt — zu analysieren |

`thunk_FUN_02e8cc` = Default/Leer-Handler für unbenutzte Interrupts.

---

## Phase 4 — FUN_000222: Hardware-Initialisierung

**Ausführungsreihenfolge nach Reset:**

```
1. Stack initialisieren  (SP, STKOV, STKUN, CP)
2. DPP-Register setzen   (DPP0=0x01, DPP1=0x12, DPP2=0x13)
3. FUN_000594            Hardware-Peripherie-Init
4. FUN_0005a0            zweite Init-Stufe
5. einit                 Interrupts aktivieren ← ISRs laufen ab hier
6. FUN_0005a6            Init nach Interrupt-Enable
7. srvwdt                Watchdog bedienen
8. FUN_002582            HAUPT-APPLIKATION (läuft endlos)
   reti                  wird nie erreicht
```

**Bestätigt durch FUN_000222:**
- DPP-Werte (DPP0=0x01, DPP1=0x12, DPP2=0x13) — direkt im Code sichtbar
- Stack liegt im internen RAM: `0xFA0C`–`0xFC00` (~500 Bytes)
- Context Pointer: `0xF754`
- Nächster Analyseschritt: `FUN_000594` (Hardware-Init)

---

## Offene Schritte (priorisiert)

1. `FUN_000594` — Hardware-Peripherie-Init
2. `FUN_0005a0` — zweite Init-Stufe
3. `FUN_0005a6` — Init nach `einit`
4. `FUN_002582` — Haupt-Applikation / Task-Scheduler
5. Die 7 aktiven ISRs identifizieren (Sensor-Lesen, Timer, etc.)

---

## Phase 5 — FUN_000594: Minimale Bus-Konfiguration

**Funktion:** 3 Instruktionen, 2 Register-Writes.

```
extr  #0x1              → Extended-Mode für nächste Instruktion (XSFR-Zugriff)
mov   0xF024,  #0x000D  → XSFR-Register (Peripherie, noch zu identifizieren)
mov   SYSCON,  #0x0514  → System Configuration Register
```

**SYSCON = 0x0514:** Steuert External-Bus-Timing und Memory-Wait-States.
Keine komplexe Hardware-Init — nur grundlegende Bus-Konfiguration.
Echter Hardware-Init wahrscheinlich in `FUN_0005a0`.

---

## Phase 5 — FUN_000594 & FUN_0005a0: Hardware-Konfiguration (mit Datenblatt verifiziert)

**FUN_000594** — 3 Instruktionen, zwei Register-Writes (via `extr` für XSFR-Zugriff):

```
XPERCON (0xF024h) = 0x000D:
  Bit 3  XRAM2EN = 1  → 8KB XRAM2 aktiviert (0x00C000–0x00DFFF)
  Bit 2  XRAM1EN = 1  → 2KB XRAM1 aktiviert (0x00E000–0x00E7FF)
  Bit 0  CAN1EN  = 1  → CAN1 aktiviert (0x00EF00–0x00EFFF)
  Bit 1  CAN2EN  = 0  → CAN2 deaktiviert
  Bit 4  RTCEN   = 0  → RTC deaktiviert

SYSCON (0xFF12h) = 0x0514:
  Bit 10  ROMEN  = 1  → Interner Flash aktiv
  Bit  8  CLKEN  = 1  → Clock Enable
  Bit  4  OWDDIS = 1  → Oscillator Watchdog deaktiviert
  Bit  2  XPEN   = 1  → X-Peripherals freigegeben (sperrt XPERCON!)
```

**Wichtig:** XPERCON kann nach XPEN-Setzen in SYSCON nicht mehr geändert werden.
Reihenfolge XPERCON zuerst, dann SYSCON ist zwingend.

**FUN_0005a0** — 1 Instruktion:

```
BUSCON0 (0xFF0Ch) = 0x049F:
  Bit 10  BUSACT0 = 1  → External Bus aktiv
  Bit 7-6 BTYP    = 10 → 16-bit Demultiplexed Bus
  Bit  5  MTTC0   = 0  → 1 Memory Tristate Wait State
  Bit  4  RWDC0   = 1  → Kein R/W-Delay
  Bit 3-0 MCTC    = 1111 → 0 Wait States (max. Geschwindigkeit)
```

**Grosse Erkenntnis — RAM-Layout jetzt vollständig klar:**

Alle `DAT_00dxxx`-Variablen die wir in der Idle-Steuerung gesehen haben liegen in
**XRAM2** (`0x00C000–0x00DFFF`). Die `DAT_00eXxx`-Variablen (IAC-Startpositionen)
liegen in **XRAM1** (`0x00E000–0x00E7FF`). Das erklärt warum diese Werte im BIN
als `0xFFFF` erscheinen — es ist initialisierter RAM, kein Flash.

---

## Phase 6 — FUN_0005a6: Watchdog-Konfiguration

**Eine Instruktion:** `mov WDTCON, #0xF401`

```
WDTCON (0xFFAEh) = 0xF401:
  Bit 15-8  WDTREL = 0xF4  → Watchdog Reload Value = 244
  Bit 0     WDTIN  = 1     → Taktteiler fCPU/128
  → Watchdog-Timeout: ~9.83ms bei 40MHz CPU-Takt
```

**Konsequenz:** Der gesamte Task-Scheduler in `FUN_002582` muss so ausgelegt
sein, dass `srvwdt` mindestens alle ~10ms ausgeführt wird. Das definiert die
maximale Zykluszeit aller periodischen Tasks.

**Init-Sequenz jetzt vollständig dokumentiert:**

| Schritt | Funktion | Was passiert |
|---|---|---|
| 1 | `FUN_000222` | Stack (SP/STKOV/STKUN/CP), DPP0/1/2 |
| 2 | `FUN_000594` | XPERCON (XRAM+CAN), SYSCON (Lock+Enable) |
| 3 | `FUN_0005a0` | BUSCON0 (External Bus 16-bit, 0 Wait States) |
| 4 | `einit` | Interrupts freigeben |
| 5 | `FUN_0005a6` | WDTCON (Watchdog ~10ms Timeout) |
| 6 | `srvwdt` | Erster Watchdog-Service |
| 7 | `FUN_002582` | Haupt-Applikation (läuft endlos) |

---

## Phase 7 — FUN_002582: Applikations-Einstieg & Bootstruktur

**Ausführungsfluss:**

```
1. FUN_0005ac        → Init (unbekannt, vor K-Line)
2. FUN_0007c6        → Init (unbekannt, vor K-Line)
3. FUN_0007e2        → K-Line Diagnostic Handler
   ├── K-Line aktiv  → FUN_0029c0 (K-Line Kommunikations-Loop)
   └── K-Line inaktiv ↓
4. DAT_00d853 == 0x10EF? → thunk_FUN_018046 (Bootloader/Flash-Modus, kein Return)
5. Normal:  FUN_003058 (r12=DAT_00d104, r13=0x3F10, r14=0) → Haupt-Task-Scheduler
6. FUN_002616 (wahrscheinlich nie erreicht)
```

**Magic-Value `0x10EF`:**
`DAT_00d853` (in XRAM2) wird auf `0x10EF` verglichen. Wenn gesetzt →
`thunk_FUN_018046` ohne Return. Klassischer Automotive-Bootloader-Trigger.
Wird wahrscheinlich über K-Line-Protokoll von aussen gesetzt.

**`FUN_003058` — Haupt-Task-Scheduler:**
Parameter:
- `r12 = DAT_00d104` → Task-Control-Tabelle (in XRAM2)
- `r13 = 0x3F10` → Task-Liste (im Flash, Segment 0)
- `r14 = 0x0` → Startwert/Counter

Das ist die wichtigste Funktion der gesamten ECU-Software.
Dort sind alle periodischen Tasks definiert.

---

## Phase 8 — FUN_003058: Control-Block-Initialisierung (nicht der Haupt-Loop)

**FUN_003058** ist eine Initialisierungsroutine für einen Task-Control-Block,
**nicht** die Haupt-Task-Loop.

```
Parameter:
  param_1 (r12) = Zeiger auf Control-Block (DAT_00d104, in XRAM2)
  param_2 (r13) = Task-Liste (0x3F10, Flash)
  param_3 (r14) = Startwert (0x0)

Ablauf:
  if param_2|param_3 != 0 → FUN_002f9a + FUN_0006d2 (Re-Init/Cleanup)
  Initialisiert Control-Block-Felder auf 0
  control_block[0x38] = 0x28A0  (= 10400 — evtl. Puffergrösse/Timer)
  FUN_002f6a(0xFA)              (= 250 — Timer oder Delay)
```

Control-Block-Grösse: mindestens 0x75+1 Bytes. Wird von 4 Stellen aufgerufen.

**Korrektur der Analyse von FUN_002582:**  
Die eigentliche Haupt-Task-Loop ist **FUN_002616**, nicht FUN_003058.

Neue Priorität: FUN_002616 analysieren.

---

## Phase 9 — FUN_002616: Haupt-Loop aufgedeckt

**Struktur:** Polling-basierte Haupt-Loop mit Timer-getriebenem Task-Executor.

```
Init: FUN_00098a, Flags DAT_00d863/DAT_00d864 = 0

LOOP (LAB_002622):
  S0RIC.bit7?  → FUN_003398    ← UART0 K-Line Daten verarbeiten
  FUN_0007d0   → FUN_002574    ← unbekannte Bedingung
  DAT_00d863 == 0? → LOOP      ← enge Schleife bis Timer-Flag gesetzt
  DAT_00d175 != 0? → LOOP      ← zweite Bedingung
  State-Machine (DAT_00d17b)   ← Boot-Sequenz
  FUN_0026e4(DAT_00d104)       ← HAUPT-TASK-EXECUTOR
  → LOOP
```

**Erkenntnisse:**

`S0RIC` (SFR 0xFF8A): UART0 Receive Interrupt Control Register.
Bit 7 = IR-Flag. Wird gepollt, kein ISR → K-Line-Kommunikation ist polling-basiert.
`FUN_003398` = UART0/K-Line Datenverarbeitungs-Funktion.

`DAT_00d863` = Timer-getriebenes Task-Trigger-Flag. Wird von einem Timer-ISR
(aus der IVT) gesetzt. Verbindet ISRs mit dem Task-Executor.

`FUN_0026e4` = eigentlicher Task-Executor. Bekommt Control-Block-Pointer
(DAT_00d104). Führt alle periodischen Tasks aus. Nächste Priorität.

**Boot-State-Machine (`DAT_00d17b`):**

| Zustand | Funktion | Übergang |
|---|---|---|
| 0x03 | FUN_000940, FUN_00068c, FUN_000a24 | → 0x14 oder 0x16 |
| 0x07 | FUN_000650(0x3F, 0x7F), FUN_0005f6 | → 0x08 |
| 0x08 | Normal-Betrieb | → FUN_0026e4 |
| 0x18 | FUN_0024b2, FUN_000650(0xB, 0x3F) | → 0x19 |

---

## Phase 10 — FUN_0026e4: Buffer-Copy, nicht Task-Dispatcher

**FUN_0026e4** kopiert 6 Bytes aus `DAT_00d63e` (XRAM2) in einen
DMA-Buffer via PEC-Channel-Zeiger bei ESFR `0xF646`.

```
r14 = *(ESFR 0xF646)                ← PEC Schreibzeiger
Loop 6×: *r14++ = DAT_00d63e[i]     ← 6 Bytes kopieren
control_block[0x5B] = 9
control_block[0x5E] = DAT_00d648
control_block[0x1F] = DAT_00d649
```

**Architektur-Korrektur:**

FUN_002616/FUN_0026e4 gehören zur K-Line/UART-Kommunikation, nicht zur
ECU-Motorsteuerung. Die periodischen Regelungs-Tasks (Zündung, Einspritzung,
Idle-Control) laufen in den **Timer-ISRs** aus der IVT — nicht in der Haupt-Loop.

Das erklärt warum die Idle-Control-Funktionen (`FUN_020f60`, `FUN_0216ce` etc.)
nicht von FUN_002616 aus aufgerufen werden.

**Neue Priorisierung:**
Die 7 aktiven ISRs aus der IVT müssen als nächstes analysiert werden.
Dort liegt die eigentliche ECU-Regelungslogik.

---

## Phase 11 — FUN_0005ac & FUN_0007c6: Bootloader-Check & Timer-Init

### FUN_0005ac — Bootloader-Trigger-Check
Liest einen gespeicherten Wert (via FUN_002df0 + FUN_000472) und prüft
ob Offset 14 des Puffers == `0x10EF` (Bootloader-Magic-Value):
- JA → `DAT_00d853 = 0x10EF` → Bootloader-Modus aktiviert
- NEIN → `DAT_00d853 = 0x0000` → Normalbetrieb

### FUN_0007c6 — Timer-Vorkonfiguration (T4, T6)
Konfiguriert zwei Timer, startet sie aber NICHT (T4R=0, T6R=0):

| Timer | Register | Wert | Prescaler | Input-Freq | Max-Periode |
|---|---|---|---|---|---|
| GPT1 T4 | T4CON (FF44h) | 0x0042 | /32 | 1.25 MHz | 52.4 ms |
| GPT2 T6 | T6CON (FF48h) | 0x0041 | /8  | 5.0 MHz  | 13.1 ms |

### ISR-Zuordnung vollständig aufgelöst (mit Datenblatt)

| IVT-Adresse | Interrupt | Peripheral | ISR-Funktion | Bedeutung |
|---|---|---|---|---|
| `0x00E8` | CC26INT | CAPCOM Register 26 | `FUN_022b5c` | Output-Compare/Capture |
| `0x00EC` | CC27INT | CAPCOM Register 27 | `FUN_022d24` | Output-Compare/Capture |
| `0x00F0` | CC28INT | CAPCOM Register 28 | `FUN_0040f0` | Output-Compare/Capture |
| `0x00F8` | T8INT   | CAPCOM Timer 8     | `FUN_02afe4` | **Basis-Timer Overflow** |
| `0x0100` | XP0INT  | CAN1 Interface     | `FUN_02a0e4` | CAN-Bus Kommunikation |
| `0x0110` | CC29INT | CAPCOM Register 29 | `FUN_027a5c` | Output-Compare/Capture |
| `0x011C` | S0TBINT | ASC0 TX-Buffer     | `FUN_02cd70` | UART0 Sende-Buffer |

**Interpretation:**
- CC26-CC28: CAPCOM Output-Compare → Zündzeitpunkt, Einspritzung, Drehzahl-Messung
- T8INT (FUN_02afe4): CAPCOM Timer 8 Overflow → wahrscheinlich der Task-Trigger (setzt DAT_00d863)
- CAN1: Dashboard-Kommunikation oder externe Schnittstelle
- S0TBINT: UART0 Senden (Empfang wird gepollt in FUN_002616)

**CAPCOM Timer 8** ist der Basis-Takt für die CAPCOM-Einheit. Sein Overflow-ISR
(FUN_02afe4) ist höchstwahrscheinlich der Task-Trigger der `DAT_00d863` setzt.

---

## Phase 12 — ISR-Wrapper-Muster & Idle-Control-Einstieg

### Universelles ISR-Wrapper-Muster (verifiziert)

Alle ISRs des ST10F269 in dieser Firmware verwenden identisches Muster:
```
push r0 → save CP context (scxt) → push/restore DPP0+DPP2 →
calls [eigentlicher Handler] → pop DPP2/DPP0/CP/r0 → reti
```

Jeder ISR hat eine eigene Register-Bank (separates CP via scxt):

| ISR-Funktion | Register-Bank | Eigentlicher Handler |
|---|---|---|
| `FUN_022b5c` (CC26) | `0xF854` | `FUN_022b84` |
| `FUN_02afe4` (T8)   | `0xF934` | `FUN_02b00c` |

### FUN_01963e — Idle-Control-Einstieg

Aufgerufen von genau einer Stelle: `0x02ed9a` (Task-Dispatcher unbekannt).

```
calls FUN_02f57c    ← Sensor-Vorbereitungs-Funktion (liest DAT_00dc75)
calls FUN_020f60    ← Idle Control Task
rets
```

`FUN_02f57c` bereitet Sensor-Achswerte für die Idle-Tabellen-Lookups vor.
Die aufrufende Funktion bei `0x02ed9a` ist der Task-Dispatcher — nächste Priorität.

---

## Phase 13 — Vollständige Software-Architektur aufgedeckt

### Kritische Korrektur: 0x10EF ist "Application Valid"-Flag

`DAT_00d853 == 0x10EF` bedeutet NICHT Bootloader-Modus, sondern **Normalbetrieb**.
- `== 0x10EF` → `FUN_018046` (Haupt-ECU-Applikation) ← NORMALBETRIEB
- `!= 0x10EF` → K-Line Diagnose-/Programmier-Modus (FUN_003058 + FUN_002616)

### T8 ISR macht nur Zähler-Inkrement

`FUN_02b00c` inkrementiert nur `DAT_00d91d` (T8-Tick-Counter). `DAT_00d863`
wird von woanders gesetzt. Der zweite ISR-Wrapper ab `0x02b018` speichert
zusätzlich MDC/MDL/MDH (MAC-Einheit) und ruft `FUN_02b04e` auf.

### FUN_02ebdc — Haupt-Task-Dispatcher (in FUN_018046)

Aufgerufen von `FUN_018046:01819a`. Enthält mehrere getrennte Sub-Routinen:

| Adresse | Funktion |
|---|---|
| `0x02ebdc` | Init: 10 Funktionen + XRAM2 löschen (0xD058–0xD1F8) |
| `0x02ec24` | Task-Gruppe 1: periodische Tasks |
| `0x02ec8a` | Task-Gruppe 2: inkl. `FUN_019660` (Idle-Initialisierung) |
| `0x02ecec` | Konditional-Task: prüft `DAT_00d703` |
| `0x02ecfe` | Sensor-Conditional: `DAT_00d929`, `DAT_00dc89`, `DAT_00db5a` |
| `0x02ed52` | Kurzcheck: `DAT_00dc89` |
| `0x02ed6a` | **Haupt-Tasks**: inkl. `FUN_01963e` (Idle-Control) |

**Idle-Control-Bedingung:** `FUN_01963e` wird nur aufgerufen wenn `DAT_00d703 != 1`.
`DAT_00d703` ist das Idle-Control-Enable-Flag.

### Vollständige ECU-Architektur

```
FUN_002582 (Applikations-Einstieg):
  ├── DAT_00d853 == 0x10EF? → FUN_018046 (NORMALBETRIEB)
  │     └── FUN_02ebdc (Task-Dispatcher)
  │           ├── Init: XRAM2 Init, Peripherie-Setup
  │           ├── Tasks: Zündung, Einspritzung, Sensoren
  │           ├── FUN_019660 → FUN_020e42 (Idle-Init)
  │           └── FUN_01963e → FUN_02f57c + FUN_020f60 (Idle-Control)
  │
  └── DAT_00d853 != 0x10EF? → K-Line Modus
        └── FUN_002616 (UART0-Polling, Diagnose)

ISR-Ebene (immer aktiv):
  T8 Overflow → DAT_00d91d++ (Tick-Counter)
  CC26/27/28  → Zündung / Einspritzung (FUN_022b84, etc.)
  CC29        → Drehzahl / Sync
  CAN1        → Dashboard
  S0TB        → UART0 Senden
```
