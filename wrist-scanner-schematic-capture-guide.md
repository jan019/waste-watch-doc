# Waste Watch — Schematic Capture Guide

Rev 0.2 — the wiring plan, sheet by sheet, with the reference document for each.
Documents referenced:
- **[ARCH]** wrist-scanner-architecture-pinmap.md
- **[PWR]** wrist-scanner-power-block.md
- **[MDM]** wrist-scanner-modem-block.md
- **[NFC]** wrist-scanner-nfc-topboard-block.md
- **[CHK]** wrist-scanner-prefreeze-checklist.md

---

## Project structure

**Base board project — 5 sheets:** Root, Power, MCU, Modem, B2B & Test
**Top board project — 2 sheets:** NFC, HMI & Connector

## Conventions (set before the first symbol)

- Net names: `VBAT`, `VSYS_PP` (BQ24074 OUT), `+3V3`, `VDD_EXT_1V8`; modem signals `M_*`,
  NFC signals `N_*`; shifted vs. unshifted domains named differently (`M_TXD_1V8` module
  side vs. `M_TXD` MCU side) so a bypassed level shifter shows up in the netlist.
- Signals crossing the B2B keep identical names in both projects.
- DNP is an attribute, never an omission — flows into the JLCPCB BOM/CPL. DNP list: second
  modem bulk cap, NFC matching trim pads, button debounce caps, ESD at antenna feed,
  Mode-B handshake shifter, nano-SIM socket (production variant populates the 1NCE MFF2).
- Every probe-worthy net gets a real test point refdes now.

---

## Step 0 — Symbols & footprints  ⟶ [MDM] §10, [NFC] §8, [CHK] §1–2

Custom parts gating everything: **EG915UEUAC** (multi-unit symbol: power / control-UART /
SIM / RF-USB), **PN7160**, **DF40 pair**, **MFF2 + nano-SIM dual footprint**. Pull footprints
from vendor files but verify pad geometry against the mechanical drawings yourself — a bad
module footprint is the most expensive mistake available. Standard-library parts: STM32,
TI power/logic, MAX17048.

## Step 1 — Base root sheet  ⟶ [ARCH] §1 (block diagram)

Five hierarchical blocks, every inter-sheet net named per the conventions. When finished it
should look like the [ARCH] §1 diagram. ~10 minutes; forces the interfaces to be explicit.

## Step 2 — Power sheet  ⟶ [PWR] §§1–5 (all of it)

Pogo pads + TVS/PTC ([PWR] §1) → BQ24074 with the OUT/BAT split topology ([PWR] §0, §2) →
MAX17048 ([PWR] §3) → TPS62840 ([PWR] §4) → TPS22916 modem branch + bulk caps ([PWR] §5,
incl. the battery-pack warning box) → battery connector (JST-SH 3-pin, NTC). Zero
dependencies on other sheets — wire fully, run ERC, done.

## Step 3 — MCU sheet  ⟶ [ARCH] §3 (pin map — this IS the spec)

Transcribe the pin tables, don't re-derive. STM32G0B1RET6 + decoupling, 32.768 kHz crystal
(PC14/15), NRST, Tag-Connect SWD, debug UART header (PA2/3),
I²C pull-ups (bus lives here). When done: diff every symbol pin against the [ARCH]
§3 tables line by line — cheapest moment to catch a transposed port.

## Step 4 — Modem sheet  ⟶ [MDM] §§1–7

Supply + bulk caps ([MDM] §1, values from [PWR] §5) → PWRKEY/RESET_N NPN stages ([MDM] §2)
→ the three fixed-direction level shifters, VCCA = VDD_EXT ([MDM] §3) → dual SIM footprint,
1NCE MFF2 = populated part, socket = DNP ([MDM] §4 + [CHK] §3 decision) → USB test pads +
USB_BOOT ([MDM] §5) → pi network + I-PEX MHF I LK receptacle ([MDM] §6, flex-antenna plan)
→ NETLIGHT test pad, unused pins per guide ([MDM] §7) → **Mode-B handshake: optional DNP
SN74AXC1T45, module GPIO ↔ MCU pin** ([ARCH] §6 item 5; pick a module GPIO that QuecPython
exposes as machine.Pin — verify on the EVB before freezing this net).
Bypass check: no modem signal may reach an MCU pin without a `_1V8` → plain rename.

## Step 5 — B2B & Test sheet  ⟶ [ARCH] §4 + [NFC] §5

Assign physical connector pins: GNDs distributed between groups, SCL/SDA beside a GND, VBAT
doubled if current rating is marginal, spares wired to PA12 + test pads. Export the pinout
table and **freeze it** — it is the contract between the two projects. Collect all test
points here.

## Step 6 — Top board project  ⟶ [NFC] §§1–5

Copy the connector pinout verbatim from Step 5. NFC sheet: PN7160 core ([NFC] §1) → EMC
filter + matching + RX footprints, placeholder values, NP0 only ([NFC] §3) → 27.12 MHz
crystal → DWL_REQ pad. Antenna drawn as a 2-pin symbol; the loop is a layout object
([NFC] §2 — the ferrite sheet and enclosure shorted-turn rule live there and in the
mechanical spec). HMI sheet: 9 switches, 10 LEDs + resistors, debounce DNP pads, mating
DF40 ([NFC] §4 — includes the IP67 keymat + proud-rim construction notes).

## Step 7 — Cross-checks before "wired"

1. ERC clean, both projects.
2. B2B pinout tables from both projects diffed against each other.
3. One full read-through per sheet with its block doc open beside it, ticking every table
   row — tedious, catches more than ERC.
4. Then the value-verification pass: work [CHK] §1 top to bottom before ordering anything.

---

