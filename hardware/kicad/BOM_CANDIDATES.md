# BOM Candidates — ESP DAC

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M Audio Platform  
**Status:** First-pass candidate list — all parts TBD until confirmed against final schematic  
**Source docs:** prototype.md, power-tree.md, lc-filter-8ohm.md

> **Legend:**
> - ✅ Locked — treat as final
> - 🔷 Strong candidate — verify availability and footprint
> - 🔸 Placeholder — function defined, part TBD
> - ❌ Blocked — depends on open decision (see OPEN_DECISIONS.md)

---

## U1 — ESP32-S3 (Microcontroller)

| Option | Part | Package | Supplier | Status |
|--------|------|---------|---------|--------|
| A (preferred) | ESP32-S3-WROOM-1-N8R8 | Module, 22.4×18mm castellated | Mouser, LCSC | ❌ TBD |
| B | ESP32-S3 (bare chip) | QFN-56 | Mouser | ❌ TBD |

**Notes:**
- Option A includes integrated antenna, RF certification, 8MB flash, 8MB PSRAM
- Option B requires external antenna and RF design expertise
- Both require external USB-UART bridge or native USB via ESP32-S3 built-in USB OTG
- Decision blocks: GPIO count verification, footprint selection, antenna keepout definition
- See OPEN_DECISIONS.md item OD-001

---

## U2 — TAS5825M (Amplifier)

| Option | Part | Package | Supplier | Status |
|--------|------|---------|---------|--------|
| A (preferred) | TAS5825MPWPR | HTSSOP-32 | Mouser, DigiKey | 🔷 Candidate |
| B | TAS5825MRGET | VQFN-32 (5×5mm) | Mouser, DigiKey | 🔷 Candidate |

**Notes:**
- Both packages are functionally identical; differ in footprint and thermal pad access
- HTSSOP-32: larger, easier to hand-solder, exposed pad on underside accessible
- VQFN-32: smaller, better for thermal via array density, requires reflow
- Check stock and lead time — TAS5825M has had availability issues
- Datasheet: https://www.ti.com/lit/ds/symlink/tas5825m.pdf
- See OPEN_DECISIONS.md item OD-003

---

## U3 — Buck Converter (Main)

| Option | Part | Type | Vin range | Vout | Iout | Package | Status |
|--------|------|------|-----------|------|------|---------|--------|
| A | TPS54360B | Non-sync step-down | 4–60V | Adj | 3.5A | SOT-23-6 | 🔸 Placeholder |
| B | LMR33630 | Sync step-down | 3.8–36V | Adj | 3A | SOIC-8 | 🔸 Placeholder |
| C | MP2307 or similar | Sync step-down | 4.75–23V | Adj | 3A | SOIC-8 | 🔸 Placeholder |

**Notes:**
- Must support input range 12–26V (protect against 26V max)
- If using PVDD ≈ 24V pass-through from 24V input, buck may only need to generate 3.3V/5V intermediate
- If board must work at 12V input AND deliver ≈24V to TAS5825M, a **boost** or **SEPIC** stage is needed for PVDD — this is an open architecture decision
- See OPEN_DECISIONS.md item OD-002
- ❌ Blocked until topology is decided

---

## U4 — 3.3V Digital LDO

| Option | Part | Vout | Iout | Package | PSRR | Status |
|--------|------|------|------|---------|------|--------|
| A | AMS1117-3.3 | 3.3V | 1A | SOT-223 | ~65dB | 🔸 Placeholder (basic) |
| B | MIC5504-3.3 | 3.3V | 500mA | SOT-23-5 | ~80dB | 🔷 Candidate |
| C | TLV75733P | 3.3V | 1A | SOT-223 | ~70dB | 🔷 Candidate |

**Notes:**
- Must supply ESP32-S3 (up to ~500mA during Wi-Fi TX), SDIO, USB logic, encoders, I2C
- Minimum 500mA; 1A recommended if Wi-Fi/BT radio is active
- Standard LDO acceptable here (noise not critical — this is digital rail, not audio clock)
- Place close to ESP32-S3 input power pin

---

## U5 — Low-Noise LDO (VDD_CLK)

| Option | Part | Vout | Iout | Noise | PSRR | Package | Status |
|--------|------|------|------|-------|------|---------|--------|
| A | LP5907MFX-3.3 | 3.3V | 250mA | 6.3µV RMS | >79dB | DSBGA-4 / SOT-23-5 | 🔷 Strong candidate |
| B | LT3042EDD | Adj (set to 3.3V) | 200mA | 0.8µV RMS | >80dB | DFN-8 | 🔷 High-performance option |
| C | NCP163 | 3.3V | 300mA | ~15µV RMS | >60dB | SOT-23-5 | 🔸 Placeholder |

**Notes:**
- Target: <20µV RMS broadband noise, PSRR >60dB at 1kHz
- LP5907 is a strong candidate — low noise, available, reasonable cost
- LT3042 is best-in-class ultra-low noise; use if jitter budget is tight
- OSC1 current draw is typically <20mA; 250mA LDO is more than adequate
- Ferrite bead (FB_CLK) on input: recommend ~600Ω at 100MHz (e.g., Murata BLM15PX601SN1)

---

## OSC1 — 24.576 MHz Oscillator

| Option | Part | Type | Vcc | Output | Jitter | Package | Status |
|--------|------|------|-----|--------|--------|---------|--------|
| A | TXS2520A-24.576MHZ | TCXO | 3.3V | LVCMOS | <1ps RMS | 2.0×1.6mm SMD | 🔷 Candidate |
| B | SIT8208AI-71-18S-24.576000X | MEMS osc | 1.8–3.3V | LVCMOS | <0.5ps RMS | 2.0×1.6mm SMD | 🔷 High-quality candidate |
| C | ECS-2532-240-BN-TR | Crystal osc | 3.3V | LVCMOS | ~100ps | 2.5×2.0mm SMD | 🔸 Basic option |

