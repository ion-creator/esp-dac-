# Open Decisions — ESP DAC

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M Audio Platform  
**Status:** Active — resolve before schematic capture begins  
**Priority:** P1 = blocks schematic | P2 = blocks layout | P3 = blocks BOM finalisation

---

## Decision Log

| ID | Title | Priority | Status | Blocking |
|----|-------|----------|--------|---------|
| OD-001 | ESP32-S3 package selection | P1 | OPEN | Schematic sheet 3 |
| OD-002 | Buck converter topology | P1 | OPEN | Schematic sheet 2 |
| OD-003 | TAS5825M package selection | P2 | OPEN | PCB footprint, thermal |
| OD-004 | USB firmware interface | P1 | OPEN | Schematic sheet 3 |
| OD-005 | Display type and interface | P2 | OPEN | Schematic sheet 5 + GPIO |
| OD-006 | Low-noise LDO (U5) placement | P2 | OPEN | Sheet 2 or sheet 5 |
| OD-007 | OSC1 part selection | P2 | OPEN | Schematic sheet 5 |
| OD-008 | GPIO pin assignment | P1 | OPEN | Schematic sheet 3 |
| OD-009 | PVDD rail strategy at 12V input | P1 | OPEN | Architecture |
| OD-010 | I2C pull-up placement | P2 | OPEN | Schematic all sheets |

---

## OD-001 — ESP32-S3 Package Selection

**Priority:** P1 — Blocks schematic sheet 3 and PCB footprint  
**Status:** OPEN

### Options

**Option A: ESP32-S3-WROOM-1 module (recommended)**
- Self-contained module with antenna, RF shield, 8MB flash, 8MB PSRAM
- Pre-certified (FCC/CE) — avoids custom RF layout
- 22.4 × 18 mm castellated LGA footprint
- Antenna keepout: defined in module datasheet (typically 3mm copper-free around antenna area)
- Fewer GPIO pins accessible than bare chip (module routes specific pins to castellations)

**Option B: ESP32-S3 bare chip (QFN-56)**
- Maximum GPIO flexibility, smallest footprint
- Requires: external antenna (PCB trace or connector + external), RF layout expertise, separate certification
- More complex schematic (external decoupling, power filter network, crystal if not TCXO)

### Recommendation
Use **Option A (WROOM-1 module)** for first prototype:
- Reduces RF design risk
- Pre-certified for FCC/CE (faster to market)
- Sufficient GPIO for this design (verify pin count against GPIO assignment table)

### Action required
1. Confirm WROOM-1 GPIO count meets all signal requirements (see SCHEMATIC_PLAN.md)
2. Download ESP32-S3-WROOM-1 KiCad footprint from Espressif GitHub or create from datasheet
3. Define antenna keepout region on PCB from module datasheet

---

## OD-002 — Buck Converter Topology

**Priority:** P1 — Blocks schematic sheet 2 and entire power architecture  
**Status:** OPEN

### The problem
The board operates from 12–24V DC input and must deliver:
- PVDD: ideally ≈ 24V for best TAS5825M performance
- +3V3: for all digital logic

**If input is 24V:** a simple buck converter can produce +3V3 (or +5V → LDO). PVDD is taken directly from input (or lightly regulated).

**If input is 12V:** PVDD must somehow be ≈24V for full dynamic headroom. A buck cannot step up. Options:
1. **Boost/SEPIC to 24V for PVDD** — more complex; boost converter required in Zone A
2. **Reduce PVDD to 12V and accept lower headroom** — simpler but limits max output power
3. **Design only for 24V operation** — simplify power tree, document 24V-only requirement

### Impact
- Option 1 (boost/SEPIC): adds components, cost, PCB area, EMI
- Option 2 (12V PVDD): TAS5825M output power at 12V PVDD is reduced; verify if acceptable
- Option 3 (24V only): simplest; may not meet spec if 12V operation is required

### Action required
1. Decide if 12V input with full dynamic headroom is a hard requirement
2. If yes → design boost or SEPIC stage in Zone A
3. If no → document minimum recommended input voltage
4. Select buck converter IC once topology is decided

---

## OD-003 — TAS5825M Package Selection

**Priority:** P2 — Blocks PCB footprint assignment and thermal via design  
**Status:** OPEN

### Options

**Option A: HTSSOP-32 (TAS5825MPWPR)**
- Exposed pad on underside (thermal connection)
- Gull-wing leads: hand-solderable, easier to inspect/rework
- 6.1 × 11 mm body, 0.65mm pitch
- Standard KiCad footprint available

