# System Architecture

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M audio platform  
**Status:** Accepted baseline — treat as implementation reference, not open for re-evaluation

---

## Goals

- Deliver superior stereo dynamics on **2 × 8 Ω** loads from a compact integrated board
- Operate reliably from **12–24 V DC input**, with best-performance mode at **24 V**
- Keep the system fully integrated: protection, power conditioning, DSP, amplification, and control on one 4-layer PCB
- Provide a stable, low-jitter audio clock domain independent of digital switching noise
- Support required connectivity/peripherals: microSD (SDIO), USB-C (firmware), 3 encoders with push, display, and UART TX/RX expansion

---

## System Block Diagram

```
                          ┌─────────────────────────────────┐
 12–24 V DC ─────────────►│         Input Protection        │
                          │  Fuse · Rev-pol MOSFET · TVS   │
                          │  EMI filter (common-mode choke) │
                          └────────────────┬────────────────┘
                                           │ Protected VIN
                          ┌────────────────▼────────────────┐
                          │         Buck Converter           │
                          │   (synchronous, high efficiency) │
                          └─┬──────────┬──────────┬─────────┘
                            │ PVDD     │ ~5V/3.3V │
                            │          │          │
              ┌─────────────▼──┐  ┌────▼────┐  ┌─▼──────────────┐
              │  TAS5825M PVDD │  │ 3.3 V   │  │  Low-Noise LDO │
              │  audio rail    │  │ digital │  │  clock branch  │
              │  (≈24 V ideal) │  │   LDO   │  └───────┬────────┘
              └───────┬────────┘  └────┬────┘          │
                      │               │          ┌──────▼──────────┐
                      │          ┌────▼────┐     │ 24.576 MHz TCXO │
                      │          │ESP32-S3 │     │  low-jitter osc │
                      │          │         │     └──────┬──────────┘
                      │          │ microSD │◄─ SDIO     │ MCLK
                      │          │ USB-C   │◄─ UART/USB │
                      │          │ 3× ENC+P│◄─ GPIO     │
                      │          │ Display │◄─ SPI/I²C  │
                      │          │ UART TX/RX ◄ UART    │
                      │          │ I²C/exp │◄─ I²C      │
                      │          └────┬────┘            │
                      │         I²S master ─────────────┘
                      │         I²C config               │
                      │               │                  │
              ┌───────▼───────────────▼──────────────────▼──┐
              │              TAS5825M                        │
              │   I²S input · DSP (EQ/limiter/SRC) · DAC   │
              │   Stereo Class-D output stage (L/R)          │
              └───────────────┬───────────────┬──────────────┘
                              │               │ Differential PWM outputs
                     ┌────────▼────────┐ ┌────▼──────────────┐
                     │ LC Output Filter │ │ LC Output Filter  │
                     │       Left       │ │      Right        │
                     └────────┬─────────┘ └────┬──────────────┘
                              │                │
                          8 Ω Speaker L     8 Ω Speaker R
```

---

## Subsystem Roles

### ESP32-S3 — Control, UI, Storage, Connectivity
- **Audio transport:** Reads audio files from microSD, feeds decoded PCM to TAS5825M via I²S
- **DSP control:** Configures TAS5825M registers over I²C (EQ, limiter, volume, mode)
- **UI:** Reads 3 rotary encoders with push switches via GPIO and drives a display module (SPI or I²C, final pick in schematic)
- **Storage:** microSD card accessed via SDIO 4-bit mode for sufficient throughput
- **Firmware:** USB-C port (USB-UART bridge or native USB) for flashing and OTA updates
- **Expansion:** I²C, UART TX/RX, and spare GPIO available on a header for peripherals

### TAS5825M — DSP / DAC / Class-D Amplifier
- Operates as **I²S slave** driven by the external 24.576 MHz audio clock
- Integrated DSP handles: EQ, loudness compensation, limiter, subsonic filter, SRC
- Stereo class-D output stage drives independent LC filters for left and right 8 Ω loads
- Configured via I²C from ESP32-S3 at startup and during playback
- PVDD rail should be kept as close to 24 V as practical for full dynamic headroom

