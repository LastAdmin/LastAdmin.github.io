---
title: Technische Referenz
parent: ecu_reverse_engineering_blog
order: 1
date: 2026-06-01
summary: Adressen, Funktionen, Speicherlayout
---

# Ducati 848 — IAW 5AM HW610: Technische Referenz

**BIN:** `Summer_Map_Backup.bin` · **Variante:** `22ADADPSMA1`

---

## 1. Hardware

| Eigenschaft | Wert | Quelle |
|---|---|---|
| ECU | Magneti Marelli IAW 5AM HW610 | Hardware |
| Prozessor | STMicroelectronics ST10F269 | Hardware |
| Architektur | C166/ST10, 16-Bit, Little Endian | Datenblatt |
| Flash | 327'680 Bytes (0x50000) | BIN-Grösse |
| EEPROM | externer 95160 | Hardware |
| Ghidra-Plugin | keyhana/c166-ghidra-module v1.1.0 | Setup |

---

## 2. Speicher-Layout

### 2.1 Flash-Bereiche (im BIN)

| Bereich | Adresse | Inhalt |
|---|---|---|
| Interrupt-Vektor-Tabelle | `0x000000`–`0x0001FF` | IVT, 4 Bytes pro Vektor |
| Init-Daten | `0x000200`–`0x000221` | Konstanten, Thunk-Daten |
| Init-Code | `0x000222`–`...` | `FUN_000222` und folgende |
| Applikations-Code | ab `~0x000594` | Funktionen, Tabellen |

### 2.2 RAM-Layout (Laufzeit, nicht im BIN)

| Bereich | Adresse | Inhalt |
|---|---|---|
| Context Pointer / Reg-Banks | ab `0xF754` | CPU-Register-Kontext |
| Stack | `0xFA0C`–`0xFC00` | ~500 Bytes, wächst abwärts |
| Interner RAM | `0xFA0C`–`0xFFFF` | Variablen, SFR-Bereich |

### 2.3 DPP-Adressschema (BESTÄTIGT durch FUN_000222)

```
DPP0 = 0x01  →  physisch 0x004000–0x007FFF  (Code-Offset 0x0000–0x3FFF)
DPP1 = 0x12  →  physisch 0x048000–0x04BFFF  (Code-Offset 0x4000–0x7FFF)
DPP2 = 0x13  →  physisch 0x04C000–0x04FFFF  (Code-Offset 0x8000–0xBFFF)
```

**Formel:** `physisch = (DPP × 0x4000) + (Offset & 0x3FFF)`

**Ghidra-Hinweis:** Tabellenreferenzen erscheinen nicht automatisch weil Ghidra
nur den 16-Bit-Offset sieht. Manuell via Search → Memory nach LE-Bytes suchen.

---

## 3. Interrupt-Vektor-Tabelle

Jeder Eintrag = 4 Bytes (`jmps Segment, Offset`), beginnend bei `0x000000`.

| IVT-Adresse | Vektor # | Handler | Identifiziert |
|---|---|---|---|
| `0x000000` | 0 (Reset) | `FUN_000222` | ✅ Hardware-Init |
| `0x000004`–`0x00E4` | 1–57 | `thunk_FUN_02e8cc` | Default (leer) |
| `0x0000E8` | 58 | `thunk_FUN_022b5c` | ❓ offen |
| `0x0000EC` | 59 | `thunk_FUN_022d24` | ❓ offen |
| `0x0000F0` | 60 | `FUN_0040f0` | ❓ offen |
| `0x0000F8` | 62 | `thunk_FUN_02afe4` | ❓ offen |
| `0x000100` | 64 | `thunk_FUN_02a0e4` | ❓ offen |
| `0x000110` | 68 | `thunk_FUN_027a5c` | ❓ offen |
| `0x00011C` | 71 | `thunk_FUN_02cd70` | ❓ offen |

---

## 4. Initialisierungs-Sequenz (FUN_000222)

