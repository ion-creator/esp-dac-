# Schematic Plan — ESP DAC

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M Audio Platform  
**Document status:** Planning reference — drives schematic capture  
**Source docs:** architecture.md, prototype.md, power-tree.md, lc-filter-8ohm.md

---

## Hierarchical Sheet Structure

```
esp-dac.kicad_sch (Root / Overview)
├── sheets/power.kicad_sch        [Sheet 2]  Zone A — Power
├── sheets/mcu.kicad_sch          [Sheet 3]  Zone B — MCU / USB / SD
├── sheets/amplifier.kicad_sch    [Sheet 4]  Zone C — Amplifier
└── sheets/clock_ui.kicad_sch     [Sheet 5]  Zone D — Clock + UI
```

---

## Sheet 1 — Root (esp-dac.kicad_sch)

**Purpose:** Block diagram overview. Shows the 4 hierarchical sheet boxes with
their interconnect labels. No individual components are placed here.

**Hierarchical sheet pin summary:**

| Sheet | Pins out (outputs from sheet) | Pins in (inputs to sheet) |
|-------|-------------------------------|--------------------------|
| Power | PVDD, +3V3, VDD_CLK | — |
| MCU_USB_SD | I2S_MCLK, I2S_BCLK, I2S_LRCLK, I2S_DATA, I2C_SDA, I2C_SCL | PVDD, +3V3 |
| Amplifier | — | PVDD, +3V3, I2S_MCLK, I2S_BCLK, I2S_LRCLK, I2S_DATA, I2C_SDA, I2C_SCL |
| Clock_UI | I2S_MCLK, I2C_SDA, I2C_SCL | +3V3, VDD_CLK |

> Note: I2S_MCLK originates from OSC1 (Clock_UI sheet) but passes through
> ESP32-S3 as a clock source. See architecture.md for the clock routing intent.
> PVDD/+3V3/VDD_CLK use global power symbols across all sheets for supply pins.

---

## Sheet 2 — Power (sheets/power.kicad_sch)

**PCB Zone:** Zone A (0–40mm W, 0–35mm H)  
**Source doc:** power-tree.md

### Component list

| Ref | Description | Package | Status |
|-----|-------------|---------|--------|
| J1 | DC input connector (barrel jack or screw terminal) | 5.5/2.5mm or 2-pin 5mm | TBD package |
| F1 | Fuse, polyfuse or blade, 3A hold / 6A blow | 1206 or blade holder | TBD part |
| Q1 | Reverse-polarity P-ch MOSFET, 30V, 5A | SOT-23 or DPAK | TBD part |
| R_Q1A | Gate pull-down resistor | 0402 | ~100kΩ |
| R_Q1B | Gate bias resistor | 0402 | ~10kΩ (divider) |
| D1 | TVS diode, unidirectional, 26–28V clamp | SMA or SMB | TBD part |
| L_EMI | Common-mode choke or EMI filter | SMD | TBD, ~8–10µH/2A |
| CX1 | X-capacitor after EMI filter | 0805 | 100nF, X2 class |
| U3 | Buck converter (main) | SOIC or exposed-pad SMD | TBD — see OPEN_DECISIONS.md |
| C_IN1 | Buck input bulk cap | Electrolytic 6.3mm | 100µF / 35V |
| C_IN2 | Buck input HF bypass | 0805 | 10µF |
| C_IN3 | Buck input NF bypass | 0402 | 100nF |
| C_OUT_PVDD1–4 | PVDD bulk cap (partial here, rest Zone C) | Polymer electrolytic | 100µF / 35V |
| C_OUT_3V31–2 | 3V3 rail bulk cap | 0805 | 10µF |
| C_OUT_3V3_NF | 3V3 rail HF bypass | 0402 | 100nF |
| U4 | 3.3V LDO, ≥500mA | SOT-223 or TO-252 | TBD part |
| C_U4_IN | LDO input bypass | 0805 | 10µF |
| C_U4_OUT1 | LDO output bulk | 0805 | 10µF |
| C_U4_OUT2 | LDO output HF bypass | 0402 | 100nF |
| U5 | Low-noise LDO for VDD_CLK | SOT-23-5 or similar | TBD — see OPEN_DECISIONS.md |
| FB_CLK | Ferrite bead, clock LDO input | 0402 | ~600Ω @ 100MHz |
| C_U5_IN | LDO input bypass | 0402 | 100nF |
| C_U5_OUT1 | LDO output cap | 0402 | 1µF |
| C_U5_OUT2 | LDO output HF bypass | 0402 | 100nF |

### Net assignments (Zone A)

| Net name | Description | Netclass |
|----------|-------------|----------|
| `VIN` | Raw DC input from J1 | Default (wide trace) |
| `PROTECTED_VIN` | After F1, Q1, D1, L_EMI | Default (wide trace) |
| `PVDD` | Buck output, ~24V | PVDD |
| `VREG_INT` | Buck intermediate rail, ~5V | Default |
| `+3V3` | LDO output, 3.3V digital | 3V3 |
| `VDD_CLK` | Low-noise LDO output | VDD_CLK |
| `GND` | Board ground | Default (L2 plane) |

