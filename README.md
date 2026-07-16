# esp-dac-

**Compact integrated audio platform — ESP32-S3 + TAS5825M**

A compact, 4-layer PCB audio player and amplifier designed for superior dynamics on 8 Ω loads, operating from a 12–24 V input with best-performance mode at 24 V.

---

## Overview

| Item | Value |
|------|-------|
| MCU / control | ESP32-S3 |
| DSP / DAC / amplifier | TAS5825M (class-D, integrated DSP) |
| PCB | 4-layer, compact form factor |
| Input voltage | 12–24 V DC |
| Best-performance mode | 24 V audio rail |
| Target load | 8 Ω, superior dynamics |
| Storage | microSD via SDIO |
| Firmware interface | USB-C (programming / OTA update) |
| User control | 3 rotary encoders |
| Expansion | I²C, UART, GPIO header |

---

## Architecture Summary

```
 12–24 V DC input
       │
  [Input protection]           fuse → reverse-polarity MOSFET → TVS → EMI filter
       │
  [Buck converter]             main switching regulator
       ├─── PVDD (≈24 V)  ──► TAS5825M power stage
       ├─── 3.3 V digital ──► ESP32-S3 · SDIO · USB bridge · UI logic
       └─── LDO low-noise ──► Audio clock oscillator domain
                                    │
                              [24.576 MHz oscillator]
                                    │
                      MCLK / BCLK / LRCLK ──► TAS5825M (I²S slave)
                                              (also drives ESP32-S3 I²S master)

  ESP32-S3  ──── I²S ────►  TAS5825M DSP ──► Class-D output ──► [LC filter] ──► 8 Ω speaker
              ── I²C ────►  TAS5825M config
              ── SDIO ───►  microSD card
              ── USB-C ──►  programming / firmware update
              ── GPIO ───►  3× rotary encoders
              ── header ──► I²C / UART / GPIO expansion
```

### PCB Stackup (4-layer)

| Layer | Role |
|-------|------|
| L1 | Components, critical signal routing (I²S, SDIO, USB, clock) |
| L2 | **Continuous GND plane** — uninterrupted across the full board |
| L3 | Power distribution (PVDD, 3.3 V, LDO branch) + secondary signals |
| L4 | Signals, auxiliary power, thermal spreading |

**Rule:** L2 must not be cut or split. All signal return currents flow on L2.

### Power Domains

| Rail | Source | Consumer |
|------|--------|----------|
| PVDD (~24 V) | Buck output or direct input | TAS5825M power stage |
| 3.3 V digital | Buck → LDO or direct buck | ESP32-S3, SDIO, USB, encoders, I²C/UART |
| VDD_CLK (low-noise) | Dedicated LDO from 3.3 V or 5 V | 24.576 MHz audio oscillator |

### Audio Clock Strategy

- Dedicated **24.576 MHz low-jitter oscillator** (TCXO preferred)
- Powers the audio reclock domain on its own filtered rail
- ESP32-S3 I²S configured with external MCLK source (`I2S_CLK_SRC_EXTERNAL`)
- TAS5825M operates as **I²S slave**, locking to the stable external clock
- Optimized for 48 kHz family (48 / 96 / 192 kHz); 44.1 kHz content handled via SRC
- `mclk_multiple` = 384 recommended for 24-bit audio to avoid sample rate error

---

## Design Goals and Priorities

1. **Superior dynamics on 8 Ω** — maximize instantaneous headroom, avoid unnecessary compression
2. **24 V audio rail** — key enabler for ~30 W continuous / ~38 W peak on 8 Ω per TAS5825M
3. **Clean power architecture** — separated PVDD, digital, and clock domains; local low-ESR bulk near TAS5825M
4. **Stable audio clock** — dedicated oscillator domain, physically and electrically isolated from switching noise
5. **Compact integration** — full functionality on a 4-layer board; no off-board supply conditioning required
6. **Reliable connectivity** — SDIO for high-throughput card access, USB-C for reliable firmware updates

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [`docs/architecture.md`](docs/architecture.md) | Full system block architecture, subsystem roles, constraints |
| [`docs/power-tree.md`](docs/power-tree.md) | Input protection chain, rail strategy, bulk/decoupling targets |
| [`docs/dsp-headroom.md`](docs/dsp-headroom.md) | DSP defaults, limiter/EQ guidance for maximum dynamics |
| [`docs/lc-filter-8ohm.md`](docs/lc-filter-8ohm.md) | LC output filter design for 8 Ω speaker load |
| [`docs/pcb-layout-checklist.md`](docs/pcb-layout-checklist.md) | 4-layer PCB layout checklist |

---

## Status

Initial architecture accepted. Documentation reflects the baseline design direction.
Hardware design, schematic, and firmware are not yet started.
