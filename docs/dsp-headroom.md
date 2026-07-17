# DSP Headroom and Dynamic Performance

**Project:** esp-dac- — Compact ESP32-S3 + TAS5825M audio platform  
**Status:** Accepted baseline

---

## Goal

Preserve maximum dynamic headroom in the audio path so that transients are reproduced with full impact and no unnecessary compression. The TAS5825M integrated DSP is powerful — the risk is over-configuring it in ways that silently reduce dynamics even when the system is technically safe.

Default settings should protect the hardware while doing the least harm to perceived dynamics.

---

## Why Headroom Matters for 8 Ω Dynamics

Dynamic headroom is the difference between the steady-state signal level and the maximum instantaneous peak the system can reproduce cleanly. On an 8 Ω load:

- TAS5825M at 24 V PVDD can deliver approximately **30 W continuous** (1% THD+N) and sustain brief peaks toward **38 W** (per datasheet figures — confirm against final hardware conditions)
- If DSP gain stages, EQ boosts, or a limiter with a fast attack are applied upstream, the apparent headroom is reduced before the output stage even engages
- Every dB of DSP gain inserted unnecessarily wastes one dB of headroom that the hardware could have provided cleanly

**Rule:** Do not add DSP features that consume headroom unless they serve a clear purpose.

---

## DSP Stage Order (TAS5825M)

The TAS5825M processes audio in a defined internal pipeline. For headroom preservation, understand the order:

```
Input (I²S) → SRC (if enabled) → Biquad EQ banks → Volume/DRC/Limiter → Class-D modulator
```

Gain introduced at any early stage reduces headroom at every later stage. Volume control should be applied as late in the chain as possible.

---

## Headroom-Preserving Defaults

### 1. Digital Volume: Start Conservative

- Set the initial digital volume to leave at least **3–6 dB of headroom** below digital full-scale
- Do not run the digital path at 0 dBFS by default; this leaves no margin for DSP processing headroom

### 2. EQ: Minimum Boost, Maximum Cut

- Avoid bass boost or loudness EQ in the default configuration
- If EQ is used for system correction (room or speaker compensation), prefer **cuts** over boosts
- Any boost consumes headroom. A +3 dB bass boost, for example, means the peak capability of the output is reduced by 3 dB relative to the unprocessed case
- If a specific EQ profile is required, ensure the overall gain structure is compensated (reduce pre-EQ level by the maximum boost amount)

### 3. Subsonic High-Pass Filter: Enable

- Enable a subsonic high-pass filter to remove content below the speaker's usable low-frequency limit
- Typical target: 3rd-order Butterworth or Linkwitz-Riley high-pass, corner frequency in the range of **30–60 Hz** for a compact enclosure (adjust to enclosure/driver tuning)
- Removing subsonic content prevents the amplifier from wasting headroom reproducing frequencies the speaker cannot reproduce, which would otherwise clip the output on deep bass transients

### 4. Limiter: Protect Without Choking Transients

The limiter's job is to protect the speaker and amplifier at sustained high levels. It should not activate during normal transients.

**Recommended limiter strategy:**

| Parameter | Guidance | Rationale |
|-----------|----------|-----------|
| Attack time | Slow (5–20 ms) | Lets short transients through; limiter does not clip drum hits or attacks |
| Release time | Moderate (100–300 ms) | Allows level recovery without pumping artifacts |
| Threshold | Set above the continuous rated power level, not at the absolute clip point | Protects against sustained overload while passing peaks cleanly |
| Ratio | 10:1 or higher (hard limiting) only above threshold | Below threshold: unity gain |
| Lookahead | Use if TAS5825M firmware supports it | Reduces overshoot before limiting kicks in |

> **Key insight:** A limiter with a fast attack (< 1 ms) will clip every loud transient, making the system sound compressed and lifeless even at moderate volumes. Slow attack is the most important parameter for preserving dynamics.

### 5. Loudness Compensation: Off by Default

- Loudness curves boost bass and treble at low listening levels
- They are useful for comfort listening but consume headroom and alter the original dynamic character
- Default: disabled; expose as a user option if needed

### 6. Bass Enhancement / Low-Frequency Boost: Off by Default

- Features like "bass boost" or virtual bass enhancement are the single most effective way to destroy headroom
- A +6 dB bass boost at 80 Hz means every bass transient consumes 4× the power headroom it would without the boost
- Default: off; if the product requires bass enhancement for a specific speaker/enclosure, tune it carefully and reduce the overall gain to compensate

---

## Safe Default Configuration Summary

```
Subsonic HP filter:    ON  — ~40 Hz, order 3 Butterworth (adjust to enclosure)
EQ:                    Flat (no boost; minor correction cuts allowed)
Bass enhancement:      OFF
Loudness:              OFF
Digital volume:        -6 dBFS relative to full-scale (leaves 6 dB margin)
Limiter attack:        10–20 ms
Limiter release:       200 ms
Limiter threshold:     Set at or slightly above continuous rated power level
Limiter ratio:         10:1 above threshold
Clipping protection:   ON (TAS5825M hardware clip limiter)
```

---

## Speaker Protection vs. Dynamics Trade-off

TAS5825M includes integrated speaker protection mechanisms. When tuning these:

- **Thermal fold-back:** Reduces gain when the device temperature rises. This is correct behavior — do not disable. Tune the system thermally so fold-back only occurs under prolonged extreme use.
- **Over-current protection:** Should be set to match the speaker's rated peak current. Too tight a threshold clips peaks; too loose risks speaker damage.
- **Clip detect:** Can be used to trigger an alert or reduce gain. Do not use it as a primary gain control.

---

## Firmware Responsibilities

The ESP32-S3 configures TAS5825M over I²C at boot and can adjust parameters during playback:

- Load a default DSP coefficient table at startup that implements the safe defaults above
- Expose user-facing controls for: volume, EQ preset, bass enhancement (if any)
- Do not apply EQ, limiting, or loudness in firmware/software on the ESP32 side — let TAS5825M DSP handle it to maintain correct gain structure
- If different speaker loads or enclosures are planned, implement selectable DSP profiles rather than hardcoding one configuration

---

## Risks

| Risk | Mitigation |
|------|------------|
| Over-aggressive limiter reducing perceived impact | Use slow attack (10–20 ms minimum); verify with transient test signal |
| Bass boost consuming headroom | Default off; if used, reduce pre-EQ gain by boost amount |
| Running digital path at 0 dBFS | Set default volume 6 dB below full-scale |
| DSP filter instability at high sample rate | Verify biquad coefficient calculation for target sample rate (48 kHz or 96 kHz) |
| Subsonic content causing clipping on bass-heavy material | Enable subsonic HP filter by default |
