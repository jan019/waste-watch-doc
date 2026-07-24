# Waste Watch — System Architecture & STM32G0B1 Pin Map

Rev 0.1 — pre-schematic planning document
Target MCU: **STM32G0B1RET6** (LQFP64, 512 KB flash dual-bank, 144 KB RAM, pin-compatible with STM32G070RBT6)

---

## 1. System block diagram

```
                          ┌────────────────────────── TOP BOARD ─────────────────────────┐
                          │                                                              │
                          │   NFC loop antenna (perimeter, ~48×36 mm)                    │
                          │      │  EMC filter + matching                                │
                          │   ┌──┴───────┐    ┌──────────┐   ┌───────────┐               │
                          │   │  PN7160  │    │ 8 buttons│   │ 8+2 LEDs  │               │
                          │   │ (I2C+IRQ │    │ (to GND) │   │ (GPIO-    │               │
                          │   │  +VEN)   │    │          │   │  driven)  │               │
                          │   └──┬───────┘    └────┬─────┘   └────┬──────┘               │
                          └──────┼─────────────────┼──────────────┼──────────────────────┘
                                 │   board-to-board connector (~30–40 pin, e.g. DF40)
                          ┌──────┼─────────────────┼──────────────┼──────────────────────┐
                          │      │ I2C1 + IRQ/VEN  │ GPIO in      │ GPIO out             │
                          │      │ + VBAT + 3V3    │              │                      │
                          │   ┌──┴──────────────────────────────────────┐                │
                          │   │            STM32G0B1RET6                │                │
                          │   │                                         │                │
                          │   │  USART1 + 1.8V level shifter ───────────┼──► EG915U-EU   │
                          │   │  GPIO: PWRKEY / RESET_N / DTR /         │    (LTE Cat    │
                          │   │        STATUS / RI ─────────────────────┼──► 1bis)       │
                          │   │  I2C1: MAX17048 fuel gauge,             │      │eSIM MFF2│
                          │   │        LIS2DW12 accel (optional)        │      │LTE ant  │
                          │   │  Spare I/O ──► test pads               │      └─keep-out│
                          │   │  USART2 ──► debug header                │                │
                          │   │  SWD ──► Tag-Connect                    │                │
                          │   └─────────────────────────────────────────┘                │
                          │                                                              │
                          │   Power:  pogo pads (5V/GND/detect) ─► TVS ─► BQ24074        │
                          │           BQ24074 ──► LiPo 503035 500mAh (PCM) = VBAT        │
                          │           VBAT ──► load switch ──► EG915U VBAT (+bulk caps)  │
                          │           VBAT ──► PN7160 TX supply (via connector)          │
                          │           VBAT ──► TPS62840 buck ──► 3V3 (MCU, logic, LEDs)  │
                          │                                            BASE BOARD        │
                          └──────────────────────────────────────────────────────────────┘
```

**Note on PN7160 placement:** the NFC controller lives on the **top board**, next to its antenna. Only I²C + IRQ + VEN + power cross the connector — clean digital/DC signals instead of a tuned RF path. If top-board space runs out, fall back to PN7160 on the base board with the antenna feed crossing the connector (works, but the matching must then account for connector parasitics).

---

## 2. Power tree

| Rail | Source | Loads | Notes |
|---|---|---|---|
| VCHG (5 V) | Dock pogo pads | BQ24074 input | TVS + series protection on exposed pads |
| VBAT (3.0–4.2 V) | LiPo 503035, 500 mAh w/ PCM (BQ24074 BAT pin) | EG915U (via load switch), PN7160 TX supply | ≥220 µF low-ESR bulk at EG915U VBAT pins; never exceeds 4.2 V < module's 4.3 V max |
| OUT (power-path) | BQ24074 OUT | TPS62840 buck input | Logic runs from dock power even with a flat battery |
| 3V3 | TPS62840 buck (always-on) | MCU, PN7160 PVDD, MAX17048, LIS2DW12, LEDs, level shifter B-side | Buck chosen for light-load efficiency (sleep dominates) |
| 1V8 (VDD_EXT) | **From EG915U** | Level shifter A-side reference only | Do not load beyond a few mA; it also tells the MCU the module is alive |

Charge current: set BQ24074 to ~250 mA (0.5 C) → full charge in ~2.5 h, good cycle life for daily cycling.

