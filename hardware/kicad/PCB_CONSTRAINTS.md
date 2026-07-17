# PCB Constraints — ESP DAC

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M Audio Platform  
**Status:** Locked constraints from architecture.md and prototype.md  
**Authority:** These constraints take priority over convenience. Do not route around them.

---

## Board Specification

| Parameter | Value | Source |
|-----------|-------|--------|
| Board outline | 100 mm × 75 mm | prototype.md |
| Layer count | 4 | prototype.md |
| Board thickness | 1.6 mm | prototype.md |
| Copper weight | 1 oz (35 µm) all layers | prototype.md |
| Surface finish | ENIG preferred; HASL lead-free acceptable | prototype.md |
| Min trace width | 0.1 mm | prototype.md |
| Min trace spacing | 0.1 mm | prototype.md |
| Min drill diameter | 0.3 mm | prototype.md |
| Copper to board edge | 0.3 mm minimum | prototype.md |
| Max component height | 10 mm on L1 (enclosure clearance) | prototype.md |

---

## Layer Stackup (4-Layer)

| Layer | Role | Constraint |
|-------|------|-----------|
| **L1 (F.Cu)** | Components + critical signal routing | I²S, SDIO, USB D+/D−, MCLK, I²C routed here |
| **L2 (In1.Cu)** | **Continuous GND plane** | **NO SPLITS. NO ROUTING CUTS. INVIOLABLE.** |
| **L3 (In2.Cu)** | Power distribution | PVDD pour (Zone C), +3V3 pour (Zone B), VDD_CLK trace |
| **L4 (B.Cu)** | Signal overflow + thermal spreading | Thermal copper fill under TAS5825M and buck converter |

### Stackup dielectric (verify with manufacturer)

```
L1  (F.Cu)   0.035 mm Cu
             0.38 mm prepreg FR4
L2  (In1.Cu) 0.035 mm Cu      ← GND plane
             0.71 mm core FR4
L3  (In2.Cu) 0.035 mm Cu      ← Power planes
             0.38 mm prepreg FR4
L4  (B.Cu)   0.035 mm Cu
─────────────────────────────
Total         1.6 mm
```

### Impedance targets (confirm with manufacturer)

| Trace type | Layer | Target impedance | Expected width |
|------------|-------|-----------------|----------------|
| Single-ended 50 Ω | L1 (ref L2) | 50 Ω | ~0.10 mm on 0.38mm dielectric |
| USB 2.0 diff pair | L1 (ref L2) | 90 Ω differential | ~0.20 mm / 0.20 mm gap |
| I2S / SDIO single-ended | L1 (ref L2) | 50–60 Ω | ~0.10–0.15 mm |

> Exact widths depend on manufacturer's process. Request controlled-impedance stackup
> and validate with their calculator.

---

## Mounting Holes

Four M3 clearance holes at board corners:

| Ref | PCB coordinates (KiCad Y-down) | Design doc coordinates (Y-up) | Net |
|-----|-------------------------------|-------------------------------|-----|
| MH1 | (3, 72) | (3, 3) from BL | GND |
| MH2 | (97, 72) | (97, 3) from BL | GND |
| MH3 | (3, 3) | (3, 72) from BL | GND |
| MH4 | (97, 3) | (97, 72) from BL | GND |

**Hole specification:**
- Drill diameter: 3.2 mm (M3 clearance)
- Pad diameter: 6.0 mm (minimum copper annular ring for GND connection)
- Keepout radius: **6 mm from hole center** — no copper, no components, no vias in this zone
- All four holes connected to GND (L2 plane) via short trace or direct fill connection

> Mounting hole keepout zones are pre-defined in `esp-dac.kicad_pcb` as rule areas.

---

## Zone Partitioning

