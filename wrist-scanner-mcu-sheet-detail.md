# Waste Watch — MCU Sheet: Detailed Component Specification

Rev 0.1 — component-level wiring for the MCU reuse block. Pin assignments come from
[ARCH] §3 (transcribe, don't re-derive); this document covers everything AROUND the MCU
that the pin map doesn't: decoupling, clock, reset, debug, bus passives.
Reference designator ranges: U1x, C1xx, R1xx, Q1xx, Y1xx, J1xx, D1xx.

---

## U101 — STM32G0B1RET6 (LQFP64)

### Supply pins & decoupling

| Item | Value / part | Placement rule |
|---|---|---|
| Per VDD-class pin | 100 nF X7R 0402, one per pin | < 2 mm from pin, via to GND at the cap pad |
| Bulk | 4.7 µF X7R 0603, one | Near the main VDD pin |
| VDDA | 100 nF + 1 µF X7R 0402/0603 | VDDA is the ADC/analog reference — even unused, decouple it properly |
| VDDA feed | 0 Ω 0402 from +3V3 (DNP alternative: ferrite bead BLM15 series) | Footprint lets you add a bead later if ADC ever gets used and is noisy |

Wire **every** VDD/VSS pair the LQFP64 provides, plus VDDA/VSSA — and check the G0B1RE
pinout for additional supply pins on this package (e.g. a separate USB/IO supply domain);
if present, tie to +3V3 with its own 100 nF. This is part of the same datasheet pass that
verifies PC/PD port extent and PD9.

### Reset & boot

| Item | Detail |
|---|---|
| NRST | 100 nF to GND close to pin; net to Tag-Connect pin 3. Internal pull-up exists — no external resistor |
| BOOT0 | **No strap needed.** STM32G0 boots via nBOOT_SEL/nBOOT0 option bits — configure once in production programming. Do not copy a BOOT0 resistor from older STM32F designs |

### Clock

| Item | Value / part | Notes |
|---|---|---|
| Y101 | 32.768 kHz crystal, **CL = 7 pF**, 3215 package (e.g. Epson FC-135R 7 pF class) | 7 pF load = lowest LSE drive/power; the RTC runs 24/7 |
| C1xx ×2 | 5.6–6.8 pF NP0 0402 | From C = 2·(CL − C_stray), C_stray ≈ 3–4 pF. **Recalculate against the exact crystal's CL before ordering** |
| Layout | PC14/PC15 traces < 5 mm, GND guard ring, nothing switching adjacent | Keep LED lines and the buck's field away |
| System clock | Internal HSI16 (+PLL if needed) — **no HSE crystal** | One less part, no accuracy requirement justifies it; UART at 115200 from HSI is fine across temp |

LSE drive level: configure LSEDRV low/medium-low in firmware for the 7 pF crystal.

---

## Debug & programming

### J101 — Tag-Connect TC2030 footprint (no connector BOM cost)

| TC2030 pad | Signal | Net |
|---|---|---|
| 1 | VCC sense | +3V3 |
| 2 | SWDIO | PA13 |
| 3 | nRESET | NRST |
| 4 | SWCLK | PA14 |
| 5 | GND | GND |
| 6 | NC | — (G0 has no SWO) |

Footprint: TC2030-NL (no-legs) — pads only, zero height, fits the IP67 enclosure story
(programming happens before final assembly; field access is not required once OTA works).
Rule: PA13/PA14 carry nothing else. They default to SWD after reset — never repurpose.

### J102 — Debug UART

PA2 (USART2_TX) / PA3 (USART2_RX) + GND → 3 test pads or a 1×3 2.54 mm footprint (pads
for production, header populated on EVT units). Optional 100 Ω series resistors (R1xx,
DNP) if EMC testing ever fingers this stub.

---

## I²C bus passives (bus "lives" on this sheet)

| Ref | Value | Net |
|---|---|---|
| R110 | 2.2 kΩ 0402 | I2C1_SCL → +3V3 |
| R111 | 2.2 kΩ 0402 | I2C1_SDA → +3V3 |

2.2 kΩ (not 4.7 k) because the bus crosses the B2B connector to the top board — extra
capacitance from connector + two boards; 2.2 k keeps rise times comfortable at 400 kHz
Fast-mode. Bus members: PN7160 (top board), MAX17048 (power sheet), LIS2DW12 (this sheet,
optional). One set of pull-ups total — nothing on the other sheets.

## LIS2DW12 accelerometer (optional-populate)

| Pin | Connection |
|---|---|
| VDD, VDD_IO | +3V3, 100 nF each |
| SCL/SDA | I2C1 bus |
| INT1 | PC13 (EXTI13) |
| INT2, unused | NC |
| SDO/SA0 | GND (sets I²C address LSB) |
| CS | +3V3 (I²C mode) |

Mark the whole cluster DNP-optional as a group in the BOM (chip + its two caps).

---

## What is deliberately NOT on this sheet

- Button pull-ups (internal, configured in FW), button debounce caps (DNP pads on top board)
- LED series resistors (top board, at the LEDs)
- Level shifters, NPN stages (modem sheet)
- I²C devices other than the accel (their sheets), their decoupling (at each device)

## Unused-pin policy

Any GPIO the pin map doesn't assign (spare PD/PF pins the LQFP64 turns out to have):
leave unconnected in schematic, configure as analog-input in firmware init (lowest power,
no floating-input current). Do NOT ground them.

## ERC / capture pitfalls specific to this sheet

1. **PA11/PA12 remap trap**: the G0 family can remap PA9/PA10 onto the PA11/PA12 pins via
   SYSCFG. We use all four as distinct signals (PA9/10 UART, PA11 BTN8, PA12 SPARE0) —
   firmware must leave the remap OFF. Add a schematic text note; this is a firmware
   contract, not a wiring change.
2. Give the crystal nets explicit names (LSE_IN/LSE_OUT) so they're findable in layout.
3. The 37 block ports: after wiring, read the port list against capture-guide Appendix A
   once, aloud, before touching the root sheet.

## Block BOM summary (excl. ports/labels)

| Qty | Part | Refs |
|---|---|---|
| 1 | STM32G0B1RET6 LQFP64 | U101 |
| ~8 | 100 nF 0402 X7R | C101… (per VDD pin + NRST + accel) |
| 1 | 4.7 µF 0603 | C110 |
| 1+1 | 1 µF + 100 nF (VDDA) | C111, C112 |
| 1 | 0 Ω 0402 (VDDA feed, bead-optional) | R101 |
| 1 | 32.768 kHz 7 pF 3215 | Y101 |
| 2 | 5.6–6.8 pF NP0 0402 | C115, C116 |
| 2 | 2.2 kΩ 0402 | R110, R111 |
| 1 | TC2030-NL footprint | J101 |
| 1 | 1×3 pads/header (debug UART) | J102 |
| 1 | LIS2DW12 + 2×100 nF (optional group) | U102, C130, C131 |
