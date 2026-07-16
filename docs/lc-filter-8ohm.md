# LC Output Filter — 8 Ω Load

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M audio platform  
**Status:** Accepted baseline

---

## Purpose

The TAS5825M class-D amplifier output is a high-frequency pulse-width modulated (PWM) signal. The LC output filter converts this PWM signal back into an analog audio waveform before it reaches the speaker. Without the filter:

- The speaker would receive high-frequency switching energy at the modulation frequency (hundreds of kHz), causing heating and potential damage
- The speaker cables would radiate EMI at switching frequencies
- The amplifier would see a capacitive load from the speaker's voice coil inductance at RF, potentially causing instability

**The LC filter is a mandatory component in any class-D design.** It is not optional.

---

## Load Assumption

This design targets a nominal **8 Ω** speaker load. This is the primary use case and the filter must be optimized for it. Significant deviation from 8 Ω (e.g., using a 4 Ω load without re-characterizing the filter) may cause:

- Underdamping or overdamping of the filter response
- Frequency response deviation in the audio band
- Increased ripple current in the inductor

---

## Filter Topology

### Differential LC (BTL — Bridge-Tied Load)

TAS5825M drives the speaker in **BTL (Bridge-Tied Load)** mode: two output pins (OUT+ and OUT–) drive the speaker differentially. Each output pin requires its own LC filter leg.

```
TAS5825M OUT+ ──[L1]──┬──── Speaker (+)
                      │
                     [C1]
                      │
                     GND  (or virtual mid-rail, if used)

TAS5825M OUT– ──[L2]──┬──── Speaker (–)
                      │
                     [C2]
                      │
                     GND
```

> For a fully differential BTL design, C1 and C2 return to the PCB's power/signal GND. The differential signal across the speaker is the sum of both half-bridge swings.

### Filter Order

A **2nd-order (single-stage) LC low-pass filter** is standard for class-D amplifiers at this power level and is appropriate for this design. It provides:

- Approximately –40 dB/decade roll-off above the corner frequency
- Sufficient attenuation of the switching carrier
- Simple, well-understood component selection
- Low parts count suitable for a compact PCB

---

## Corner Frequency Target

The corner frequency (f₀) must be:

- **Well above** the audio band upper limit (~20 kHz) to avoid phase shift and attenuation in the audio band
- **Well below** the switching frequency to adequately attenuate the carrier
- Damped correctly for the 8 Ω load

For a 2nd-order LC filter targeting 8 Ω:

```
f₀ = 1 / (2π × √(L × C))

Damping factor:  ζ = (1/2) × √(C/L) × R_load

For Butterworth (maximally flat, ζ = 0.707):
  L × C = 1 / (2π × f₀)²
  C = L / R_load²  (from ζ = 0.707, R_load = 8 Ω)
```

**Design target range for f₀: 40–80 kHz** (verify against TAS5825M switching frequency in application note)

> The exact corner frequency should be confirmed using TAS5825M application guidance and the actual switching frequency. Component values below are **illustrative targets only** and must be verified with SPICE simulation and bench measurement.

### Illustrative Design Targets (8 Ω, f₀ ≈ 60 kHz)

| Component | Illustrative Target | Notes |
|-----------|---------------------|-------|
| L (per leg) | ~3–10 µH | Low DCR; rated for peak output current |
| C (per leg) | ~100–470 nF | Film or C0G/NP0 ceramic; not X7R for audio |
| R_load (assumed) | 8 Ω | Nominal speaker impedance |

> **These are placeholder targets.** Use TAS5825M application note values as the starting point, then adjust for the actual switching frequency, minimum load impedance, and EMI requirements.

---

## Component Selection Criteria

### Inductor (L)

- **Inductance tolerance:** ±10% or better
- **DCR (DC resistance):** As low as practical to minimize power loss at rated current; aim for < 50 mΩ per leg
- **Saturation current:** Must exceed the peak output current at maximum power
  - At 24 V PVDD into 8 Ω: peak output current ≈ √(2 × P_peak / R_load); verify against actual power target
- **Core type:** Powdered iron or ferrite; must have low core loss at switching frequency
- **Self-resonant frequency:** Must be well above the switching frequency
- **Form factor:** SMD shielded inductor recommended for EMI; through-hole acceptable for prototyping

### Capacitor (C)

- **Dielectric:** Film capacitor or C0G/NP0 ceramic preferred for audio path
  - Avoid X7R or Y5V for filter capacitors in audio: they have significant voltage-dependent capacitance variation that alters the filter response with signal amplitude
- **Voltage rating:** Must exceed maximum voltage swing across the capacitor at full output; in BTL, the capacitor sees the full differential swing, so rate generously
- **ESR:** Low ESR to avoid damping loss and heating
- **Tolerance:** ±5% or better for predictable filter corner frequency

---

## Placement and Routing

### Placement

- Place L1, L2, C1, C2 **immediately adjacent to TAS5825M output pins (OUT+, OUT–)**
- The filter must be the first thing after the amplifier output — not after the speaker connector or after a via transition
- Keep the filter components in a tight, compact footprint

### Routing

- Traces from OUT+ and OUT– to the inductor must be **short and wide** (high current; minimize impedance)
- After the inductor, the trace to the speaker terminal should be direct
- Filter capacitor return path: connect C1 and C2 returns to the **local power/signal GND pour**, not via a long trace back to the main GND
- Maintain **differential symmetry**: L1/C1 and L2/C2 should be mirror-image in layout to match timing and impedance on both legs

### EMI Considerations

- Class-D output traces between TAS5825M and the filter carry switching-frequency current — route on inner or bottom layers where possible, or use tight differential routing
- After the filter, the traces to the speaker connector carry audio-frequency current and are much more benign from an EMI perspective
- Do not route sensitive signals (clock, I²S, SDIO) parallel to the pre-filter output traces

---

## Load Sensitivity

| Load | Effect on Filter |
|------|-----------------|
| 8 Ω (target) | Filter tuned correctly; Butterworth response |
| 4 Ω | Load damping doubles; filter corner shifts; verify stability |
| 16 Ω | Under-damped response; potential for ringing in audio band |
| Open circuit (no speaker) | No load damping; filter becomes resonant; TAS5825M protection should engage |
| Capacitive / reactive speaker | Complex impedance interacts with LC filter; can cause stability issues at high frequency |

**Design for 8 Ω as the primary use case.** If 4 Ω or 16 Ω use is desired, re-characterize the filter or select L/C values that provide acceptable response across the intended impedance range.

---

## Summary

| Requirement | Specification |
|-------------|--------------|
| Topology | 2nd-order LC BTL (one filter leg per output) |
| Corner frequency | 40–80 kHz (confirm vs. switching frequency) |
| Primary load | 8 Ω |
| Inductor saturation | Exceeds peak output current at 24 V / 8 Ω |
| Capacitor type | Film or C0G/NP0 ceramic (no X7R) |
| Placement | Immediately adjacent to TAS5825M output pins |
| Routing | Short, wide, differential, symmetrical; away from clock and digital signals |

---

## Risks

| Risk | Mitigation |
|------|------------|
| Filter underdamped with non-8 Ω load | Clearly document 8 Ω as primary use case; re-characterize for alternate loads |
| Inductor saturation at peak output | Select inductor with saturation current rating above maximum peak current |
| X7R capacitor causing non-linearity | Use film or C0G capacitor; avoid X7R in the signal path |
| Filter placed far from TAS5825M | Enforce placement rule: filter immediately after output pins |
| High-frequency ringing on open circuit | TAS5825M over-voltage protection should engage; do not operate without load in sustained mode |
