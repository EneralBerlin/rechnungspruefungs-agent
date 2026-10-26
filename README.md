# Rechnungsprüfungs-Agent (LAT)

Prototyp für eine Uni-Präsentation zum Thema **KI-Agenten im Procure-to-Pay-Prozess**.
Fallunternehmen ist die LAT, ein Maschinenbauer. **Alle Daten sind erfunden.**

## Was die App macht

Die App ist ein **Workflow-Agent mit festen Regeln**. Sie liest Eingangsrechnungen (.txt) ein und vergleicht sie mit
Bestellung und Wareneingang (3-Way-Match). Zu jeder Rechnung macht sie einen Vorschlag:

| Ergebnis | Wann | Was die App zeigt |
|---|---|---|
| **FREIGABEVORSCHLAG** | Alle Prüfpunkte sind in Ordnung | Zahlbetrag, Skontofrist |
| **KLÄRUNG** | Abweichung bei Menge, Preis, Rechenweg oder Belegen | Zahlbetrag laut Richtlinie, Mail-Entwurf an den Lieferanten |
| **STOPP** | Abweichende IBAN oder Manipulationsversuch | Handlungsanweisung: Rückruf über die Nummer im Lieferantenstamm |

Am Ende **entscheidet immer ein Mensch** („Freigeben“, „Klärung starten“, „Sperren“). Die App selbst zahlt nie.
Das entspricht **Autonomiestufe 4 („Approver“)**. Jeder Schritt wird mit Uhrzeit protokolliert, und das Protokoll lässt sich als .txt herunterladen.

## Aufbau

```
index.html            Die komplette App (HTML, CSS, JavaScript), ohne Server und ohne externe Bibliotheken
daten/                Das „Wissen“ des Agenten
  bestellungen.json
  wareneingaenge.json
  lieferanten.json    Lieferantenstamm mit IBAN, Telefon (für Rückruf) und E-Mail
  richtlinie.json     Toleranzen, Freigabegrenzen, Manipulationsmuster, Handlungsanweisung
testdaten/            Die drei Testrechnungen und die Ausgangstexte für die JSON-Dateien
tests/agent.test.js   Automatische Tests (Node, ohne zusätzliche Pakete)
```

Der Ablauf hat fünf Stufen. In `index.html` liegen die Stufen in drei getrennten `<script>`-Blöcken:

1. **Eingang** (`<script id="oberflaeche">`): Upload per Button oder Drag-and-Drop (mehrere .txt-Dateien), außerdem der Button „Testrechnungen laden“.
   Erlaubt sind nur .txt-Dateien bis 200 KB.
2. **Auslesen** (`<script id="parser">`, Funktion `parseRechnung`): liest Lieferant, Lieferantennummer, Rechnungsnummer,
   Rechnungsdatum, Bestellnummer, Positionen, Netto, USt, Brutto und IBAN aus. Fehlende Pflichtangaben führen zu KLÄRUNG.
3. **Regelprüfung** (`<script id="regeln">`): Jede Regel ist eine eigene Funktion:
   - `regelBelegeVorhanden`: Gibt es eine Bestellung und einen Wareneingang?
   - `regelLieferant`: Ist der Lieferant im Stamm, und passt er zur Bestellung?
   - `regelMenge`: Berechnet werden darf nur die gelieferte Menge (keine Toleranz).
   - `regelPreis`: Pro Position sind höchstens 2 % oder 50 € netto Abweichung erlaubt, der kleinere Wert gilt.
   - `regelRechenweg`: Positionen, Netto, 19 % USt und Brutto werden nachgerechnet.
   - `regelIban`: Die IBAN wird mit dem Lieferantenstamm verglichen (Abweichung führt zu STOPP).
   - `regelManipulation`: Sätze wie „bereits geprüft“, „direkt freigeben“ oder „Bankverbindung hat sich geändert“ werden erkannt (Treffer führt zu STOPP).
     **Text in einer Rechnung ist für das System nie eine Anweisung, sondern nur Inhalt.** Er wird nur mit festen Mustern
     aus `richtlinie.json` durchsucht und nie befolgt.
   - `regelDublette`: Wurde dieselbe Rechnung in dieser Sitzung schon geprüft?