Effort estimate: Step 0 ≈ a day (module symbols are slow); Steps 1–5 ≈ a day; Step 6 ≈ a
half day; Step 7 ≈ 2 h. Anything that contradicts the docs during capture — a pin the
symbol has that the spec doesn't mention, a strap the datasheet wants that isn't covered —
gets resolved against the block doc before it fossilizes into the netlist.

---

## Appendix A — Complete port lists per reuse block

Create each block with ALL ports in one sitting (minimizes Update-Block-Symbols churn).
Convention: **GND is a global net** via power symbols on every sheet — no GND ports.
The rails (+3V3, VBAT, VBAT_MODEM) ARE ports, so the contract stays explicit.
Wide groups use bus notation: `BTN[0..8]`, `LED[0..7]`.

### Block: POWER  (13 ports)

| Port | Dir | Goes to | Note |
|---|---|---|---|
| +3V3 | out | MCU, B2B | TPS62840 output |
| VBAT | out | B2B | raw battery node — PN7160 TX supply on top board |
| VBAT_MODEM | out | MODEM | post-TPS22916 load switch |
| MODEM_SW_EN | in | MCU (PA7) | load-switch enable, default-on pull |
| CHG_N | out | MCU (PA0) | BQ24074 /CHG, open-drain + pull-up |
| PGOOD_N | out | MCU (PA1) | BQ24074 /PGOOD = dock detect |
| GAUGE_ALRT_N | out | MCU (PA15) | MAX17048 ALRT |
| I2C1_SCL, I2C1_SDA | bidir | MCU, B2B | MAX17048 sits on the shared bus |
| *(bus expansion)* | | | BTN/LED buses do not touch this block |

Internal only (no ports): VCHG pogo input, VSYS_PP (charger OUT → buck), battery
connector (B+/NTC/B−), TS, ISET/ILIM/TMR.

### Block: MCU  (37 ports)

| Port | Dir | Goes to | Note |
|---|---|---|---|
| +3V3 | in | POWER | |
| I2C1_SCL, I2C1_SDA | bidir | POWER, B2B | pull-ups live on this sheet |
| M_TXD | out | MODEM | PA9 → shifter → module RXD |
| M_RXD | in | MODEM | module TXD → shifter → PA10 |
| M_PWRKEY | out | MODEM | PB14 → NPN stage |
| M_RESET_N | out | MODEM | PB11 → NPN stage |
| M_DTR | out | MODEM | PB15, sleep control |
| M_RI | in | MODEM | PB12, EXTI12 |
| M_STATUS | in | MODEM | PB13 |
| M_HSK | bidir | MODEM | PD9 (EXTI9) — B-side of DNP Mode-B handshake shifter |
| MODEM_SW_EN | out | POWER | PA7 |
| CHG_N | in | POWER | PA0 |
| PGOOD_N | in | POWER | PA1 |
| GAUGE_ALRT_N | in | POWER | PA15 |
| N_IRQ | in | B2B | PA8, EXTI8 |
| N_VEN | out | B2B | PB10 |
| BTN[0..8] | in | B2B | PB0–PB7 + PA11; internal pull-ups |
| LED[0..7] | out | B2B | PC0–PC7 |
| STAT_A, STAT_B | out | B2B | PA4, PA5 |
| SPARE0 | bidir | B2B | PA12 (9th-button LED / reserve) |

Internal only: SWD/Tag-Connect, debug UART (PA2/PA3) header, PA6 spare test pad,
32.768 kHz crystal, NRST, decoupling.

### Block: MODEM  (10 ports)

| Port | Dir | Goes to | Note |
|---|---|---|---|
| VBAT_MODEM | in | POWER | bulk caps at module on this sheet |
| +3V3 | in | POWER | shifter VCCB side |
| M_TXD | in | MCU | |
| M_RXD | out | MCU | |
| M_PWRKEY | in | MCU | |
| M_RESET_N | in | MCU | |
| M_DTR | in | MCU | |
| M_RI | out | MCU | |
| M_STATUS | out | MCU | |
| M_HSK | bidir | MCU | module GPIO ↔ DNP AXC1T45 |

Internal only: VDD_EXT_1V8 (generated by module, consumed by shifter VCCA — never
leaves this sheet), all `*_1V8` shifted nets, USIM lines + dual SIM footprint, USB
test pads + USB_BOOT, pi network + I-PEX, NETLIGHT test pad.

### Block: B2B_TEST  (28 ports)

| Port | Dir | Goes to | Note |
|---|---|---|---|
| +3V3 | in | POWER | to connector |
| VBAT | in | POWER | to connector (double pins if rating tight) |
| I2C1_SCL, I2C1_SDA | bidir | MCU, POWER | |
| N_IRQ | out | MCU | from connector |
| N_VEN | in | MCU | to connector |
| BTN[0..8] | out | MCU | from connector |
| LED[0..7] | in | MCU | to connector |
| STAT_A, STAT_B | in | MCU | to connector |
| SPARE0 | bidir | MCU | to connector spare pin |

Internal only: the DF40 itself, remaining spare pins → test pads, all test points
collected from other nets (test points connect via net labels, not ports).

### Top-board project (mirror contract)

| Block | Ports |
|---|---|
| NFC | +3V3, VBAT, I2C1_SCL, I2C1_SDA, N_IRQ, N_VEN (6) — antenna/matching/crystal/DWL_REQ internal |
| HMI_CONN | +3V3, VBAT, I2C1_SCL, I2C1_SDA, N_IRQ, N_VEN, BTN[0..8], LED[0..7], STAT_A, STAT_B, SPARE0 (28) — holds the mating DF40, so the NFC-bound signals pass through it |

Port names crossing the B2B are IDENTICAL in both projects — diff the two DF40 pin
tables as the final contract check (capture guide Step 7).