### Audio Clock Domain
- **24.576 MHz TCXO/oscillator** on a dedicated low-noise power rail
- Provides MCLK to both ESP32-S3 (external clock source) and TAS5825M
- Isolated from switching regulator noise and high-speed digital domains
- Traces must be short, referenced to continuous GND on L2, routed away from buck switch nodes and SDIO

### Power System
Covered in detail in [`power-tree.md`](power-tree.md). Summary:
- Integrated input protection (fuse, reverse-polarity, TVS, EMI)
- Main buck converter feeds PVDD and a lower-voltage intermediate rail
- Separate LDO for 3.3 V digital
- Separate low-noise LDO for the audio clock domain

### Output Stage
Covered in [`lc-filter-8ohm.md`](lc-filter-8ohm.md). Summary:
- LC low-pass filter between TAS5825M outputs and speaker terminals
- Optimized for 8 Ω load impedance
- Placed immediately adjacent to TAS5825M output pins

---

## PCB Architecture

### Stackup

| Layer | Role | Notes |
|-------|------|-------|
| L1 | Signal + components | ESP32-S3, TAS5825M, oscillator, connectors, passives |
| L2 | **Continuous GND plane** | No splits, no cuts; full-board reference plane |
| L3 | Power planes + secondary signals | PVDD pour, 3.3 V pour, short signal stubs |
| L4 | Signals + auxiliary power | Routing overflow, thermal relief connections |

### Physical Zone Partitioning

```
┌──────────────────────────────────────────────────────────┐
│  [Input protection + Buck]  │ [ESP32-S3 + SDIO + USB-C] │
│─────────────────────────────│────────────────────────────│
│ [TAS5825M + L/R output filt]│ [Clock + encoders + display]│
└──────────────────────────────────────────────────────────┘
```

- Power and switching components occupy one quadrant with their own local return paths
- Clock domain is physically separated from buck switch nodes and class-D output traces
- TAS5825M is placed to minimize PVDD trace length and maximize proximity to the LC filter
- ESP32-S3 antenna keep-out zone respected if Wi-Fi/BT radio is used

### Critical Routing Rules
1. L2 GND plane must be continuous — no routing cuts beneath high-frequency paths
2. MCLK trace: short, uninterrupted L2 reference underneath, away from switching edges
3. I²S traces: matched length, routed together, L2 reference intact below
4. PVDD power path: short, wide, with pour on L3; local bulk capacitors on L1 near TAS5825M
5. Buck switch node (SW): confined to local area; no sensitive signal routing nearby
6. LC filter: directly adjacent to TAS5825M OUT pins; differential layout, symmetric where possible
7. SDIO: length-matched, differential pairs, direct path to microSD connector
8. USB-C data lines: routed as a differential pair with appropriate impedance
9. Keep display bus and UART TX/RX away from class-D output and buck switch node

---

## Constraints and Risks

| Constraint | Detail |
|------------|--------|
| PVDD maximum | TAS5825M supports PVDD up to 26.4 V; do not exceed |
| Thermal | TAS5825M dissipates significant heat at high output; thermal via array to L4 copper or bottom pad required |
| Clock halt | TAS5825M reports clock error if MCLK/BCLK/LRCLK ratios are incorrect; verify firmware I²S settings |
| SDIO routing | Long or poorly terminated SDIO traces cause CRC errors; minimize trace length and stubs |
| 44.1 kHz content | System clock is optimized for 48 kHz family; 44.1 kHz requires SRC conversion (TAS5825M has on-chip SRC) |
| EMI | Class-D L/R output traces carry high-frequency switching current; keep both LC filters local, route speaker wires after filters |

---

## Implementation Priorities

1. Get the power tree correct before laying out signal routing
2. Establish and preserve continuous L2 GND — revisit at every layout iteration
3. Place TAS5825M, LC filter, and PVDD bulk capacitors first
4. Route the audio clock domain next: short, clean, isolated
5. Route I²S, I²C, and SDIO with length matching and solid GND reference
6. Verify thermal design before finalizing component placement
7. Confirm MCLK/BCLK/LRCLK ratios in firmware before hardware tape-out

---

## PCB Image (Initial Placement View)

An initial top-view PCB image with stereo outputs, 3 encoders with push, display, and UART TX/RX is available here:

- [`pcb-topview.svg`](pcb-topview.svg)