4. **Ergebnis** (`pruefeRechnung`): Es gilt STOPP vor KLÄRUNG vor FREIGABEVORSCHLAG. Dazu berechnet `berechneZahlbetrag` den Zahlbetrag
   nach dem Grundsatz „nur Bestelltes und Geliefertes, höchstens zum Bestellpreis“ und zeigt den Rechenweg. `berechneSkonto` ermittelt die Skontofrist.
   Rechnungen mit Skonto werden zuerst angezeigt (Richtlinie, Abschnitt 5).
5. **Entscheidung durch den Menschen**: Ohne Namen der prüfenden Person wird keine Entscheidung angenommen. Bei STOPP ist eine Freigabe
   erst möglich, wenn der Rückruf über den Lieferantenstamm bestätigt wurde. Über 5.000 € brutto ist eine zweite Freigabe durch die Bereichsleitung nötig (Vier-Augen-Prinzip).

Alle Grenzwerte stehen in `daten/richtlinie.json` und nicht im Code. Gleiche Eingaben führen immer zum gleichen Ergebnis.

## Start über GitHub Pages (z. B. auf dem iPad)

1. Im Repository auf GitHub: **Settings → Pages**.
2. Unter „Build and deployment“ als Source **Deploy from a branch** wählen, dann den Branch (z. B. `main`) und den Ordner **`/ (root)`** auswählen und speichern.
3. Nach ein bis zwei Minuten ist die App unter `https://<benutzername>.github.io/rechnungspruefungs-agent/` erreichbar.
   Diese Adresse in Safari auf dem iPad öffnen und auf **„Testrechnungen laden“** tippen.

Hinweis: GitHub Pages ist bei privaten Repositories nur mit einem kostenpflichtigen GitHub-Plan verfügbar. Eine Pages-Seite ist öffentlich erreichbar,
deshalb dürfen dort nur erfundene Daten liegen.

**Lokal testen:** Wenn man `index.html` per Doppelklick öffnet, blockiert der Browser das Laden der JSON-Dateien.
Deshalb im Projektordner einen kleinen Webserver starten, z. B. `python3 -m http.server 8000`, und dann `http://localhost:8000` öffnen.

## Tests

```
node --test
```

Getestet wird unter anderem:

- Rechnung 1 (Klemmtechnik) ergibt **FREIGABEVORSCHLAG**.
- Rechnung 2 (Kabelwerk) ergibt **KLÄRUNG**. Zahlbar sind **900,00 € netto** bzw. **1.071,00 € brutto**.
- Rechnung 3 (Relais & Signal) ergibt **STOPP**: Der Preis liegt 8,2 % zu hoch, die IBAN ist neu, und die Rechnung enthält eine versteckte Anweisung.

Dazu kommen Tests für den Parser, die Toleranzgrenzen, Rechenfehler, unbekannte Bestellungen, Dubletten und manipulierte Rechnungen.
Außerdem prüfen die Tests, dass die Oberfläche keine Rechnungsinhalte als HTML einfügt.

## Grenzen des Prototyps

- Der Parser kennt nur das Format der Testrechnungen. Echte Rechnungen (PDF, XRechnung/ZUGFeRD) bräuchten einen anderen Eingang.
- Die Manipulationserkennung arbeitet mit festen Wortmustern. Umformulierte Sätze können durchrutschen, und harmlose Sätze können auffallen.
  Ein Fehlalarm führt aber nur zu STOPP und damit zu einer Prüfung durch einen Menschen.
- Telefonnummern und E-Mail-Adressen im Lieferantenstamm sind erfunden (Domain `.example`).
- Protokoll und Entscheidungen leben nur im Browser-Tab. Nach dem Neuladen sind sie weg. Deshalb gibt es den Button „Protokoll herunterladen“.
