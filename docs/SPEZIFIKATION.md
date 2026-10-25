# civ-tester4icom – Spezifikation Version 1

**English version: [SPECIFICATION.md](SPECIFICATION.md)**

Stand 25.10.2026 · freigegeben. Dies ist die freigegebene Originalfassung; die englische Fassung ist eine Übersetzung. Bei Abweichungen gilt diese deutsche Fassung.

## 1 Zweck und Abgrenzung

civ-tester4icom ist ein offenes Mess- und Testprogramm für Icom-CI-V-Busse und Geräte mit CI-V-Schnittstelle, gedacht als Werkzeug für Funkamateure und Entwickler von Eigenbau-Zubehör. Anders als der CI-V Prüfer enthält es keine fest eingebauten Messabläufe: Der Anwender stellt seine Abläufe in der Oberfläche selbst zusammen, speichert sie und führt sie aus. Mitgelieferte Abläufe sind nur Vorlagen.

**Kernfunktionen Version 1**

- Bus mithören, dekodieren, mitschneiden, offline auswerten
- Abläufe aus Katalogbefehlen und eigenen Befehlen bauen, mit Antwortfenstern, Timings und Bewertung
- Messunsicherheit selbst bestimmen und bei jedem Zeitwert angeben
- Kapazitätstest, Konformitätstest, Simulator mit Fehlerinjektion
- S-Meter- und Leistungskalibrierung als geführte Abläufe
- Einstellungsprofile (z. B. FT8) mit Fragebogen
- Geräteeinstellungen sichern und wiederherstellen

**Nicht Ziel:** vollständige Radio-Bedienoberfläche, Abgleich von Geräten im Service-/Abgleichmodus, analoge Pegel- oder Flankenmessung am Bus.

**Lizenz:** MIT, Vermerk „Copyright (c) 2026 UnImAtrIXzEr0“. Rufzeichen und Name des Autors stehen nicht als Voreinstellung im Programm; das Feld „Bediener“ bleibt leer, bis der Anwender es ausfüllt.

**Haftung:** Bereitstellung ohne Gewährleistung (Lizenztext). Zusätzlich Hinweis in README und Programm: Abläufe mit sendeauslösenden Befehlen nur auf eigene Verantwortung und nur an einer Kunstantenne.

**Unabhängigkeits- und Markenhinweis** (README oben, „Über“-Dialog, Fußzeile jedes Berichts, Anleitung; keine Icom-Logos oder Icom-Gestaltung):

> civ-tester4icom ist ein unabhängiges, offenes Projekt von Funkamateuren. Es steht in keiner Verbindung zur Icom Inc. oder ihren Tochtergesellschaften und wird von diesen weder unterstützt noch geprüft. „Icom“, „CI-V“ und die Gerätebezeichnungen sind Marken bzw. Bezeichnungen der Icom Inc. und werden hier nur verwendet, um die Geräte zu benennen, mit denen das Programm zusammenarbeitet. Alle Beschreibungen von Befehlen und Messbedingungen sind eigene Formulierungen; maßgeblich sind die Originalunterlagen des Herstellers.

> civ-tester4icom is an independent open-source project by radio amateurs. It is not affiliated with, endorsed by, or reviewed by Icom Inc. or its subsidiaries. "Icom", "CI-V" and model names are trademarks or designations of Icom Inc. and are used here only to identify the equipment this software works with. All descriptions of commands and measurement conditions are our own wording; the manufacturer's original documentation is authoritative.

## 2 Plattform und Architektur

Das Programm wird in C++17 mit Qt 6 (Widgets) als klassische Desktopanwendung gebaut, nicht als Browseranwendung. Grund: Der CI-V Prüfer wurde im Langzeitbetrieb instabil, weil Browser inaktive Tabs und deren Herzschlag schlafen legen.

**Aufbau in drei Teilen**

| Teil | Aufgabe |
| --- | --- |
| Kernbibliothek (ohne Oberfläche) | Transport, Busschicht, Katalog, Ablauf-Engine, Simulator, Kalibrierung, Bericht |
| Desktopanwendung | Oberfläche, Not-Aus-Fenster, Freigabedialoge |
| Kommandozeilenprogramm | gespeicherte Abläufe ohne Oberfläche, für Langzeitläufe und Raspberry Pi ohne Bildschirm |

- Serieller Verkehr läuft in einem eigenen Ein-/Ausgabe-Thread mit hochauflösenden Zeitstempeln, unabhängig von der Oberfläche.
- Während eines Laufs verhindert das Programm den Ruhezustand (Windows `SetThreadExecutionState`, Linux systemd-inhibit).
- Die Quelle der Zeitstempel ist austauschbar, damit später ein Hardware-Messkopf nachgerüstet werden kann (in Version 1 nicht vorgesehen).

**Zielsysteme (nur 64 Bit)**

| System | Paket | Hinweis |
| --- | --- | --- |
| Windows 10 / 11 | Installer und portable ZIP | Windows 10 so lange, wie Qt es unterstützt |
| Linux x86\_64 | AppImage | gebaut auf Ubuntu 24.04, läuft ab 24.04 |
| Raspberry Pi | AppImage arm64 | Raspberry Pi OS 64 Bit (Bookworm), empfohlen Pi 4/5 |

Builds laufen über GitHub Actions. Alle gelieferten Dateien tragen die Versionsnummer im Namen, z. B. `civ-tester4icom_1.0.0_windows_x64.zip`.

## 3 Transport und Anschlussarten

Version 1 verbindet sich über serielle Ports (USB und Bluetooth Classic SPP) und TCP; die Transportschicht ist austauschbar, BLE folgt in Version 2. Das Programm legt sich auf keinen bestimmten Adapter fest.

