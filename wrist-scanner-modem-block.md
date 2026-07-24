# Waste Watch — EG915U-EU Modem Block Specification

Rev 0.1 — companion to architecture/pin-map and power block documents.
Signals are referenced by Quectel net name; take exact pad numbers from the EG915U
Hardware Design Guide (keep the current rev next to you during capture).

---

## 0. Block summary

```
                       ┌──────────────────────────────────────────┐
 BAT ──► TPS22916 ──►──┤ VBAT (all pins tied, bulk caps at pins)  │
                       │                                          │
 MCU PA9  ─► shifter ─►┤ MAIN_RXD          ANT_MAIN ├─ pi-net ─► LTE antenna
 MCU PA10 ◄─ shifter ◄─┤ MAIN_TXD                                 │
 MCU PB15 ─► shifter ─►┤ DTR (sleep ctrl)  USIM_* ◄──► eSIM MFF2  │
 MCU PB12 ◄─ shifter ◄─┤ RI                        (+ dual nano-  │
 MCU PB13 ◄─ shifter ◄─┤ STATUS                     SIM footprint │
 MCU PB14 ─► NPN     ─►┤ PWRKEY                     for dev)      │
 MCU PB11 ─► NPN     ─►┤ RESET_N                                  │
                       │ VDD_EXT (1.8 V ref out) ─► shifter VCCA  │
                       │ USB_DP/DM/VBUS/BOOT ─► test pads         │
                       └──────────────────────────────────────────┘
```

---

## 1. Supply (see power doc §5 for the branch upstream)

| Item | Value / rule |
|---|---|
| VBAT pins | Tie all VBAT pads together on a wide pour; all GND pads to solid ground with via stitching under the module |
| Bulk | 330 µF polymer + footprint for second (DNP) — placed at the module, not the charger |
| RF decoupling | 100 nF + 33 pF + 10 pF, closest to VBAT pads, per Quectel reference |
| Supply range check | Module spec 3.4–4.3 V; our BAT node is 3.0–4.2 V — firmware lockout at ~3.5 V keeps us off the 3.4 V floor under burst (power doc §5 warning applies) |

---

## 2. Power-on / reset control (PWRKEY, RESET_N)

Both are driven with the standard Quectel open-collector stage — never push-pull from a
3.3 V GPIO (the pins are internally pulled to a VBAT-domain rail):

```
MCU GPIO ─ 4.7 kΩ ─┬─ base MMBT3904 (or any small NPN / N-FET)
                   └─ 47 kΩ ─ GND        collector → PWRKEY (or RESET_N), emitter → GND
```

- **PWRKEY (PB14):** MCU drives GPIO high → transistor pulls PWRKEY low. Hold per design
  guide timing (typ. ≥ 500 ms to power on; longer pulse for power off — take exact values
  from the guide). Note the logic inversion: GPIO high = key pressed.
- **RESET_N (PB11):** identical stage. Emergency use only; normal recovery order in FW is
  AT-retry → PWRKEY cycle → RESET_N → PA7 load-switch cut.
- Add 100 nF from each control pin to GND close to the module (glitch immunity).

---

## 3. Level shifting (1.8 V ⇄ 3.3 V)

The EG915U logic I/O is **1.8 V (VDD_EXT domain)**. A 1.8 V high into a 3.3 V STM32 input
does **not** meet VIH (0.7 × 3.3 = 2.31 V) — every modem→MCU signal must be shifted, not
just the MCU→modem ones.

| Part | Channels | Direction | Signals |
|---|---|---|---|
| SN74AXC2T45 #1 | 2 | modem→MCU (single DIR strap) | MAIN_TXD → PA10, RI → PB12 |
| SN74AXC2T45 #2 | 2 | MCU→modem (single DIR strap) | PA9 → MAIN_RXD, PB15 → DTR |
| SN74AXC1T45 | 1 | modem→MCU | STATUS → PB13 |
| SN74AXC1T45 (**DNP**) | 1 | modem→MCU default, DIR strap reversible | M_HSK: module GPIO25/Pin39 (nominated) → **PD9 (EXTI9)** — Mode-B handshake, populate only if Mode B ships |

- **Channels are grouped by direction, deliberately**: the 2T45 has ONE DIR pin for both
  bits, so each package must point one way — which is why the UART pair is split across
  the two packages. DIR is a direction-control input (A→B vs B→A) meant for runtime-
  switched buses; here it's strapped as a build-time constant per the datasheet table,
  never driven. Grouping by direction means 2× the identical part number, one glance-
  checkable strap per package, and no per-bit strap traps (the alternative — an
  SN74AXC2T245 with per-bit DIR holding TXD+RXD together — works but adds a second part
  number and a silent-failure strap risk for purely cosmetic layout benefit).

- VCCA = **VDD_EXT** (from the module — this is deliberate: shifter outputs float safe when
  the module is off, and VDD_EXT presence itself tells the MCU the module is alive).
  Load on VDD_EXT is µA-level, far under its budget. 100 nF decoupling on VCCA and VCCB.
