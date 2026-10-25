# civ-tester4icom – Specification Version 1

**Deutsche Fassung: [SPEZIFIKATION.md](SPEZIFIKATION.md)**

Status 2026-10-25 · approved. This English text is a translation of the approved German original ([SPEZIFIKATION.md](SPEZIFIKATION.md)); where the two differ, the German version prevails.

**Changes after approval (2026-10-25, accepted by the user):** three-level decision rule with uniform terms (section 4); self-test pattern without `FE`/`FD`/`FC` (section 4); transmit lock from package 8, release in package 15 (sections 7 and 14); run control (section 14).

## 1 Purpose and scope

civ-tester4icom is an open measurement and test program for Icom CI-V buses and equipment with a CI-V interface, intended as a tool for radio amateurs and developers of home-built accessories. Unlike the earlier CI-V Prüfer, it contains no hard-coded test sequences: users assemble their own sequences in the user interface, save them and run them. Bundled sequences are templates only.

**Core functions of version 1**

- Monitor, decode, record and analyse bus traffic offline
- Build sequences from catalog commands and user-defined commands, with response windows, timing and evaluation
- Determine the measurement uncertainty itself and state it with every time value
- Capacity test, conformance test, simulator with fault injection
- S-meter and power calibration as guided sequences
- Settings profiles (e.g. FT8) with a questionnaire
- Save and restore device settings

**Out of scope:** a complete radio control panel, alignment of equipment in service/adjustment mode, analog level or edge measurements on the bus.

**Licence:** MIT, notice "Copyright (c) 2026 UnImAtrIXzEr0". The author's call sign and name are not preset anywhere in the program; the "Operator" field stays empty until the user fills it in.

**Liability:** provided without warranty (licence text). In addition, README and program state: sequences containing transmit-triggering commands are run at the user's own risk and only into a dummy load.

**Independence and trademark notice** (top of README, "About" dialog, footer of every report, manual; no Icom logos or Icom styling):

> civ-tester4icom ist ein unabhängiges, offenes Projekt von Funkamateuren. Es steht in keiner Verbindung zur Icom Inc. oder ihren Tochtergesellschaften und wird von diesen weder unterstützt noch geprüft. „Icom“, „CI-V“ und die Gerätebezeichnungen sind Marken bzw. Bezeichnungen der Icom Inc. und werden hier nur verwendet, um die Geräte zu benennen, mit denen das Programm zusammenarbeitet. Alle Beschreibungen von Befehlen und Messbedingungen sind eigene Formulierungen; maßgeblich sind die Originalunterlagen des Herstellers.

> civ-tester4icom is an independent open-source project by radio amateurs. It is not affiliated with, endorsed by, or reviewed by Icom Inc. or its subsidiaries. "Icom", "CI-V" and model names are trademarks or designations of Icom Inc. and are used here only to identify the equipment this software works with. All descriptions of commands and measurement conditions are our own wording; the manufacturer's original documentation is authoritative.

## 2 Platform and architecture

The program is built in C++17 with Qt 6 (Widgets) as a classic desktop application, not as a browser application. Reason: the CI-V Prüfer became unstable in long-term operation because browsers put inactive tabs and their heartbeat to sleep.

**Three-part structure**

| Part | Responsibility |
| --- | --- |
| Core library (no UI) | transport, bus layer, catalog, sequence engine, simulator, calibration, report |
| Desktop application | user interface, emergency-stop window, release dialogs |
| Command-line program | saved sequences without UI, for long-term runs and headless Raspberry Pi |

- Serial traffic runs in a dedicated I/O thread with high-resolution timestamps, independent of the UI.
- During a run the program prevents system sleep (Windows `SetThreadExecutionState`, Linux systemd-inhibit).
- The timestamp source is exchangeable so that a hardware probe can be added later (not planned for version 1).

**Target systems (64-bit only)**

| System | Package | Note |
| --- | --- | --- |
| Windows 10 / 11 | installer and portable ZIP | Windows 10 for as long as Qt supports it |
| Linux x86\_64 | AppImage | built on Ubuntu 24.04, runs on 24.04 and later |
| Raspberry Pi | AppImage arm64 | Raspberry Pi OS 64-bit (Bookworm), Pi 4/5 recommended |

Builds run on GitHub Actions. Every delivered file carries the version number in its name, e.g. `civ-tester4icom_1.0.0_windows_x64.zip`.