| Transport | Version | Beispiele | Hinweis |
| --- | --- | --- | --- |
| Seriell (USB) | 1 | FTDI, CH340, CP210x, Pico, CT-17-Nachbau, Radio-USB-Port |  |
| Bluetooth Classic (SPP) | 1 | eingebauter oder USB-Bluetooth-Adapter am PC; IC-705, ID-4100, ID-5200, ID-52, ID-52 PLUS | erscheint als COM-Port bzw. `/dev/rfcommX`; größere Latenz |
| TCP | 1 | WLAN-Adapter (z. B. ESP32), ser2net | gleiche Einschränkungen wie Bluetooth |
| Bluetooth LE | 2 | IC-705 (GATT Serial over LE laut Icom-Datenblatt) | Dienst und Kennungen werden beim Verbinden ermittelt; erster Test am IC-705 |

Das Icom-Netzwerkprotokoll (RS-BA1) wird nicht nachgebaut; dafür gibt es den virtuellen COM-Port der Icom Remote Utility.

**Anschlussarten** (beim Verbinden erkannt oder abgefragt, bestimmen die Auswertung):

1. **Bus-Adapter mit Echo**: am CI-V-Bus, eigenes Echo vorhanden, Kollisionen möglich.
2. **Adapter ohne Echo**: getrennte RX/TX-Leitungen.
3. **Direktverbindung**: Radio-USB-Port, Bluetooth zum Gerät. Kein Bus, keine Kollisionen; Kollisions- und Kapazitätstest abgeschaltet oder gekennzeichnet.

**Feste Regeln**

- Ports werden immer 8N1 ohne Handshake und mit **DTR und RTS inaktiv** geöffnet (Schutz vor ungewolltem Senden bei USB-SEND/Keying).
- Bei den ID-Geräten ist „Echo Back“ am Gerät wählbar; der Selbsttest erkennt das.
- Steht ein ID-Gerät auf „DV Data“, kommt kein CI-V: Meldung „keine CI-V-Antwort – Serialport Function am Gerät prüfen“ statt nur „Zeitüberschreitung“.
- Bei Verbindungsabbruch automatisch neu verbinden; die Lücke wird in Mitschnitt und Bericht markiert.
- Mehrere Ports gleichzeitig (z. B. Port A sendet am Bus, Port B hört am Radio-USB mit).

## 4 Busschicht und Messunsicherheit

Die Busschicht folgt dem Icom CI-V Reference Manual v3.2 (Abschn. 1-6, 5-4, 5-5) und gibt jeden Zeitwert mit seiner selbst bestimmten Messunsicherheit an. Ein Zeitwert ohne Unsicherheit wird nie ausgegeben.

**Rahmen und Voreinstellungen**

| Punkt | Voreinstellung | Herkunft |
| --- | --- | --- |
| Präambel Controller | 2 × `FE` | CI-V-Handbuch 5-5 |
| Controller-Adresse | E0h, als Variable änderbar (E1h–EFh bei Adresskonflikt) | CI-V-Handbuch |
| Rahmenende | `FD`; Rahmen ohne `FD` werden verworfen und gezählt | CI-V-Handbuch |
| Kollision | Echo ≠ gesendet → Abbruch, Bus frei abwarten, **5 × `FC`**, Bus frei abwarten, Wiederholung | CI-V-Handbuch 1-6, 5-4 |
| Empfang von `FC` | laufenden Rahmen verwerfen | CI-V-Handbuch 5-4 |
| Bus frei vor Senden | 5 ms, je Schritt änderbar | eigene Festlegung |
| Wiederholungen nach Kollision | max. 3, Backoff 10–40 ms | eigene Festlegung |
| Einschalt-Präambel (`18 01`) | Anzahl `FE` je Baudrate aus dem Geräteprofil | Reference Guide des Geräts |

Unaufgeforderte Rahmen (Transceive-Rundrufe an 00, fremde Master) werden immer mitgeschnitten. Je Erwartungsschritt wählbar: ignorieren und zählen, oder als Fehler werten.

**Adresskonflikt am Controller:** E0h ist auch die Vorgabe der meisten anderen PC-Programme. Bus-Inventur und Monitor erkennen, wenn ein anderer Teilnehmer ebenfalls als E0h fragt, und schlagen eine freie Adresse im Bereich E1h–EFh vor. Antworten an die eigene Adresse, die zu keiner eigenen Frage passen, werden als „fremd“ gezählt und nie als Antwort gewertet.

**Adapter-Selbsttest** (vor jeder Zeitmessung, während Langzeitläufen alle 30 min wiederholt, änderbar)

1. Einige hundert Bytes senden, Zeit bis zum Echo messen.
2. Kleinster Wert = feste Latenz; Streuung = Jitter (Typ A).
3. Häufung der Empfangszeitpunkte in Rastern = Quantisierung, z. B. FTDI-Latenztimer (Typ B, Rechteckverteilung).
4. Kombinierte Unsicherheit, **erweitert mit k = 2** nach GUM (JCGM 100).
5. Bei FTDI-Adaptern Hinweis auf den Latenztimer (Werkseinstellung 16 ms, empfohlen 1 ms).

Ohne Echo (Direktverbindung) wird nur eine Obergrenze aus der Streuung der Antwortzeiten geschätzt und als **„Schätzung, nicht gemessen“** gekennzeichnet. Verschlechtert sich die Unsicherheit während eines Laufs deutlich, wird das im Bericht markiert.

**Darstellung:** „Latenz 12,4 ms ± 1,1 ms (k = 2)“. Ist ein Wert kleiner als seine Unsicherheit, erscheint „nicht auflösbar“. Histogrammklassen sind nie feiner als U.

**Bestanden / nicht bestanden** nach ILAC-G8: „bestanden“ nur, wenn Messwert plus U innerhalb der Grenze liegt, sonst „nicht entscheidbar“.

**Fehlerraten** nach der Dreierregel: 0 Fehler in n Rahmen belegen eine Fehlerrate unter 3/n bei 95 %. Das Programm rechnet daraus die nötige Testdauer (Beispiel: unter 0,1 % braucht 3000 fehlerfreie Rahmen).

## 5 Gerätekatalog

Der Katalog ist datengetrieben: Jedes Gerät ist eine eigene YAML-Datei, das Programm selbst enthält kein Gerätewissen. Neue Geräte brauchen kein Programm-Update.

