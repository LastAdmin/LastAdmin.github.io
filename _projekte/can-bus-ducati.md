---
title: CAN-Bus-Reverse-Engineering & ECU-Analyse
order: 2
category: Privates Projekt · Embedded
summary: Analyse und Anpassung der Motorsteuerung einer Ducati, mit einem selbst gebauten Datenlogger auf Basis von Raspberry Pi Pico und MCP2515-CAN-Modul.
tags: [CAN-Bus, Raspberry Pi Pico, MCP2515, Reverse Engineering, ECU]
facts:
  - label: Hardware
    value: Raspberry Pi Pico
    note: + MCP2515-CAN-Modul
  - label: Fahrzeug
    value: Ducati
  - label: Art
    value: Privates Projekt
  - label: Schwerpunkt
    value: Datenanalyse
---

## Ausgangslage

Ich wollte verstehen, welche Daten die Motorsteuerung (ECU) meiner Ducati über den CAN-Bus austauscht, und die Steuerung gezielt anpassen.

<!-- Ergänzen: Modell / Baujahr, Auslöser für das Projekt. -->

## Umsetzung

- Eigenbau eines Datenloggers mit einem **Raspberry Pi Pico** und einem **MCP2515-CAN-Modul**
- Aufzeichnung der CAN-Nachrichten während der Fahrt
- Auswertung der Fahrzeugdaten und Zuordnung der Nachrichten-IDs

<!-- Ergänzen: Firmware (MicroPython / C?), Auswertung (z.B. Python, SavvyCAN), Stromversorgung und Gehäuse, Fotos des Aufbaus: ![Datenlogger](/assets/img/projekte/can-logger.jpg) -->

## Erkenntnisse

<!-- Ergänzen: Was konnte entschlüsselt werden? Was wurde an der ECU angepasst? Was habe ich gelernt (Bus-Protokolle, Hardware-Debugging, systematisches Vorgehen)? -->

Das Projekt verbindet Hardware, Programmierung und systematische Analyse – dieselbe Herangehensweise, die ich bei der Fehlersuche in IT-Systemen anwende.
