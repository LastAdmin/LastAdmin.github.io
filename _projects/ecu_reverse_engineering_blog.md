---
title: Ducati 848 ECU Reverse Engineering
category: Motorrad
date: 2026-06-01
status: Laufend
summary: Reverse Engineering der Ducati 848 Superbike ECU - Magneti Marelli IAW5AM HW610
tech:
  - Elektronik
  - Reverse Engineering
  - Ghidra
# cover: /assets/images/projects/motorrad.jpg  # Bild in assets/images/projects/ ablegen und einkommentieren
---

# Was im Steuergerät meiner Ducati 848 wirklich steht

Es fing mit einem Ärgernis an, das jeder Twin-Fahrer kennt: ein zickiger Leerlauf und ein Knallen im Ansaugtrakt, kalt und beim Gaswegnehmen. Werkstattantwort: „Steht halt so im Mapping." Mir hat das nicht gereicht – also habe ich das Steuergerät auseinandergenommen. Nicht mechanisch, sondern Byte für Byte.

Im 848er sitzt eine Magneti Marelli IAW 5AM (HW610) auf Basis eines ST10F269 – ein 16-Bit-Controller aus der C166/ST10-Familie mit segmentiertem Speicher. Heißt: Jede Tabellenadresse muss man von Hand umrechnen, bevor man überhaupt weiß, wo man hinschaut. Auf die fertigen XDF-Maps konnte ich mich nicht verlassen; die sind selbst reverse-engineered, lückenhaft und an den entscheidenden Stellen schlicht falsch.

Was beim Zerlegen herauskam, hat mich überrascht. Das angeblich „stockmäßige" Mapping ist an mehreren Stellen kaputt: Die Zündkarte steht im Leerlauf auf 0° vor OT, die Temperaturachse ist nicht monoton (was die Interpolation im Kaltlauf bricht), und der Kaltstart-Timer läuft nach gut 12 statt 90–180 Sekunden ab. Kein einzelner Fehler also, sondern ein Zusammenspiel mehrerer fehlerhafter Kalibrierwerte.

Das ist die Kurzfassung. Wie der Aufrufgraph der Leerlaufregelung aussieht, wie man RAM von Flash unterscheidet und wie ich die einzelnen Tabellen Schritt für Schritt aufgespürt habe, steht in der technischen Doku:

- **[Entdeckungen – chronologisch](ducati_848_ecu/ducati_848_ecu_entdeckungen.md)** – der Weg durch Ghidra, Fund für Fund
- **[Technische Referenz](ducati_848_ecu/ducati_848_ecu_technisch.md)** – Adressen, Funktionen, Speicherlayout

Wer es genau wissen will, findet dort alles. Die eigentliche Erkenntnis bleibt aber auch ohne Hex-Dump bestehen: Eine moderne Maschine ist nur so lange eine Blackbox, wie man sie als eine behandelt.