---

## Sheet 3 — MCU / USB / SD (sheets/mcu.kicad_sch)

**PCB Zone:** Zone B (40–100mm W, 0–35mm H)  
**Source doc:** architecture.md, prototype.md

### Component list

| Ref | Description | Package | Status |
|-----|-------------|---------|--------|
| U1 | ESP32-S3 | Module or QFN-56 | TBD — see OPEN_DECISIONS.md |
| J2 | USB-C receptacle | SMD, board edge | TBD part |
| D_USB | USB-C ESD protection | SOT-363 or similar | Optional, recommended |
| R_CC1 | USB-C CC1 resistor, 5.1kΩ | 0402 | — |
| R_CC2 | USB-C CC2 resistor, 5.1kΩ | 0402 | — |
| R_VBUS | VBUS sense resistor (upper) | 0402 | ~100kΩ |
| R_VBUS2 | VBUS sense resistor (lower) | 0402 | ~100kΩ (divider) |
| J3 | microSD push-push connector | SMD | TBD part |
| R_SD0–3 | SDIO pull-ups 10kΩ | 0402 | ×4 |
| R_SDCMD | SDIO CMD pull-up 10kΩ | 0402 | — |
| C_SD | microSD decoupling | 0402 | 100nF |
| J4 | UART header, 4-pin 2.54mm | TH or SMD | GND/+3V3/TX/RX |
| J6 | I2C/GPIO expansion header | TH or SMD | TBD pinout |
| C_U1_VDD1–N | ESP32-S3 VDD decoupling | 0402 | 10µF + 100nF per VDD pin |

### GPIO assignments (TBD — pending ESP32-S3 package selection)

| Function | GPIO | Notes |
|----------|------|-------|
| I2S_MCLK | TBD | External clock input from OSC1 |
| I2S_BCLK | TBD | I2S bit clock output |
| I2S_LRCLK | TBD | I2S word select output |
| I2S_DATA | TBD | I2S data output to TAS5825M |
| I2C_SDA | TBD | Open-drain; 4.7kΩ pull-up |
| I2C_SCL | TBD | Open-drain; 4.7kΩ pull-up |
| SDIO_CLK | TBD | Length-matched group |
| SDIO_CMD | TBD | Length-matched group |
| SDIO_D0 | TBD | Length-matched group |
| SDIO_D1 | TBD | Length-matched group |
| SDIO_D2 | TBD | Length-matched group |
| SDIO_D3 | TBD | Length-matched group |
| USB_DP | Fixed (ESP32-S3 native USB) | Or to USB-UART bridge |
| USB_DM | Fixed (ESP32-S3 native USB) | Or to USB-UART bridge |
| ENC1_A | TBD | 10kΩ pull-up to +3V3 |
| ENC1_B | TBD | 10kΩ pull-up to +3V3 |
| ENC1_SW | TBD | 10kΩ pull-up to +3V3 |
| ENC2_A | TBD | 10kΩ pull-up to +3V3 |
| ENC2_B | TBD | 10kΩ pull-up to +3V3 |
| ENC2_SW | TBD | 10kΩ pull-up to +3V3 |
| ENC3_A | TBD | 10kΩ pull-up to +3V3 |
| ENC3_B | TBD | 10kΩ pull-up to +3V3 |
| ENC3_SW | TBD | 10kΩ pull-up to +3V3 |
| AMP_PDN | TBD | TAS5825M power-down, active low |
| AMP_FAULT | TBD | TAS5825M /FAULT, open-drain input |
| DISP_* | TBD | Display SPI or I2C signals |

---

## Sheet 4 — Amplifier (sheets/amplifier.kicad_sch)

**PCB Zone:** Zone C (0–70mm W, 35–75mm H)  
**Source docs:** lc-filter-8ohm.md, dsp-headroom.md, architecture.md

### Component list

