# Waste Watch — Power Block Schematic Specification

Rev 0.1 — companion to the architecture/pin-map document
Covers: dock input → charger → battery → rails, with component values and layout notes.

---

## 0. Topology decision (read this first)

The BQ24074 is a **power-path** charger: loads on its OUT pin run from the dock while the
battery charges independently with correct termination. But OUT can sit at ~4.4 V when
docked, and the **EG915U's absolute max supply is 4.3 V** — so the modem cannot hang off OUT.

Split the loads:

- **OUT → TPS62840 buck → 3V3** (MCU, NFC logic, gauge, LEDs). Logic gets full power-path
  benefit: the watch boots and runs on the dock even with a dead battery.
- **BAT → EG915U (via load switch) + PN7160 TX supply.** The battery never exceeds 4.2 V,
  safely under the modem's 4.3 V limit.

Consequence: modem current drawn while docked flows through the BAT node and can blur the
charger's end-of-charge detection — the charger terminates when BAT-pin current tapers below
I_CHG/10 = 25 mA and cannot tell battery current from load current, so any BAT-node load must
stay well under 25 mA for termination to work. Modem sleep (~1.5 mA) satisfies this 15× over.
**Decision: handled in firmware** (HW alternatives — clamping LDO, lower charge voltage, or a
dedicated buck-boost — each cost brown-out margin, capacity, or RF-adjacent complexity for a
failure mode the safety timer already backstops). **Firmware rule: while /PGOOD is active,
keep the modem in sleep and stretch upload intervals.**

```
Pogo 5V ──► ESD/TVS ──► BQ24074 IN                 OUT ──► TPS62840 ──► 3V3
                              │                     │
                             BAT ◄──► LiPo 503035 (PCM + NTC)
                              │
                              ├──► TPS22916 load switch ──► EG915U VBAT (+ bulk caps)
                              └──► PN7160 VBAT (TX driver, via B2B connector)
                             TS ◄── pack NTC        MAX17048 VDD ◄── BAT
```

---

## 1. Dock input & protection

| Ref | Part / value | Purpose |
|---|---|---|
| J-PADS | 2× pogo target pads (5 V, GND), gold ENIG, ≥Ø3 mm | GND pad larger / annular around mount if possible — reduces mis-mate risk |
| D-TVS | SMF5.0A (or PESD5V0X1BT for space) 5 V ↔ GND | ESD + surge clamp on exposed contact |
| F1 | PTC resettable fuse, 750 mA hold (e.g. 0603/1206 polyfuse) | Short-circuit protection on exposed pad |
| C-IN | 4.7 µF X7R 25 V + 100 nF at BQ24074 IN | Input decoupling |

The BQ2407x input tolerates well above 5 V with internal OVP, so a mis-wired or noisy dock
supply won't kill the charger. Dock side: any regulated 5 V ≥ 500 mA supply.

---

## 2. Charger — BQ24074 (VQFN-16)

| Pin | Connection | Value / note |
|---|---|---|
| IN | From PTC/TVS input | |
| OUT | To buck input | C-OUT: 10 µF X7R + 100 nF |
| BAT | Battery + (JST-SH 3-pin pack connector: B+, NTC, B−) | C-BAT: 10 µF X7R |
| ISET | R-ISET to GND | **3.57 kΩ 1 % → I_CHG ≈ 250 mA (0.5 C)**; formula I_CHG = K_ISET / R_ISET, K_ISET ≈ 890 A·Ω — verify constant in your datasheet rev |
| ILIM | R-ILIM to GND | **3.09 kΩ 1 % → input limit ≈ 500 mA**; I_INmax = K_ILIM / R_ILIM, K_ILIM ≈ 1550 A·Ω — verify |
| TS | Pack NTC (10 k, β≈3435) to GND | Buy packs **with NTC lead** — daily-cycled wearable deserves temp-qualified charging. If a no-NTC pack is forced, 10 kΩ fixed resistor to GND defeats the check (not preferred) |
| TMR | R-TMR to GND | Size per datasheet formula for a **≈4 h safety window** (~29 kΩ ballpark — confirm K_TMR from datasheet before ordering) |
| /CHG | → MCU PA0 | Open-drain; 100 kΩ pull-up to 3V3 |
| /PGOOD | → MCU PA1 | Open-drain; 100 kΩ pull-up to 3V3; doubles as dock-detect |
| SYSOFF | Tie to GND | Normal power-path operation |
| EN1/EN2 | Strap per datasheet for "use ILIM resistor" mode | |

