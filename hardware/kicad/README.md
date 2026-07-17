# ESP DAC — KiCad Project

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M Stereo Audio Platform  
**PCB:** 100 mm × 75 mm, 4-layer  
**Status:** Skeleton — Schematic capture in progress  
**KiCad version:** 7.x (file format 20230121)

---

## Directory Contents

```
hardware/kicad/
├── esp-dac.kicad_pro          KiCad project file (netclasses, design rules)
├── esp-dac.kicad_sch          Root schematic (hierarchical overview)
├── esp-dac.kicad_pcb          PCB skeleton (board outline, mounting holes, zone fills)
├── sheets/
│   ├── power.kicad_sch        Sheet 2: Input protection, buck converter, LDOs
│   ├── mcu.kicad_sch          Sheet 3: ESP32-S3, microSD/SDIO, USB-C, UART
│   ├── amplifier.kicad_sch    Sheet 4: TAS5825M, LC output filters, speaker connectors
│   └── clock_ui.kicad_sch     Sheet 5: 24.576 MHz oscillator, display, 3× encoders
├── libraries/
│   ├── esp-dac.kicad_sym      Custom symbol library (placeholders for key ICs)
│   └── esp-dac.pretty/        Custom footprint library (add project-specific FPs here)
│       └── README.md          Footprint naming guide
├── README.md                  ← this file
├── SCHEMATIC_PLAN.md          Hierarchical sheet plan, net groups, pin assignments
├── PCB_CONSTRAINTS.md         Board setup, stackup, design rules, keepouts
├── BOM_CANDIDATES.md          First-pass BOM with candidate part numbers
└── OPEN_DECISIONS.md          Unresolved engineering decisions that block capture
```

---

## Quick Start for Hardware Designers

### 1. Open the project

Open `esp-dac.kicad_pro` in KiCad 7 (File → Open Project).

### 2. Review documentation first

Before touching the schematic or PCB, read these files in order:
1. [`OPEN_DECISIONS.md`](OPEN_DECISIONS.md) — resolve blocking decisions
2. [`SCHEMATIC_PLAN.md`](SCHEMATIC_PLAN.md) — understand the sheet hierarchy
3. [`PCB_CONSTRAINTS.md`](PCB_CONSTRAINTS.md) — understand hard layout rules
4. [`BOM_CANDIDATES.md`](BOM_CANDIDATES.md) — review component candidates

### 3. Resolve open decisions

See [`OPEN_DECISIONS.md`](OPEN_DECISIONS.md). The three highest-priority items are:
- ESP32-S3 package (module vs. bare chip) — affects footprint and GPIO count
- Buck converter topology — affects power sheet schematic
- TAS5825M package (HTSSOP-32 vs. VQFN-32) — affects thermal via layout

### 4. Schematic capture workflow

1. Start with `sheets/power.kicad_sch` — place and wire power components
2. Move to `sheets/mcu.kicad_sch` — assign GPIO pins, wire ESP32-S3
3. Complete `sheets/amplifier.kicad_sch` — TAS5825M + LC filters
4. Finish `sheets/clock_ui.kicad_sch` — oscillator, encoders, display
5. Run ERC on root schematic `esp-dac.kicad_sch`

### 5. PCB layout workflow

Follow [`../../docs/pcb-layout-checklist.md`](../../docs/pcb-layout-checklist.md) strictly.

Key sequence:
1. Import netlist into `esp-dac.kicad_pcb`
2. Zone placement (A → B → C → D)
3. TAS5825M + LC filter + PVDD bulk first
4. ESP32-S3 with antenna keepout enforced
5. Audio clock domain (OSC1) — short, isolated
6. All I2S, I2C, SDIO routing with L2 GND reference intact
7. USB-C differential pair (90 Ω target)
8. Fill L2 GND plane (should be uninterrupted)
9. Fill L3 PVDD and 3V3 pours

---

## PCB Summary

| Parameter | Value |
|-----------|-------|
| Board size | 100 mm × 75 mm |
| Layers | 4 (L1: signal, L2: GND, L3: power, L4: signal/thermal) |
| Thickness | 1.6 mm |
| Copper weight | 1 oz (all layers) |
| Surface finish | ENIG (preferred) or HASL lead-free |
| Min trace/space | 0.1 mm / 0.1 mm |
| Min drill | 0.3 mm |
| Edge clearance | 0.3 mm minimum |

### Mounting Holes

| Ref | Position | Drill | Keepout radius | Net |
|-----|----------|-------|----------------|-----|
| MH1 | (3, 3) mm from bottom-left | 3.2 mm | 6 mm | GND |
| MH2 | (97, 3) mm | 3.2 mm | 6 mm | GND |
| MH3 | (3, 72) mm | 3.2 mm | 6 mm | GND |
| MH4 | (97, 72) mm | 3.2 mm | 6 mm | GND |

> Coordinates are from the bottom-left corner of the board with Y-up convention.
> In KiCad (Y-down), MH1=(3,72), MH2=(97,72), MH3=(3,3), MH4=(97,3).

---

## Net Classes (defined in `esp-dac.kicad_pro`)

| Netclass | Pattern | Track width | Clearance | Notes |
|----------|---------|-------------|-----------|-------|
| Default | — | 0.25 mm | 0.20 mm | General signals |
| PVDD | `PVDD*` | 1.0 mm | 0.30 mm | TAS5825M power stage |
| 3V3 | `+3V3*` | 0.5 mm | 0.20 mm | Digital supply |
| VDD_CLK | `VDD_CLK*` | 0.25 mm | 0.20 mm | Ultra-low noise clock supply |
| I2S | `I2S_*` | 0.2 mm | 0.20 mm | Length-matched I2S bus |
| SDIO | `SDIO_*` | 0.2 mm | 0.20 mm | Length-matched SDIO bus |
| USB_D | `USB_D*` | 0.2 mm | 0.15 mm | 90 Ω diff pair |
| SPK_OUT | `SPK_*` | 0.8 mm | 0.20 mm | Pre-filter class-D outputs |

---

## Source Documentation

All design intent is derived from:

| Document | Topic |
|----------|-------|
| [`docs/architecture.md`](../../docs/architecture.md) | System block architecture |
| [`docs/prototype.md`](../../docs/prototype.md) | PCB dimensions, zones, key components |
| [`docs/power-tree.md`](../../docs/power-tree.md) | Power rail strategy |
| [`docs/lc-filter-8ohm.md`](../../docs/lc-filter-8ohm.md) | LC filter design |
| [`docs/dsp-headroom.md`](../../docs/dsp-headroom.md) | TAS5825M DSP config |
| [`docs/pcb-layout-checklist.md`](../../docs/pcb-layout-checklist.md) | Layout checklist |