**Inhalt eines Geräteprofils**

- Geräteklasse, Werksadresse, Baudraten, Einschalt-Präambel je Baudrate, Frequenzlänge (5 oder 6 Byte)
- Befehle mit Sub-Befehlen, Datenformat, Wertebereich, Antworttyp, Bedeutung in eigenen Worten
- Sendestufe je Befehl (siehe Abschnitt 7)
- Herstellerstützpunkte der Meter (z. B. S0/S9/S9+60, PO 0/50/100 %)
- Nennleistung und Leistungsstützpunkte je Band, Betriebsarten mit Dauerträger
- Abgleichbedingungen aus dem Servicehandbuch (sofern vorhanden)
- Reihenfolge für das Wiederherstellen von Einstellungen
- Anzahl Empfänger (MAIN/SUB)
- Quellenangabe je Eintrag: Dokumenttitel, Ausgabe, Seite, ohne Link und ohne Textauszug

**Geräteklassen:** Basis (gemeinsamer Befehlssatz), Transceiver, Empfänger, D-STAR, Endstufe, Zubehör, Eigenbau. Keine Klasse setzt Funkgeräte-Annahmen voraus (die IC-PW2 hat z. B. keine Frequenz).

**Datentypen** für Eingabe und Dekoder: BCD-Frequenz 5/6 Byte, Pegel 0000–0255, Modus und Filter, Bitfelder, Zeit/Datum, Text, Rohbytes. Eingabe „14,074 MHz“ erzeugt den fertigen Rahmen; Rohbytes werden lesbar dekodiert.

**Eigene Befehle und Befehlssätze** legt der Anwender im Programm an und speichert sie als eigene Profildatei.

**Quellenregel:** Maßgeblich ist jeweils die neueste Ausgabe; liegt ein Dokument auf Englisch und Deutsch vor, gilt die englische Fassung.

**Erstumfang**

| Gerät | Klasse | Werksadresse | Stand in Version 1 |
| --- | --- | --- | --- |
| Basisprofil | Basis | – | vollständig |
| IC-7610 | Transceiver | 98h | vollständig, Servicehandbuch |
| IC-9700 | Transceiver | A2h | vollständig, Servicehandbuch |
| IC-7300MK2, IC-7760, IC-705, IC-905, IC-2730 | Transceiver | B6h, B2h, A4h, ACh, 90h | nur Basisbefehle, Profil wird später einzeln vervollständigt |
| ID-50, ID-52, ID-52 PLUS, ID-5200 | D-STAR | AEh, A6h, B4h, B8h | nur Basisbefehle, später einzeln |
| IC-R8600, IC-R15, IC-R30 | Empfänger | 96h, B0h, 9Ch | nur Basisbefehle, später einzeln |
| IC-PW2 | Endstufe | AAh | Werksadresse und Klasse, später einzeln |
| Altgeräte aus CI-V-Handbuch v3.2 | Transceiver/Empfänger | je Gerät | nur Basisbefehle |

Entschieden 25.10.2026: Version 1 enthält nur IC-7610 und IC-9700 vollständig; weitere Geräte werden danach einzeln vervollständigt, jedes in einer eigenen kurzen Arbeitssitzung.

## 6 Ablauf-Editor

Abläufe werden aus strukturierten Schritten in der Oberfläche zusammengestellt, als lesbare YAML-Datei gespeichert und in Desktop und Kommandozeile gleich ausgeführt. Eine Skriptsprache gibt es in Version 1 nicht; Bedingungen nutzen einfache Ausdrücke.

| Schritttyp | Zweck |
| --- | --- |
| Senden | Katalogbefehl oder Rohbytes, Ziel- und Absenderadresse frei |
| Erwarten | exakt, Muster mit Platzhaltern, dekodierter Wert im Bereich, OK/NG, „keine Antwort“ als Sollergebnis; Fenster frühestens/spätestens; Umgang mit fremden Rahmen; Verhalten bei Fehlschlag (wiederholen, abbrechen, weiter) |
| Warten | feste Zeit oder bis ein bestimmter Rahmen erscheint |
| Variable | Wert aus einer Antwort übernehmen und weiterverwenden |
| Schleife, Bedingung | Wiederholungen, Verzweigung |
| Unterablauf | wiederverwendbare Bausteine, z. B. „Zustand sichern“ |
| Hintergrundaufgabe | parallele zyklische Abfrage, z. B. als Last |
| Bedieneraufforderung | Text plus Bestätigung, mit Höchstdauer |
| Fehlerinjektion | verfälschte Rahmen gezielt senden (Abschnitt 8) |
| Aufräumblock | läuft immer, auch bei Abbruch und Not-Aus |
| Kommentar, Markierung | erscheint in Mitschnitt und Bericht |

**Parameter je Sendeschritt:** Bus frei seit (ms), Abstand zwischen Bytes (ms), Verhalten bei Kollision.

**Ablaufparameter:** Geräteadresse, Controller-Adresse, Baudrate und Ports sind Variablen, damit derselbe Ablauf an verschiedenen Geräten läuft.

**Prüfung vor dem Start:** Zeitfenster kleiner als die Messunsicherheit werden gemeldet; sendeauslösende Schritte werden aufgelistet (Abschnitt 7).

**Direktkonsole:** einzelne Befehle senden und die Antwort dekodiert sehen, ohne Ablauf.

## 7 Sicherheit

Die Sendersperre ist immer aktiv und wird nie gespeichert; gesendet wird nur nach ausdrücklicher manueller Freigabe für genau einen Lauf in der Oberfläche. Das Programm sendet nie selbstständig.

**Sendestufen im Katalog**

| Stufe | Beispiele | bei aktiver Sperre |
| --- | --- | --- |
| löst HF aus | `1C 00 01` Senden, `1C 01 02` Tuner abstimmen, CW-Text `17`, Sprachspeicher senden | blockiert |
| sendet indirekt | VOX, USB-SEND/Keying über DTR/RTS | blockiert |
| sendebezogener Parameter | Sendeleistung, Split, Sendefrequenz | erlaubt, gekennzeichnet |