```
Adresse   Instruktion              Bedeutung
000222    mov STKOV, #0xFA0C      Stack-Overflow-Grenze
000226    mov STKUN, #0xFC00      Stack-Underflow-Grenze
00022a    mov SP,    #0xFC00      Stack Pointer (Top of Stack)
00022e    mov CP,    #0xF754      Context Pointer (Register-Bank)
000234    mov DPP0,  #0x01        Adress-Segment 0
000238    mov DPP1,  #0x12        Adress-Segment 1
00023c    mov DPP2,  #0x13        Adress-Segment 2
000246    calls FUN_000594        ← Hardware-Peripherie-Init     [TODO]
00024a    calls FUN_0005a0        ← RAM/Peripherie-Init Stufe 2  [TODO]
00024e    einit                   Interrupts aktivieren
000252    calls FUN_0005a6        ← Post-einit Init              [TODO]
000256    srvwdt                  Watchdog-Reset
00025a    calls FUN_002582        ← Haupt-Applikation            [TODO]
00025e    reti                    (nie erreicht)
```

---

## 5. Bekannte Kalibrier-Tabellen (Flash) — VERIFIZIERT

Direkt aus Hex-Analyse des BIN, unabhängig von Ghidra-Code-Analyse.

| Physische Adresse | Grösse | Inhalt | Defekt |
|---|---|---|---|
| `0x04C300` | 16×16 word | Ignition Map (RPM × TPS) | ⚠️ 10° BTDC in idle/low-load |
| `0x04A940` | 16 × word | Idle RPM Target (kalt→warm) | ✅ korrekt |
| `0x04A960` | 16 × word | Idle Temp-Achse | ⚠️ non-monoton |
| `0x04A93A` | word | Cold-Start Timer | ⚠️ ~12s statt 90–180s |

---

## 6. Noch zu analysieren

### 6.1 Init-Funktionen (nächste Schritte, in Reihenfolge)

| Funktion | Aufgerufen bei | Hypothese |
|---|---|---|
| `FUN_000594` | `0x000246` | Hardware-Peripherie-Init (SFR-Konfiguration) |
| `FUN_0005a0` | `0x00024a` | RAM-Init oder weitere Peripherie |
| `FUN_0005a6` | `0x000252` | Post-einit (läuft mit aktiven Interrupts) |
| `FUN_002582` | `0x00025a` | Haupt-Applikation / Task-Scheduler |

### 6.2 Aktive ISRs (Interrupt-Service-Routinen)

Alle müssen noch identifiziert werden. Wahrscheinliche Kandidaten basierend
auf ST10F269-Vektor-Nummern:

| Handler | Vektor | Wahrscheinliche Funktion |
|---|---|---|
| `thunk_FUN_022b5c` | 58 | Timer oder ADC |
| `thunk_FUN_022d24` | 59 | Timer oder ADC |
| `FUN_0040f0` | 60 | Timer oder Serial |
| `thunk_FUN_02afe4` | 62 | ADC oder CAN |
| `thunk_FUN_02a0e4` | 64 | ADC oder CAN |
| `thunk_FUN_027a5c` | 68 | Serial / K-Line |
| `thunk_FUN_02cd70` | 71 | Serial / K-Line |

---

## Legende

✅ Verifiziert — direkt aus Code oder BIN bestätigt  
⚠️ Defekt — bekanntes Problem  
❓ Offen — Hypothese, noch zu bestätigen  
[TODO] — nächste Analyse-Schritte  

---

## 7. FUN_000594 — Bus-Konfiguration ✅

**Adresse:** `0x000594` · **Aufgerufen von:** `FUN_000222` bei `0x000246`

| Register | Adresse | Wert | Bedeutung |
|---|---|---|---|
| XSFR | `0xF024` | `0x000D` | Peripherie-Register (via extr, noch zu identifizieren) |
| SYSCON | `0xFF12` | `0x0514` | External-Bus-Timing, Memory-Wait-States |

Minimale Funktion — nur 3 Instruktionen. Keine Sensor- oder Peripherie-Konfiguration.

---

## 8. Hardware-Konfiguration (aus Init-Sequenz, mit Datenblatt verifiziert)

### 8.1 XPERCON — X-Peripheral Configuration (0xF024h)