---

## 3. STM32G0B1RET6 pin map

### Port A

| Pin | Function | Dir | Notes |
|---|---|---|---|
| PA0 | /CHG from BQ24074 | in | Open-drain, pull-up to 3V3; polled |
| PA1 | /PGOOD from BQ24074 | in | Dock-present detect; polled |
| PA2 | USART2_TX (debug) | out | Debug header |
| PA3 | USART2_RX (debug) | in | Debug header |
| PA4 | Status LED A (e.g. charge) | out | On-board or via connector |
| PA5 | Status LED B (e.g. network) | out | |
| PA6 | Spare (test pad) | — | Freed — haptic feature removed |
| PA7 | EG915U load-switch EN | out | Hard power recovery only; normally on |
| PA8 | PN7160 IRQ | in | **EXTI8** |
| PA9 | USART1_TX → modem RXD | out | Through 3V3↔1V8 shifter |
| PA10 | USART1_RX ← modem TXD | in | Through shifter |
| PA11 | BTN 8 — Dirty | in | **EXTI11**, internal pull-up, switch to GND |
| PA12 | Spare (LED 8 / RTS) | — | HW flow control dropped — not needed at 115200 baud AT link |
| PA13 | SWDIO | — | Tag-Connect |
| PA14 | SWCLK | — | Tag-Connect |
| PA15 | MAX17048 ALRT | in | **EXTI15**, low-battery interrupt |

### Port B

| Pin | Function | Dir | Notes |
|---|---|---|---|
| PB0 | BTN 0 — Clean | in | **EXTI0**, internal pull-up, switch to GND |
| PB1 | BTN 1 — Empty | in | **EXTI1** |
| PB2 | BTN 2 — 0 % | in | **EXTI2** |
| PB3 | BTN 3 — 25 % | in | **EXTI3** |
| PB4 | BTN 4 — 50 % | in | **EXTI4** |
| PB5 | BTN 5 — 75 % | in | **EXTI5** |
| PB6 | BTN 6 — 100 % | in | **EXTI6** |
| PB7 | BTN 7 — SOS | in | **EXTI7**; firmware requires long-press; consider recessed cap |
| PB8 | I2C1_SCL | — | PN7160, MAX17048, LIS2DW12; 2.2–4.7 kΩ pull-ups to 3V3 |
| PB9 | I2C1_SDA | — | |
| PB10 | PN7160 VEN | out | NFC enable/reset |
| PB11 | EG915U RESET_N | out | Open-drain drive (NPN/MOSFET per Quectel ref design) |
| PB12 | EG915U RI | in | **EXTI12**, wake on incoming data/URC |
| PB13 | EG915U STATUS | in | Polled; module power state |
| PB14 | EG915U PWRKEY | out | Open-drain drive per ref design |
| PB15 | EG915U DTR | out | Sleep control (DTR high = module may sleep) |

### Port C

| Pin | Function | Dir | Notes |
|---|---|---|---|
| PC0–PC7 | LED 0–7 (button indicators) | out | ~2–4 mA each, resistor per LED; via connector |
| PC13 | LIS2DW12 INT1 | in | **EXTI13**, wake-on-motion (optional part) |
| PC14 | OSC32_IN | — | 32.768 kHz crystal for RTC |
| PC15 | OSC32_OUT | — | |

### Port D

| Pin | Function | Dir | Notes |
|---|---|---|---|
| PD9 | M_HSK — Mode-B handshake | in (default) | **EXTI9**; B-side of DNP SN74AXC1T45; default direction module→MCU ("event pending" from QuecPython app), DIR strap reversible by resistor. Module side: GPIO25/Pin39 nominated, pending HW-guide cross-check + EVB test ([MDM] §11). **Verify PD9 present on LQFP64 in the G0B1RE pinout table** |

EXTI check: lines 0–9, 11, 12, 13, 15 used, each from exactly one port — no conflicts.
Boot: STM32G0 boots via nBOOT_SEL option bits — no external BOOT0 strap needed.

---

## 4. Board-to-board connector (base ↔ top)

Suggested part class: Hirose DF40 / Molex SlimStack, 0.4 mm pitch, 30–40 pos, stack height per enclosure.