## 3 Transport and connection types

Version 1 connects via serial ports (USB and Bluetooth Classic SPP) and TCP; the transport layer is exchangeable, BLE follows in version 2. The program is not tied to any particular adapter.

| Transport | Version | Examples | Note |
| --- | --- | --- | --- |
| Serial (USB) | 1 | FTDI, CH340, CP210x, Pico, CT-17 clone, radio USB port |  |
| Bluetooth Classic (SPP) | 1 | built-in or USB Bluetooth adapter on the PC; IC-705, ID-4100, ID-5200, ID-52, ID-52 PLUS | appears as COM port or `/dev/rfcommX`; higher latency |
| TCP | 1 | Wi-Fi adapter (e.g. ESP32), ser2net | same limitations as Bluetooth |
| Bluetooth LE | 2 | IC-705 (GATT Serial over LE according to the Icom data sheet) | service and identifiers are discovered on connect; first test on the IC-705 |

The Icom network protocol (RS-BA1) is not re-implemented; the virtual COM port of the Icom Remote Utility covers that case.

**Connection types** (detected or asked for when connecting; they determine the evaluation):

1. **Bus adapter with echo**: on the CI-V bus, own echo present, collisions possible.
2. **Adapter without echo**: separate RX/TX lines.
3. **Direct connection**: radio USB port, Bluetooth to the device. No bus, no collisions; collision and capacity tests are disabled or flagged.

**Fixed rules**