- Rohbytes und eigene Befehle werden vor dem Senden gegen den Katalog geprüft; unbekannte Befehle an sendefähige Geräte gelten als „möglicherweise HF-auslösend“.
- Fehlerinjektion und Konformitätstest erzeugen bei aktiver Sperre nie Muster, die einen gültigen Sendebefehl ergeben könnten.

**Freigabe**

- Dialog listet jeden HF-auslösenden Schritt, Höchstleistung, Stoßdauer und gesamte Sendezeit.
- Bestätigung „Kunstantenne angeschlossen“.
- Gilt nur für diesen Lauf; erlischt am Laufende, bei Abbruch und bei Programmende.
- **Kommandozeile:** verweigert jeden Ablauf mit sendeauslösendem Schritt.

**Not-Aus**

1. Solange eine Freigabe aktiv ist, gibt es keine sperrenden (modalen) Dialoge; Rückfragen erscheinen als Leiste im Fenster.
2. Eigenes, immer obenliegendes Not-Aus-Fenster mit großem Knopf, unabhängig vom Hauptfenster.
3. Not-Aus-Taste (Pause oder F12, nicht Esc) über einen anwendungsweiten Ereignisfilter; kein Eingabefeld oder Dialog kann sie abfangen.
4. Wirkt direkt im Ein-/Ausgabe-Thread: nach dem laufenden Byte sofort `1C 00 00`, Ablauf beenden, Empfang zurücklesen.
5. Systemweite Not-Aus-Taste, wo das Betriebssystem es erlaubt (Windows, Linux/X11; unter Wayland nicht zuverlässig, so dokumentiert).
6. Kein externer Hardware-Not-Aus; letzter Ausweg laut Anleitung: Stromversorgung des Geräts trennen.

**Wächter**

- Sendestoß endet spätestens nach der Höchstdauer (Kalibrierung: 7 s) mit `1C 00 00` und Kontrolle.
- Reagiert die Oberfläche während des Sendens länger als 1 s nicht, beendet der Ein-/Ausgabe-Thread das Senden selbst.
- SWR-Meter (`15 12`) wird während jedes Stoßes gelesen; über der Grenze (Vorgabe 2:1) sofortiger Abbruch.
- Bei Verbindungsverlust während des Sendens mehrfach `1C 00 00` versuchen; Bericht meldet „Sendezustand unbekannt“.

**Schnappschüsse und Wiederherstellung**

- Standard: Das Programm liest vor dem Lauf genau die Werte, die der Ablauf setzt. Optional: vollständiger Schnappschuss aller lesbaren Einstellungen laut Katalog.
- Der Schnappschuss wird als Datei gespeichert, bevor der erste Setzbefehl gesendet wird. Nach Absturz oder Stromtrennung fragt das Programm beim Start: „Ungesicherter Gerätezustand gefunden – wiederherstellen?“
- Wiederherstellung am Laufende, bei Abbruch und von Hand, in der Reihenfolge laut Profil; danach zurücklesen und vergleichen, Abweichungen in den Bericht.
- Nur setzbare, nicht lesbare Werte werden markiert und nicht wiederhergestellt.
- Benannte Schnappschüsse dienen als Ausgangszustand für reproduzierbare Tests.

**Abgleichmodus:** wird nie unterstützt, auch nicht lesend. Das Programm ändert keine Abgleichwerte.

## 8 Mitgelieferte Prüfungen und Simulator

Alle mitgelieferten Prüfungen sind normale, änderbare Abläufe; Kapazitäts- und Konformitätstest sind Teil von Version 1.

**Vorlagen**

1. Adapter-Selbsttest (Abschnitt 4)
2. Bus-Inventur: passiv mithören, dann `19 00` an mögliche Adressen → wer ist mit welcher Adresse am Bus
3. Baudraten-Erkennung durch Mithören
4. Grundfunktionen je Gerät: lesen, setzen, wiederherstellen, ohne HF-auslösende Befehle
5. Transceive-Verhalten: Zustand ändern, Rundruf prüfen
6. Kapazitätstest
7. Langzeittest mit Testdauer nach der Dreierregel
8. Konformitätstest

**Kapazitätstest** (nur an Bus-Anschlüssen)

- **Theorie:** Leitungszeiten bei der eingestellten Baudrate (19200 Bd: 1 Byte 0,52 ms, `03`-Abfrage 3,1 ms, Antwort 5,7 ms).
- **Phase 1, passiv (Vorgabe 10 min):** Busauslastung, Verteilung der Lücken, periodische Fremd-Master mit Periode und Phase, Rundrufaufkommen, Teilnehmer ohne erkennbare Kollisionsvermeidung.
- **Phase 2, aktiv:** Abfrageintervall stufenweise verkürzen (Vorgabe 500 → 50 ms, je 200 Abfragen); Antwortquote, Latenz P95 und Kollisionen je Stufe; Knickpunkt bestimmen.
- **Ergebnis:** empfohlenes Intervall = Knickpunkt × 1,5 plus 2 × U; bei periodischen Fremd-Mastern empfohlene Phasenlage. Hinweis: bei geänderter Busbelegung wiederholen.

**Konformitätstest** (Prüfling: Icom-Gerät oder Eigenbau; Adresse und Rolle werden vorgegeben: antwortet nur, fragt selbst ab, beides)

- verwirft verstümmelte Rahmen (ohne `FD`, zu kurz, Müllbytes)
- reagiert richtig auf `FC`
- erkennt Kollisionen und wiederholt danach (das Programm erzeugt gezielt Kollisionen)
- toleriert Baudabweichungen (Vorgabe ±2 %)
- übersteht Fehlerinjektion ohne Absturz

Jede Prüfung nennt ihre Grundlage (z. B. CI-V-Handbuch 5-4) und endet mit „erfüllt“, „nicht erfüllt“ oder „nicht entscheidbar“.

