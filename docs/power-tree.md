# Power Tree

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M audio platform  
**Status:** Accepted baseline

---

## Goals

- Accept **12–24 V DC input** from a wall adapter or bench supply
- Best-performance mode: **24 V** input for maximum audio headroom on 8 Ω
- Protect the board against common field failures: reverse polarity, voltage spikes, EMI
- Deliver three separated power rails to avoid cross-domain interference:
  - **PVDD** for the TAS5825M class-D power stage
  - **3.3 V digital** for ESP32-S3 and all logic
  - **VDD_CLK** low-noise rail for the audio clock oscillator

---

## Input Specification

| Parameter | Minimum | Nominal (best mode) | Maximum |
|-----------|---------|---------------------|---------|
| Input voltage | 12 V | 24 V | 26 V |
| Input current (at 24 V, full load) | — | ~2–3 A (design target) | — |
| Connector | Barrel jack or screw terminal | — | — |

> **Note:** All current values above are design targets based on expected load. Verify with actual component selection and thermal simulation.

---

## Input Protection Chain

All protection elements are placed before the main converter, as close to the input connector as practical.

```
VIN ──┬── [F1: Polyfuse or blade fuse]
      │         |
      │   [Q1: P-channel or N-channel MOSFET — reverse polarity protection]
      │         |
      │   [D1: TVS or bidirectional TVS — voltage spike clamp]
      │         |
      │   [L1 + CX1: EMI filter — common-mode choke + X-capacitor]
      │         |
      └──────── PROTECTED_VIN
```

### Protection Elements

| Element | Function | Design Guidance |
|---------|----------|-----------------|
| F1 — Fuse | Overcurrent protection; prevents fire on catastrophic failure | Select for ~3–4 A hold, ~6–8 A blow; polyfuse acceptable for bench use |
| Q1 — Reverse polarity | Prevents reverse input from damaging the board | P-channel MOSFET in series with VIN; gate driven by a resistor divider from GND |
| D1 — TVS | Clamps transients from inductive loads or hot-plug events | Unidirectional TVS; clamp voltage above max normal VIN but below component ratings |
| L1 / CX1 — EMI filter | Reduces conducted interference entering and leaving the board | Common-mode choke + ceramic X-cap; also reduces switching regulator noise back to supply |

> **Placement rule:** All of F1, Q1, D1, and L1 must be in the input zone, before the main converter. Keep the high-frequency current loops of the buck converter away from the input filter.

---

## Rail Strategy

### Overview

```
PROTECTED_VIN (12–24 V)
       │
  ┌────▼─────────────────────────────────────┐
  │         Buck Converter (main)            │
  │   Synchronous, adjustable or fixed output│
  └───┬───────────────┬───────────────────────┘
      │               │
      │ PVDD (≈24 V)  │ VREG_INT (5 V or 3.3 V intermediate)
      │               │
      │          ┌────▼──────────┐     ┌──────────────────────┐
      │          │  3.3 V LDO    │     │   Low-Noise LDO      │
      │          │  (digital)    │     │   (clock domain)     │
      │          └────┬──────────┘     └──────────┬───────────┘
      │               │                            │
   TAS5825M      ESP32-S3                  24.576 MHz oscillator
   power stage   SDIO, USB                VDD_CLK rail
                 encoders, I²C
```

### PVDD Rail — TAS5825M Power Stage

- **Target voltage:** As close to 24 V as practical (TAS5825M max: 26.4 V)
- **Source:** Either the direct protected input at 24 V, or a boosted/regulated rail from the buck
- **Local bulk capacitance:** Required immediately adjacent to TAS5825M PVDD pins
  - Target: 2–4 × 100 µF low-ESR polymer or electrolytic (design target; verify against ripple current spec)
  - Plus: 2–4 × 10 µF MLCC + 100 nF MLCC per PVDD pin for high-frequency decoupling