Gesetzt in **FUN_000594** via `extr` vor SYSCON. Wert: `0x000D`.

| Bit | Name | Wert | Bedeutung |
|---|---|---|---|
| 4 | RTCEN | 0 | Real Time Clock deaktiviert |
| 3 | XRAM2EN | 1 | 8KB XRAM2 aktiv (`0x00C000–0x00DFFF`) |
| 2 | XRAM1EN | 1 | 2KB XRAM1 aktiv (`0x00E000–0x00E7FF`) |
| 1 | CAN2EN | 0 | CAN2 deaktiviert |
| 0 | CAN1EN | 1 | CAN1 aktiv (`0x00EF00–0x00EFFF`) |

### 8.2 SYSCON — System Configuration (0xFF12h)

Gesetzt in **FUN_000594** nach XPERCON. Wert: `0x0514`. Danach unveränderlich.

| Bit | Name | Wert | Bedeutung |
|---|---|---|---|
| 10 | ROMEN | 1 | Interner Flash aktiv |
| 8 | CLKEN | 1 | Clock Enable |
| 4 | OWDDIS | 1 | Oscillator Watchdog deaktiviert |
| 2 | XPEN | 1 | X-Peripherals freigegeben (sperrt XPERCON!) |

### 8.3 BUSCON0 — Bus Configuration (0xFF0Ch)

Gesetzt in **FUN_0005a0**. Wert: `0x049F`.

| Bit | Name | Wert | Bedeutung |
|---|---|---|---|
| 10 | BUSACT0 | 1 | External Bus aktiv |
| 7-6 | BTYP | 10 | 16-bit Demultiplexed Bus |
| 5 | MTTC0 | 0 | 1 Memory Tristate Wait State |
| 4 | RWDC0 | 1 | Kein R/W-Delay |
| 3-0 | MCTC | 1111 | 0 Wait States (max. Geschwindigkeit) |

### 8.4 Vollständiges RAM-Layout (VERIFIZIERT)

| Bereich | Adresse | Grösse | Inhalt |
|---|---|---|---|
| XRAM2 | `0x00C000–0x00DFFF` | 8KB | Runtime-Variablen (`DAT_00dxxx`) |
| XRAM1 | `0x00E000–0x00E7FF` | 2KB | Runtime-Variablen (`DAT_00eXxx`) |
| CAN1 | `0x00EF00–0x00EFFF` | 256B | CAN1-Peripheral-Register |
| Stack | `0x00FA0C–0x00FC00` | ~500B | CPU-Stack |
| SFR/ESFR | `0xF000–0xFFFF` | 4KB | Hardware-Register |

**Konsequenz:** Alle `DAT_00dxxx` und `DAT_00eXxx` die wir in der Idle-Analyse
gesehen haben sind **XRAM-Laufzeitvariablen**, nicht Flash-Kalibrierdata.
Der Wert `0xFFFF` im BIN = nicht initialisierter Speicher.

---

## 9. WDTCON — Watchdog Timer (0xFFAEh) ✅

Gesetzt in **FUN_0005a6** nach `einit`. Wert: `0xF401`.

| Feld | Wert | Bedeutung |
|---|---|---|
| WDTREL (Bit 15-8) | 0xF4 (244) | Reload-Wert High-Byte |
| WDTIN (Bit 0) | 1 | Taktteiler fCPU/128 |
| **Timeout** | **~9.83ms** | **bei 40MHz CPU-Takt** |

Die `srvwdt`-Instruktion muss mindestens alle ~10ms ausgeführt werden.
Das ist die harte Zeitschranke für den gesamten Task-Scheduler.

### 9.1 Vollständige Init-Sequenz ✅

| Reihenfolge | Funktion | Register | Wert |
|---|---|---|---|
| 1 | `FUN_000222` | SP | `0xFC00` |
| 1 | `FUN_000222` | DPP0/1/2 | `0x01 / 0x12 / 0x13` |
| 2 | `FUN_000594` | XPERCON | `0x000D` |
| 2 | `FUN_000594` | SYSCON | `0x0514` |
| 3 | `FUN_0005a0` | BUSCON0 | `0x049F` |
| 4 | `einit` | — | Interrupts freigeben |
| 5 | `FUN_0005a6` | WDTCON | `0xF401` |
| 6 | `srvwdt` | — | Erster Watchdog-Service |
| 7 | `FUN_002582` | — | Haupt-Applikation |