**Simulator** (nur allgemeine Demo-Geräte, keine projektspezifischen wie TRXCom oder S/P-Meter)

- generischer Transceiver, Empfänger und Endstufe aus jedem Katalogprofil, mit Zustandsmodell (Frequenz, Modus, Antenne, Split, Transceive an/aus, ausgeschaltet bis Präambel + `18 01`)
- frei konfigurierbarer Abfrage-Teilnehmer: Adresse, Befehl, Intervall, Jitter, mit oder ohne Kollisionsvermeidung
- Antwortverhalten: Verzögerung fest oder verteilt; unbekannte Befehle mit NG oder ignorieren
- mehrere simulierte Teilnehmer an einem Port; Szenarien speicherbar
- Bus-Eignungstest mit zwei Ports (A sendet, B empfängt am anderen Leitungsende): Fehlerrate je Baudrate, Toleranz bei Baudabweichung
- Voraussetzung am echten Bus: Adapter mit Open-Collector-Ausgang; nie gegen ein echtes Gerät mit gleicher Adresse

**Fehlerinjektion** (Anteile in %): NG statt OK, keine Antwort, verspätete Antwort, Rahmen ohne `FD`, einzelnes `FE` oder Müllbyte, Lücke im Rahmen, absichtliche Kollision.

## 9 S-Meter-Kalibrierung

Der geführte Ablauf stellt das Radio per CI-V auf die Icom-Abgleichbedingungen ein, führt den Anwender Stufe für Stufe durch die 15 IARU-Pegel und speichert die Tabelle Pegel → Rohwert. Gelesen wird der Rohwert von `15 02` (0–255); das ist ein aufbereiteter Meterwert, kein direkter ADC-Wert.

**Pegelstufen (Vorgabe, IARU Region 1 R.1, Pegel am Lastende/PD)**

| Stufe | unter 30 MHz | über 30 MHz |
| --- | --- | --- |
| S1 | −121 dBm | −141 dBm |
| S2 | −115 dBm | −135 dBm |
| S3 | −109 dBm | −129 dBm |
| S4 | −103 dBm | −123 dBm |
| S5 | −97 dBm | −117 dBm |
| S6 | −91 dBm | −111 dBm |
| S7 | −85 dBm | −105 dBm |
| S8 | −79 dBm | −99 dBm |
| S9 | −73 dBm | −93 dBm |
| S9+10 … S9+60 | −63 … −13 dBm in 10-dB-Schritten | −83 … −33 dBm in 10-dB-Schritten |

Wahlweise S0 (Messsender aus) am Anfang und Kontrollmessung S9 am Ende. Feinere Zwischenpunkte sind zusätzlich wählbar. Die Pegeltabelle richtet sich nach der **Messfrequenz**, nicht nach dem Gerät (6 m beim IC-7610 = VHF-Pegel).

**Messbedingungen**

| Parameter | Wert | Herkunft |
| --- | --- | --- |
| Frequenz KW | 14,1515 MHz | IC-7610-Servicehandbuch |
| Frequenz 2 m / 70 cm / 23 cm | 146,0215 / 435,0215 / 1280,0215 MHz | IC-9700-Servicehandbuch |
| Signal | unmodulierter Träger, Pegel am Lastende (PD) | Servicehandbücher |
| Betriebsart, Filter, AGC | CW, 1,2 kHz, AGC MID | eigene Festlegung (Icom nennt nichts) |
| Vorverstärker, Abschwächer | AUS; weitere Tabellen für PRE1/PRE2 wählbar | Icom gleicht PRE getrennt ab |
| RF-Gain, NB/NR/Notch, PBT | Maximum / aus / Mitte | eigene Festlegung |
| Einschwingzeit, Messwerte | 2 s, 20 Werte je Stufe | eigene Festlegung |

**Rückfallregel:** Geräte ohne eigene Abgleichdaten im Profil nutzen auf KW/6 m die IC-7610-Bedingungen. Auf VHF/UHF gibt es keinen Rückfall: Ohne Servicehandbuch im Profil ist die Messung gesperrt. Ersatzwerte werden im Bericht gekennzeichnet; fehlt eine Einstellung am Zielgerät, wird die nächstliegende genommen und vermerkt.

**Ablauf**

1. Gerät wählen; Profil lädt Abgleichbedingungen (angezeigt, änderbar).
2. Messsender eintragen: Einheit (dBm, dBµV, µV, PD oder EMF), Vorschaltdämpfung, Pegelgenauigkeit, kleinster erreichbarer Pegel.
3. Sicherheits-Checkliste (Antenne ab, Messsender aus).
4. Schnappschuss, dann Gerät einstellen und zurücklesen.
5. Je Stufe: „Lege −97 dBm an (S5)“ mit dem am Messsender einzustellenden Wert → Anwender bestätigt → Programm liest N Werte → Mittelwert und Streuung → bei zu großer Streuung wiederholen, übernehmen (markiert) oder überspringen.
6. Jederzeit: Zurück, Wiederholen, Zwischenpunkt, Abbrechen (Teilergebnis wird als „unvollständig“ gespeichert).
7. Abschluss: Plausibilität (monoton, Sättigung, S0-Grundwert), Vergleich mit Icom-Stützpunkten, Wiederherstellung, ZIP.

**Besonderheiten**

- IC-7610: Icom gleicht S9 und S9+60 ab; diese beiden Punkte werden als Prüfpunkte für den Werksabgleich markiert.
- Zwei Empfänger (MAIN/SUB): wahlweise beide messen, eine Tabelle je Empfänger.
- IC-9700: kein S-Meter-Abgleich im Servicehandbuch; Vergleich nur mit den Stützpunkten des Reference Guide.
- Stufen unter dem kleinsten Pegel des Messsenders werden vorab als „nicht messbar“ markiert. Unterscheidet sich der Rohwert nicht vom S0-Wert oder steigt nicht, wird der Punkt „nicht auflösbar“ und nicht in die Tabelle übernommen.

## 10 Leistungskalibrierung

