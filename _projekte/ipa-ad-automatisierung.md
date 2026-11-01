# Active Directory automatisieren: Meine IPA

*Ein PowerShell-Tool, das Benutzerkonten im Active Directory selbstständig anlegt, anpasst und deaktiviert, zuerst vollautomatisch aus SharePoint, später auch per CSV-Import für den Support.*

---

## Worum geht es?

Jedes Unternehmen mit einer Windows-Infrastruktur verwaltet seine Mitarbeitenden im **Active Directory (AD)**. Dort hat jede Person ein Benutzerkonto, über das sie sich anmeldet, Mails erhält und Zugriff auf Laufwerke, Programme und Gruppen bekommt.

Bei jedem Eintritt, jedem Abteilungswechsel und jedem Austritt muss dieses Konto gepflegt werden. Von Hand ist das mühsam und fehleranfällig: Ein Tippfehler im Namen, eine falsche Abteilung oder eine vergessene Gruppenmitgliedschaft fallen oft erst auf, wenn die neue Person am ersten Arbeitstag nicht arbeiten kann. Noch heikler ist der umgekehrte Fall, wenn ein Konto nach dem Austritt aktiv bleibt.

In meiner **IPA** (Individuelle Praktische Arbeit, die Abschlussarbeit der Informatik-Lehre in der Schweiz) habe ich ein Tool entwickelt, das diese drei Vorgänge automatisiert:

| Vorgang | Was im AD passiert |
|---|---|
| **Eintritt** | Neues Benutzerkonto anlegen, in die richtige Organisationseinheit (OU) einordnen, Gruppen zuweisen, Initialpasswort setzen |
| **Mutation** | Daten des bestehenden Kontos ändern, bei einem Abteilungswechsel in die neue OU verschieben, Gruppen hinzufügen oder entfernen |
| **Austritt** | Konto aus allen Gruppen entfernen, deaktivieren und mit dem Deaktivierungsdatum versehen |

---

## Ausgangslage: Die Daten liegen in SharePoint

Die Personendaten und Änderungsanträge wurden im Betrieb bereits in **SharePoint-Listen** erfasst. Ein Antrag durchläuft dort einen Genehmigungsprozess, bevor er umgesetzt werden darf. Die Daten sind auf mehrere Listen verteilt, die sich gegenseitig referenzieren, ähnlich wie Tabellen in einer Datenbank:

- **Person**: Name, Geburtsdatum, Funktion, E-Mail, zugehörige Einheit
- **Change**: der eigentliche Antrag (Eintritt, Mutation oder Austritt), Fälligkeitsdatum, Genehmigungsstatus, zu vergebende Gruppen
- **Unit**: Organisationseinheit mit Standort, Abteilung und Adresse
- **Feature** und **Roles**: Zusatzinformationen wie Postfach-Features oder Rollen mit ihren Berechtigungen

Bis dahin wurden diese Anträge von Hand gelesen und im AD umgesetzt. Mein Auftrag war, diesen Schritt zu automatisieren.

---

## Version 1: Vollautomatisch (Stand der IPA)

Während der IPA war das Tool für den **unbeaufsichtigten Betrieb auf einem Server** gebaut. Es läuft als geplanter Task, holt sich die Daten selbst und arbeitet alle fälligen Anträge ab, ohne dass jemand eingreifen muss.

Der Ablauf besteht aus vier Schritten:

```
 ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐    ┌──────────────┐
 │  1. Laden    │ →  │ 2. Zusammenführen│ →  │ 3. Prüfen    │ →  │ 4. Ausführen │
 │  SharePoint  │    │  temporäre       │    │  fehlerhafte │    │  Eintritt /  │
 │  Listen      │    │  Task-Liste      │    │  Tasks raus  │    │  Mutation /  │
 │              │    │                  │    │              │    │  Austritt    │
 └──────────────┘    └──────────────────┘    └──────────────┘    └──────────────┘
```

### 1. Daten laden

Das Script verbindet sich über das Modul **PnP.PowerShell** mit SharePoint Online. Die Anmeldung läuft „App-Only“ über ein Zertifikat. Es gibt also kein gespeichertes Benutzerpasswort, und die Verbindung ist an eine registrierte Anwendung in Microsoft Entra ID gebunden. Anschliessend werden alle fünf Listen gelesen.

### 2. Eine temporäre Task-Liste aufbauen

Aus den Rohdaten baut das Tool eine **temporäre Liste mit allen Aufgaben**, die verarbeitet werden sollen. Dafür nimmt es jeden offenen Antrag aus der Change-Liste als Ausgangspunkt und löst seine Verweise auf: Der Antrag zeigt auf eine Person, die Person auf eine Einheit, die Einheit liefert Adresse und Abteilung. Am Ende steht pro Antrag ein einzelnes, flaches Objekt, das alle nötigen Informationen enthält.

