# Prototype Specification — esp-dac-

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M audio platform  
**Status:** Prototype phase — baseline design locked, PCB dimensions confirmed

---

## PCB Dimensions

| Parameter | Value |
|-----------|-------|
| Width | 100 mm |
| Height | 75 mm |
| Layers | 4 (L1 signal, L2 GND plane, L3 power, L4 signal/thermal) |
| Board thickness | 1.6 mm (standard) |
| Copper weight | L1/L4: 1 oz, L2/L3: 1 oz (verify with manufacturer) |
| Surface finish | HASL lead-free or ENIG |
| Min trace/space | 0.1 mm / 0.1 mm |
| Min drill | 0.3 mm |
| Edge clearance | 0.3 mm minimum from copper to board edge |

---

## Mounting Holes

Four M3 mounting holes, one at each corner:

| Hole | Position (from board origin, bottom-left) |
|------|-------------------------------------------|
| MH1 | (3 mm, 3 mm) |
| MH2 | (97 mm, 3 mm) |
| MH3 | (3 mm, 72 mm) |
| MH4 | (97 mm, 72 mm) |

- Hole diameter: 3.2 mm (M3 clearance)
- Copper keep-out radius: 6 mm from hole center
- Connect mounting hole pads to GND (L2) via short trace

---

## Zone Partitioning (100 × 75 mm board)

```
┌──────────────────────────────────────────────────────────────────────┐  75mm
│  ZONE A (0–40mm W, 0–35mm H)       │  ZONE B (40–100mm W, 0–35mm H) │
│  Input Protection + Buck            │  Digital / Control              │
│  • Fuse, RevPol MOSFET, TVS, EMI   │  • ESP32-S3                    │
│  • Buck converter                   │  • microSD (SDIO)              │
│  • PVDD bulk capacitors             │  • USB-C connector             │
│  • Input connector (board edge)     │  • UART TX/RX header           │
├─────────────────────────────────────┼────────────────────────────────┤
│  ZONE C (0–70mm W, 35–75mm H)      │  ZONE D (70–100mm W, 35–75mm H)│
│  Audio Power (Stereo)               │  UI + Clock                    │
│  • TAS5825M                         │  • Display connector           │
│  • LC Filter Left + Right           │  • 3× Rotary encoders + push   │
│  • Speaker connectors (board edge)  │  • 24.576 MHz oscillator       │
│  • PVDD pour, thermal vias          │  • Low-noise LDO (VDD_CLK)     │
└──────────────────────────────────────────────────────────────────────┘ 100mm
```

Zone boundaries are approximate guides. Final placement follows routing efficiency and thermal constraints.

---

## Key Components

### Processing & Amplification

| Reference | Component | Package | Zone |
|-----------|-----------|---------|------|
| U1 | ESP32-S3 (module or bare chip) | LGA/module | B |
| U2 | TAS5825M | HTSSOP-32 / VQFN-32 | C |

### Power

| Reference | Component | Value / Type | Zone |
|-----------|-----------|-------------|------|
| F1 | Fuse | 3 A, 250 V, 1206 | A |
| Q1 | Reverse polarity MOSFET | P-channel, 30 V, 5 A | A |
| D1 | TVS diode | Unidirectional, 24 V clamping | A |
| L_EMI | Common-mode choke / EMI filter | 8–10 µH / 2 A | A |
| U3 | Buck converter | Sync, 12–26 V in, 24 V / 3.3 V out | A |
| U4 | LDO 3.3 V digital | 3.3 V, ≥500 mA, low dropout | A/B |
| U5 | LDO low-noise VDD_CLK | 3.3 V or 3.0 V, ultra-low noise | D |

### Audio Clock

| Reference | Component | Value | Zone |
|-----------|-----------|-------|------|
| OSC1 | TCXO / oscillator | 24.576 MHz, 3.3 V, low jitter | D |
| C_OSC1 | Bypass capacitor | 100 nF, 0402 | D |
| C_OSC2 | Bulk bypass | 1 µF, 0402 | D |

### Output Stage (per channel)