| Group | Signals | Count |
|---|---|---|
| Power | 3V3, VBAT (PN7160 TX supply), GND ×3–4 | 5–6 |
| NFC | SCL, SDA, IRQ, VEN | 4 |
| Buttons | BTN0–BTN8 (incl. Dirty) | 9 |
| LEDs | LED0–LED7, STATUS A, STATUS B | 10 |
| Spare | future use / extra GND | 2–4 |

Total ≈ 30–33 → pick a 34- or 40-pin connector for margin.

---

## 5. Bill of major components

| Block | Part | Package | Notes |
|---|---|---|---|
| MCU | STM32G0B1RET6 | LQFP64 | Dual-bank flash → clean OTA |
| Cellular | Quectel EG915U-EU | LGA | Cat 1bis, single antenna; sleep-registered strategy |
| SIM | eSIM MFF2 (soldered) | MFF2 | No socket = reliability + space |
| NFC | NXP PN7160 | VFBGA/HVQFN | NCI, same driver family as PN7150 |
| Charger | TI BQ24074 | VQFN16 | Power-path, ISET → 250 mA |
| Fuel gauge | MAX17048 | TDFN | I²C, battery % to LED + server |
| Buck 3V3 | TPS62840 | tiny WCSP/SOT | 60 nA Iq — matters in sleep |
| Load switch | e.g. SiP32431 / TPS22916 | small | EG915U hard-reset path |
| Level shifter | SN74AXC2T45 ×2 (or 4-ch AXC4T774) | tiny | UART + control lines 3V3↔1V8 |
| Accel (opt.) | LIS2DW12 | LGA12 | Wake-on-motion, future man-down |
| Battery | LiPo 503035, 500 mAh, with PCM | 30×35×5 mm | Multi-vendor commodity size |
| LTE antenna | Chip (Ignion/Johanson) or flex in shell | — | Keep-out zone at board short edge; VNA tune on-arm |
| Crystal | 32.768 kHz, 12.5 pF | 3215 | RTC timestamps for offline buffering |
| ESD | TVS on pogo pads, USB/debug lines | — | Exposed-contact protection |

---

## 6. Firmware-relevant hardware decisions (baked into this map)

1. **Modem stays registered, sleeping via DTR** — SOS latency ~1–2 s, ~1.5 mA sleep cost. Load switch (PA7) is a recovery hammer, not a duty-cycle tool.
2. **All 9 buttons on EXTI** — MCU sits in STOP mode; any button, NFC IRQ, RI, or accel motion wakes it.
3. **Scan records buffered in internal flash** with RTC timestamps — survives coverage holes and reboots; uploaded in batches.
4. **Fuel gauge ALRT** wired to EXTI → low-battery warning LED even in sleep.
5. **Dual-brain option preserved (Mode A / Mode B).** Mode A (default): STM32 runs the
   application, AT commands to stock module FW. Mode B: module runs QuecPython application;
   STM32 acts as I/O + NFC coprocessor speaking an event/command protocol over the same
   UART (DTR wake, PWRKEY/STATUS wiring identical; QuecPython flashed via the USB test
   pads). NFC stays on the STM32 in both modes (C driver, deterministic timing). Hardware
   delta: one optional DNP interconnect — module GPIO ↔ MCU pin via an extra SN74AXC1T45 —
   as a Mode-B handshake line. Firmware rule: layer the STM32 FW so the I/O/NFC service
   layer exposes one internal API consumed by either the local app layer (A) or a UART
   bridge (B). Ship one mode; keep the other as prototyping path and fallback.

---

## 7. Open questions before schematic capture

1. ~~Clean vs. dirty UX~~ — **resolved: dedicated Dirty button (BTN 8 on PA11), 9 buttons total.**
2. **"Empty" vs. "0 %"** — are these distinct events (bin emptied vs. bin observed empty)? Affects nothing electrically, but confirms the button count.
3. **LED scheme** — one LED per button vs. a smaller set (5 fill LEDs + clean/dirty + status)? Current map reserves 10 lines (+PA12 spare for a 9th button LED), so either works.
4. **Dock contacts** — 2 pads (5 V/GND) or 3 (＋dock-ID/detect)? Map assumes detect comes from /PGOOD, so 2 pads suffice.
5. **Enclosure ingress/temp rating** — IP54? IP67? Determines button/seal choices and whether the debug header is a Tag-Connect footprint only.