---

## 10. Applikations-Bootstruktur (FUN_002582) ✅

### 10.1 Ausführungsreihenfolge

| Schritt | Funktion | Bedingung | Bedeutung |
|---|---|---|---|
| 1 | `FUN_0005ac` | immer | Init vor K-Line |
| 2 | `FUN_0007c6` | immer | Init vor K-Line |
| 3 | `FUN_0007e2` | immer | K-Line Diagnostic Handler |
| 4 | `FUN_0029c0` | K-Line aktiv | K-Line Kommunikations-Loop |
| 5 | `thunk_FUN_018046` | `DAT_00d853 == 0x10EF` | Bootloader/Flash-Modus (kein Return) |
| 6 | `FUN_003058` | Normalbetrieb | **Haupt-Task-Scheduler** |

### 10.2 Bootloader-Trigger

| Variable | Adresse | Magic-Wert | Effekt |
|---|---|---|---|
| `DAT_00d853` | XRAM2 | `0x10EF` | Springt in Flash-Programmiermodus |

### 10.3 Task-Scheduler Parameter

| Register | Wert | Bedeutung |
|---|---|---|
| r12 | `DAT_00d104` | Task-Control-Tabelle (XRAM2) |
| r13 | `0x3F10` | Task-Liste (Flash) |
| r14 | `0x0` | Startwert |

### 10.4 Noch zu analysieren (priorisiert)

| Funktion | Priorität | Grund |
|---|---|---|
| `FUN_003058` | ★★★ | Haupt-Task-Scheduler — enthält alle Tasks |
| `FUN_0005ac` | ★★ | Init vor K-Line — evtl. Timer/ADC-Config |
| `FUN_0007c6` | ★★ | Init vor K-Line — evtl. Timer/ADC-Config |
| `FUN_0029c0` | ★ | K-Line Kommunikations-Loop |
| `thunk_FUN_018046` | ★ | Bootloader/Flash-Programmiermodus |

---

## 11. FUN_003058 — Control-Block-Initialisierung ✅

**Adresse:** `0x003058`  
**Aufgerufen von:** `FUN_002582`, `FUN_0030cc`, `FUN_003aca` (×2)

Initialisiert den Task-Control-Block an `DAT_00d104` (XRAM2).

### Control-Block-Felder (DAT_00d104 + Offset)

| Offset | Wert | Bedeutung |
|---|---|---|
| `+0x04` | — | Zeiger, via FUN_0006d2 verwendet |
| `+0x1E` | 0 | initialisiert |
| `+0x2A` | 0 | initialisiert |
| `+0x32` | 0 | initialisiert |
| `+0x34` | 0 | initialisiert |
| `+0x36` | 0 | initialisiert (Byte) |
| `+0x37` | 0 | initialisiert (Byte) |
| `+0x38` | `0x28A0` | einziger Nicht-Null-Wert = 10400 |
| `+0x3A` | 0 | initialisiert |
| `+0x4A` | 0 | initialisiert |
| `+0x5A` | 0 | initialisiert (Byte) |
| `+0x5B` | 0 | initialisiert (Byte) |
| `+0x5C` | 0 | initialisiert |
| `+0x6E` | 0 | initialisiert (Byte) |
| `+0x72` | 0 | initialisiert (Byte) |
| `+0x73` | 0 | initialisiert (Byte) |
| `+0x75` | 0 | initialisiert (Byte, letztes bekanntes Feld) |

### Neue Sub-Funktionen

| Funktion | Parameter | Hypothese |
|---|---|---|
| `FUN_002f9a` | Control-Block-Zeiger | Reset/Cleanup vorheriger Zustand |
| `FUN_0006d2` | `control_block[4]` | unbekannt |
| `FUN_002f6a` | `0xFA` (250) | Timer/Delay-Funktion |