Das Programm stellt am Radio die Leistung in Prozent ein, der Anwender löst jeden Sendestoß selbst aus und trägt den abgelesenen Wattmeterwert mit seiner Ungenauigkeit ein. Daraus entstehen zwei Tabellen: Einstellung → Leistung und PO-Rohwert (`15 11`) → Leistung.

**Messpunkte:** 1–14 % in 1-%-Schritten, 15–100 % in 5-%-Schritten (32 Punkte). Gesetzt wird der nächstliegende Rohwert von `14 0A` (0–255); in der Tabelle stehen Nennwert in % und gesendeter Rohwert. Optional fragt das Programm bei den ersten Punkten: „Zeigt das Radio x % an?“

**Betriebsarten**

| Betriebsart | Aufbau | Bericht |
| --- | --- | --- |
| RTTY (Standard) | kein Ton; Dauerträger; FM nur manuell als Ausweich | eigene Messbedingung |
| Herstelleraufbau | USB; Tongenerator 1,5 kHz Sinus an Mikrofonbuchse, NF-Millivoltmeter parallel; Pegel laut Servicehandbuch (IC-7610: 10 mV, IC-9700: 30 mV); Mic-Gain fest, Kompressor und VOX aus | „nach Icom-Servicehandbuch, Betriebsart USB angenommen“ |
| PC-Ton über USB-Audio | USB-D, MOD-Eingang USB; Programm erzeugt 1,5 kHz nur während eines Stoßes; Pegel in dBFS | eigene Messbedingung |

Bei Herstelleraufbau und PC-Ton wird das ALC-Meter (`15 13`) mitgelesen; Punkte ohne ALC-Regelung werden als „Ansteuerung zu gering“ markiert. Ob RTTY am IC-7610 einen sauberen Dauerträger liefert, wird beim ersten Test geprüft; zeigt PO beim ersten Punkt praktisch nichts, meldet das Programm „kein Träger – Betriebsart prüfen“.

**Frequenz:** KW 14,150 MHz, 6 m 50,150 MHz (abseits von Bakenfrequenzen), weitere Bänder wählbar. Icom gibt für den Leistungsabgleich keine Frequenz an. Filterbreite bleibt unverändert und wird nur protokolliert.

**Vorbereitung**

1. Gerät wählen (Nennleistung und Icom-Stützpunkte je Band).
2. Leistungsmesser: Typ, Messbereiche, Frequenzbereich, Koppel- oder Vorschaltdämpfung. Kunstantenne: Belastbarkeit, Frequenzbereich. Messpunkte außerhalb dieser Grenzen werden gesperrt.
3. Sicherheits-Checkliste: Kunstantenne über Leistungsmesser an ANT1, keine Antenne, Tuner aus, Mikrofon/Taste/PTT ab (außer Herstelleraufbau), USB-SEND/Keying aus.
4. Schnappschuss, Gerät einstellen (Frequenz, Betriebsart, ANT1, Tuner/Split/VOX aus, Leistung minimal), zurücklesen, Empfang prüfen.
5. Sendefreigabe mit Not-Aus-Fenster.

**Je Messpunkt**

1. Programm setzt x %, zeigt „Punkt 7/32: 7 %“.
2. Anwender drückt „Senden“. Träger höchstens **7 s** mit sichtbarem Countdown; **„Fertig“** (groß, neben dem Countdown, auch Eingabetaste) beendet früher. Nach ca. 1 s Einschwingzeit wird PO gemittelt, SWR überwacht.
3. Empfang, zurücklesen.
4. Eingabe: Messwert in **W oder dBm**; Ungenauigkeit **± % vom Messwert** oder **± % vom Endwert** plus Messbereich (vorbelegt mit dem letzten Wert).
5. Plausibilitätsprüfung (steigen Rohwert und Leistung?), Zeile erscheint sofort in beiden Tabellen.
6. Weiter erst nach Bestätigung; keine erzwungene Pause. Unverbindliche Anzeige von bisheriger Sendezeit und mittlerer Leistung.
7. Lief die Zeit ab ohne „Fertig“: „Wert abgelesen?“ → Eingabe oder Wiederholen.

Ist die Unsicherheit eines Punktes größer als 10 % vom Messwert, fordert das Programm einen kleineren Messbereich an oder markiert den Punkt „nicht ausreichend genau“.

**Abschluss:** Sendeaus geprüft, Schnappschuss wiederhergestellt, Icom-Stützpunkte markiert (IC-7610: 1 / 10 / 50 W und 100 % = 103 W; IC-9700 je Band laut Servicehandbuch), Unsicherheit je Punkt, ZIP. Zeitbedarf ca. 15–20 min je Band.

## 11 Einstellungsprofile

Einstellungsprofile versetzen den Transceiver in eine geprüfte Grundeinstellung, z. B. für FT8; sie sind ein eigener Programmteil neben den Messabläufen. Eingestellt wird nur der Transceiver, nicht die PC-Software.

**Fragebogen vor dem Anwenden** (Antworten als Stationsprofil gespeichert)

1. Gerät (möglichst automatisch erkannt; danach nur Optionen, die das Gerät kennt)
2. Steuerverbindung: USB-Port, CI-V-Buchse, LAN/WLAN über Icom Remote Utility, Bluetooth
3. Audioweg: USB-Audio, ACC mit externem Interface, Mikrofonbuchse
4. Sendetastung: CAT, USB-Steuerleitung (DTR/RTS), ACC-SEND, VOX
5. Anwendung: FT8/FT4, RTTY (AFSK oder FSK), PSK, SSTV, Packet/APRS, Winlink/VARA, Fernbetrieb
6. PC-Software (nur für Hinweise): WSJT-X, JTDX, fldigi u. a.
7. Weitere Geräte am CI-V-Bus (Tuner, Endstufe, Meter) → Transceive, CI-V Output (for ANT), CI-V USB Port
8. Mehrere Programme gleichzeitig am Gerät?

**Anwenden**

1. Ist-Zustand lesen, Vorschau Ist → Soll zeigen; erst nach Bestätigung schreiben.
2. Vorher automatisch Schnappschuss („Rückgängig“ jederzeit).
3. Danach zurücklesen und vergleichen.