Dabei werden auch SharePoint-spezifische Feldtypen in normale Werte übersetzt. Ein Nachschlagefeld liefert zum Beispiel nicht direkt den Text, sondern ein Objekt mit ID und Anzeigewert, und ein Taxonomiefeld liefert ein Label.

### 3. Jede Aufgabe prüfen

Bevor etwas im AD verändert wird, durchläuft jede Aufgabe eine Reihe von Prüfungen. Nur fehlerfreie Aufgaben gehen weiter:

- **Genehmigt?** Nur genehmigte Anträge werden umgesetzt.
- **Noch offen?** Bereits erledigte Anträge werden übersprungen.
- **Fällig?** Ein Eintritt am 1. des Folgemonats wird erst an diesem Tag ausgeführt.
- **Vollständig?** Pflichtfelder wie Vorname, Nachname, Einheit und E-Mail müssen gefüllt sein.
- **Gültige Zeichen?** Namen, E-Mail und Funktion werden gegen eine Whitelist erlaubter Zeichen geprüft. Das AD reagiert je nach Attribut unterschiedlich auf Sonderzeichen, deshalb ist die Prüfung bewusst streng.
- **Gültige Länge?** Der Anmeldename (`sAMAccountName`) darf im AD höchstens 20 Zeichen lang sein und wird bei Bedarf gekürzt.

Zusätzlich erzeugt das Tool die abgeleiteten Felder: Anzeigename, Benutzerprinzipalname (UPN, z.B. `vorname.nachname@domain`), Anmeldename und, falls keines mitgegeben wurde, ein Initialpasswort.

Fehlerhafte Aufgaben werden nicht stillschweigend verworfen. Sie landen im Log, und im automatischen Betrieb ging zusätzlich eine **Benachrichtigung an einen Teams-Kanal** (über einen Power-Automate-Webhook), damit der Support eingreifen konnte.

### 4. Im AD ausführen

Je nach Art des Antrags wird das passende Modul aufgerufen:

- **Eintritt**: Zuerst prüft das Tool, ob es den Benutzer schon gibt. Falls ja, bricht es ab, denn ob es sich um einen Wiedereintritt oder eine neue Person mit gleichem Namen handelt, muss ein Mensch entscheiden. Danach prüft es, ob die Ziel-OU existiert. Erst dann wird das Konto mit allen Attributen angelegt (Name, Funktion, Vorgesetzte Person, Abteilung, Standort, Adresse). Bei der ersten Anmeldung muss das Passwort geändert werden.
- **Mutation**: Das bestehende Konto wird angepasst. Ändert sich die Einheit, wird es in die neue OU verschoben, und Gruppen werden hinzugefügt oder entfernt.
- **Austritt**: Das Konto wird aus allen Gruppen entfernt, deaktiviert und erhält in der Beschreibung das Deaktivierungsdatum. Ist es bereits deaktiviert, passiert nichts. Der Vorgang lässt sich also gefahrlos wiederholen.

### Das Initialpasswort

Ein neues Konto braucht ein Startpasswort, das die Person sicher erreicht, ohne dass es irgendwo im Klartext abgelegt wird. Das Tool erzeugt es deshalb **deterministisch**: Aus einem geheimen Schlüssel und einigen Personendaten wird ein SHA-256-Hash berechnet, aus dem das Passwort gebildet wird. Ein anderes System, das denselben Schlüssel kennt, kann dasselbe Passwort unabhängig berechnen, ohne dass es je übertragen werden muss. Schlecht unterscheidbare Zeichen wie `l`/`1` oder `0`/`O` werden dabei ersetzt, damit sich das Passwort fehlerfrei abtippen lässt.

---

## Version 2: Weiterentwicklung nach der IPA

Nach der IPA hat sich gezeigt, dass der Support das Tool auch **direkt** nutzen wollte, zum Beispiel für einen einzelnen Eintritt, der nicht über SharePoint erfasst wurde, oder um einen Vorgang gezielt anzustossen. Ich habe das Tool deshalb um einen **manuellen Modus** erweitert:

1. Der Supporter startet das Tool per Doppelklick auf eine Startdatei.
2. Ein Dateidialog öffnet sich, und er wählt eine **CSV-Datei** mit den Daten der Person aus.
3. Die Daten werden genau wie im automatischen Modus zusammengeführt und geprüft.
4. Das Tool fragt nach den Admin-Anmeldedaten und führt den Vorgang aus.
5. Alle Ausgaben erscheinen live in der Konsole und werden zusätzlich in eine Log-Datei geschrieben.