**Korrektur:** Die Haupt-Task-Loop ist **FUN_002616** (nach FUN_003058 aufgerufen).

---

## 12. FUN_002616 — Haupt-Loop ✅

**Adresse:** `0x002616`  
**Aufgerufen von:** `FUN_002582`, `FUN_0025c2`

### 12.1 Loop-Struktur

```
Init:  FUN_00098a + Flags = 0

LOOP (0x002622):
  1. UART0 poll: S0RIC.bit7? → FUN_003398  (K-Line Daten)
  2. FUN_0007d0? → FUN_002574
  3. DAT_00d863 == 0? → zurück zu 1         (Timer-Flag)
  4. DAT_00d175 != 0? → zurück zu 1
  5. Boot-State-Machine (DAT_00d17b)
  6. FUN_0026e4(DAT_00d104)                 (Task-Executor)
  → zurück zu 1
```

### 12.2 Wichtige Variablen

| Variable | Adresse | Typ | Bedeutung |
|---|---|---|---|
| `S0RIC` | SFR `0xFF8A` | Hardware-Reg | UART0 Receive Interrupt Control |
| `DAT_00d863` | XRAM2 | Byte-Flag | Timer-getriebenes Task-Trigger-Flag |
| `DAT_00d175` | XRAM2? | Byte-Flag | zweite Loop-Bedingung |
| `DAT_00d17b` | XRAM2? | Byte | Boot-State-Machine Zustand |
| `DAT_00d864` | XRAM2 | Byte-Flag | Init-Flag |

### 12.3 Identifizierte Funktionen

| Funktion | Bedeutung | Status |
|---|---|---|
| `FUN_003398` | UART0/K-Line Datenverarbeitung | ✅ identifiziert |
| `FUN_0026e4` | Haupt-Task-Executor | ★★★ nächste Priorität |
| `FUN_00098a` | Einmaliger Init vor Loop | ❓ offen |
| `FUN_0007d0` | Loop-Bedingungsprüfung | ❓ offen |
| `FUN_002574` | Reaktion auf FUN_0007d0 | ❓ offen |

### 12.4 Timer-ISR Verbindung

`DAT_00d863` wird von einem der 7 aktiven ISRs aus der IVT gesetzt.
Das ist die Verbindung zwischen Hardware-Timer und Software-Task-Scheduling.
Welcher ISR das ist, wird bei der ISR-Analyse klar.

---

## 13. Software-Architektur — Korrigiertes Gesamtbild ✅

### 13.1 Zwei parallele Ausführungsstränge

**Strang 1: Haupt-Loop (FUN_002616) — Kommunikation**
```
FUN_002616 (Haupt-Loop):
  ├── UART0/K-Line Polling → FUN_003398
  ├── Boot-State-Machine
  └── FUN_0026e4 (Buffer-Copy für PEC/DMA)
```

**Strang 2: Timer-ISRs — Motorsteuerung**
```
Timer-ISR (aus IVT):
  └── setzt DAT_00d863 (Task-Trigger)
       └── ruft periodische Tasks auf:
            ├── FUN_01963e → FUN_020f60 → Idle-Control
            ├── Sensor-Lesen (ADC)
            ├── Zündung
            └── Einspritzung
```

### 13.2 FUN_0026e4 — Buffer-Copy Utility

| Element | Wert | Bedeutung |
|---|---|---|
| Quelle | `DAT_00d63e` (XRAM2) | 6-Byte Quell-Array |
| Ziel-Zeiger | ESFR `0xF646` | PEC-Channel Schreibzeiger |
| Länge | 6 Bytes | fix |
| CB-Feld [0x5B] | `0x09` | fix gesetzt |
| CB-Feld [0x5E] | `DAT_00d648` | dynamisch aus XRAM2 |
| CB-Feld [0x1F] | `DAT_00d649` | dynamisch aus XRAM2 |

### 13.3 Nächste Prioritäten (neu bewertet)