**Regeln**

- Welche Menüpunkte es gibt und wie sie kodiert sind, kommt ausschließlich aus dem Reference Guide des Geräts.
- Jeder Wert trägt seine Herkunft: Icom-Vorgabe, Erfahrungswert oder Anwender. Erfahrungswerte werden als Startpunkt gezeigt, nicht als Vorgabe.
- Einstellungen, die das Senden betreffen (USB-Keying, VOX), fallen unter „sendet indirekt“ und brauchen die Freigabe.
- Warnung bei Sendetastung über DTR/RTS: Das Gerät kann beim Öffnen des COM-Ports durch andere Programme auf Sendung gehen.
- Empfehlungen als Text, z. B. ALC in Datenbetriebsarten nicht ansprechen lassen, Kompressor aus, „COM-Port kann nur von einem Programm gleichzeitig genutzt werden“.
- Profile sind teilbare Dateien.

**Recherche vor dem Bauen:** Reference Guides und Advanced Manuals (Menüpunkte), Icom-Einrichtungshinweise für Datenbetrieb, Anleitungen von WSJT-X und fldigi. Forenwerte nur als „Erfahrungswert“.

## 12 Ergebnisse, Oberfläche, Sprachen

Jeder Lauf und jede Kalibrierung endet mit einer ZIP-Datei, deren Name Programmversion, Art, Gerät, Frequenz und Zeit enthält, z. B. `civ-tester4icom_1.0.0_S-Meter_IC-7610_14152kHz_2026-11-02_1930.zip`.

**Inhalt der ZIP-Datei**

| Datei | Inhalt |
| --- | --- |
| `bericht.html` / `bericht.pdf` | Prüfling, Adapter und Messmittel, Bedingungen, Ergebnisse mit Unsicherheit, Konformitätsaussage, Entscheidungsregel, Hinweise, Markenhinweis |
| `tabelle.csv` | Ergebnistabelle (bei Kalibrierung: Pegel bzw. % → Rohwert/Leistung mit Streuung, Anzahl, Markierungen) |
| `messwerte.csv` | alle Einzelwerte |
| `ergebnisse.json` | alles maschinenlesbar |
| `telegramme.csv` | CI-V-Mitschnitt mit Zeitstempeln |
| Code-Exporte | optional über Vorlagen; mitgeliefert eine Beispielvorlage für Kennlinien-Tabellen in C |

CSV nach RFC 4180, Zeitstempel nach ISO 8601. Mitschnitte sind in einem eigenen Format offline wieder einlesbar und anders auswertbar.

**Oberfläche** (Entwurf vom 23.10.2026 freigegeben)

1. Ablauf-Editor: Projektbaum, Schrittliste, Eigenschaften, Busmonitor-Kurzansicht
2. Gerätekatalog mit Wert-Eingabe und Dekoder
3. Lauf und Busmonitor mit Kennzahlen, Histogramm, Selbsttest, Filtern
4. Simulator
5. Kapazitätstest
6. Kalibrierung S-Meter und Leistung (geführt)
7. Einstellungsprofile mit Fragebogen

- Hell/Dunkel: „folgt dem System“ (Standard), hell oder dunkel fest wählbar. Status nie nur über Farbe, immer auch Text oder Symbol.
- Sprachen: Deutsch und Englisch, zur Laufzeit umschaltbar, Standard = Systemsprache; Berichte in der gewählten Sprache. Code und Kommentare englisch, Anleitung in beiden Sprachen.
- Bezeichnungen in Oberfläche und Anleitung sind identisch (Lehre aus dem CI-V Prüfer: kein Knopf, der nur in der Anleitung existiert).

## 13 Repo, Veröffentlichung, Bauablauf

Das Repo `civ-tester4icom` wird nach dem ersten Commit mit LICENSE, README und .gitignore öffentlich; alle Herstellerunterlagen und eigenen Mitschnitte bleiben im privaten Repo `civ-werkstatt`.

**Regeln für das öffentliche Repo**

- Baut und läuft vollständig ohne das private Repo.
- Keine Icom-PDFs, keine Auszüge daraus, keine Mitschnitte der eigenen Anlage, keine Zugangsdaten, keine lokalen Pfade; `.gitignore` schließt PDFs, Mitschnitte und Berichte aus.
- Quellenangabe nur als Text: „Icom, IC-7610 CI-V Reference Guide (2025), S. 12, erhältlich auf der Icom-Website“.
- Umstellen auf öffentlich und Einschalten von Secret Scanning mit Push-Schutz macht der Repo-Inhaber selbst.

**Bauablauf (Variante B)**

1. Diese Spezifikation freigeben.
2. Grundgerüst bauen: Transport, Busschicht, Selbsttest, Busmonitor, Builds für alle Plattformen.
3. Zwischentest beim Anwender (ca. 15 min): Adapter anstecken, Selbsttest und Monitor, Ergebnis zurückmelden.
4. Rest am Stück bauen, Commit nach jedem Abschnitt.
5. Gesamttest am IC-7610, inklusive erster RTTY-Trägerprüfung bei 5 % an der Kunstantenne.

**Version 2 (geplant):** Bluetooth LE, erster Test am IC-705.

**Spätere Erweiterungen (nicht geplant):** SWR-Meter-Kalibrierung, ALC-Kalibrierung, Strom/Spannung der Endstufe, Hardware-Messkopf mit Zeitstempeln, Skriptsprache, Import von Logikanalysator-Mitschnitten.

**Offene Punkte**

- [x] Umfang „vollständig“ im Gerätekatalog (Abschnitt 5)
- [x] Englische Advanced Manuals für IC-7300, IC-7760 und die ID-Geräte (deutsche liegen vor)
- [ ] Testadapter vorerst nur das eigene Pico-Interface (civ-werkstatt/pico-interface); Adapterliste für die Testmatrix später
- [x] Busmitschnitte als Testdaten (privat)
- [ ] Termin für den Baubeginn