```
(0,0)──────────────────────────────────────────────(100,0)
│  ZONE A                       │  ZONE B                 │
│  0–40mm W, 0–35mm H           │  40–100mm W, 0–35mm H   │
│  Input Protection + Buck       │  ESP32-S3 + microSD     │
│  All protection components     │  USB-C + UART header    │
│  Buck converter               │  Antenna keepout zone    │
│  PVDD bulk (partial)           │  SDIO routing            │
│  3V3 LDO + VDD_CLK LDO        │                          │
├───────────────────────────────┼──────────────────────────┤ (y=35mm)
│  ZONE C                       │  ZONE D                  │
│  0–70mm W, 35–75mm H          │  70–100mm W, 35–75mm H   │
│  TAS5825M + LC filters        │  Display connector       │
│  Speaker connectors            │  3× Rotary encoders      │
│  PVDD pour (L3)               │  24.576 MHz oscillator   │
│  Thermal vias (TAS5825M)       │  Low-noise LDO (U5)     │
(0,75)──────────────────────────────────────────────(100,75)
```

> Zone boundaries are guidelines for placement. Final placement follows routing
> efficiency and thermal constraints. Do not violate hard constraints for zone convenience.

---

## Connector Placement Constraints (LOCKED)

The following connectors **must** be placed within 3 mm of the specified board edge:

| Ref | Description | Board edge | Notes |
|-----|-------------|-----------|-------|
| J1 | DC input (barrel jack / screw terminal) | Left edge (x=0) | Zone A |
| J2 | USB-C receptacle | Right or top edge | Zone B |
| J_SPK_L | Left speaker screw terminal | Bottom edge (y=75) | Zone C |
| J_SPK_R | Right speaker screw terminal | Bottom edge (y=75) | Zone C |

> Enforced by enclosure mechanical design. Cannot be moved inboard.

---

## Keepout Zones (LOCKED)

### ESP32-S3 Antenna Keepout

- **Location:** Antenna end of ESP32-S3 module/chip
- **Requirement:** Minimum 3 mm copper-free zone around antenna
- **Applies to:** ALL copper layers (F.Cu, In1.Cu, In2.Cu, B.Cu)
- **Reference:** ESP32-S3 datasheet hardware design guide
- **Pre-defined in PCB:** Rule area `ESP32_ANTENNA_KEEPOUT` (approximate — refine after U1 placement)
- **Note:** If using a module (WROOM-1), the module datasheet defines the exact keepout. If using bare chip + PCB trace antenna, an RF engineer must define the antenna geometry and keepout.

### Mounting Hole Keepouts

- **Rule:** No copper, no components, no vias within 6 mm radius of each mounting hole center
- **Pre-defined in PCB:** Rule areas `MH1_KEEPOUT_6mm` through `MH4_KEEPOUT_6mm`
- **Note:** The circular keepout approximation uses KiCad polygon. Verify exact circular boundary during DRC.

### Buck Converter Switch Node (SW)

- **Rule:** SW node trace area must be ≤ 50 mm²
- **Rule:** Surround SW node with GND stitching vias on all layers
- **Rule:** No sensitive signals (clock, I2S, SDIO) within 10 mm of SW node
- **Not pre-defined in PCB:** Add after buck converter placement

---

## Routing Constraints (LOCKED)

### L2 GND Plane

```
RULE: L2 (In1.Cu) must be a continuous, uninterrupted copper pour
      across the ENTIRE board area.

PROHIBITED:
  - Any routing cut that interrupts the return current path
  - Any via that splits the plane
  - Any trace on L2 (other than GND)
  - Splitting L2 for any reason

ENFORCEMENT:
  - The L2 GND zone in esp-dac.kicad_pcb covers 99.4% of board area
  - Run DRC after every pour fill to confirm no plane interruptions
  - Check at every layout iteration
```

### MCLK Routing

- Route on L1 (F.Cu)
- L2 GND reference must be intact and continuous below the trace
- Do not route MCLK near: buck SW node, SDIO data lines, class-D output traces
- Maximum trace length: minimize — shorter is better for jitter
- Via transitions: minimize — zero via transitions preferred

### I2S Bus (I2S_MCLK, I2S_BCLK, I2S_LRCLK, I2S_DATA)