- VCCB = 3V3.
- Do **not** substitute TXB/TXS auto-direction parts here — they misbehave with the modem's
  drive strengths and slow edges; fixed-direction AXC parts are the boring, reliable choice.
- MCU firmware: configure PA10/PB12/PB13 without pull-ups (shifter drives them); treat
  "VDD_EXT absent → shifter outputs low" as "module off" in the state machine.

---

## 4. SIM — eSIM MFF2 with dev-friendly dual footprint

| Item | Detail |
|---|---|
| Production part | eUICC MFF2 (soldered), ordered **pre-provisioned** with your MVNO/roaming profile — confirm provisioning path with the SIM vendor before committing; a soldered SIM you can't provision is a paperweight |
| Dev provision | Place a **dual footprint**: MFF2 pads *and* a push-push nano-SIM socket wired to the same USIM lines. Populate the socket on dev/EVT units, the MFF2 in production. Never populate both |
| Wiring | USIM_VDD, USIM_DATA, USIM_CLK, USIM_RST, GND per design guide |
| Passives | 33 pF to GND on DATA/CLK/RST near the module (RF immunity); series 0–22 Ω resistors optional per guide |
| ESD | TVS array (e.g. low-cap ESDA type) only needed at the *socket* variant — pads exposed during handling |

---

## 5. USB test interface (do not skip this)

Route USB_VBUS, USB_DP, USB_DM, GND and USB_BOOT to **test pads** (or a 5-pad Tag-Connect-
style footprint). This is your path for:

- Quectel module firmware updates (QFlash) — modules ship with assorted FW revs and you
  *will* need to normalize them,
- AT console + network debugging with QNavigator/QCOM during bring-up,
- emergency download mode via USB_BOOT if a module FW update bricks.

