# Waste Watch — Consolidated Pre-Freeze Checklist

Rev 0.1 — every open verification, decision, and external dependency from the four block
documents (architecture/pin-map, power, modem, NFC/top-board), in one place.
Work top to bottom; items marked ⛔ block schematic freeze, items marked 🔶 block layout,
items marked 🟢 can slip to bring-up.

---

## 1. Datasheet lookups (desk work, ~half a day total)

- [ ] ⛔ **BQ24074**: confirm K_ISET, K_ILIM, K_TMR constants and EN1/EN2 strapping for
      "resistor-set current limit" mode → finalize R-ISET (target 250 mA), R-ILIM
      (target 500 mA), R-TMR (target ≈4 h safety window)
- [ ] ⛔ **TPS62840**: pick fixed-3.3 V variant vs. VSET-strapped — decide by what LCSC
      actually stocks (see §2); confirm inductor recommendation (2.2 µH assumed)
- [ ] ⛔ **EG915U HW Design Guide**: exact PWRKEY/RESET_N timings; whether RESET_N wiring
      is mandatory; VDD_EXT current budget; recommended VBAT bulk capacitance for Cat 1bis
      TX (our 330 µF + DNP 470 µF must exceed it); USIM series/shunt passive values
- [ ] ⛔ **PN7160 datasheet**: IRQ polarity default, I²C address straps, DWL_REQ logic
      level, PVDD 3.3 V host-domain confirmation, VBAT minimum for acceptable TX field,
      crystal load capacitance
- [ ] ⛔ **PN7160 antenna app note**: note number + NXP design tool inputs; recommended
      EMC filter cutoff for its TX mode
- [ ] 🔶 **STM32G0B1RET6 datasheet**: confirm LQFP64 pin availability for every assignment
      in the pin map (PA/PB/PC0–7/PC13–15 used — sanity-check against the actual pinout
      table, especially PC port extent) — **and confirm PD9 exists on LQFP64** (M_HSK,
      EXTI9); fallback if absent: PA12 poll-only (EXTI12 taken by M_RI)

## 2. Sourcing / stock checks (LCSC + distributor, ~2 h)

- [ ] ⛔ EG915U-EU orderable part number — **decided: EG915UEUAC-N05-SNNSA (JLCPCB stock,
      assembled in-line)**. Residual checks: (a) confirm with Quectel FAE that **QuecPython
      firmware exists for the EUAC revision** — if not, Mode B is Mode-A-only on production
      hardware unless EUAB is consigned; (b) ask FAE about the reported **onboard eUICC**
      (AT+QESIM) on EG915U-EU — could replace the external MFF2 footprint; (c) note the
      Core EVB is EUAB → firmware branches differ from production units (AT behavior
      identical, but re-verify timing/quirks on an EUAC sample before pilot)
- [ ] 🟢 **EG800K-EU / EG800AK-EU dual evaluation** — order a sample/EVB alongside the
      EG915U EVB; the factory site survey decides: if LTE-only coverage is solid in target
      facilities AND QuecPython-EU support checks out, EG800K becomes the v2 size/cost/
      power optimization (17.7 × 15.8 mm, no 2 A GSM bursts → relaxed brown-out budget);
      if any site shows marginal LTE, the EG915U's 2G fallback is justified with data
- [ ] ⛔ PN7160 package availability (HVQFN40 preferred) and orderable number at
      LCSC/Mouser; check price/stock vs. the VFBGA variant
- [ ] ⛔ STM32G0B1RET6 stock (the G0B1 has had allocation waves before — if dry, the
      G0B1RCT6 256 KB is the fallback, still dual-bank)
- [ ] 🔶 TPS62840 variant per §1; SN74AXC2T45 ×2 + AXC1T45; TPS22916; MAX17048; BQ24074
- [ ] 🔶 B2B connector — **selected: DF40C-50DP-0.4V(51) plug on base board (LCSC
      C424645) + DF40HC(3.5)-50DS-0.4V(51) receptacle on top board (fallback: DF40HC(3.0),
      C3652838) → 3.5 mm (or 3.0 mm) mated height.** Verify live JLCPCB assembly stock of
      the chosen receptacle before layout. Mechanical requirements: (a) DF40 is NOT
      polarized — screw bosses/standoffs MUST be asymmetric so reversed stacking is
      physically impossible; (b) standoffs carry all shock/shear loads (C/HC series has no
      reinforcing metal fittings); (c) at 3.0 mm height, top-board underside keep-out
      directly above the EG915U (2.4 mm tall) is mandatory
- [ ] 🔶 Ferrite sheet for the NFC loop (13.56 MHz spec: permeability/loss, adhesive,
      0.1–0.2 mm) — Würth WE-FSFS class; get cut samples
- [ ] 🟢 LTE **flex antenna** candidates (2–3 adhesive monopole/IFA types, ~35–45 mm, EU
      bands incl. B20) with I-PEX pigtails + MHF I **LK** locking receptacles; check each
      vendor's minimum-ground-plane spec against the ~50 mm base board