**Option B: VQFN-32 (TAS5825MRGET)**
- Larger exposed thermal pad area (potentially better thermal performance)
- 5 × 5 mm body, 0.5mm pitch, leadless (harder to inspect/rework)
- No leads to reflow: requires controlled paste stencil
- Standard KiCad footprint available

### Recommendation
**Option A (HTSSOP-32)** for first prototype:
- Hand-solder/reflow rework is possible
- Thermal performance is acceptable with proper thermal via array
- Easier to bring up and debug on first spin

### Action required
1. Confirm TAS5825MPWPR availability from supplier
2. Add footprint to `esp-dac.kicad_sym` and assign footprint property
3. Define thermal via array pattern in PCB footprint

---

## OD-004 — USB Firmware Interface

**Priority:** P1 — Affects USB-C schematic, boot mode circuitry, and potentially J4 UART  
**Status:** OPEN

### Options

**Option A: ESP32-S3 Native USB (USB OTG)**
- Uses ESP32-S3 built-in USB OTG hardware
- No additional IC required (saves BOM/space)
- Firmware: implement USB CDC ACM or JTAG via TinyUSB
- D+/D− connect directly from J2 to ESP32-S3 USB GPIO pins
- Need: CC1/CC2 resistors (5.1kΩ), optional ESD protection
- Limitation: only USB 2.0 FS (12 Mbps)

**Option B: External USB-UART Bridge (e.g., CP2102N, CH340)**
- Adds one IC to BOM
- Simpler firmware (standard UART for flashing)
- ESP32-S3 USB GPIO pins not used (freed for other functions)
- May be more reliable for initial bring-up

**Option C: Combined — native USB + UART header**
- Native USB for firmware/OTA
- J4 UART header remains for debug/backup flashing

### Recommendation
**Option A** (native USB) + **J4 UART header** (Option C):
- Reduces BOM
- ESP32-S3 native USB is well-supported in ESP-IDF 5.x
- UART header provides backup path during bring-up

### Action required
1. Select USB interface approach
2. Design boot mode circuitry (GPIO0 strapping, EN reset button)
3. Route D+/D− from J2 with 90Ω differential pair rule enforced

---

## OD-005 — Display Type and Interface

**Priority:** P2 — Affects clock_ui.kicad_sch and GPIO count  
**Status:** OPEN

### Options