DP/DM as a 90 Ω differential pair, kept short; VBUS pad fed externally from the debug rig
(do not connect to the watch's 5 V — the dock pads are power-only).

---

## 6. Antenna feed — **I-PEX connector + flex antenna in the case (primary plan)**

| Item | Detail |
|---|---|
| Connector | I-PEX MHF I **LK (locking)** receptacle on ANT_MAIN; if plain MHF I, design a foam press-pad in the lid over the mated plug. Strain-relieve/tape the pigtail |
| ANT_MAIN trace | 50 Ω CPWG, short: module → pi network → connector |
| Matching | Pi network placeholder retained (series 0 Ω + two shunt DNP) — flex antennas still need trimming to the real enclosure + arm; iterate by swapping antenna candidates + retuning, no respin |
| Antenna | Small adhesive flex monopole/IFA, ~35–45 mm class, LTE EU bands incl. **B20 (800 MHz — key for factory indoor penetration)**. **These are ground-referenced**: the base board ground plane remains the counterpoise — keep it solid and maximal; check each candidate's minimum-ground-plane spec against the ~50 mm board |
| Enclosure keep-out (replaces the PCB keep-out) | Antenna zone: plastic only, no metal features, no battery behind it — radiator on the outer face / outward side wall, battery toward the skin side; ≥ a few mm from the NFC loop plane |
| Production test | EOL check: network registration + RSSI threshold → catches unmated/forgotten pigtails |
| Expectations | Low-band efficiency on a watch-sized device is physics-limited regardless of antenna type; the flex + away-from-arm placement buys a few dB over a chip antenna, not phone-grade performance. Pilot in a real factory early |
| ESD | Optional DNP shunt ESD footprint at the feed — populate only if ESD testing demands it |
| Future cost-down | At volume, a stamped/LDS antenna in the case with spring-finger contacts removes the cable + hand assembly — tooling spend for later, not v1 |

---

## 7. Misc module pins

| Pin | Treatment |
|---|---|
| NETLIGHT | Test pad only (1.8 V domain). Network state comes to the MCU via URC/AT instead of burning a shifter channel |
| ADC | Unconnected (battery telemetry comes from MAX17048) |
| All unused I/O | Leave floating per Quectel guide (do not ground unknowns) |

---

## 8. Firmware bring-up crib (defined by this hardware)

1. Enable load switch (PA7 high default) → wait VDD_EXT present → pulse PWRKEY → wait
   STATUS asserted → AT sync at 115200 (send `AT` until echo; then lock baud, disable echo).
2. Sleep mode: `AT+QSCLK=1`, then DTR high (PB15) lets the module sleep registered;
   DTR low wakes it. RI (PB12, EXTI12) signals incoming data/URCs during sleep — configure
   RI behavior via `AT+QCFG` during provisioning.
3. SOS path: DTR low → socket open → POST → confirm → DTR high. Target < 2 s registered.
4. Recovery ladder: 3× AT timeout → PWRKEY off/on cycle → RESET_N pulse → PA7 power cut
   (≥ modem off-time) → repeat with backoff. Log each escalation in the next upload.

---

## 9. Certification note (for the CE/RED file)

EG915U-EU carries its own RED certification as a module, which simplifies but does not
remove end-product obligations: EN 301 489-1/-52 EMC applies to the final watch (same
family as the fill-sensor work), plus SAR consideration — this is a **body-worn
transmitter**, so check EN 50566/EN 62209-2 applicability early rather than at test-lab
booking time. Antenna gain must stay within what Quectel's module cert assumed.

---

## 10. Verify against the current EG915U Hardware Design Guide before freeze

- Exact PWRKEY/RESET timing values and whether RESET_N is required or optional to wire.
- VDD_EXT current budget (assumed ≥ 10 mA here; we draw µA).
- Recommended USIM series/shunt values and whether the guide's reference includes ESD parts.
- Bulk capacitance recommendation for Cat 1bis TX (our 330 µF + DNP 470 µF should exceed it).
- Any RF pins (ANT_DIV/GNSS variants) present on the exact ordered part number — order code
  matters; confirm the -EU data-only variant string with your distributor.

---

## 11. QuecPython GPIO ↔ physical pin mapping (EG915U) — for the Mode-B handshake pad

Source: developer.quectel.com → QuecPython API reference → machine.Pin, module selector =
EG915U series. **Identification anchor:** Quectel support confirmed for EG915U-EU that
physical pin P114 = GPIO39 — which matches this table. Cross-verify on the EVB REPL and
with the module selector before freeze (the doc page stacks all module families' tables;
selecting the wrong module is the main hazard).

| GPIO number | Physical pin | Shared-I/O constraint |
|---|---|---|
| GPIO1 | Pin4 | not together with GPIO41 |
| GPIO2 | Pin5 | not together with GPIO36 |
| GPIO3 | Pin6 | not together with GPIO35 |
| GPIO4 | Pin7 | not together with GPIO24 |
| GPIO5 | Pin18 | — |
| GPIO6 | Pin19 | — |
| GPIO7 | Pin1 | not together with GPIO37 |
| GPIO8 | Pin38 | — |
| GPIO9 | Pin25 | — |
| GPIO10 | Pin26 | — |
| GPIO11 | Pin27 | not together with GPIO32 |
| GPIO12 | Pin28 | not together with GPIO31 |
| GPIO13 | Pin40 | — |
| GPIO14 | Pin41 | — |
| GPIO15 | Pin64 | — |
| GPIO16 | Pin20 | not together with GPIO30 |
| GPIO17 | Pin21 | — |
| GPIO20 | Pin30 | — |
| GPIO21 | Pin88 | — |
| GPIO22 | Pin36 | not together with GPIO40 |
| GPIO23 | Pin37 | not together with GPIO38 |
| GPIO24 | Pin16 | not together with GPIO4 |
| GPIO25 | Pin39 | — |
| GPIO26 | Pin42 | not together with GPIO27 |
| GPIO27 | Pin78 | not together with GPIO26 |
| GPIO28 | Pin83 | not together with GPIO33 |
| GPIO30 | Pin92 | not together with GPIO16 |
| GPIO31 | Pin95 | not together with GPIO12 |
| GPIO32 | Pin97 | not together with GPIO11 |
| GPIO33 | Pin98 | not together with GPIO28 |
| GPIO34 | Pin104 | — |
| GPIO35 | Pin105 | not together with GPIO3 |
| GPIO36 | Pin106 | not together with GPIO2 |
| GPIO37 | Pin108 | listed as "not with GPIO4" in the source — by pairing logic (GPIO7↔Pin1) this is likely a doc typo for GPIO7; treat both as suspect until EVB-verified |
| GPIO38 | Pin111 | not together with GPIO23 |
| GPIO39 | Pin114 | — (anchor: confirmed = P114 by Quectel support) |
| GPIO40 | Pin115 | not together with GPIO22 |
| GPIO41 | Pin116 | not together with GPIO1 |

Notes: GPIO18, GPIO19 and GPIO29 are absent from the EG915U table (not mapped). "Shared-
I/O constraint" = the two GPIO numbers drive the same internal I/O; use at most one of the
pair and record the exclusion in the schematic.

**Handshake pad selection procedure:**
1. Cross this table against the EG915U Hardware Design Guide pinout; strike every row whose
   physical pin is USIM, MAIN UART, USB, PWRKEY/RESET_N/STATUS/DTR/RI, VDD_EXT, or RF.
2. From the survivors, prefer an unconstrained row (no shared-I/O note, no boot/strap
   role in the HW guide, marked as plain GPIO).
3. Record BOTH numbers in the schematic net name/comment, e.g. `M_HSK (GPIO25 / Pin39)`.
4. EVB verification before freeze: `from machine import Pin;
   Pin(Pin.GPIOn, Pin.OUT, Pin.PULL_DISABLE, 0).write(1)` with a meter on the pad.
5. If the module→MCU direction ever reverses (module as interrupt input), re-check the pad
   against QuecPython's ExtInt support — not every mapped GPIO is interrupt-capable.

**Selection (pending EVB verification):** M_HSK = module **GPIO25 / Pin39** ↔ DNP
SN74AXC1T45 ↔ MCU **PD9 (EXTI9)**. Default direction module→MCU; DIR strap via resistor
option. Verify: (a) Pin39 free of conflicting function in the HW design guide, (b) GPIO25
drives on the EVB REPL, (c) PD9 exists on the LQFP64 G0B1RE pinout.