| Reference | Component | Value | Zone |
|-----------|-----------|-------|------|
| L_OUT_L, L_OUT_R | Output inductors | ~10 µH, rated ≥ 4 A | C |
| C_OUT_L, C_OUT_R | Filter capacitors | ~1 µF, 50 V, X7R, 1210 | C |
| J_SPK_L, J_SPK_R | Speaker connectors | 2-pin, 5 mm pitch, board edge | C |

### Connectors & UI

| Reference | Component | Zone |
|-----------|-----------|------|
| J1 | DC input (barrel jack or screw terminal) | A, board edge |
| J2 | USB-C connector | B, board edge |
| J3 | microSD slot | B |
| J4 | UART TX/RX expansion header (2.54 mm, 4-pin) | B |
| J5 | Display header (SPI/I²C, 2.54 mm) | D |
| J6 | I²C/GPIO expansion header | B/D |
| ENC1–ENC3 | Rotary encoders with push switch | D |

---

## Critical Mechanical Constraints

1. **Board edge connectors** (J1, J2, J_SPK_L/R) must be placed ≤ 3 mm from their respective board edges to maintain alignment within an enclosure
2. **ESP32-S3 antenna keep-out**: minimum 3 mm copper-free zone around antenna end of the module/chip (see ESP32-S3 datasheet)
3. **TAS5825M thermal pad**: via array to L4 copper fill is mandatory; minimum 9 thermal vias (3 × 3), 0.3 mm drill, 0.5 mm pad
4. **Buck converter switch node**: keep SW node trace area under 50 mm² to limit EMI; surround with ground vias
5. **Minimum keep-out** around each M3 mounting hole: 6 mm radius from hole center (no copper, no components)
6. **Component height budget**: stay within 10 mm total component height on L1 for typical enclosure clearance

---

## Layer Assignments Summary

| Layer | Purpose | Notes |
|-------|---------|-------|
| L1 (Top) | Components + critical signals | I²S, SDIO, USB D+/D–, MCLK, I²C |
| L2 | **Continuous GND plane** | No splits — inviolable rule |
| L3 | Power distribution | PVDD pour, 3.3 V pour, VDD_CLK trace |
| L4 (Bottom) | Signal overflow + thermal | Thermal spreading copper under TAS5825M and buck |

---

## Prototype Build Targets

| Target | Value |
|--------|-------|
| Prototype quantity | 5 boards (first spin) |
| Assembly | Hand solder + reflow (no BGA) |
| Test objectives | Power-up sequencing, I²S audio output, SDIO card access, USB firmware flash, encoder GPIO, display SPI/I²C |
| First-spin acceptance | Audio playback from SD card at 48 kHz / 24-bit on 8 Ω dummy load |

---

## Next Steps

1. **Schematic** — Create schematic in KiCad (or equivalent) using the architecture and component list above
2. **BOM** — Finalize component selection; confirm part numbers and availability
3. **PCB layout** — Follow [`pcb-layout-checklist.md`](pcb-layout-checklist.md) strictly; start with zone placement, then GND plane, then power, then signals
4. **DRC / ERC** — Run electrical rule check and design rule check before Gerber export
5. **Gerber review** — Visual verification of all layers in a Gerber viewer
6. **Prototype order** — Submit to PCB manufacturer; order stencil for SMD paste application
7. **Assembly and bring-up** — Power supply sequencing test before full assembly; test point probing at each rail
8. **Firmware** — Port / develop ESP-IDF firmware for I²S master, TAS5825M I²C init, SDIO FAT read, encoder GPIO, display driver

---

## References

- [`architecture.md`](architecture.md) — Full system block architecture
- [`power-tree.md`](power-tree.md) — Input protection and rail strategy
- [`dsp-headroom.md`](dsp-headroom.md) — TAS5825M DSP configuration
- [`lc-filter-8ohm.md`](lc-filter-8ohm.md) — LC output filter design
- [`pcb-layout-checklist.md`](pcb-layout-checklist.md) — Layout checklist for prototype sign-off
- [`pcb-topview.svg`](pcb-topview.svg) — PCB top-view layout diagram