Why 250 mA / 0.5 C: fills the 500 mAh cell in ~2.5 h (meets the "couple of hours in the
dock" requirement), keeps the pack cool, and preserves cycle life on a battery that will be
cycled ~250×/year.

---

## 3. Fuel gauge — MAX17048 (TDFN-8)

| Pin | Connection |
|---|---|
| VDD | BAT node, 100 nF decoupling close to pin (it senses cell voltage through VDD) |
| SDA / SCL | I2C1 bus (PB9 / PB8) — pull-ups already on the bus, 2.2–4.7 kΩ to 3V3 |
| ALRT | → MCU PA15, 100 kΩ pull-up to 3V3; configure low-SOC alert (e.g. 15 %) |
| QSTRT | GND (or per datasheet default) |

ModelGauge needs no sense resistor — one less layout headache. Report SOC in every server
upload; drive the low-battery LED from the ALRT interrupt even in STOP mode.

---

## 4. 3V3 rail — TPS62840 buck

| Item | Value |
|---|---|
| VIN | From BQ24074 OUT; 10 µF X7R input cap |
| VOUT | 3.3 V (set via VSET pin resistor/strap per datasheet variant) |
| L | 2.2 µH, ≥1 A Isat, small 2016/2520 power inductor |
| C-OUT | 22 µF X7R (6.3 V, ≥0805 to keep effective C under DC bias) |
| EN | Tie to VIN — always on |
| MODE/STOP | GND (auto PFM) — the 60 nA I_Q + PFM efficiency at µA loads is exactly why this part is here |

Load budget: MCU ≤ 25 mA active, PN7160 PVDD ≤ 20 mA, LEDs ≤ 40 mA worst case, gauge/accel
≈ µA → < 100 mA peak, far inside the part's rating.

---

## 5. Modem power branch (BAT → EG915U)

| Ref | Part / value | Note |
|---|---|---|
| U-SW | TPS22916 (or SiP32431) load switch, ≥2 A | EN ← MCU PA7, default ON (pull-up on EN); this is the "unstick a wedged modem" hammer, not a duty-cycle tool |
| C-BULK | **330 µF polymer** (6.3 V, low ESR) at EG915U VBAT | Polymer, not MnO₂ tantalum — consistent with your fill-sensor/lock component policy; add a second footprint (DNP) for 470 µF if field testing shows droop |
| C-RF | 100 nF + 33 pF + 10 pF close to module VBAT pins | Per Quectel reference design |
| Trace | VBAT route ≥ 1.5 mm wide, short | 1 A-class bursts; measure droop at the pin during TX, spec ≥ 3.4 V at battery cutoff |

Brown-out math: at 3.5 V battery and ~150 mΩ total path (PCM + switch + trace), a 1 A burst
drops ~0.15 V → 3.35 V at the module — right at the edge at end-of-discharge. This is why the
PCM choice, load-switch R_on, and trace width all get a line item here; keep total path
resistance ≤ 150 mΩ and set the firmware low-battery lockout (no uploads, LED warning) at
~3.5 V / ~10 % SOC.

> ⚠️ **BATTERY PACK SELECTION REQUIREMENT — brown-out budget.** The 503035 pack MUST be
> selected with the protection circuit (PCM) resistance in mind: require the vendor to
> spec **PCM path resistance ≤ 80 mΩ** (protection FETs + sense) and confirm the cell's
> pulse capability (≥ 2 C / 1 A bursts) at 0 °C. Cheap packs with 200–300 mΩ PCMs will
> consume the entire 150 mΩ path budget alone and cause modem brown-outs below ~30 % SOC —
> a failure that looks like random reboots in the field and is unfixable without a battery
> swap. Measure the actual pack resistance (1 A load step, scope at the connector) during
> incoming inspection of the first batch, and re-verify at −10 °C if cold-storage factories
> are a target environment.

PN7160 TX supply taps the same BAT node through the B2B connector: give it its own 10 µF +
100 nF on the top board; TX bursts are ~100 mA and brief.

---

## 6. Layout notes (base board, power corner)

1. Charger + buck + bulk caps cluster near the pogo pads / battery connector — keep the
   high-current loop (pads → charger → battery) tight and off the antenna end of the board.
2. The EG915U bulk cap bank sits at the module, not at the charger.
3. Kelvin-ish sense: MAX17048 VDD trace taps the battery connector pin, not the far end of
   the BAT plane.
4. NTC wires stay away from the buck inductor.
5. Nothing from this block intrudes into the LTE antenna keep-out zone at the far short edge.

---

## 7. Firmware hooks defined by this block

- /PGOOD (PA1) low → docked: enter charge UI mode, modem to sleep, stretch uploads.
- /CHG (PA0) → charge-in-progress LED state; both high while docked = charge complete.
- MAX17048 ALRT (PA15) → low-battery warn at 15 %, hard lockout of modem TX below ~10 %.
- PA7 low pulse (≥ module off-time) → full modem power cycle after N failed AT recoveries.

---

## 8. To verify against datasheets before schematic freeze

- K_ISET, K_ILIM, K_TMR constants and EN1/EN2 strapping for the exact BQ24074 variant.
- TPS62840 fixed-voltage variant vs. VSET-strapped — pick whichever LCSC actually stocks.
- PCM internal resistance of the chosen 503035 pack (goes into the 150 mΩ budget).
- EG915U-EU hardware design guide: exact recommended cap set and VBAT min under burst.
