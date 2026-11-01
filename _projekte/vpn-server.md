---
title: VPN-Server im Eigenbetrieb
order: 4
category: Privates Projekt · Netzwerk & Security
summary: Betrieb verschiedener VPN-Lösungen (Tailscale, NetBird, WireGuard) für den sicheren Fernzugriff auf das Heimnetz.
tags: [WireGuard, Tailscale, NetBird, VPN, Netzwerk-Security]
facts:
  - label: Lösungen
    value: 3 im Vergleich
    note: Tailscale, NetBird, WireGuard
  - label: Zweck
    value: Fernzugriff
  - label: Art
    value: Privates Projekt
  - label: Status
    value: In Betrieb
---

## Ziel

Sicherer Zugriff von unterwegs auf Dienste im Heimnetz, ohne Ports ungeschützt ins Internet zu öffnen.

## Umsetzung

Ich betreibe und vergleiche verschiedene VPN-Lösungen:

| Lösung    | Merkmal                                  |
| --------- | ---------------------------------------- |
| WireGuard | Schlankes VPN-Protokoll, selbst betrieben |
| Tailscale | Mesh-VPN auf Basis von WireGuard          |
| NetBird   | Mesh-VPN auf Basis von WireGuard, selbst hostbar |

<!-- Ergänzen: Wo laufen die Server (ESXi-VM, Raspberry Pi)? Wie sind Zugriffsregeln / ACLs umgesetzt? Welche Lösung setze ich heute primär ein und warum? -->

## Erkenntnisse

<!-- Ergänzen: Vor- und Nachteile der Lösungen aus eigener Erfahrung. -->
