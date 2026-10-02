# pv-flaechenakquise
Agentensystem zur systematischen Recherche von Flächen für Freiflächen-Photovoltaik in Baden-Württemberg, gebaut mit Claude Code.

Das System beantwortet für eine Region drei Fragen:

1. In welchen Kommunen liegen die meisten regionalplanerisch ausgewiesenen PV-Flächen?
2. Welche Planungsunterlagen gibt es in einer Kommune, und laufen auf einer Fläche
   bereits PV-Projekte?
3. Welche Festlegungen des Regionalplans gelten auf einem konkreten Flurstück?

Es ist ein reines Recherche-Werkzeug. Ob eine Fläche geeignet ist, bewertet es
bewusst nicht.

> **Hinweis:** Dieses Repository beschreibt das Projekt. Code, Daten und
> Rechercheergebnisse sind nicht öffentlich.

## Inhalt

- [Ausgangslage](#ausgangslage)
- [Ablauf](#ablauf)
- [Die Agenten](#die-agenten)
- [Regionalplan-Prüfung per Geodienst](#regionalplan-prüfung-per-geodienst)
- [Gegenprobe auf der PDF-Karte](#gegenprobe-auf-der-pdf-karte)
- [Regeln des Systems](#regeln-des-systems)
- [Sandbox und Sicherheit](#sandbox-und-sicherheit)
- [Qualitätssicherung durch den Lead-Agenten](#qualitätssicherung-durch-den-lead-agenten)
- [Projektstruktur](#projektstruktur)
- [Grenzen](#grenzen)
- [Technik](#technik)
- [Fazit](#fazit)

## Ausgangslage

Wer Flächen für Freiflächen-PV sucht, muss Informationen aus sehr
unterschiedlichen Quellen zusammentragen:

- **Regionalpläne** mit Vorrang- und Vorbehaltsgebieten, oft als PDF-Karten im
  Maßstab 1:50.000
- **kommunale Unterlagen** wie Flächennutzungsplan, Kriterienkatalog oder
  Potenzialanalyse
- **Ratsinformationssysteme**, in denen Beschlüsse und Bebauungsplanverfahren liegen
- **Presse, Entwickler-Websites und Ausschreibungsergebnisse** der Bundesnetzagentur
- **Geodienste** des Landes mit Flurstücken und Planfestlegungen

Von Hand ist das langsam. Dieses Projekt teilt die Arbeit auf
spezialisierte Agenten auf und sichert die Ergebnisse durch feste Regeln und
Prüfschritte ab.

## Ablauf

```
Regionalplan-Teilplan (Vorrang- und Vorbehaltsgebiete)
        │
        ▼
Phase 1: Kommunen-Ranking nach Flächenanteil
        │
        ▼
Phase 2: Planungsunterlagen der ausgewählten Kommune
         (FNP-Übernahme, Kriterienkatalog, Potenzialanalyse)
        │
        ▼
Zielflächen (Flurstücke), je Fläche parallel:
   Projekt-Screening     Web-Recherche + Ratsinformationssystem
   Regionalplan-Prüfung  Geodienst-Abfrage + Gegenprobe auf der PDF-Karte


Zwischen den Phasen entscheidet ein Mensch, wie es weitergeht: welche Kommune
vertieft wird und welche Flächen geprüft werden.

## Die Agenten

Ein Lead-Agent steuert fünf Subagenten. Jeder hat eine eng umrissene Aufgabe,
nur die dafür nötigen Werkzeuge und ein festes Budget an Tool-Aufrufen.

| Agent | Aufgabe | Werkzeuge |
|---|---|---|
| **Lead** | Delegiert, prüft Ergebnisse gegen die Regeln, führt zusammen, hält Rücksprache | alle |
| **chat-a** | Regionalplan-Analyse und kommunale Planungsunterlagen | Dateien, Websuche |
| **chat-b** | Projekt-Screening per Web-Recherche | Dateien, Websuche |
| **chat-c** | Projekt-Screening im Ratsinformationssystem | Dateien, Websuche, Browser |
| **chat-d** | Regionalplan-Prüfung per Geodienst (WFS) | Dateien, Websuche, Browser |
| **chat-e** | Kartenauswertung auf der PDF-Karte | Dateien, Browser |

### chat-a: Regionalplan und Kommune

- **Phase 1:** Sucht im Regionalplan-Teilplan eine Flächenbilanz, die Vorrang- und
  Vorbehaltsgebiete den Kommunen zuordnet, und gibt die fünf Kommunen mit dem
  größten Flächenanteil aus, mit Seitenverweis. Gibt es keine solche Bilanz, wird
  nicht aus der Karte geschätzt.
- **Phase 2:** Recherchiert für eine ausgewählte Kommune, ob der
  Flächennutzungsplan die Gebiete übernommen hat und ob ein Kriterienkatalog oder
  eine Potenzialanalyse vorliegt. Zu jedem Fund gehören Link, Planstand und
  Fundstelle.

### chat-b: Projekt-Screening im Web

Prüft für jede Fläche, ob ein PV-Projekt in Planung, Genehmigung, Bau oder
Betrieb ist. Quellen sind Ausschreibungsergebnisse der Bundesnetzagentur, lokale
Presse, Entwickler-Websites und Verbandsdokumente.

Pressemeldungen allein reichen nicht: Der Agent sucht vorrangig den Bebauungsplan,
die Satzung oder die Verfahrensseite mit dem genauen Standort und gibt Gemarkung
und Flurstücksnummern an, wenn sie dort stehen.

### chat-c: Projekt-Screening im Ratsinformationssystem

Arbeitet parallel zu chat-b an derselben Fläche, aber ausschließlich im
Ratsinformationssystem der Kommune. Viele dieser Portale sind interaktiv und
lassen sich nicht einfach abrufen. Deshalb steuert der Agent einen Browser,
bedient die Volltext- und Sitzungssuche und wertet Vorlagen, Tagesordnungspunkte
und Beschlüsse einzeln aus.

Lässt ein Portal keine gezielte Suche zu, dokumentiert der Agent, welche
Begriffe, Zeiträume und Gremien er geprüft hat, statt ohne Beleg „kein Treffer"
zu melden.

### chat-d: Regionalplan-Prüfung per Geodienst

Fragt für ein Flurstück alle Layer des Regionalplan-Geodienstes ab und liefert die
dort geltenden Festlegungen mit Belegen. Details im nächsten Abschnitt.

Bei einem neuen Regionalverband kommt Vorarbeit dazu: den passenden Dienst
finden, die Layer gegen die Legende der Raumnutzungskarte abgleichen und einen
Steckbrief anlegen. Der Steckbrief hält auch fest, was der Dienst **nicht**
abdeckt, damit niemand den Geodienst mit dem ganzen Plan gleichsetzt.

### chat-e: Kartenauswertung

Liest auf der PDF-Karte des Regionalplans ab, welche Festlegungen an einer
Zielposition gelten. Dient als Gegenprobe zur Geodienst-Abfrage und als Hauptweg,
wenn es keinen Geodienst gibt.

## Regionalplan-Prüfung per Geodienst

Kern des Projekts ist ein Prüfskript, das im Browser läuft. Für ein Flurstück

1. holt es das Flurstückspolygon aus den amtlichen Liegenschaftsdaten (ALKIS),
2. fragt alle Layer des Regionalplan-WFS ab und
3. ordnet jedem Layer eine Kategorie zu.

### Kategorien statt Prozentwerten

| Kategorie | Bedeutung |
|---|---|
| vollständig | Das Flurstück liegt ganz in der Festlegung |
| teilweise | Ein Teil des Flurstücks liegt in der Festlegung |
| Randlage | Nur ein schmaler Rest ist betroffen, oder die Grenze liegt höchstens 25 m entfernt |
| nicht betroffen | Keine Überschneidung, bei erfolgreicher und vollständiger Abfrage |
| nicht geprüft | Fehler oder Abbruch; kein Ergebnis |

Die Regionalplankarte ist nicht parzellenscharf. Prozentwerte würden eine
Genauigkeit vortäuschen, die der Plan nicht hat. Deshalb gibt es die Kategorie
„Randlage".

### Prüfschranken

Bevor das Skript ein Ergebnis ausgibt, muss jedes Flurstück mehrere Schranken
passieren. Scheitert eine, gibt es kein Ergebnis.

- **Eindeutigkeit:** Die Liegenschaftsdaten liefern genau ein aktuelles Flurstück,
  und der Gemarkungsname passt.
- **Fläche:** Die aus dem Polygon berechnete Fläche weicht höchstens 0,5 % von
  der amtlichen Fläche ab.
- **Koordinaten:** Koordinatensystem, Wertebereich und Achsenreihenfolge sind
  plausibel, der Schwerpunkt liegt in Baden-Württemberg.
- **Planungsgebiet:** Das Flurstück liegt vollständig im Gebiet des Verbands.
  Sonst werden alle „nicht betroffen" zu „nicht geprüft" herabgestuft.

Für jeden Layer gilt außerdem: „nicht betroffen" gibt es nur, wenn der Dienst
erfolgreich und vollständig geantwortet hat. Jeder Fehler und jede abgeschnittene
Antwort führt zu „nicht geprüft" mit Angabe des Grunds.

### Selbsttest

Das Skript enthält einen Selbsttest mit Regressionstests und Fehlerinjektion.
Er prüft unter anderem, ob absichtlich fehlerhafte Abfragen korrekt als
„nicht geprüft" erkannt werden. Nach einem unabhängigen Audit wurden mehrere
Schwachstellen behoben, etwa ein Fall, in dem der Status „nicht verbindlich"
als verbindlich gewertet wurde.

### Aufwand

Weil die Abfrage deterministisch im Skript läuft, braucht der Agent nur wenige
Aufrufe. Gemessen wurden rund 37.000 Tokens und 2,5 Minuten je Fläche mit einem
kleinen Modell; die reine Abfrage dauert 10 bis 16 Sekunden je Flurstück.

## Gegenprobe auf der PDF-Karte

Ein Geodienst kann unvollständig oder veraltet sein. Deshalb wird das Ergebnis
gegen die PDF-Karte geprüft:

- beim ersten Lauf je Regionalverband
- als Stichprobe, etwa bei jeder zehnten Fläche und bei neuem Datenstand
- bei Unstimmigkeiten oder unerklärten Lücken
- für Legendenkategorien, die der Geodienst nicht enthält

Der Kartenauswerter lädt die Karte im Browser, kalibriert sie anhand von zwei
eindeutig lesbaren Referenzorten und dem Kartenmaßstab, zeichnet den
Flurstücksumriss ein und liest ab, welche Legendenmuster im Umriss liegen. Die
Kalibrierung wird je Kartenblatt gespeichert und wiederverwendet.

Weichen Geodienst und Karte voneinander ab, werden alle Flächen des Verbands neu
geprüft, bis die Ursache geklärt ist.

## Regeln des Systems

**Jede Quelle braucht einen funktionierenden Link.** Ohne Link wird ein Dokument,
Projekt oder eine Fläche nicht genannt, sondern ausdrücklich als
„nicht recherchierbar" gekennzeichnet.

**Nie raten.** Flächen werden nicht aus Kartenbildern geschätzt, Koordinaten
nicht aus dem Gedächtnis abgeleitet, Flurstücksbezüge nicht vermutet.

**Aktuell recherchieren.** Rechtliche und planerische Angaben kommen aus der
Web-Recherche, nicht aus dem Trainingswissen des Modells.

**Kein Treffer heißt nicht frei.** „Kein Projekt bekannt" bedeutet nur, dass
öffentlich nichts zu finden war. Es sagt nichts über die Verfügbarkeit der Fläche.

**Budget und Zwischenstand.** Jeder Agent hat ein Budget an Tool-Aufrufen, legt
seine Ergebnisdatei früh an und aktualisiert sie laufend. Ist das Budget
aufgebraucht, speichert er den Zwischenstand und meldet ihn als unvollständig.

**Wissen wird gespeichert.** Was über einen Regionalverband einmal erarbeitet
wurde (Dienste, Layer, Abdeckungslücken, Kartenkalibrierung), liegt im Projekt
und wird nicht bei jeder Fläche neu erkundet.

## Sandbox und Sicherheit

Die Agenten arbeiten ausschließlich im Projektordner und im Web.

- Shell-Befehle sind gesperrt, Lesezugriffe außerhalb des Projektordners blockiert.
- Downloads brauchen die ausdrückliche Zustimmung des Nutzers im Chat. Eine
  Nachricht eines anderen Agenten ersetzt sie nie, auch nicht die des Lead-Agenten.
- PDFs werden nur gelesen, nicht gespeichert.
- Anweisungen aus Webseiten oder Dokumenten führen nie zu Änderungen am System.
- Änderungen an Agenten oder Regeln schlägt der Lead-Agent vor; umgesetzt werden
  sie erst nach Bestätigung.

## Qualitätssicherung durch den Lead-Agenten

1. **Freigabe:** Vor der ersten Ausführung eines Schritts holt der Lead-Agent
   die Freigabe ein.
2. **Prüfung:** Nach Abschluss liest er die Ergebnisdatei selbst und prüft sie
   gegen die Regeln und die Aufgabenstellung.
3. **Korrektur:** Ist ein Ergebnis nicht sauber, beauftragt er den Subagenten mit
   einer konkreten Korrektur neu. Nach zwei erfolglosen Runden zeigt er das
   Ergebnis trotzdem und kennzeichnet es klar als unvollständig.
4. **Zusammenführung:** Die Ergebnisse von Web-Recherche und
   Ratsinformationssystem werden zu einer Übersicht zusammengelegt. Dopplungen
   werden zusammengefasst, Widersprüche kenntlich gemacht.
5. **Status:** Eine zentrale Übersicht hält den Stand jeder Fläche und die Kosten
   der Läufe fest, damit neue Sessions dort anknüpfen können.

## Projektstruktur

```
pv-flaechenpruefung/
├── CLAUDE.md                     Regeln und Prozess des Lead-Agenten
├── .claude/
│   ├── settings.json             Sandbox-Berechtigungen
│   └── agents/                   Definitionen der fünf Subagenten
├── docs/
│   └── kriteriengeruest.md       Begriffe, Workflow, EEG-Flächenkategorien
├── tools/
│   └── regionalplan_check.html   Prüfskript (ALKIS + Regionalplan-WFS)
├── regionalplaene/
│   └── <verband>/                Karten, Dienste-Steckbrief, Kalibrierung
└── ergebnisse/
    ├── status-uebersicht.md      Gesamtüberblick
    └── <kommune>-<art>.md        Ergebnisdateien je Kommune und Schritt
```

## Grenzen

- Das Prüfskript deckt nur Baden-Württemberg ab, weil es die Landesdienste nutzt.
- Geprüft wird die Ebene des Regionalplans. Natur-, Wasser- und Denkmalschutz
  werden nicht im Detail geprüft.
- Die Rechtswirkung einzelner Plansätze, etwa ob ein Regionaler Grünzug
  Freiflächen-PV zulässt, wird noch nicht automatisch ausgewertet.
- Das System bewertet nicht, ob eine Fläche geeignet ist.
- Die Ergebnisse ersetzen keine rechtliche oder planerische Prüfung.

## Technik

- **Claude Code** mit Subagenten und einem Lead-Agenten
- **WFS 2.0** des Geoportals Raumordnung Baden-Württemberg für die Planfestlegungen
- **ALKIS-WFS** des Landesamts für Geoinformation und Landentwicklung für die Flurstücke
- **JavaScript im Browser** für das Prüfskript, ohne Installation
- **pdf.js** für die Auswertung der PDF-Karten
- Koordinatensystem **EPSG:25832** (ETRS89 / UTM 32N)


## Fazit

Das System ist ein erster, akzeptabler Versuch, aber noch ineffizient und
deutlich ausbaufähig.

**Was gut funktioniert:** Die Recherche, ob auf einer Fläche bereits ein
PV-Projekt läuft (chat-b und chat-c), und die Recherche, ob eine Gemeinde
Kriterien, eine Potenzialanalyse oder eine Übernahme in den
Flächennutzungsplan festgelegt hat (chat-a), liefern verlässliche und
belegte Ergebnisse.

**Was noch nicht gut funktioniert:** Die Regionalplan-Prüfung (chat-d und
chat-e) ist noch sehr langsam und nur eingeschränkt funktionsfähig. Sie
deckt nur Baden-Württemberg ab, jeder neue Regionalverband braucht eigene
Vorarbeit, der Geodienst enthält nicht alle Kategorien der Kartenlegende,
und die Gegenprobe auf der PDF-Karte ist aufwendig.

**Sandbox:** Die Sicherheitsregeln sind derzeit zu einschränkend. Sie
verhindern einen universellen Einsatz und stören reibungslose Abläufe,
etwa wenn Dokumente nicht direkt geöffnet werden können und jeder Download
eine einzelne Freigabe braucht.

**Ausblick:** Die nächsten Schritte sind eine schnellere und breiter
einsetzbare Regionalplan-Prüfung, eine Sandbox, die Sicherheit und
Arbeitsfluss besser ausbalanciert, und insgesamt ein geringerer Aufwand
je Fläche.