- Route as a group; keep traces parallel and adjacent
- **Length-match** BCLK, LRCLK, DATA within ±5 mm of each other
- MCLK length-match is less critical (it is the master reference clock)
- Route on L1 with solid L2 GND reference beneath
- Netclass: `I2S` (0.2 mm track)
- Do not route I2S parallel to class-D output traces or buck SW node

### SDIO Bus (CLK, CMD, D0–D3)

- Route as a length-matched group (all within ±5 mm of each other)
- **No stubs**, no branch points, no unused via stubs
- Route directly from ESP32-S3 to J3 (microSD); minimize via transitions
- Do not route SDIO beneath or parallel to the buck SW node
- Netclass: `SDIO` (0.2 mm track)

### USB Differential Pair (USB_DP, USB_DM)

- Route as a differential pair: keep D+ and D− traces **together** at all times
- Maintain **0.2 mm gap** between the two traces throughout
- Target: **90 Ω differential impedance** (verify with manufacturer's calculator)
- Length-match D+ and D− (within ±0.1 mm)
- Place ESD protection (D_USB) between J2 and ESP32-S3 U1, before any branch
- Do not route the USB pair beneath or near switching regulators
- Netclass: `USB_D` (0.2 mm track, 0.15 mm clearance)

### PVDD Routing

- Primary path: Buck output → L3 pour (Zone C) → TAS5825M PVDD pins
- Trace width: 1.0 mm minimum; use fills/pours where possible
- Local bulk caps (Zone C) must be placed within 5 mm of TAS5825M PVDD pins
- PVDD pour on L3 must not share copper with the +3V3 pour
- Netclass: `PVDD` (1.0 mm track, 0.3 mm clearance)

### Speaker Output (Pre-Filter)

- Class-D output traces (OUT+/OUT− before LC filter) carry switching-frequency current
- Route short and wide; minimize loop area
- Do not route these traces near MCLK, I2S, SDIO, or I2C signals
- Keep filter components (L/C) within 5 mm of U2 output pins
- Netclass: `SPK_OUT` (0.8 mm track)

---

## TAS5825M Thermal Via Array (MANDATORY)

The TAS5825M has an exposed pad on the underside that must be connected to L4 copper for thermal dissipation.

| Parameter | Requirement |
|-----------|-------------|
| Minimum via count | 9 (3×3 grid) |
| Via drill diameter | 0.3 mm |
| Via pad diameter | 0.5 mm |
| Grid spacing | Equal spacing within the pad area |
| L4 copper fill | Must cover the full pad area and extend for heat spreading |
| Net | GND (thermal pad is GND-connected) |

> Without the thermal via array, TAS5825M will overheat at moderate output power.
> This constraint is non-negotiable.

**How to implement in KiCad:**
1. Place a custom footprint for U2 that includes the thermal via array in the pad definition, OR
2. After placing U2, manually add thermal vias within the courtyard area
3. Verify on the L4 copper fill that adequate copper area exists for spreading

---

## Design Rule Checks to Run Before Gerber Export

All of the following must pass before submitting for manufacture:

- [ ] ERC (Electrical Rules Check) on all 5 schematic sheets — 0 errors
- [ ] DRC (Design Rules Check) on PCB — 0 errors
- [ ] L2 GND plane continuity — visually verify in Copper Fill view
- [ ] L3 PVDD and +3V3 pours do not overlap or share copper
- [ ] All keepout zones respected (mounting holes, antenna, SW node)
- [ ] All connector placement constraints met (J1, J2, J_SPK_L/R within 3mm of edge)
- [ ] USB differential pair impedance verified with manufacturer
- [ ] TAS5825M thermal via array present and connected to L4 fill
- [ ] Board outline is a closed polygon with 0.05 mm line width on Edge.Cuts
- [ ] All component courtyard boundaries checked for overlaps
- [ ] Silkscreen reference designators readable and outside courtyard
- [ ] Gerber output includes: all Cu layers, F/B SilkS, F/B Mask, Edge.Cuts, drill file

See also: [`../../docs/pcb-layout-checklist.md`](../../docs/pcb-layout-checklist.md)