- [ ] 🟢 **Quectel Core EVB** (EG915UEUAB-CORE-QPYTHON-EVB) ×1–2 as dev tooling:
      (a) validate eSIM/MVNO profile + registration, (b) prototype & time the AT sleep/
      wake/SOS sequences from modem doc §8, (c) measure real TX burst currents against the
      power budget, (d) on-site RSSI/RSRP survey per band (esp. B20) in a real customer
      factory. Also the Mode-B (QuecPython) prototyping platform

## 3. Vendor conversations (start now — longest lead times)

- [ ] ⛔ **Battery pack vendor**: 503035, 500 mAh, PCM **≤ 80 mΩ path resistance**
      (written spec, not verbal), ≥ 2 C pulse at 0 °C, **NTC lead** (3-wire JST-SH),
      cycle-life data at 0.5 C daily cycling. Get 2 vendors quoted — this part has the
      brown-out budget riding on it
- [ ] ⛔ **eSIM/MVNO** — **decided for v1: populate the 1NCE MFF2 eSIM (already in JLCPCB
      stock) on the dual footprint** → functional out of the box, no RSP infrastructure
      needed for EVT. Dual footprint rule: MFF2 *or* nano-SIM socket populated, never both;
      keep a few EVT units with the socket instead for arbitrary test SIMs. Parallel
      track (cost-down, not blocking): (a) FAE — do EUAC units ship with a GSMA-provisioned,
      QESIM-enabled onboard eUICC (real EID, not FFFF)? (b) 1NCE — can they deliver SGP.22
      profiles with activation codes for download to third-party eUICCs? Both yes in
      writing → SIM parts drop from the BOM in a later revision
- [ ] 🔶 **Test lab (CE/RED)**: body-worn transmitter → does SAR (EN 62209-2 / EN 50566)
      apply at EG915U power class? Answer influences antenna placement and max-power
      config — needed before layout, not after. Same conversation: confirm EN 301 489-1/
      -52 plan (family work with the fill sensor) + EN 301 489-3 for the NFC side
- [ ] 🟢 Dock supply: any regulated 5 V ≥ 500 mA; decide dock connector/pogo pin part
      (mating side) together with the enclosure design

## 4. Design decisions still open

- [ ] ⛔ **"Empty" vs. "0 %" buttons** — distinct events or redundant? (Electrically done
      either way — 9 buttons wired — but firmware/server semantics need the answer)
- [ ] 🔶 **LED scheme** — proposal on the table: LED0–4 fill, LED5 clean, LED6 dirty,
      LED7 SOS, STATUS A battery, STATUS B network; PA12 spare covers a 9th if needed.
      Confirm or edit
- [ ] 🔶 **Enclosure ingress/temp rating** (IP54 vs IP67; cold-storage factories yes/no?)
      → switch sealing choice, debug via Tag-Connect-only, battery cold-pulse verification
      temperature (§5)
- [ ] 🟢 **Scan interaction model** — A: button-first (recommended) vs B: tap-first/LPCD.
      Hardware supports both; decide before firmware freeze, can A/B on pilots

## 5. Measurements & tests (need hardware/samples)

- [ ] 🔶 Battery pack incoming inspection procedure: 1 A load step, scope at connector,
      total path resistance ≤ 150 mΩ budget check; repeat at −10 °C if cold environments
      are in scope
- [ ] 🟢 NFC loop L/Q measurement **of the mated stack in the enclosure mockup** (VNA) →
      NXP tool → matching values (footprints already generic + DNP trim pads)
- [ ] 🟢 LTE antenna on-arm VNA characterization + matching iteration (expect 2–3 passes);
      verify against the gain assumed by the module's certification
- [ ] 🟢 EG915U VBAT droop measurement at the module pin during TX burst at 3.5 V battery
- [ ] 🟢 NFC read range with **actual bin tags** through a glove at 0.5–2 cm; verify at
      3.5 V battery, not just full charge

## 6. Mechanical gates (block top-board routing)

- [ ] 🔶 Enclosure button layout frozen (plunger positions, SOS recess/guard, light pipes)
      → gates switch placement and therefore top-board routing
- [ ] 🔶 Stack-up drawing: top board / B2B height / base board / battery / back shell with
      pogo pads — confirms connector stack height (§2) and ferrite sheet clearance
- [ ] 🔶 50 × 38 mm wearability mockup (3D print with dummy mass ≈ 60–80 g) on-arm with a
      glove — cheap sanity check before the outline is locked into two PCBs and a tool
- [ ] 🟢 Tap-surface ergonomics test: top face vs. wrist-side tap against tags at real bin
      mounting heights → confirms NFC antenna face

---

**Suggested order of attack:** §3 vendor items first (lead times), §1 in one sitting, §2
same day, then the two ⛔ decisions in §4 — at that point the schematic can be captured in
full while §5/§6 proceed in parallel.