Damit gibt es zwei Wege zu denselben Aktionen:

```
  Automatisch (Server)              Manuell (Supporter)
  ────────────────────              ───────────────────
  SharePoint-Listen                 CSV-Datei
         │                                 │
         └────────────┐       ┌────────────┘
                      ▼       ▼
                ┌───────────────────┐
                │      Mapping      │   ← übersetzt beide Quellen
                └─────────┬─────────┘     in ein einheitliches Format
                          ▼
                   Prüfen → Ausführen     ← identischer Code für beide Wege
```

### Das Problem: Zwei Quellen, zwei Formate

Die CSV-Datei und SharePoint enthalten dieselben Informationen, aber in unterschiedlicher Form:

| Information | SharePoint | CSV |
|---|---|---|
| Nachname | `Lastname` | `Surname` |
| Vorname | `Firstname` | `GivenName` |
| E-Mail | `EmailOrg` | `E-Mail` |
| Gruppen | Nachschlagefeld mit Objekten | Text, mit `;` getrennt |
| Einheit | Verweis auf eine andere Liste | direkt als Code in der Zeile |
| Datumsformat | US-Format | je nach Excel-Einstellung unterschiedlich |

Die naheliegende Lösung wäre gewesen, für die CSV einfach ein zweites Script zu schreiben. Dann hätte aber jede Korrektur an der Prüf- oder AD-Logik doppelt gemacht werden müssen, und früher oder später hätten sich die beiden Versionen auseinanderentwickelt.

### Die Lösung: Ein Mapping

Stattdessen habe ich ein **Mapping** gebaut, eine zentrale Konfigurationsdatei, die für jede Datenquelle beschreibt, *wo* ein Feld liegt, *wie* es ausgelesen wird und *welche Regeln* dafür gelten. Der Rest des Programms arbeitet ausschliesslich mit einheitlichen, internen Feldnamen wie `PersonLastname` oder `UnitCode` und weiss nicht, woher die Daten ursprünglich kommen.

Ein Ausschnitt (vereinfacht):

```powershell
# SharePoint: Wo liegt das Feld und welchen Typ hat es?
Person = @{
    Lastname = @{ Field = "Lastname"; Type = "Direct" }
    Unit     = @{ Field = "Unit";     Type = "ID"; isIndex = $true
                  Pointer = @{ Table = "Unit"; Key = "ID" } }
    JobTitle = @{ Field = "JobTitle"; Type = "Taxonomy" }
}

# CSV: Gleiches internes Feld, anderer Spaltenname
CSV = @{
    PersonLastname   = @{ Field = "Surname";    Mandatory = $true; CharCheck = $true }
    ChangePrincipals = @{ Field = "Principals"; Mandatory = $true; MultiValue = $true }
}
```

Das Mapping legt dabei mehrere Dinge fest:

- **Feldname in der Quelle**: `Surname` in der CSV wird zu `PersonLastname`.
- **Feldtyp**: Soll der Wert direkt übernommen, aus einem Nachschlage- oder Taxonomiefeld gelesen oder als Verweis auf eine andere Liste aufgelöst werden?
- **Verknüpfungen**: Über `isIndex` und `Pointer` weiss das Tool, welche Liste über welchen Schlüssel mit welcher anderen verbunden ist. So kann es die Verweise in zwei Ebenen automatisch auflösen (Antrag → Person → Einheit).
- **Prüfregeln pro Feld**: Ob ein Feld Pflicht ist, auf Sonderzeichen geprüft wird, einen bestimmten Wert haben muss (z.B. „genehmigt“) oder gerade *leer* sein muss (ein erledigter Antrag hat ein Abschlussdatum und wird übersprungen).
- **Sonderfälle**: Felder, die eine eigene Logik brauchen, etwa ein Geburtsdatum, das in verschiedenen Formaten ankommen kann, gehen an einen separaten Ausnahme-Handler.

Für das Programm bedeutet das:

- Kommt ein neues Feld dazu, muss es nur im Mapping eingetragen werden.
- Heisst eine Spalte in SharePoint oder in der CSV plötzlich anders, wird eine einzige Zeile angepasst.
- Prüfungen und AD-Aktionen existieren genau einmal und funktionieren für beide Quellen.
- Eine weitere Datenquelle, zum Beispiel ein HR-System, liesse sich anbinden, indem man ein weiteres Mapping ergänzt, ohne die Kernlogik anzufassen.

---

## Aktueller Stand

Mit dem Mapping wurde die interne Datenstruktur grundlegend umgebaut. Alle Module mussten deshalb auf die neue Architektur umgestellt werden. Bis zu meinem Austritt aus dem Betrieb war das so weit:

| Funktion | Stand |
|---|---|
| Eintritt im manuellen Modus (CSV) | Fertig umgebaut und im Einsatz |
| Mapping und Validierung für SharePoint und CSV | Fertig umgebaut |
| Mutation und Austritt | In der IPA-Version umgesetzt, aber noch nicht auf die neue Architektur umgebaut |
| Vollautomatischer Betrieb auf dem Server | Technisch vorbereitet, aber mit der neuen Architektur noch nicht abschliessend getestet |

Dass nicht alles fertig wurde, lag nicht an der Technik. Wohin sich der Prozess künftig entwickeln sollte, war intern noch offen, ebenso, welche Datenquelle langfristig die führende sein würde. Statt Funktionen auf eine Richtung hin auszubauen, die sich später wieder ändern könnte, lag der Fokus auf dem Teil, der im Alltag sofort gebraucht wurde: dem Eintritt per CSV. Dank des Mappings ist die Grundlage für die übrigen Module gelegt. Sie müssen nur noch auf die einheitlichen Feldnamen umgestellt werden, eine neue Datenanbindung brauchen sie nicht.

---

## Sicherheit und Betrieb

Ein Tool, das mit Admin-Rechten Benutzerkonten verändert, muss vorsichtig gebaut sein. Einige Massnahmen:

- **Module einzeln schaltbar**: Anlegen, Mutieren, Deaktivieren sowie der automatische und der manuelle Modus lassen sich in der Konfiguration einzeln ein- und ausschalten. So kann ein Modul, das noch nicht fertig getestet ist, nicht versehentlich laufen.
- **Prüfen vor Handeln**: Existiert der Benutzer schon? Gibt es die Ziel-OU? Im Zweifel bricht das Tool ab und meldet den Fall, statt zu raten.
- **Signierte Scripts**: Alle PowerShell-Dateien werden mit einem Code-Signing-Zertifikat signiert, damit nur unveränderter Code ausgeführt wird.
- **Keine gespeicherten Admin-Passwörter**: Im manuellen Modus werden die Anmeldedaten bei jedem Lauf abgefragt, im automatischen Modus läuft die SharePoint-Anbindung über ein Zertifikat.
- **Nachvollziehbares Logging**: Jeder Lauf schreibt ein eigenes Log mit Zeitstempel. Es gibt mehrere Debug-Stufen, wobei die detaillierteste auch sensible Werte zeigt und deshalb nur bei der Fehlersuche aktiviert wird.
- **Versionierung**: Das Repository folgt einem eigenen Versionsschema mit Alpha-, Beta- und Stable-Releases und einer gepflegten Änderungshistorie.

---

## Technologien

- **PowerShell 7** als Sprache und Laufzeitumgebung
- **ActiveDirectory-Modul** für `New-ADUser`, `Set-ADUser`, `Move-ADObject`, Gruppenverwaltung usw.
- **PnP.PowerShell** für den Zugriff auf SharePoint Online
- **Microsoft Entra ID** App-Registrierung mit zertifikatsbasierter Anmeldung
- **Power Automate / Microsoft Teams** für Benachrichtigungen im automatischen Betrieb
- **Authenticode** Code-Signing für die Scripts
- **Git / GitHub** für Versionsverwaltung und Dokumentation

---

## Was ich daraus mitnehme

**Daten sind das eigentliche Problem.** Die AD-Befehle selbst sind schnell geschrieben. Den grössten Teil der Arbeit machte der Umgang mit Daten aus, die unvollständig sind, Sonderzeichen enthalten, in verschiedenen Formaten ankommen oder auf mehrere Listen verteilt sind. Die Validierung ist deshalb der umfangreichste Teil des Tools.

**Lieber abbrechen als raten.** Bei Benutzerkonten ist ein falsch angelegtes Konto schlimmer als ein fehlendes. Das Tool ist deshalb so gebaut, dass es im Zweifel stoppt und einen Menschen informiert.

**Trennen, was sich ändert, von dem, was gleich bleibt.** Das Mapping war ursprünglich nicht geplant. Es entstand erst, als eine zweite Datenquelle dazukam. Es hat sich aber als die wichtigste Designentscheidung herausgestellt: Die Herkunft der Daten ist Konfiguration, die Verarbeitung ist Code, und so bleibt das Tool erweiterbar, ohne dass es auseinanderfällt.

**Ein Tool lebt nach der Abgabe weiter.** Die IPA war ein Meilenstein, nicht das Ende. Erst im Alltag zeigte sich, wie die Supporter das Tool tatsächlich nutzen wollten, und die wichtigste Erweiterung kam danach.