| Funktion | Priorität | Grund |
|---|---|---|
| Timer-ISRs (IVT 0xE8–0x11C) | ★★★ | Motorsteuerungs-Regelkreise |
| `FUN_01963e` (Caller von Idle-Control) | ★★★ | wo wird Idle-Control aufgerufen? |
| `FUN_0005ac` / `FUN_0007c6` | ★★ | Init vor K-Line (Timer/ADC Config?) |
| `FUN_003398` | ★ | K-Line Protokoll-Handler |

---

## 14. Timer & ISR Architektur (VERIFIZIERT mit Datenblatt) ✅

### 14.1 Timer-Konfiguration (FUN_0007c6)

| Timer | Register | Adresse | Wert | Prescaler | Freq | Max-Periode | Status |
|---|---|---|---|---|---|---|---|
| GPT1 T4 | T4CON | FF44h | 0x0042 | /32 | 1.25 MHz | 52.4 ms | Gestoppt |
| GPT2 T6 | T6CON | FF48h | 0x0041 | /8  | 5.0 MHz  | 13.1 ms | Gestoppt |

Beide werden an anderer Stelle gestartet (Bit 7 in TxCON setzen).

### 14.2 Bootloader-Trigger (FUN_0005ac)

| Variable | Magic-Wert | Quelle | Effekt |
|---|---|---|---|
| `DAT_00d853` | `0x10EF` | Gespeicherter Wert in Puffer[14] | Bootloader-Modus |

Lesefunktionen: `FUN_002df0` (Interface-Init) + `FUN_000472` (Datenlesen, 4 Bytes).

### 14.3 Vollständige ISR-Zuordnung ✅

| IVT | Interrupt | Peripheral | Funktion | ECU-Bedeutung |
|---|---|---|---|---|
| `0x00E8` | CC26INT | CAPCOM Ch. 26 | `FUN_022b5c` | Zündzeitpunkt / Einspritzung |
| `0x00EC` | CC27INT | CAPCOM Ch. 27 | `FUN_022d24` | Zündzeitpunkt / Einspritzung |
| `0x00F0` | CC28INT | CAPCOM Ch. 28 | `FUN_0040f0` | Zündzeitpunkt / Einspritzung |
| `0x00F8` | T8INT   | CAPCOM Timer 8 | `FUN_02afe4` | **Basis-Takt / Task-Trigger** |
| `0x0100` | XP0INT  | CAN1 | `FUN_02a0e4` | CAN-Bus (Dashboard?) |
| `0x0110` | CC29INT | CAPCOM Ch. 29 | `FUN_027a5c` | Drehzahl / Sync |
| `0x011C` | S0TBINT | ASC0 TX-Buf | `FUN_02cd70` | UART0 Senden (K-Line) |

### 14.4 Vollständige Software-Architektur ✅

```
HARDWARE-EBENE:
  CAPCOM Timer 8 (T8) → läuft frei, generiert Overflow-ISR
  CAPCOM Ch. 26/27/28/29 → Output-Compare auf T8-Basis

ISR-EBENE (interrupt-gesteuert):
  FUN_02afe4 (T8 Overflow) → Task-Trigger: setzt DAT_00d863
  FUN_022b5c (CC26) → Zündung / Einspritzung Zylinder 1?
  FUN_022d24 (CC27) → Zündung / Einspritzung Zylinder 2?
  FUN_0040f0 (CC28) → Zündung / Einspritzung
  FUN_027a5c (CC29) → Drehzahl-Sync / Kurbelwellen-Position
  FUN_02a0e4 (CAN1) → CAN-Bus Handler
  FUN_02cd70 (S0TB) → UART0 Sende-Buffer

APPLIKATIONS-EBENE (Task-gesteuert via DAT_00d863):
  FUN_002616 (Haupt-Loop):
    ├── UART0 poll → FUN_003398 (K-Line Empfang)
    └── DAT_00d863 gesetzt? → FUN_026e4 (Buffer-Copy)

MOTOR-STEUERUNG (via ISR oder separatem Task-System):
  FUN_01963e → FUN_020f60 → Idle-Control
  [noch zu lokalisieren im Aufrufbaum]
```

### 14.5 Nächste Prioritäten

