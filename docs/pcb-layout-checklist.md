# PCB Layout Checklist — 4-Layer Audio Platform

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M audio platform  
**Status:** Active reference — verify each item before layout handoff

---

## How to Use This Checklist

Work through each section in order. Earlier sections (power, GND plane) must be correct before later sections (signal routing) are meaningful. Mark each item when verified; escalate any that cannot be satisfied before proceeding.

---

## 1. Stackup Verification

- [ ] 4-layer stackup confirmed with PCB manufacturer: L1 / L2 / L3 / L4
- [ ] L2 is designated as GND plane in the design tool — no power pours assigned to L2
- [ ] Controlled impedance requirements for USB differential pairs and SDIO confirmed with manufacturer
- [ ] Board thickness and copper weight per layer specified and confirmed

---

## 2. Component Placement — Power Stage

- [ ] Input connector (barrel jack or screw terminal) placed at board edge
- [ ] F1 (fuse), Q1 (reverse polarity MOSFET), D1 (TVS), and input EMI filter placed in input zone, in protection order, before the main converter
- [ ] Buck converter placed with tight input/output capacitor loop (short switching node)
- [ ] Buck exposed pad (if present) has thermal via array to L4 copper or GND fill
- [ ] PVDD bulk capacitors (≥ 2 × 100 µF + MLCC) placed immediately adjacent to TAS5825M PVDD pins
- [ ] Buck switching node (SW pin trace) does not pass under or near clock, I²S, SDIO, or USB routing

---

## 3. Component Placement — TAS5825M

- [ ] TAS5825M placed with thermal pad vias to L4 copper fill for heat spreading
- [ ] LC output filter (L + C per output leg) placed immediately adjacent to TAS5825M OUT+/OUT– pins
- [ ] PVDD decoupling capacitors (10 µF + 100 nF per pin) placed within 1–2 mm of each PVDD pin
- [ ] GVDD and DVDD bypass capacitors placed per TAS5825M datasheet application diagram
- [ ] I²C (SDA/SCL) traces to ESP32-S3 are short and not near output switching traces
- [ ] BST (bootstrap) capacitors placed per datasheet

---

## 4. Component Placement — Audio Clock Domain

- [ ] 24.576 MHz oscillator placed in the clock zone, physically away from buck switch node and TAS5825M output stage
- [ ] Low-noise LDO for VDD_CLK placed adjacent to oscillator
- [ ] Ferrite bead (if used) placed on VDD_CLK supply trace between 3.3 V and LDO input
- [ ] 100 nF + 1 µF bypass capacitors placed immediately at oscillator supply pin
- [ ] MCLK output trace from oscillator is short (< 20 mm recommended) before reaching TAS5825M and ESP32-S3
- [ ] No switching power or class-D output traces routed parallel to or near MCLK trace

---

## 5. Component Placement — ESP32-S3

- [ ] ESP32-S3 placed with antenna region clear of copper (respect antenna keep-out zone in ESP32-S3 datasheet/module spec)
- [ ] Decoupling capacitors (10 µF + 100 nF per VDD pin) placed adjacent to each power pin
- [ ] USB-C connector placed at board edge; D+/D– traces route directly to ESP32-S3 USB pins with minimal via transitions
- [ ] microSD connector placed for short SDIO routing to ESP32-S3; mechanical slot orientation confirmed for target enclosure
- [ ] 3× rotary encoder connectors or footprints placed according to mechanical design
- [ ] I²C/UART/GPIO expansion header placed at a convenient board edge or mounting location

---

## 6. GND Plane — L2

- [ ] L2 is 100% GND pour with no cuts, splits, or poured power islands
- [ ] No routing vias penetrate L2 in a pattern that creates significant slots or breaks in the plane beneath critical signal paths
- [ ] All GND connections (device pads, chassis ground, shield cans if used) connect to L2 via short vias
- [ ] The buck switching current return path does not create a wide loop: verify that the input cap GND, switch GND, and output cap GND all connect closely to L2
- [ ] No ground splits at board edges that would break continuity under high-frequency signals

---

## 7. Power Routing — L3