| Ref | Description | Package | Status |
|-----|-------------|---------|--------|
| U2 | TAS5825M | HTSSOP-32 or VQFN-32 | TBD package |
| C_PVDD_BULK1–4 | PVDD bulk caps (Zone C) | Polymer electrolytic | 100µF / 35V each |
| C_PVDD_HF1–4 | PVDD HF bypass | 1206 X7R | 10µF / 35V each |
| C_PVDD_NF1–N | PVDD per-pin bypass | 0402 | 100nF per PVDD pin |
| C_DVDD | DVDD (3V3) bypass | 0402 | 100nF |
| C_VCOM | VCOM bypass | per datasheet | per TI app note |
| C_GVDD | GVDD bypass | per datasheet | per TI app note |
| C_BST_L | Bootstrap cap, left | 0402 | 100nF ceramic |
| C_BST_R | Bootstrap cap, right | 0402 | 100nF ceramic |
| R_I2C_SDA | I2C SDA pull-up | 0402 | 4.7kΩ (or on MCU sheet) |
| R_I2C_SCL | I2C SCL pull-up | 0402 | 4.7kΩ (or on MCU sheet) |
| R_PDN | AMP_PDN pull-down | 0402 | 100kΩ (keeps amp off at power-up) |
| R_FAULT | /FAULT pull-up | 0402 | 10kΩ to +3V3 |
| L_OUT_L1 | Output inductor, left OUT+ | SMD shielded | ~3–10µH, rated ≥4A |
| L_OUT_L2 | Output inductor, left OUT– | SMD shielded | ~3–10µH, rated ≥4A |
| L_OUT_R1 | Output inductor, right OUT+ | SMD shielded | ~3–10µH, rated ≥4A |
| L_OUT_R2 | Output inductor, right OUT– | SMD shielded | ~3–10µH, rated ≥4A |
| C_OUT_L1 | Filter cap, left OUT+ | 1210 C0G/NP0 or film | ~100–470nF, no X7R |
| C_OUT_L2 | Filter cap, left OUT– | 1210 C0G/NP0 or film | ~100–470nF, no X7R |
| C_OUT_R1 | Filter cap, right OUT+ | 1210 C0G/NP0 or film | ~100–470nF, no X7R |
| C_OUT_R2 | Filter cap, right OUT– | 1210 C0G/NP0 or film | ~100–470nF, no X7R |
| J_SPK_L | Left speaker connector | 2-pin, 5mm pitch, board edge | screw terminal |
| J_SPK_R | Right speaker connector | 2-pin, 5mm pitch, board edge | screw terminal |

### Critical placement notes

1. LC filter components (L_OUT_*, C_OUT_*) must be placed **immediately** after U2 OUT pins
2. Thermal via array under U2 thermal pad: minimum 3×3 = 9 vias, 0.3mm drill, 0.5mm pad
3. PVDD bulk caps: split between Zone A (near buck output) and Zone C (near U2 pins)
4. I2C pull-ups: only ONE set for the whole bus; confirm placement with MCU sheet

---

## Sheet 5 — Clock + UI (sheets/clock_ui.kicad_sch)

**PCB Zone:** Zone D (70–100mm W, 35–75mm H)  
**Source docs:** architecture.md, prototype.md

### Component list

| Ref | Description | Package | Status |
|-----|-------------|---------|--------|
| OSC1 | 24.576 MHz oscillator / TCXO | 2.0×1.6mm or 2.5×2.0mm SMD | TBD part |
| C_OSC1 | OSC1 supply bypass (NF) | 0402 | 100nF |
| C_OSC2 | OSC1 supply bypass (bulk) | 0402 | 1µF |
| FB_CLK | Ferrite bead, OSC1 supply | 0402 | ~600Ω @ 100MHz |
| ENC1 | Rotary encoder + push switch | TH or SMD | TBD part |
| ENC2 | Rotary encoder + push switch | TH or SMD | TBD part |
| ENC3 | Rotary encoder + push switch | TH or SMD | TBD part |
| R_ENC1A–3SW | Encoder pull-up resistors (9 total) | 0402 | 10kΩ each |
| C_ENC1A–3SW | Encoder debounce caps (9 total) | 0402 | 100nF each |
| J5 | Display header | 2.54mm TH or SMD | TBD — SPI or I2C |
| U5 | Low-noise LDO (may also be here) | SOT-23-5 | TBD — see power sheet |

---

## Net Groups Summary

```
Power domain        Nets
──────────────────────────────────────────────────
PVDD group:         PVDD, PVDD_LOCAL (Zone C)
3V3 group:          +3V3, VREG_INT
Clock supply:       VDD_CLK
Ground:             GND (L2 plane, no splits)

Audio signals       Nets
──────────────────────────────────────────────────
I2S bus:            I2S_MCLK, I2S_BCLK, I2S_LRCLK, I2S_DATA
I2C bus:            I2C_SDA, I2C_SCL
SDIO bus:           SDIO_CLK, SDIO_CMD, SDIO_D0, SDIO_D1, SDIO_D2, SDIO_D3
USB pair:           USB_DP, USB_DM
Speaker (L):        SPK_L_P, SPK_L_N  (pre-filter: OUT_L_P, OUT_L_N)
Speaker (R):        SPK_R_P, SPK_R_N  (pre-filter: OUT_R_P, OUT_R_N)

Control             Nets
──────────────────────────────────────────────────
Amp control:        AMP_PDN, /AMP_FAULT
Encoder signals:    ENC1_A, ENC1_B, ENC1_SW, ENC2_A, ENC2_B, ENC2_SW,
                    ENC3_A, ENC3_B, ENC3_SW
Display:            DISP_* (TBD, SPI or I2C)
```
