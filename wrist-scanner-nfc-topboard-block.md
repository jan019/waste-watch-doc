# Waste Watch — PN7160 NFC + Top Board Block Specification

Rev 0.1 — companion to architecture/pin-map, power, and modem documents.
Chip decision: **PN7160** (I²C variant, HVQFN40 preferred for assembly/sourcing — confirm
the exact orderable part number and package with LCSC stock before footprint work).

---

## 0. Block summary (top board)

```
        ┌────────────────────────── TOP BOARD ──────────────────────────┐
        │  ╔═ NFC loop, 2 turns, board perimeter (~48 × 36 mm) ═╗       │
        │  ║   ┌───────────────────────────────────────────┐    ║       │
        │  ║   │  9 tact switches   10 LEDs (+resistors)   │    ║       │
        │  ║   │                                           │    ║       │
        │  ║   │  ┌──────────┐  EMC filter + matching      │    ║       │
        │  ║   │  │  PN7160  ├──────────────────────────►──╫────╢       │
        │  ║   │  │ 27.12MHz │  (TX1/TX2 diff + RX tap)    │    ║       │
        │  ║   │  └───┬──────┘                             │    ║       │
        │  ║   └──────┼────────────────────────────────────┘    ║       │
        │  ╚══════════╪═════════════════════════════════════════╝       │
        │             │ B2B connector: 3V3, VBAT, GND×n, SCL, SDA,      │
        │             │ IRQ, VEN, BTN0–8, LED0–7, STATUS A/B, spares    │
        └─────────────┼──────────────────────────────────────────────────┘
                      ▼ base board (MCU I2C1 = PB8/PB9, IRQ = PA8, VEN = PB10)
```

---

## 1. PN7160 core connections

| Signal | Connection | Notes |
|---|---|---|
| VBAT | Battery node via B2B connector | 10 µF + 100 nF at pin; supplies the TX driver — field strength tracks battery voltage, verify read range at 3.5 V, not just full charge |
| PVDD | 3V3 rail | Host-interface I/O domain matches MCU — no shifting needed |
| SCL / SDA | I2C1 (PB8/PB9) through connector | Pull-ups live on the base board (2.2–4.7 kΩ); keep only the chip on this stub |
| IRQ | → PA8 (EXTI8) | Active-high per NCI convention; no pull needed (push-pull) — verify polarity/config in datasheet |
| VEN | ← PB10 | Full power-down control; VEN low = chip off (≈ zero current) |
| DWL_REQ | Test pad + 10 kΩ pull-down (+ optional wire to PA12 spare) | Forces firmware-download mode; you'll want it the day an NFC-controller FW update goes sideways |
| I2C address straps | Strap per datasheet for default address | Only one NFC device on the bus |
| XTAL1/XTAL2 | 27.12 MHz crystal + load caps per datasheet CL | Tight placement; keep away from the loop antenna trace |
| Unused RF/misc | Per datasheet — do not ground unknowns | |

---

## 2. Loop antenna (the actual engineering here)

**Geometry starting point:** 2 turns around the top-board perimeter, outer dimension
~48 × 36 mm, trace 0.8 mm, gap 0.5 mm, corners chamfered. Expected inductance ≈ 0.5–0.8 µH
— inside the PN7160's comfortable range. One turn will likely be too low-L, three probably
unnecessary; the board layout should make dropping to 1 or up to 3 turns a copper-only
change (keep the matching footprints generic).

**Rules inside and under the loop:**

1. **No copper pours/planes inside the loop area on the top board.** Route button/LED
   signals as thin individual traces, no closed conductive loops, crossings roughly
   perpendicular to the antenna trace.
2. **The base board is the problem.** It sits 2–4 mm below the loop, full of ground pour —
   a classic eddy-current sink that will crush the Q and detune the antenna. Two
   mitigations, in order of preference:
   - **Ferrite sheet** (sintered, 0.1–0.2 mm, e.g. Würth WE-FSFS class) laminated on the
     top board's underside, under the loop track ring. Budget it in the BOM and stack
     height *now* — it's the difference between 3–4 cm read range and "must touch the tag
     perfectly."
   - Copper pullback on the base board's outer 3–4 mm ring (helps, but fights with the
     base board's own layout needs; do it where free).
3. **Enclosure rule (write into the ID/mechanical spec):** no closed conductive ring
   concentric with the loop — no metal bezel, trim ring, or continuous metal gasket
   carrier around the face. A closed ring is a shorted turn and will collapse the field
   entirely. No metal faceplate or dome-carrier sheet over the aperture; discrete tact
   switches, silicone keymat, plastic light pipes, and small corner screws are all fine.
4. Measure L and Q **of the real assembled stack** (both boards mated, battery in, inside
   the enclosure mock-up) with the VNA before computing matching values. The loop in free
   air is a different antenna.

**Read-range reality check:** a wrist device tapping an NFC tag on a bin wants reliable
coupling at 0.5–2 cm through a glove — comfortably achievable with this loop size *if* the
ferrite sheet is in place. Test with the actual bin tags (tag antenna size matters as much
as ours).

---

## 3. EMC filter + matching network (footprints now, values later)

Standard NXP differential topology — place all footprints, compute values with the NXP
antenna design tool from measured L/Q:

```
TX1 ──[L0]──┬──[Cs]──┬───────────────► loop ►───────────┐
           [C0]     [Cp]                                 │
            ⏚        ⏚                                   │
TX2 ──[L0]──┬──[Cs]──┬───────────────► loop ◄───────────┘
           [C0]     [Cp]
                     RX ◄──[Crx]──[Rrx]── tap at antenna node (per NXP app note)
```

