# Package plan / Paketplan

English first, German below each item. Source: specification section 14 ([SPECIFICATION.md](SPECIFICATION.md) / [SPEZIFIKATION.md](SPEZIFIKATION.md)).
Englisch zuerst, Deutsch jeweils darunter. Grundlage: Spezifikation Abschnitt 14.

## Run control / Laufsteuerung (as of 2026-10-25, overrides spec section 14 on this point)

**EN**

1. Every day at 22:00 (Europe/Berlin) a scheduled task starts a fresh session. It reads this file and `STAND.md` and determines the first open item.
2. If that item is a package, it asks the user whether this package shall be built tonight. Only an explicit "yes" received before 01:30 schedules the build for 01:30 in the same session. "No", no answer, or an answer after 01:30 means: no build that night.
3. If that item is a hold point (⏸), it does not offer a build. It asks for the test result instead. Only the user's own report closes a hold point: the report is recorded in `STAND.md`, the hold point is ticked here, both are committed.
4. A build run builds exactly one package (or the next part of one, see rule 6), then ends.

**DE**

1. Täglich um 22:00 Uhr (Europe/Berlin) startet eine geplante Aufgabe eine frische Sitzung. Sie liest diese Datei und `STAND.md` und ermittelt den ersten offenen Punkt.
2. Ist dieser Punkt ein Paket, fragt sie den Anwender, ob es in der folgenden Nacht gebaut werden soll. Nur ein ausdrückliches „Ja“, das vor 01:30 Uhr eingeht, plant den Bau für 01:30 Uhr in derselben Sitzung ein. „Nein“, keine Antwort oder eine Antwort nach 01:30 Uhr heißt: in dieser Nacht kein Bau.
3. Ist dieser Punkt ein Haltepunkt (⏸), bietet sie keinen Bau an, sondern fragt nach dem Testergebnis. Ein Haltepunkt wird nur durch die Rückmeldung des Anwenders selbst geschlossen: Die Rückmeldung wird in `STAND.md` eingetragen, der Haltepunkt hier abgehakt, beides committet.
4. Ein Baulauf baut genau ein Paket (oder den nächsten Teil davon, siehe Regel 6) und endet dann.

## Rules for every run / Regeln für jeden Lauf

**EN**

1. Take the first unticked item from top to bottom. Hold points are separate items placed after the package they test; that package itself is built normally.
2. Read only the files needed for the package. Build and test (simulator, unit tests, build).
3. Commit to `main` with the package number in the message, e.g. `P04: frame parser, echo, collision, jammer`. Author and committer: `UnImAtrIXzEr0 <146923538+UnImAtrIXzEr0@users.noreply.github.com>`. Then push.
4. Add an entry to `STAND.md` (done, open, observations) and tick the package here — only when it is complete.
5. End immediately afterwards, even if quota is left.
6. If a package becomes too large: stop at a clean point with a commit, leave the box unticked, note in `STAND.md` exactly where the next run continues.
7. A run never: builds beyond a hold point, builds more than one package, creates a release, tag or version 1.0, changes the private repository `civ-werkstatt`, puts manufacturer documents, excerpts from them, recordings, credentials or local paths into this repository.
8. `civ-werkstatt` may be read as reference (e.g. Reference Guides for device profiles), never written. Nothing is copied from it; profiles use own wording and source references as text only (title, edition, page, no link).

**DE**

1. Den ersten nicht abgehakten Punkt von oben nehmen. Haltepunkte sind eigene Punkte hinter dem Paket, das sie testen; dieses Paket selbst wird normal gebaut.
2. Nur die für das Paket nötigen Dateien lesen. Bauen und testen (Simulator, Unit-Tests, Build).
3. Commit auf `main` mit Paketnummer in der Nachricht, z. B. `P04: frame parser, echo, collision, jammer`. Autor und Committer: `UnImAtrIXzEr0 <146923538+UnImAtrIXzEr0@users.noreply.github.com>`. Danach pushen.
4. Eintrag in `STAND.md` (erledigt, offen, Auffälligkeiten) und Paket hier abhaken – nur, wenn es vollständig ist.
5. Danach sofort beenden, auch wenn Kontingent übrig ist.
6. Wird ein Paket zu groß: an sauberer Stelle mit Commit enden, Kästchen offen lassen, in `STAND.md` genau festhalten, wo der nächste Lauf weitermacht.
7. Ein Lauf tut nie: über einen Haltepunkt hinausbauen, mehr als ein Paket bauen, ein Release, einen Tag oder Version 1.0 erstellen, das private Repo `civ-werkstatt` ändern, Herstellerunterlagen, Auszüge daraus, Mitschnitte, Zugangsdaten oder lokale Pfade in dieses Repo bringen.
8. `civ-werkstatt` darf als Referenz gelesen werden (z. B. Reference Guides für Geräteprofile), nie beschrieben. Daraus wird nichts kopiert; Profile verwenden eigene Formulierungen und Quellenangaben nur als Text (Titel, Ausgabe, Seite, ohne Link).