**Option A: SPI display (e.g., 0.96" OLED SSD1306 SPI, or small TFT)**
- Requires: SPI_CLK, SPI_MOSI, SPI_CS_DISP, DC, RST (5 GPIO signals + +3V3/GND)
- Can share SPI bus with other devices (different CS)
- Higher data rate possible

**Option B: I2C display (e.g., 0.96" OLED SSD1306 I2C)**
- Uses existing I2C_SDA/I2C_SCL bus (no additional GPIO)
- Limited to 2 devices on bus: TAS5825M (0x4C or 0x4D) + display (0x3C or 0x3D)
- Simpler wiring, fewer GPIO required
- Lower data rate

### Recommendation
**Decide based on GPIO availability** after OD-001 is resolved:
- If GPIO is tight → Option B (I2C, shares existing bus)
- If GPIO is available → Option A (SPI, more flexibility)

### Action required
1. Resolve OD-001 first (ESP32-S3 package → available GPIO count)
2. Select display module; obtain pinout
3. Update clock_ui.kicad_sch with actual J5 pinout

---

## OD-006 — Low-Noise LDO (U5) Placement

**Priority:** P2 — Affects which schematic sheet contains U5  
**Status:** OPEN

### Options

**Option A: U5 in Zone A (on power sheet)**
- Pros: all power generation in one place; shorter trace from +3V3 to U5 input
- Cons: VDD_CLK trace must travel from Zone A to Zone D (OSC1 location); more path for noise pickup

**Option B: U5 in Zone D (on clock_ui sheet)**
- Pros: U5 is physically close to OSC1; shortest VDD_CLK trace; minimal noise injection on LDO output
- Cons: U5 input supply (+3V3) must travel from Zone A/B to Zone D

### Recommendation
**Option B** (U5 in Zone D, close to OSC1):
- VDD_CLK trace is the most noise-sensitive path → minimize its length
- +3V3 is a relatively quiet rail; routing it to Zone D is acceptable with bypass caps

### Action required
1. Confirm OSC1 and U5 fit in Zone D (70–100mm W, 35–75mm H)
2. Move U5 to clock_ui.kicad_sch schematic if confirmed
3. Add ferrite bead (FB_CLK) on VDD_CLK input at U5 input pin

---

## OD-007 — OSC1 Part Selection

**Priority:** P2 — Affects clock_ui.kicad_sch footprint and jitter budget  
**Status:** OPEN

### Requirements
- Frequency: exactly 24.576 MHz
- Supply: 3.3V (or 1.8V — confirm against final U5 output voltage)
- Output: LVCMOS compatible with ESP32-S3 and TAS5825M
- Jitter: as low as practical; target <100 ps RMS; TCXO/MEMS preferred over XO

### Candidate parts
See BOM_CANDIDATES.md — OSC1 section  
MEMS oscillator (SiTime SIT8208 or similar) preferred for low jitter + small size

### Action required
1. Confirm OSC1 supply voltage (3.3V vs. 1.8V) based on U5 decision
2. Select part; verify KiCad footprint availability
3. Verify MCLK fanout loading (OSC1 drives both ESP32-S3 and TAS5825M)
4. Check if a buffer is needed between OSC1 and MCLK loads

---

## OD-008 — GPIO Pin Assignment (ESP32-S3)

**Priority:** P1 — Required before schematic capture of sheet 3 can be completed  
**Status:** OPEN — depends on OD-001

### Action required
1. Resolve OD-001 (module vs. bare chip)
2. Download ESP32-S3 datasheet and GPIO matrix table
3. Map all required signals to specific GPIO numbers:
   - I2S: MCLK, BCLK, LRCLK, DATA (4 pins) — use I2S0 peripheral pins
   - I2C: SDA, SCL (2 pins) — use I2C0 peripheral
   - SDIO: CLK, CMD, D0–D3 (6 pins) — must use SDMMC-compatible GPIO
   - USB: DP, DM (2 fixed pins — GPIO19/GPIO20 for native USB)
   - Encoders: A/B/SW × 3 (9 pins) — any GPIO
   - AMP_PDN: 1 pin
   - AMP_FAULT: 1 pin (interrupt-capable GPIO preferred)
   - Display: 2–5 pins depending on interface
   - Boot strapping: GPIO0, GPIO46 (fixed)
   - Total: ~28–35 GPIO required
4. Check against WROOM-1 available castellations (~36–43 accessible GPIO)
5. Update SCHEMATIC_PLAN.md GPIO table with final assignments

---

## OD-009 — PVDD Rail Strategy at 12V Input

**Priority:** P1 — Directly impacts Zone A power architecture  
**Status:** OPEN — see OD-002

See OD-002 for full discussion. Summary:
- If 12V input AND full headroom required → boost/SEPIC stage needed
- If 24V input only → simple pass-through or single buck step-down
- **Decision here determines the entire Zone A schematic**

---

## OD-010 — I2C Pull-up Placement

**Priority:** P2 — Only one set of pull-ups per I2C bus (avoid duplicating)  
**Status:** OPEN

### Context
The I2C bus is shared by:
- TAS5825M (amplifier sheet)
- Display module if I2C (clock_ui sheet)
- Pulled up on MCU sheet or amplifier sheet

### Problem
Each sheet has a natural place to add I2C pull-ups. But the bus requires **exactly one** set of pull-ups (4.7kΩ to +3V3 on SDA and SCL). If multiple sheets each add pull-ups, the effective pull-up resistance becomes too low (parallel combination), violating I2C spec.

### Options
- Pull-ups on MCU sheet: close to master (ESP32-S3)
- Pull-ups on amplifier sheet: close to TAS5825M
- Pull-ups on a separate net tie area on clock_ui sheet

### Recommendation
Place the single set of **4.7kΩ pull-ups on the amplifier sheet** (sheet 4), physically close to TAS5825M on the PCB. The I2C bus is relatively short and the ESP32-S3 is nearby. This avoids routing long open-drain traces without pull-up across zone boundaries.

### Action required
1. Confirm I2C bus length after component placement
2. Add 2 × 4.7kΩ pull-up resistors (R_I2C_SDA, R_I2C_SCL) on amplifier sheet
3. Remove any duplicate pull-ups from other sheets
4. If display is I2C and is far from amplifier, reconsider placement — may need pull-ups midpoint or at MCU

---

## Resolved Decisions (none yet)

*All decisions above are open. This section will be populated as decisions are made.*

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-07-17 | Skeleton generator | Initial open decisions from documentation review |