| Funktion | Priorität | Grund |
|---|---|---|
| `FUN_02afe4` (T8 ISR) | ★★★ | Task-Trigger — setzt DAT_00d863? |
| `FUN_022b5c` (CC26 ISR) | ★★★ | Zündung/Einspritzung — Kern der ECU |
| `FUN_027a5c` (CC29 ISR) | ★★ | Kurbelwellen-Sync / Drehzahl |
| Caller von FUN_01963e | ★★★ | wo wird Idle-Control aufgerufen? |

---

## 15. ISR-Wrapper-Muster (ST10F269 Standard) ✅

### 15.1 ISR-Struktur (gilt für alle aktiven ISRs)

```asm
ISR_Name:
  push  r0                    ; Stackpointer sichern
  mov   DAT_00fXXX, r0        ; r0 in ISR-Kontext-Area
  scxt  CP, #0xFXXX           ; Wechsel zu ISR-eigener Register-Bank
  push  DPP0
  mov   DPP0, #0x01           ; DPP0 wiederherstellen
  push  DPP2
  mov   DPP2, #0x13           ; DPP2 wiederherstellen
  calls FUN_handler            ; eigentliche ISR-Logik
  pop   DPP2
  pop   DPP0
  pop   CP
  nop
  pop   r0
  reti
```

### 15.2 ISR-Register-Banken

| ISR | CP-Adresse | Handler |
|---|---|---|
| CC26 `FUN_022b5c` | `0xF854` | `FUN_022b84` |
| T8   `FUN_02afe4` | `0xF934` | `FUN_02b00c` |

### 15.3 Idle-Control-Aufrufkette (verifiziert)

```
0x02ed9a  [Task-Dispatcher — noch zu identifizieren]
  └── FUN_01963e
        ├── FUN_02f57c  (Sensor-Vorbereitung, liest DAT_00dc75)
        └── FUN_020f60  (Idle Control Task)
              └── FUN_0211f6 + FUN_0216ce + ...
```

### 15.4 Nächste Analyse-Schritte

| Funktion | Priorität | Warum |
|---|---|---|
| `FUN_02b00c` | ★★★ | T8-ISR-Handler — setzt Task-Trigger DAT_00d863? |
| Funktion an `0x02ed9a` | ★★★ | Task-Dispatcher für alle Motor-Tasks |
| `FUN_022b84` | ★★ | CC26-Handler — Zündung/Einspritzung Logik |
| `FUN_02f57c` | ★★ | Sensor-Vorbereitung vor Idle-Control |

---

## 16. Vollständige Software-Architektur ✅

### 16.1 Boot-Entscheidung (korrigiert)

| `DAT_00d853` | Modus | Entry-Point |
|---|---|---|
| `== 0x10EF` | **NORMALBETRIEB** | `FUN_018046` (Haupt-ECU-App) |
| `!= 0x10EF` | K-Line Diagnose/Flash | `FUN_002616` (UART0-Polling) |

`0x10EF` = "Application Valid"-Flag, gesetzt nach erfolgreichem Flash.

### 16.2 T8 ISR (FUN_02b00c) ✅

Inkrementiert nur `DAT_00d91d` (T8-Tick-Counter). Kein Task-Trigger.
Zweiter ISR-Wrapper (0x02b018) mit MAC-Sicherung ruft `FUN_02b04e` auf.

### 16.3 FUN_02ebdc — Task-Dispatcher ✅

Aufgerufen von `FUN_018046:01819a`.