- **Rail separation:** PVDD must not share a PCB pour with the 3.3 V digital supply
- **Purpose:** Provides instantaneous current for class-D output peaks; low impedance at audio frequencies is critical for dynamic headroom

> **Key insight:** Insufficient local bulk capacitance causes PVDD sag during loud transients, reducing headroom and increasing distortion. This is one of the most common causes of poor subjective dynamics.

### 3.3 V Digital Rail

- **Source:** Buck converter or a post-buck LDO
- **Consumers:** ESP32-S3 (core + IO), microSD (SDIO), USB bridge/logic, rotary encoders, I²C peripherals
- **Local decoupling:** 10 µF + 100 nF per power pin on ESP32-S3; 100 nF per logic device
- **Rail separation:** Separate pour from PVDD; not shared with VDD_CLK
- **Note:** ESP32-S3 can draw significant instantaneous current during Wi-Fi/BT transmit if radio is used; account for this in bulk capacitance if radio is active

### VDD_CLK — Low-Noise Audio Clock Rail

- **Source:** Dedicated low-noise LDO fed from the 3.3 V rail or a separate intermediate rail
- **Target voltage:** Per oscillator/TCXO specification (typically 3.3 V or 1.8 V)
- **Noise requirement:** Very low output noise LDO (target: <20 µV RMS broadband noise, LDO PSRR >60 dB at 1 kHz)
- **Filtering:** Add ferrite bead in series between 3.3 V and LDO input; 100 nF + 1 µF MLCC at LDO output, placed immediately adjacent to oscillator
- **Physical isolation:** LDO and oscillator must be placed away from the buck switch node and TAS5825M output stage; clock zone on the PCB should have no high-frequency power paths nearby

---

## Buck Converter Guidelines

- **Topology:** Synchronous step-down (buck); fixed or adjustable output
- **Switching frequency:** Choose high enough to reduce output ripple with compact inductors, but ensure it does not fall in the audible band or create beat frequencies with audio sample rates
- **Input capacitor:** Low-ESR ceramic close to converter input pins; additional electrolytic for hold-up
- **Output capacitor:** Per converter datasheet; low-ESR types; place close to output pins
- **PCB:** Keep the high-current switching loop (input cap → switch → inductor → output cap) as tight as possible; no signal routing within this loop
- **Thermal:** Ensure adequate copper area and/or thermal via under the converter if it uses an exposed pad package

---

## Bulk and Decoupling Summary

| Rail | Location | Bulk (design target) | HF Decoupling |
|------|----------|----------------------|---------------|
| PVDD | Adjacent to TAS5825M PVDD pins | 2–4 × 100 µF polymer | 2–4 × 10 µF + 100 nF per pin |
| 3.3 V digital | Adjacent to ESP32-S3 VDD pins | 2 × 10 µF | 100 nF per power pin |
| VDD_CLK | Adjacent to oscillator supply pin | 1 × 1 µF | 100 nF |
| Buck input | Adjacent to converter input pins | 1 × 100 µF electrolytic | 100 nF + 10 µF MLCC |
| Buck output | Adjacent to converter output pins | Per datasheet | Per datasheet |

> All capacitor values above are design targets. Final values must be validated against the specific component selection, ripple current ratings, and simulation results.

---

## Risks

| Risk | Mitigation |
|------|------------|
| Reverse polarity input | Q1 MOSFET protection; test before deployment |
| PVDD over-voltage (>26.4 V) | TVS clamp D1; do not use unregulated supplies exceeding 26 V |
| PVDD sag during loud transients | Adequate local bulk capacitance; low-ESR types; short, wide traces |
| Clock oscillator noise | Dedicated low-noise LDO; physical separation; ferrite bead filtering |
| Buck switching noise coupling to audio | Separate pours; L2 GND plane continuity; physical separation in layout |
| Reverse current from PVDD to digital rail | Separate rail pours; no direct connection between PVDD and 3.3 V |