- [ ] PVDD pour covers TAS5825M area adequately; not shared with 3.3 V digital pour
- [ ] 3.3 V digital pour covers ESP32-S3, SDIO, and logic areas; no overlap with PVDD pour
- [ ] VDD_CLK trace/pour confined to clock zone; isolated from PVDD and main 3.3 V pours
- [ ] All power pours have adequate clearance from each other and from board edge
- [ ] Power traces are sized for rated current (minimum width per current rating and thermal requirements)

---

## 8. LC Output Filter Routing

- [ ] OUT+ and OUT– traces from TAS5825M to inductors are short (< 5 mm) and wide
- [ ] LC filter components are symmetric: L1/C1 (OUT+ leg) mirrors L2/C2 (OUT– leg) in layout
- [ ] Filter capacitor GND return connects to local GND via, not via a long trace
- [ ] Post-filter traces (inductor to speaker connector) do not run parallel to pre-filter switching traces
- [ ] Speaker connector placed at board edge; post-filter traces kept away from oscillator and I²S routing

---

## 9. Audio Clock Routing

- [ ] MCLK trace routed on L1 with continuous GND reference on L2 directly beneath
- [ ] MCLK trace length minimized; no sharp corners; 45° or curved bends only
- [ ] MCLK trace not parallel to SDIO, USB D+/D–, or switching power traces for more than a few mm
- [ ] MCLK via transitions (if necessary) are minimal; each transition has a nearby GND via

---

## 10. I²S Routing

- [ ] MCLK, BCLK, LRCLK, SDIN routed together as a group from ESP32-S3 to TAS5825M
- [ ] Traces are length-matched within group (or close enough that timing slack is not exceeded)
- [ ] Group routed away from buck switch node and class-D output traces
- [ ] Continuous L2 GND reference beneath entire I²S trace group

---

## 11. SDIO Routing

- [ ] SDIO data lines (D0–D3), CLK, and CMD are length-matched within group
- [ ] No stubs or branch points on SDIO traces
- [ ] Traces are direct from ESP32-S3 to microSD connector; no unnecessary via transitions
- [ ] SDIO traces do not pass beneath or parallel to buck switch node

---

## 12. USB-C Routing

- [ ] D+ and D– routed as a differential pair with target impedance (90 Ω differential; confirm with manufacturer)
- [ ] D+/D– pair length-matched
- [ ] Pair routed with continuous GND reference on L2 beneath both traces
- [ ] ESD protection device (if used) placed immediately after USB-C connector, before D+/D– reach ESP32-S3
- [ ] VBUS sense resistor divider and CC resistors placed per USB-C application requirements

---

## 13. I²C and GPIO

- [ ] I²C (SDA/SCL) pull-up resistors placed close to ESP32-S3 or TAS5825M (confirm placement with actual bus length)
- [ ] I²C traces not routed parallel to class-D output or buck switching traces
- [ ] GPIO expansion header pins labeled in silkscreen

---

## 14. Thermal

- [ ] TAS5825M thermal pad via array provides adequate thermal path to L4 copper fill
- [ ] Buck converter exposed pad (if any) has thermal via array
- [ ] Board area around TAS5825M has sufficient copper fill on L4 for heat spreading
- [ ] Enclosure / heatsink contact zones confirmed (if any) and free of conflicting components

---

## 15. Silkscreen and Fabrication

- [ ] All connectors labeled with function and pin 1 indicator
- [ ] Board revision and project identifier in silkscreen
- [ ] Polarity markers on electrolytic and polarized capacitors
- [ ] Test points for: PVDD, 3.3 V, VDD_CLK, GND, MCLK, I²S signals
- [ ] Board outline and mounting holes placed and confirmed
- [ ] Minimum trace/space, drill sizes, and copper-to-edge clearances confirmed with manufacturer DRC rules

---

## 16. Final DRC and Sign-off

- [ ] ERC (electrical rule check) passed with no errors
- [ ] DRC (design rule check) passed for selected manufacturer rules
- [ ] Net list continuity verified: no unconnected nets, no unintended short circuits
- [ ] Gerber files generated and reviewed in Gerber viewer before submission
- [ ] BOM cross-checked against schematic for all placed components