| Group | Footprints | Starting ballpark (replace after measurement) |
|---|---|---|
| EMC filter | L0 ×2 (0603, high-SRF), C0 ×2 (0402 NP0) | L0 ≈ 470–560 nH, C0 ≈ 180–270 pF; cutoff per NXP app note for the chip's TX mode |
| Matching | Cs ×2, Cp ×2 (0402 NP0, ±2 %) | From the design tool; give each position a parallel DNP pad for trim without respins |
| RX | Crx, Rrx | Per PN7160 app-note values |
| Damping | Optional series R across loop (DNP) | Only if Q is too high and pulse shape fails |

All NP0/C0G dielectric in the RF path — no X7R here. This mirrors your PN7150 lock design
methodology; only the computed values and the chip's app note change.

---

## 4. Buttons (×9) and LEDs (×10)

| Item | Spec |
|---|---|
| Switches | 6×6 or 6×3.5 mm SMT tact, ≥ 160 gf actuation (glove use — firmer is better), IP-rated variants if the enclosure doesn't seal the plungers itself |
| SOS | Same switch, but the enclosure gives it a recessed/guarded cap; FW long-press ≥ 1.5 s |
| Wiring | One side to BTNx line, other to GND; MCU internal pull-ups; optional 100 nF DNP pads per button for HW debounce if EMC testing shows glitches |
| LEDs | 0603, high-brightness (visible in factory lighting through light pipes); series R ≈ 1 kΩ from 3V3-driven GPIO (~1.5–2.5 mA, plenty with a proper light pipe) |
| LED map (proposal) | LED0–4 = fill levels, LED5 = clean, LED6 = dirty, LED7 = SOS armed/sent, STATUS A = battery/charge, STATUS B = network/upload. PA12 spare covers an "empty" LED if the scheme needs all nine |
| Layout | All switches/LEDs inside the antenna loop; thin traces, no loops, no pours (see §2) |

Mechanical coupling to the enclosure (plunger alignment, light pipes, membrane vs. discrete
caps) will drive exact switch positions — freeze the enclosure button layout before routing
this board.

**IP67 construction (decided):** single molded silicone keymat over all nine plungers +
light-pipe windows, perimeter flange compressed between top shell and internal ledge — one
gasket seals the whole face; SOS guard ring molded into the same mat. **Tap-surface rule:**
the enclosure rim stands ~1–1.5 mm proud of the button tops so tapping the watch flat
against a bin tag lands on the rim, not the keys; firmware additionally ignores button
transitions while the NFC field is engaged with a tag. Back-side pogo pads: hard gold
plating, slight dome/drain geometry so water doesn't pool on the contacts.

---

## 5. Board-to-board connector (final assignment)

DF40-class, 0.4 mm pitch, 34–40 pos; pick stack height to match the battery + component
clearance. Assignment principle: GND pins distributed among signal groups; SCL/SDA adjacent
to a GND; VBAT (NFC TX supply) gets two pins if the connector current rating is marginal.
Keep one pin group as spares wired to PA12 + test pads.

---

## 6. Scan interaction — hardware supports both models (decide before FW freeze)

| Model | How this HW does it | Power cost |
|---|---|---|
| A: Button-first ("select fill %, then tap bin to commit") | VEN low all shift; button press → VEN high → NCI init (~tens of ms) → poll → read tag → VEN low | NFC ≈ 0 between scans — negligible |
| B: Tap-first (field always sniffing) | VEN high all shift; PN7160 LPCD/ULPCD cycles autonomously, IRQ wakes MCU on tag | LPCD average ≈ 10–20 µA class — still small, but nonzero and config-sensitive |

Model A is the recommendation from the workflow standpoint (unambiguous commit action, no
accidental scans brushing past tags) and it's also the simpler firmware. Nothing in the
schematic changes either way — VEN and IRQ are wired for both.

---

## 7. Firmware porting notes (PN7150 → PN7160)

1. Transport (I²C framing, IRQ/VEN handshake) — unchanged from your lock driver.
2. NCI init — update to NCI 2.0 CORE_RESET/CORE_INIT flow; contained change.
3. RF configuration — regenerate with NXP's config tool for the new chip *and* the new
   antenna; do not carry the lock's RF settings blob over.
4. ISO-DEP / DESFire layer — untouched; the EV1 three-pass AES auth ports as-is (relevant
   if bin tags are ever DESFire; for plain NTAG/Type 2 bins it's simpler still).
5. Add DWL_REQ handling to your provisioning jig scripts for controller FW updates.

---

## 8. Verify against PN7160 datasheet/app notes before freeze

- Exact package/orderable part number availability (HVQFN40 vs VFBGA) at LCSC/Mouser.
- IRQ polarity/configuration default, I²C address strap pins, DWL_REQ logic level.
- PVDD range confirmation for 3.3 V host domain; VBAT min for acceptable TX field at 3.5 V.
- Crystal load capacitance and drive level; whether your clock could instead come from an
  external source (not worth it here — keep the crystal).
- NXP antenna design app note number for PN7160 (tool inputs: measured L, Q, target
  operating distance) and the recommended EMC cutoff for its TX mode.
- Ferrite sheet: permeability/loss spec at 13.56 MHz, adhesive thickness, min bend radius.