**Notes:**
- Frequency must be exactly 24.576 MHz (= 512 × 48 kHz)
- Low jitter critical for audio quality; TCXO or MEMS oscillator preferred over simple XO
- LVCMOS output compatible with ESP32-S3 and TAS5825M MCLK input directly
- Supply from VDD_CLK (U5 output)
- Enable/disable pin: leave tied high (always on) or connect to GPIO for power management

---

## F1 — Fuse

| Option | Part | Rating | Package | Status |
|--------|------|--------|---------|--------|
| A | Schurter 0034.3712 or similar | 3A / 250V fast-blow | 1206 SMD | 🔸 Placeholder |
| B | Littelfuse 0452003.MR | 3A / 125V SMD fuse | 1206 | 🔸 Placeholder |

**Notes:**
- Select 3A hold, ~6A blow for 24V input with 2×25W output + margins
- SMD 1206 fuse holder acceptable for prototype; use blade fuse holder for production

---

## Q1 — Reverse Polarity MOSFET

| Option | Part | Type | Vds | Id | Rds_on | Package | Status |
|--------|------|------|-----|----|--------|---------|--------|
| A | AO3401A | P-ch | -30V | -4A | 40mΩ | SOT-23 | 🔷 Candidate |
| B | DMP3098LSD | P-ch | -30V | -4.6A | 55mΩ | SOT-23 | 🔷 Candidate |

**Notes:**
- Gate-source threshold: must be reliably ON at minimum VIN=12V (Vgs = 0 to -Vt)
- P-channel MOSFET in series with VIN; gate driven via resistor divider to GND
- Vds rating: must exceed maximum VIN (26V) with margin → select ≥30V rated

---

## D1 — TVS Diode (Input Protection)

| Option | Part | Clamp | Vbr | Package | Status |
|--------|------|-------|-----|---------|--------|
| A | SMBJ26A | 26V / 42V clamp | 28.9V | SMA | 🔷 Candidate |
| B | P6KE27A | 27V / 43.5V clamp | 29.7V | DO-15 TH | 🔸 Through-hole option |

**Notes:**
- Unidirectional; clamp voltage must be above max normal VIN (26V) but below component max ratings (~28V)
- SMD preferred; check peak pulse power against expected transient energy

---

## Output Inductors (L_OUT_L1/L2, L_OUT_R1/R2)

| Option | Inductance | DCR | Isat | Package | Type | Status |
|--------|-----------|-----|------|---------|------|--------|
| A | 10µH | <40mΩ | ≥4A | 7×7mm or 7×4mm SMD | Shielded ferrite | 🔸 Placeholder |
| B | 6.8µH | <30mΩ | ≥4.5A | 6×6mm SMD | Shielded ferrite | 🔸 Placeholder |

**Notes:**
- Final value depends on TAS5825M switching frequency and target LC corner frequency
- Must consult TAS5825M application note for recommended L values
- Isat must exceed peak output current at 24V / 8Ω
- 4 inductors total (×2 per channel for BTL differential)
- Candidates: Würth 744 series, Coilcraft SER/XAL series, Bourns SRR series

---

## Output Filter Capacitors (C_OUT_L1/L2, C_OUT_R1/R2)

| Option | Value | Dielectric | Voltage | Package | Status |
|--------|-------|-----------|---------|---------|--------|
| A | 220nF | C0G/NP0 ceramic | 50V | 1210 | 🔷 Candidate |
| B | 100nF | Film | 63V | Through-hole or SMD | 🔷 Candidate |

**Notes:**
- **DO NOT use X7R ceramic** in audio filter — voltage-dependent capacitance causes distortion
- C0G (NP0) ceramic: stable, good for HF, low ESR, preferred for this application
- Film: excellent but physically larger; through-hole acceptable for prototype
- Voltage rating: must withstand full differential output swing at 24V PVDD
- 4 capacitors total; same value for all four (L/R symmetry)

---

## Connectors

| Ref | Description | Candidate | Status |
|-----|-------------|-----------|--------|
| J1 | DC input barrel jack 5.5/2.5mm | CUI PJ-102BH or similar | 🔸 TBD |
| J2 | USB-C receptacle | GCT USB4135-GF-A or similar | 🔸 TBD |
| J3 | microSD push-push | Hirose DM3D-SF or similar | 🔸 TBD |
| J4 | UART 2.54mm 4-pin header | Molex 22-28-4040 or similar | 🔸 TBD |
| J5 | Display header 2.54mm | Per display module spec | 🔸 TBD |
| J6 | I2C/GPIO header 2.54mm | TBD pinout | 🔸 TBD |
| J_SPK_L/R | Speaker screw terminal 5mm pitch | Phoenix Contact PT-1.5/2-5.0H | 🔷 Candidate |

---

## Passive Components (general)

| Type | Standard values to stock | Notes |
|------|--------------------------|-------|
| Resistors 0402 | 100Ω, 1kΩ, 4.7kΩ, 10kΩ, 100kΩ | Pull-ups, dividers |
| Caps 0402 100nF | X7R, 25V | Bypass caps (digital) |
| Caps 0402 1µF | X7R, 10V | Bulk bypass |
| Caps 0805 10µF | X7R, 25V | Power supply bulk |
| Caps 1210 100–470nF | C0G, 50V | LC filter (must be C0G, NOT X7R) |
| Polymer electrolytic 100µF | 35V | PVDD bulk |

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-07-17 | Skeleton generator | Initial first-pass BOM candidate table |