| Adresse | Sub-Routine | Schlüssel-Funktionen |
|---|---|---|
| `0x02ebdc` | Init | FUN_020212, FUN_01973c, FUN_018be4, FUN_022326, FUN_01d574, FUN_0270fe, FUN_02204c, FUN_0242a2, FUN_01ba0a, FUN_019728 + XRAM2-Clear |
| `0x02ec24` | Task-Gruppe 1 | FUN_019838 (×2), FUN_027e88, FUN_022fb8, FUN_0225dc, FUN_018cf6, FUN_02949a, FUN_01cdda, FUN_026540, FUN_02e0ac, FUN_02e1c2, FUN_02c684, FUN_019656, FUN_0185b4 (×3), FUN_0185d6, FUN_018410 |
| `0x02ec8a` | Task-Gruppe 2 | FUN_0185d6 (×2), FUN_030cc4, FUN_027e1c, FUN_022d82, FUN_0224b2, FUN_0225dc, **FUN_019660** (Idle-Init!), FUN_0185b4 (×3), FUN_0185d6, FUN_018410 |
| `0x02ecec` | Konditional | prüft `DAT_00d703 == 1` → FUN_01bb40 oder skip |
| `0x02ecfe` | Sensor-Check | `DAT_005d54`, `DAT_00d929`, `DAT_00dc89 == 4`, `DAT_00db5a` → FUN_02da6c, FUN_019622, FUN_019634 |
| `0x02ed52` | Kurzcheck | `DAT_00dc89 == 4` → FUN_019838 + FUN_019648 |
| `0x02ed6a` | **Haupt-Tasks** | FUN_019838, FUN_0243c6, FUN_01d51e, FUN_01f4ec + (wenn `DAT_00d703 == 1`: FUN_01fbcc, sonst: FUN_0272d2 + FUN_0263b2 + FUN_0201ac + FUN_01a1be + **FUN_01963e**) + FUN_01d24a + FUN_0257f8 + FUN_025b20 + FUN_01984c + FUN_01d48c + FUN_018410 |

### 16.4 Neue wichtige Variablen

| Variable | Adresse | Bedeutung |
|---|---|---|
| `DAT_00d853` | XRAM2 | "Application Valid"-Flag (`0x10EF` = gültig) |
| `DAT_00d91d` | XRAM2 | T8-Tick-Counter (inkrementiert bei jedem T8-Overflow) |
| `DAT_00d703` | XRAM2 | Idle-Control-Enable-Flag (`!= 1` → Idle-Control aktiv) |
| `DAT_00dc89` | XRAM2 | Sensor/Mode-Status (Wert 4 hat besondere Bedeutung) |
| `DAT_00db5a` | XRAM2 | weiterer Status |
| `DAT_00d929` | XRAM2 | Sensor-Zustand (0xFF = Initialwert) |
| `DAT_005d54` | Flash? | Konfigurations-Flag |

### 16.5 Vollständige Software-Hierarchie

```
HARDWARE:
  CAPCOM Timer 8 (T8) → Tick-Counter DAT_00d91d++
  CAPCOM CC26/27/28   → Zündung/Einspritzung (FUN_022b84 etc.)
  CAN1                → Dashboard (FUN_02a0e4)
  ASC0 TX-Buffer      → K-Line Senden (FUN_02cd70)

SOFTWARE (Normalbetrieb, DAT_00d853 = 0x10EF):
  FUN_018046 (Haupt-ECU-App)
    └── FUN_02ebdc (Task-Dispatcher)
          ├── [Init] 10 Startup-Funktionen + XRAM2-Clear
          ├── [Task-Gr. 1] Periodische Tasks
          ├── [Task-Gr. 2] + FUN_019660 → FUN_020e42 (Idle-Init)
          ├── [Konditional] DAT_00d703-basiert
          ├── [Sensor-Check] Mehrere Sensor-Konditionals
          └── [Haupt] FUN_01963e → FUN_02f57c + FUN_020f60 (Idle-Control)
                       (nur wenn DAT_00d703 != 1)

SOFTWARE (K-Line Modus, DAT_00d853 != 0x10EF):
  FUN_002616 (UART0-Polling, Diagnose/Flash)
```

### 16.6 Nächste Analyse-Schritte

| Funktion | Priorität | Grund |
|---|---|---|
| `FUN_018046` | ★★★ | Haupt-App: wie ruft sie FUN_02ebdc auf? Timing? |
| `FUN_022b84` | ★★★ | CC26-Handler: Zündung/Einspritzungs-Logik |
| `FUN_02b04e` | ★★ | zweiter T8-ISR-Handler |
| `FUN_019660` | ★★ | Idle-Init-Kette (ruft FUN_020e42 auf) |
| `DAT_00d703` | ★★ | wer setzt Idle-Control-Enable-Flag? |