## Packages and hold points / Pakete und Haltepunkte

Week numbers count build weeks from the start, without waiting times. / Wochen ab Start, ohne Wartezeiten.

### Week 1 / Woche 1

- [ ] **P01** – Repository foundation: LICENSE, README, .gitignore, structure, CMake
  Repo-Grundlage: LICENSE, README, .gitignore, Struktur, CMake
- [ ] **P02** – Build pipeline Windows, Linux, Raspberry Pi
  Build-Pipeline Windows, Linux, Raspberry Pi
- [ ] **P03** – Serial transport, DTR/RTS off, port selection
  Transport seriell, DTR/RTS aus, Port-Auswahl

### Week 2 / Woche 2

- [ ] **P04** – Frame parser, echo, collision, jammer
  Rahmenparser, Echo, Kollision, Jammer
- [ ] **P05** – Adapter self-test, measurement uncertainty
  Adapter-Selbsttest, Messunsicherheit
- [ ] **P06** – Bus monitor in the UI, recording
  Busmonitor in der Oberfläche, Mitschnitt
- [ ] **⏸ H1** – Intermediate test with the Pico (approx. 15 min) — waits for the user's report
  Zwischentest mit dem Pico (ca. 15 min) — wartet auf Rückmeldung des Anwenders

### Week 3 / Woche 3

- [ ] **P07** – Corrections from the intermediate test
  Korrekturen aus dem Zwischentest
- [ ] **P08** – Catalog format, data types, decoder, base profile
  Katalogformat, Datentypen, Dekoder, Basisprofil
- [ ] **P09** – IC-7610 profile part 1
  IC-7610-Profil Teil 1

### Week 4 / Woche 4

- [ ] **P10** – IC-7610 profile part 2, direct console
  IC-7610-Profil Teil 2, Direktkonsole
- [ ] **P11** – IC-9700 profile
  IC-9700-Profil
- [ ] **P12** – Sequence engine: send, expect, wait, variables
  Ablauf-Engine: Senden, Erwarten, Warten, Variablen

### Week 5 / Woche 5

- [ ] **P13** – Sequence engine: loops, sub-sequences, background, clean-up block
  Ablauf-Engine: Schleifen, Unterabläufe, Hintergrund, Aufräumblock
- [ ] **P14** – Sequence editor (UI)
  Ablauf-Editor (Oberfläche)
- [ ] **P15** – Transmit lock, release, emergency stop, watchdogs
  Sendersperre, Freigabe, Not-Aus, Wächter

### Week 6 / Woche 6

- [ ] **P16** – Snapshots, restore, report and ZIP
  Schnappschüsse, Wiederherstellung, Bericht und ZIP
- [ ] **⏸ H2** – Test pre-release on the IC-7610 — waits for the user's report
  Vorabversion am IC-7610 testen — wartet auf Rückmeldung des Anwenders
- [ ] **P17** – Corrections from the pre-release test
  Korrekturen aus dem Vorabtest
- [ ] **P18** – TCP, Bluetooth SPP, connection types, command line
  TCP, Bluetooth-SPP, Anschlussarten, Kommandozeile

### Week 7 / Woche 7

- [ ] **P19** – Simulator and fault injection
  Simulator und Fehlerinjektion
- [ ] **P20** – Capacity test
  Kapazitätstest
- [ ] **P21** – Conformance test
  Konformitätstest

### Week 8 / Woche 8

- [ ] **P22** – S-meter calibration
  S-Meter-Kalibrierung
- [ ] **P23** – Power calibration, tone output
  Leistungskalibrierung, Tonausgabe
- [ ] **P24** – Settings profiles with questionnaire and research
  Einstellungsprofile mit Fragebogen und Recherche

### Week 9–10 / Woche 9–10

- [ ] **P25** – Languages, light/dark, installer, manual
  Sprachen, Hell/Dunkel, Installer, Anleitung
- [ ] **⏸ H3** – Complete test incl. calibration — waits for the user's report
  Gesamttest inkl. Kalibrierung — wartet auf Rückmeldung des Anwenders
- [ ] **P26** – Corrections from the complete test (may take several runs; ticked when all corrections listed in `STAND.md` are done)
  Korrekturen aus dem Gesamttest (kann mehrere Läufe brauchen; abgehakt, wenn alle in `STAND.md` aufgeführten Korrekturen erledigt sind)
- [ ] **⏸ H4** – Approval of version 1.0 by the user — no run publishes a release
  Freigabe Version 1.0 durch den Anwender — kein Lauf veröffentlicht ein Release