**Quellen** (privat in `civ-werkstatt/doku/referenz`): Icom CI-V Reference Manual v3.2 (2002); CI-V Reference Guides IC-7610 (2025), IC-7300MK2, IC-7760 (2025), IC-9700, IC-705 (2023), IC-905, IC-R8600, IC-R15, IC-2730, IC-PW2, ID-50, ID-52, ID-52 PLUS, ID-5200; Servicehandbücher IC-7610 (2017) und IC-9700 (2019); IC-705 Bluetooth/WLAN-Spezifikation; Advanced Manuals IC-705, IC-R30, ID-4100, ID-5200, ID-52, ID-52 PLUS; IARU Region 1 Empfehlung R.1; JCGM 100 (GUM); ILAC-G8.

## 14 Bauplan in Nachtläufen

Gebaut wird in 26 Paketen, je ein Paket pro Lauf, montags, mittwochs und freitags um 01:30 Uhr. Damit ist Version 1.0 nach etwa 9 bis 10 Wochen fertig, zuzüglich der Tage, an denen ein Lauf auf einen Test wartet.

> Hinweis 25.10.2026: Die Laufsteuerung wurde nach der Freigabe geändert (tägliche Abfrage um 22 Uhr statt fester Wochentage). Maßgeblich für die Laufsteuerung ist [PAKETPLAN.md](PAKETPLAN.md).

**Ablauf eines Laufs**

1. Geplante Aufgabe startet eine frische, kurze Sitzung (nicht diese Unterhaltung).
2. Sie liest im Repo `docs/PAKETPLAN.md` und nimmt das erste offene Paket.
3. Ist das Paket ein Haltepunkt (⏸) ohne Rückmeldung des Anwenders: kein Bau, nur Eintrag „wartet auf Test“, Ende.
4. Sonst: nur die für das Paket nötigen Dateien lesen, bauen, testen (Simulator, Unit-Tests, Build).
5. Commit mit Paketnummer, Eintrag in `docs/STAND.md` (was erledigt, was offen, Auffälligkeiten), Paket in `PAKETPLAN.md` abhaken.
6. Sofort beenden, auch wenn Kontingent übrig ist. Wird ein Paket zu groß, an sauberer Stelle mit Commit enden; der nächste Lauf macht dort weiter.

**Ein Lauf tut nie:** über einen Haltepunkt hinausbauen, mehr als ein Paket bauen, ein Release oder Version 1.0 veröffentlichen, das private Repo `civ-werkstatt` ändern, Herstellerunterlagen ins öffentliche Repo bringen.

**Paketplan** (Wochen ab Start, ohne Wartezeiten)

| Woche | Lauf | Paket | Haltepunkt |
| --- | --- | --- | --- |
| 1 | 1 | Repo-Grundlage: LICENSE, README, .gitignore, Struktur, CMake |  |
| 1 | 2 | Build-Pipeline Windows, Linux, Raspberry Pi |  |
| 1 | 3 | Transport seriell, DTR/RTS aus, Port-Auswahl |  |
| 2 | 4 | Rahmenparser, Echo, Kollision, Jammer |  |
| 2 | 5 | Adapter-Selbsttest, Messunsicherheit |  |
| 2 | 6 | Busmonitor in der Oberfläche, Mitschnitt | ⏸ Zwischentest mit dem Pico (ca. 15 min) |
| 3 | 7 | Korrekturen aus dem Zwischentest |  |
| 3 | 8 | Katalogformat, Datentypen, Dekoder, Basisprofil |  |
| 3 | 9 | IC-7610-Profil Teil 1 |  |
| 4 | 10 | IC-7610-Profil Teil 2, Direktkonsole |  |
| 4 | 11 | IC-9700-Profil |  |
| 4 | 12 | Ablauf-Engine: Senden, Erwarten, Warten, Variablen |  |
| 5 | 13 | Ablauf-Engine: Schleifen, Unterabläufe, Hintergrund, Aufräumblock |  |
| 5 | 14 | Ablauf-Editor (Oberfläche) |  |
| 5 | 15 | Sendersperre, Freigabe, Not-Aus, Wächter |  |
| 6 | 16 | Schnappschüsse, Wiederherstellung, Bericht und ZIP | ⏸ Vorabversion am IC-7610 testen |
| 6 | 17 | Korrekturen aus dem Vorabtest |  |
| 6 | 18 | TCP, Bluetooth-SPP, Anschlussarten, Kommandozeile |  |
| 7 | 19 | Simulator und Fehlerinjektion |  |
| 7 | 20 | Kapazitätstest |  |
| 7 | 21 | Konformitätstest |  |
| 8 | 22 | S-Meter-Kalibrierung |  |
| 8 | 23 | Leistungskalibrierung, Tonausgabe |  |
| 8 | 24 | Einstellungsprofile mit Fragebogen und Recherche |  |
| 9 | 25 | Sprachen, Hell/Dunkel, Installer, Anleitung | ⏸ Gesamttest inkl. Kalibrierung |
| 9–10 | 26+ | Korrekturen aus dem Gesamttest | ⏸ Freigabe Version 1.0 |

**Haltepunkte:** Der Anwender meldet das Testergebnis in einer Unterhaltung; dort wird es als Eintrag in `docs/STAND.md` festgehalten. Erst dann baut der nächste Lauf weiter.

**Steuerung durch den Anwender**

- Nach Woche 1: Wochenlimit in der Nutzungsanzeige prüfen, dann Anzahl der Nächte anpassen (2 bis 5 pro Woche).
- Einzelne Nächte auslassen, pausieren oder beenden: kurze Nachricht genügt.
- Morgens Stand lesen: `docs/STAND.md` im Repo oder die Benachrichtigung der geplanten Aufgabe.

**Vor dem ersten Lauf:** Spezifikation freigegeben, als `docs/SPEZIFIKATION.md` mit `PAKETPLAN.md` und `STAND.md` im Repo, geplante Aufgabe eingerichtet, Starttermin vereinbart.