- Ports are always opened 8N1 without handshake and with **DTR and RTS inactive** (protection against unintended transmission via USB SEND/keying).
- On the ID series, "Echo Back" is selectable on the device; the self-test detects it.
- If an ID device is set to "DV Data", no CI-V arrives: the message is "no CI-V response – check Serialport Function on the device" instead of just "timeout".
- On connection loss, reconnect automatically; the gap is marked in the recording and the report.
- Several ports at the same time (e.g. port A transmits on the bus, port B listens on the radio's USB port).

## 4 Bus layer and measurement uncertainty

The bus layer follows the Icom CI-V Reference Manual v3.2 (sections 1-6, 5-4, 5-5) and states every time value together with its self-determined measurement uncertainty. A time value without uncertainty is never output.

**Frames and defaults**

| Item | Default | Source |
| --- | --- | --- |
| Controller preamble | 2 × `FE` | CI-V manual 5-5 |
| Controller address | E0h, changeable as a variable (E1h–EFh on address conflict) | CI-V manual |
| End of frame | `FD`; frames without `FD` are discarded and counted | CI-V manual |
| Collision | echo ≠ sent → abort, wait for bus idle, **5 × `FC`**, wait for bus idle, retry | CI-V manual 1-6, 5-4 |
| Reception of `FC` | discard the current frame | CI-V manual 5-4 |
| Bus idle before sending | 5 ms, changeable per step | own definition |
| Retries after collision | max. 3, back-off 10–40 ms | own definition |
| Power-on preamble (`18 01`) | number of `FE` per baud rate from the device profile | device Reference Guide |

Unsolicited frames (transceive broadcasts to 00, foreign masters) are always recorded. Selectable per expect step: ignore and count, or treat as an error.

**Address conflict at the controller:** E0h is also the default of most other PC programs. Bus inventory and monitor detect when another participant also polls as E0h and suggest a free address in the range E1h–EFh. Replies to our own address that do not match any of our own requests are counted as "foreign" and never evaluated as a response.

**Adapter self-test** (before every timing measurement, repeated every 30 min during long-term runs, changeable)

1. Send a few hundred bytes, measure the time until the echo. The pattern never contains the bytes `FE`, `FD` or `FC`, so no device can form a frame from it and no ongoing transmission is disturbed; it is sent only when the bus is idle, under the same rules as any send step. A unit test checks the pattern.
2. Smallest value = fixed latency; spread = jitter (type A).
3. Clustering of reception times on a grid = quantisation, e.g. FTDI latency timer (type B, rectangular distribution).
4. Combined uncertainty, **expanded with k = 2** according to GUM (JCGM 100).
5. For FTDI adapters, a note on the latency timer (factory setting 16 ms, recommended 1 ms).

Without echo (direct connection) only an upper bound is estimated from the spread of response times and labelled **"estimate, not measured"**. If the uncertainty degrades noticeably during a run, this is marked in the report.

**Presentation:** "Latency 12.4 ms ± 1.1 ms (k = 2)". If a value is smaller than its uncertainty, "not resolvable" is shown. Histogram bins are never finer than U.

**Decision rule** according to ILAC-G8:09/2019, U = expanded uncertainty (k = 2), x = measured value, LL/UL = lower/upper limit:

| Limit | met | not met | undecidable |
| --- | --- | --- | --- |
| upper limit UL only | x + U ≤ UL | x − U > UL | otherwise |
| lower limit LL only | x − U ≥ LL | x + U < LL | otherwise |
| window [LL, UL] (e.g. response window earliest/latest) | x − U ≥ LL and x + U ≤ UL | x + U < LL or x − U > UL | otherwise |

Equality at the limit counts as "met". A result based on an estimate (direct connection) keeps the label "estimate, not measured". Error rates do not fall under this rule but under the rule of three. Throughout the program, reports and manual the terms "met", "not met", "undecidable" (German „erfüllt“, „nicht erfüllt“, „nicht entscheidbar“) are used uniformly.

**Error rates** by the rule of three: 0 errors in n frames demonstrate an error rate below 3/n at 95 %. The program derives the required test duration from this (example: below 0.1 % requires 3000 error-free frames).

## 5 Device catalog

The catalog is data-driven: every device is a separate YAML file, the program itself contains no device knowledge. New devices do not require a program update.

**Content of a device profile**

- Device class, default address, baud rates, power-on preamble per baud rate, frequency length (5 or 6 bytes)
- Commands with sub-commands, data format, value range, response type, meaning in our own words
- Transmit level per command (see section 7)
- Manufacturer reference points of the meters (e.g. S0/S9/S9+60, PO 0/50/100 %)
- Rated power and power reference points per band, modes with continuous carrier
- Alignment conditions from the service manual (where available)
- Order for restoring settings
- Number of receivers (MAIN/SUB)
- Source reference per entry: document title, edition, page; no link and no text excerpt

**Device classes:** base (common command set), transceiver, receiver, D-STAR, amplifier, accessory, home-built. No class presupposes radio assumptions (e.g. the IC-PW2 has no frequency).

**Data types** for input and decoder: BCD frequency 5/6 bytes, level 0000–0255, mode and filter, bit fields, time/date, text, raw bytes. Entering "14.074 MHz" produces the complete frame; raw bytes are decoded into readable form.

**User-defined commands and command sets** are created in the program and saved as a separate profile file.

**Source rule:** the newest edition is always authoritative; if a document exists in English and German, the English version applies.

**Initial scope**

| Device | Class | Default address | Status in version 1 |
| --- | --- | --- | --- |
| Base profile | base | – | complete |
| IC-7610 | transceiver | 98h | complete, service manual |
| IC-9700 | transceiver | A2h | complete, service manual |
| IC-7300MK2, IC-7760, IC-705, IC-905, IC-2730 | transceiver | B6h, B2h, A4h, ACh, 90h | base commands only, profile completed individually later |
| ID-50, ID-52, ID-52 PLUS, ID-5200 | D-STAR | AEh, A6h, B4h, B8h | base commands only, individually later |
| IC-R8600, IC-R15, IC-R30 | receiver | 96h, B0h, 9Ch | base commands only, individually later |
| IC-PW2 | amplifier | AAh | default address and class, individually later |
| Legacy devices from CI-V manual v3.2 | transceiver/receiver | per device | base commands only |

Decided 2026-10-25: version 1 contains only IC-7610 and IC-9700 complete; further devices are completed individually afterwards, each in its own short work session.

## 6 Sequence editor

Sequences are assembled from structured steps in the user interface, saved as a readable YAML file and executed identically by the desktop and the command-line program. Version 1 has no scripting language; conditions use simple expressions.

| Step type | Purpose |
| --- | --- |
| Send | catalog command or raw bytes, destination and source address freely selectable |
| Expect | exact, pattern with placeholders, decoded value within range, OK/NG, "no response" as the expected result; window earliest/latest; handling of foreign frames; behaviour on failure (retry, abort, continue) |
| Wait | fixed time or until a specific frame appears |
| Variable | take a value from a response and reuse it |
| Loop, condition | repetitions, branching |
| Sub-sequence | reusable building blocks, e.g. "save state" |
| Background task | parallel cyclic polling, e.g. as load |
| Operator prompt | text plus confirmation, with maximum duration |
| Fault injection | deliberately send corrupted frames (section 8) |
| Clean-up block | always runs, including on abort and emergency stop |
| Comment, marker | appears in the recording and the report |

**Parameters per send step:** bus idle since (ms), gap between bytes (ms), behaviour on collision.

**Sequence parameters:** device address, controller address, baud rate and ports are variables, so the same sequence runs on different devices.

**Check before start:** time windows smaller than the measurement uncertainty are reported; transmit-triggering steps are listed (section 7).

**Direct console:** send single commands and see the decoded response, without a sequence.

## 7 Safety

The transmit lock is always active and is never saved; transmission happens only after an explicit manual release for exactly one run in the user interface. The program never transmits on its own.

**Transmit levels in the catalog**

| Level | Examples | with lock active |
| --- | --- | --- |
| triggers RF | `1C 00 01` transmit, `1C 01 02` tune, CW text `17`, transmit voice memory | blocked |
| transmits indirectly | VOX, USB SEND/keying via DTR/RTS | blocked |
| transmit-related parameter | output power, split, transmit frequency | allowed, flagged |

- Raw bytes and user-defined commands are checked against the catalog before sending; unknown commands to transmit-capable devices are treated as "possibly RF-triggering".
- Fault injection and the conformance test never generate patterns that could form a valid transmit command while the lock is active.
- All send paths pass through one single check point in the I/O path. This transmit lock is built in package 8 without any release option; release, emergency stop and watchdogs follow in package 15 and build on it. Before package 8 the program sends only the self-test pattern and `FC`.

**Release**

- The dialog lists every RF-triggering step, maximum power, burst duration and total transmit time.
- Confirmation "dummy load connected".
- Valid for this run only; expires at the end of the run, on abort and on program exit.
- **Command line:** refuses every sequence that contains a transmit-triggering step.

**Emergency stop**

1. While a release is active there are no blocking (modal) dialogs; questions appear as a bar inside the window.
2. A separate, always-on-top emergency-stop window with a large button, independent of the main window.
3. Emergency-stop key (Pause or F12, not Esc) via an application-wide event filter; no input field or dialog can intercept it.
4. Acts directly in the I/O thread: after the current byte, immediately `1C 00 00`, end the sequence, read back the receive state.
5. System-wide emergency-stop key where the operating system allows it (Windows, Linux/X11; not reliable under Wayland, documented as such).
6. No external hardware emergency stop; last resort according to the manual: disconnect the device's power supply.

**Watchdogs**

- A transmit burst ends no later than the maximum duration (calibration: 7 s) with `1C 00 00` and a check.
- If the UI does not respond for more than 1 s while transmitting, the I/O thread ends the transmission by itself.
- The SWR meter (`15 12`) is read during every burst; above the limit (default 2:1) immediate abort.
- On connection loss while transmitting, `1C 00 00` is attempted several times; the report states "transmit state unknown".

**Snapshots and restore**

- Default: before the run the program reads exactly the values the sequence will set. Optional: full snapshot of all readable settings according to the catalog.
- The snapshot is saved as a file before the first set command is sent. After a crash or power loss the program asks at start-up: "Unsaved device state found – restore?"
- Restore at the end of the run, on abort and manually, in the order given by the profile; afterwards read back and compare, deviations go into the report.
- Values that can only be set but not read are marked and not restored.
- Named snapshots serve as the starting state for reproducible tests.

**Adjustment mode:** never supported, not even read-only. The program does not change any alignment values.

## 8 Bundled tests and simulator

All bundled tests are ordinary, editable sequences; capacity and conformance tests are part of version 1.

**Templates**

1. Adapter self-test (section 4)
2. Bus inventory: listen passively, then `19 00` to possible addresses → who is on the bus with which address
3. Baud-rate detection by listening
4. Basic functions per device: read, set, restore, without RF-triggering commands
5. Transceive behaviour: change state, check broadcast
6. Capacity test
7. Long-term test with test duration according to the rule of three
8. Conformance test

**Capacity test** (bus connections only)

- **Theory:** line times at the configured baud rate (19200 Bd: 1 byte 0.52 ms, `03` query 3.1 ms, response 5.7 ms).
- **Phase 1, passive (default 10 min):** bus load, distribution of gaps, periodic foreign masters with period and phase, broadcast volume, participants without recognisable collision avoidance.
- **Phase 2, active:** shorten the polling interval step by step (default 500 → 50 ms, 200 queries each); response rate, latency P95 and collisions per step; determine the knee point.
- **Result:** recommended interval = knee point × 1.5 plus 2 × U; with periodic foreign masters, a recommended phase offset. Note: repeat when the bus population changes.

**Conformance test** (device under test: Icom device or home-built; address and role are specified: responds only, polls itself, both)

- discards corrupted frames (without `FD`, too short, garbage bytes)
- reacts correctly to `FC`
- detects collisions and retries afterwards (the program deliberately generates collisions)
- tolerates baud-rate deviations (default ±2 %)
- survives fault injection without crashing

Every check names its basis (e.g. CI-V manual 5-4) and ends with "met", "not met" or "undecidable".

**Simulator** (general demo devices only, no project-specific ones such as TRXCom or the S/P meter)

- generic transceiver, receiver and amplifier from any catalog profile, with a state model (frequency, mode, antenna, split, transceive on/off, powered off until preamble + `18 01`)
- freely configurable polling participant: address, command, interval, jitter, with or without collision avoidance
- response behaviour: fixed or distributed delay; unknown commands answered with NG or ignored
- several simulated participants on one port; scenarios can be saved
- bus suitability test with two ports (A transmits, B receives at the other end of the line): error rate per baud rate, tolerance to baud-rate deviation
- prerequisite on a real bus: adapter with open-collector output; never against a real device with the same address

**Fault injection** (shares in %): NG instead of OK, no response, late response, frame without `FD`, single `FE` or garbage byte, gap within the frame, deliberate collision.

## 9 S-meter calibration

The guided sequence sets the radio via CI-V to the Icom alignment conditions, takes the user step by step through the 15 IARU levels and saves the table level → raw value. The raw value is read with `15 02` (0–255); this is a processed meter value, not a direct ADC value.

**Level steps (default, IARU Region 1 R.1, level at the load end/PD)**

| Step | below 30 MHz | above 30 MHz |
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
| S9+10 … S9+60 | −63 … −13 dBm in 10 dB steps | −83 … −33 dBm in 10 dB steps |

Optionally S0 (signal generator off) at the start and a check measurement at S9 at the end. Finer intermediate points can be added. The level table follows the **measurement frequency**, not the device (6 m on the IC-7610 = VHF levels).

**Measurement conditions**

| Parameter | Value | Source |
| --- | --- | --- |
| Frequency HF | 14.1515 MHz | IC-7610 service manual |
| Frequency 2 m / 70 cm / 23 cm | 146.0215 / 435.0215 / 1280.0215 MHz | IC-9700 service manual |
| Signal | unmodulated carrier, level at the load end (PD) | service manuals |
| Mode, filter, AGC | CW, 1.2 kHz, AGC MID | own definition (Icom specifies nothing) |
| Preamp, attenuator | OFF; additional tables for PRE1/PRE2 selectable | Icom aligns PRE separately |
| RF gain, NB/NR/notch, PBT | maximum / off / centre | own definition |
| Settling time, readings | 2 s, 20 readings per step | own definition |

**Fallback rule:** devices without their own alignment data in the profile use the IC-7610 conditions on HF/6 m. There is no fallback on VHF/UHF: without a service manual in the profile, the measurement is locked. Substitute values are flagged in the report; if a setting does not exist on the target device, the closest one is used and noted.

**Procedure**

1. Select the device; the profile loads the alignment conditions (shown, changeable).
2. Enter the signal generator: unit (dBm, dBµV, µV, PD or EMF), external attenuation, level accuracy, lowest achievable level.
3. Safety checklist (antenna disconnected, signal generator off).
4. Snapshot, then set up the device and read back.
5. Per step: "Apply −97 dBm (S5)" with the value to set on the signal generator → user confirms → program reads N values → mean and spread → if the spread is too large: repeat, accept (marked) or skip.
6. At any time: back, repeat, intermediate point, abort (partial result is saved as "incomplete").
7. Completion: plausibility (monotonic, saturation, S0 baseline), comparison with Icom reference points, restore, ZIP.

**Special cases**

- IC-7610: Icom aligns S9 and S9+60; these two points are marked as check points for the factory alignment.
- Two receivers (MAIN/SUB): optionally measure both, one table per receiver.
- IC-9700: no S-meter alignment in the service manual; comparison only with the reference points of the Reference Guide.
- Steps below the lowest level of the signal generator are marked "not measurable" beforehand. If the raw value does not differ from the S0 value or does not rise, the point is "not resolvable" and is not taken into the table.

## 10 Power calibration

The program sets the radio's power in percent, the user triggers every transmit burst personally and enters the wattmeter reading together with its inaccuracy. Two tables result: setting → power, and PO raw value (`15 11`) → power.

**Measurement points:** 1–14 % in 1 % steps, 15–100 % in 5 % steps (32 points). The nearest raw value of `14 0A` (0–255) is set; the table lists the nominal value in % and the raw value sent. Optionally the program asks at the first points: "Does the radio show x %?"

**Modes**

| Mode | Setup | Report |
| --- | --- | --- |
| RTTY (default) | no tone; continuous carrier; FM only manually as a fallback | own measurement condition |
| Manufacturer setup | USB; tone generator 1.5 kHz sine into the microphone socket, AF millivoltmeter in parallel; level according to the service manual (IC-7610: 10 mV, IC-9700: 30 mV); mic gain fixed, compressor and VOX off | "according to Icom service manual, mode USB assumed" |
| PC tone via USB audio | USB-D, MOD input USB; program generates 1.5 kHz only during a burst; level in dBFS | own measurement condition |

With manufacturer setup and PC tone the ALC meter (`15 13`) is read as well; points without ALC action are marked "drive too low". Whether RTTY on the IC-7610 delivers a clean continuous carrier is checked during the first test; if PO shows practically nothing at the first point, the program reports "no carrier – check mode".

**Frequency:** HF 14.150 MHz, 6 m 50.150 MHz (away from beacon frequencies), further bands selectable. Icom gives no frequency for the power alignment. The filter width remains unchanged and is only logged.

**Preparation**

1. Select the device (rated power and Icom reference points per band).
2. Power meter: type, ranges, frequency range, coupling or external attenuation. Dummy load: power rating, frequency range. Measurement points outside these limits are locked.
3. Safety checklist: dummy load via power meter on ANT1, no antenna, tuner off, microphone/key/PTT disconnected (except manufacturer setup), USB SEND/keying off.
4. Snapshot, set up the device (frequency, mode, ANT1, tuner/split/VOX off, minimum power), read back, check receive state.
5. Transmit release with emergency-stop window.

**Per measurement point**

1. Program sets x %, shows "Point 7/32: 7 %".
2. User presses "Transmit". Carrier for at most **7 s** with a visible countdown; **"Done"** (large, next to the countdown, also the Enter key) ends earlier. After about 1 s settling time PO is averaged, SWR is monitored.
3. Receive, read back.
4. Input: reading in **W or dBm**; inaccuracy **± % of reading** or **± % of full scale** plus range (pre-filled with the last value).
5. Plausibility check (do raw value and power rise?); the row appears immediately in both tables.
6. Continue only after confirmation; no enforced pause. Non-binding display of accumulated transmit time and average power.
7. If the time ran out without "Done": "Value read?" → enter or repeat.

If the uncertainty of a point exceeds 10 % of the reading, the program asks for a smaller range or marks the point "not accurate enough".

**Completion:** transmit-off verified, snapshot restored, Icom reference points marked (IC-7610: 1 / 10 / 50 W and 100 % = 103 W; IC-9700 per band according to the service manual), uncertainty per point, ZIP. Time required approx. 15–20 min per band.

## 11 Settings profiles

Settings profiles put the transceiver into a verified basic configuration, e.g. for FT8; they are a separate part of the program alongside the measurement sequences. Only the transceiver is configured, not the PC software.

**Questionnaire before applying** (answers are saved as a station profile)

1. Device (detected automatically where possible; afterwards only options the device supports)
2. Control connection: USB port, CI-V jack, LAN/Wi-Fi via Icom Remote Utility, Bluetooth
3. Audio path: USB audio, ACC with external interface, microphone socket
4. Transmit keying: CAT, USB control line (DTR/RTS), ACC SEND, VOX
5. Application: FT8/FT4, RTTY (AFSK or FSK), PSK, SSTV, packet/APRS, Winlink/VARA, remote operation
6. PC software (for hints only): WSJT-X, JTDX, fldigi and others
7. Further devices on the CI-V bus (tuner, amplifier, meter) → Transceive, CI-V Output (for ANT), CI-V USB Port
8. Several programs on the device at the same time?

**Applying**

1. Read the current state, show a preview current → target; write only after confirmation.
2. Automatic snapshot beforehand ("Undo" at any time).
3. Afterwards read back and compare.

**Rules**

- Which menu items exist and how they are encoded comes exclusively from the device's Reference Guide.
- Every value carries its origin: Icom default, empirical value or user. Empirical values are shown as a starting point, not as a default.
- Settings affecting transmission (USB keying, VOX) fall under "transmits indirectly" and require the release.
- Warning for transmit keying via DTR/RTS: the device can go into transmit when other programs open the COM port.
- Recommendations as text, e.g. do not let ALC act in data modes, compressor off, "a COM port can only be used by one program at a time".
- Profiles are shareable files.

**Research before building:** Reference Guides and Advanced Manuals (menu items), Icom set-up notes for data modes, manuals of WSJT-X and fldigi. Values from forums only as "empirical value".

## 12 Results, user interface, languages

Every run and every calibration ends with a ZIP file whose name contains program version, type, device, frequency and time, e.g. `civ-tester4icom_1.0.0_S-Meter_IC-7610_14152kHz_2026-11-02_1930.zip`.

**Content of the ZIP file**

| File | Content |
| --- | --- |
| `bericht.html` / `bericht.pdf` | device under test, adapter and measuring equipment, conditions, results with uncertainty, conformity statement, decision rule, notes, trademark notice |
| `tabelle.csv` | result table (for calibration: level or % → raw value/power with spread, count, markers) |
| `messwerte.csv` | all individual readings |
| `ergebnisse.json` | everything machine-readable |
| `telegramme.csv` | CI-V recording with timestamps |
| Code exports | optional via templates; a sample template for characteristic-curve tables in C is included |

CSV according to RFC 4180, timestamps according to ISO 8601. Recordings can be re-loaded offline in a dedicated format and analysed differently.

**User interface** (draft of 2026-10-23 approved)

1. Sequence editor: project tree, step list, properties, compact bus monitor
2. Device catalog with value input and decoder
3. Run and bus monitor with key figures, histogram, self-test, filters
4. Simulator
5. Capacity test
6. Calibration of S-meter and power (guided)
7. Settings profiles with questionnaire

- Light/dark: "follow system" (default), light or dark selectable as fixed. Status is never conveyed by colour alone, always also by text or symbol.
- Languages: German and English, switchable at runtime, default = system language; reports in the selected language. Code and comments in English, manual in both languages.
- Labels in the UI and the manual are identical (lesson from the CI-V Prüfer: no button that exists only in the manual).

## 13 Repository, publication, build sequence

The repository `civ-tester4icom` becomes public after the first commit with LICENSE, README and .gitignore; all manufacturer documents and our own recordings stay in the private repository `civ-werkstatt`.

**Rules for the public repository**

- Builds and runs completely without the private repository.
- No Icom PDFs, no excerpts from them, no recordings of our own station, no credentials, no local paths; `.gitignore` excludes PDFs, recordings and reports.
- Source references as text only: "Icom, IC-7610 CI-V Reference Guide (2025), p. 12, available on the Icom website".
- Switching to public and enabling secret scanning with push protection is done by the repository owner personally.

**Build sequence (variant B)**

1. Approve this specification.
2. Build the skeleton: transport, bus layer, self-test, bus monitor, builds for all platforms.
3. Intermediate test by the user (approx. 15 min): plug in the adapter, self-test and monitor, report the result.
4. Build the rest in one go, commit after each section.
5. Complete test on the IC-7610, including the first RTTY carrier check at 5 % into the dummy load.

**Version 2 (planned):** Bluetooth LE, first test on the IC-705.

**Later extensions (not planned):** SWR meter calibration, ALC calibration, amplifier current/voltage, hardware probe with timestamps, scripting language, import of logic-analyser recordings.

**Open items**

- [x] Scope of "complete" in the device catalog (section 5)
- [x] English Advanced Manuals for IC-7300, IC-7760 and the ID devices (German ones available)
- [ ] Test adapter for now only the own Pico interface (civ-werkstatt/pico-interface); adapter list for the test matrix later
- [x] Bus recordings as test data (private)
- [ ] Date for the start of construction

**Sources** (private in `civ-werkstatt/doku/referenz`): Icom CI-V Reference Manual v3.2 (2002); CI-V Reference Guides IC-7610 (2025), IC-7300MK2, IC-7760 (2025), IC-9700, IC-705 (2023), IC-905, IC-R8600, IC-R15, IC-2730, IC-PW2, ID-50, ID-52, ID-52 PLUS, ID-5200; service manuals IC-7610 (2017) and IC-9700 (2019); IC-705 Bluetooth/WLAN specification; Advanced Manuals IC-705, IC-R30, ID-4100, ID-5200, ID-52, ID-52 PLUS; IARU Region 1 Recommendation R.1; JCGM 100 (GUM); ILAC-G8:09/2019.

## 14 Build plan in nightly runs

Construction is split into 26 packages, one package per run, on Mondays, Wednesdays and Fridays at 01:30. Version 1.0 is thus complete after about 9 to 10 weeks, plus the days on which a run waits for a test.

> Note 2026-10-25: run control was changed after approval (daily question at 22:00 instead of fixed weekdays). [PAKETPLAN.md](PAKETPLAN.md) is authoritative for run control.

**Course of a run**

1. The scheduled task starts a fresh, short session (not this conversation).
2. It reads `docs/PAKETPLAN.md` in the repository and takes the first open package.
3. If the package is a hold point (⏸) without feedback from the user: no build, only the entry "waiting for test", end.
4. Otherwise: read only the files needed for the package, build, test (simulator, unit tests, build).
5. Commit with package number, entry in `docs/STAND.md` (what was done, what is open, observations), tick the package in `PAKETPLAN.md`.
6. End immediately, even if quota is left. If a package becomes too large, end at a clean point with a commit; the next run continues there.

**A run never:** builds beyond a hold point, builds more than one package, publishes a release or version 1.0, changes the private repository `civ-werkstatt`, brings manufacturer documents into the public repository.

**Package plan** (weeks from start, without waiting times)

| Week | Run | Package | Hold point |
| --- | --- | --- | --- |
| 1 | 1 | Repository foundation: LICENSE, README, .gitignore, structure, CMake |  |
| 1 | 2 | Build pipeline Windows, Linux, Raspberry Pi |  |
| 1 | 3 | Serial transport, DTR/RTS off, port selection |  |
| 2 | 4 | Frame parser, echo, collision, jammer |  |
| 2 | 5 | Adapter self-test, measurement uncertainty |  |
| 2 | 6 | Bus monitor in the UI, recording | ⏸ Intermediate test with the Pico (approx. 15 min) |
| 3 | 7 | Corrections from the intermediate test |  |
| 3 | 8 | Catalog format, data types, decoder, base profile, central transmit lock (no release) |  |
| 3 | 9 | IC-7610 profile part 1 |  |
| 4 | 10 | IC-7610 profile part 2, direct console |  |
| 4 | 11 | IC-9700 profile |  |
| 4 | 12 | Sequence engine: send, expect, wait, variables |  |
| 5 | 13 | Sequence engine: loops, sub-sequences, background, clean-up block |  |
| 5 | 14 | Sequence editor (UI) |  |
| 5 | 15 | Release, emergency stop, watchdogs (building on the transmit lock from package 8) |  |
| 6 | 16 | Snapshots, restore, report and ZIP | ⏸ Test pre-release on the IC-7610 |
| 6 | 17 | Corrections from the pre-release test |  |
| 6 | 18 | TCP, Bluetooth SPP, connection types, command line |  |
| 7 | 19 | Simulator and fault injection |  |
| 7 | 20 | Capacity test |  |
| 7 | 21 | Conformance test |  |
| 8 | 22 | S-meter calibration |  |
| 8 | 23 | Power calibration, tone output |  |
| 8 | 24 | Settings profiles with questionnaire and research |  |
| 9 | 25 | Languages, light/dark, installer, manual | ⏸ Complete test incl. calibration |
| 9–10 | 26+ | Corrections from the complete test | ⏸ Approval of version 1.0 |

**Hold points:** the user reports the test result in a conversation; there it is recorded as an entry in `docs/STAND.md`. Only then does the next run continue building.

**Control by the user**

- After week 1: check the weekly limit in the usage display, then adjust the number of nights (2 to 5 per week).
- Skip single nights, pause or stop: a short message is enough.
- Read the status in the morning: `docs/STAND.md` in the repository or the notification of the scheduled task.

**Before the first run:** specification approved, in the repository as `docs/SPEZIFIKATION.md` together with `PAKETPLAN.md` and `STAND.md`, scheduled task set up, start date agreed.
