<!---

This file is used to generate your project datasheet. Please fill in the information below and delete any unused
sections.

You can also include images in this folder and reference them in the markdown. Each image must be less than
512 kb in size, and the combined size of all images must be less than 1 MB.
-->

## How it works

This is a noise-shaping SAR ADC: an 8-bit differential successive-approximation (SAR) converter with a first-order
error-feedback (EF) noise-shaping loop, designed for low-frequency sensor signals.

- **SAR core.** Two 8-bit binary-weighted capacitor DACs (256 x 34.6 fF MiM unit capacitors per side, 8.9 pF) sample
  the differential input on their top plates (top-plate switch to VCM). A comparator (pre-amplifier, StrongArm latch,
  SR latch) and a digital SAR controller then resolve the 8 bits, MSB first. A conversion takes 12 clock cycles.
- **Noise shaping.** After each conversion, the residue voltage left on the DAC top plates is amplified x30 by a
  switched-capacitor amplifier and stored on a residue capacitor (Cres). During the next conversion Cres is
  charge-shared onto the top plates, so the previous quantization error is subtracted from the next sample. This
  pushes the quantization noise to high frequencies (noise transfer function 1 - z^-1). After digital low-pass
  filtering and decimation (e.g. sinc^2 with an oversampling ratio of 8) the in-band resolution rises well above
  8 bits.
- **Modes.**
  - `NS_EN` (ui[0]): 1 = noise shaping on; 0 = plain 8-bit SAR.
  - `AMP_SHARE` (ui[1]): 0 = separate comparator pre-amplifier and residue amplifier; 1 = one amplifier does both.
- **Bias.** A single bias pin, `IBIAS` (ua[5]): 1 uA is drawn *out of* the pin. An on-chip mirror derives the residue
  amplifier and comparator bias currents from it.
- **Timing.** With a 96 kHz clock, the ADC converts at 8 kS/s. `EOC` (uio[0]) rises when a new code is on `uo[7:0]`
  (B7 = MSB). With OSR = 8 that gives a 1 kS/s output after decimation.
- **Simulated performance** (transistor level, typical corner, 0.9 x full-scale sine): about 7.5 bits ENOB raw,
  11.5-11.9 bits ENOB after sinc^2 decimation with OSR = 8 (1 kS/s, 500 Hz bandwidth). With `NS_EN` = 0 it is a plain
  8-bit SAR (about 7.9 bits ENOB at Nyquist).

## How to test

1. **Supplies and references** (low-impedance sources; each input samples onto about 9 pF, so buffer the inputs):
   - ua[2] `VCM` = 0.9 V
   - ua[3] `VREF_P` = 1.8 V
   - ua[4] `VREF_N` = 0 V (or ground)
   - ua[5] `IBIAS`: draw 1 uA out of the pin, with a current sink to ground or a resistor to ground (about
     0.8 MOhm; the pin sits at roughly 0.7-1.0 V). Adjust for 1 uA.
2. **Input:** differential signal on ua[1] `V_P` and ua[0] `V_N`, both centred at 0.9 V. Full scale is
   +-1.8 V differential (VREF_P - VREF_N); keep the amplitude to at most 0.9 x full scale, i.e. up to 0.81 V peak on
   each pin, in antiphase. Example: a 256 Hz sine.
3. **Digital setup:** clock 96 kHz on `clk`, `ena` = 1, `ui[0]` (NS_EN) = 1, `ui[1]` (AMP_SHARE) = 0, then release
   `rst_n` (low -> high).
4. **Capture:** read `uo[7:0]` on every rising edge of `EOC` (uio[0]), at 8 kS/s. Record, for example, 4096
   consecutive codes. Discard the first few conversions after reset.
5. **Evaluate:**
   - Raw spectrum: FFT of the 8-bit codes. With noise shaping on, the noise floor rises towards fs/2 (shaped noise).
   - Noise-shaped result: decimate by 8 with a sinc^2 filter (moving average of 8, applied twice, keep every 8th
     sample), then compute SNDR/ENOB in the 0-500 Hz band.
   - Compare with `NS_EN` = 0 (plain SAR) and with `AMP_SHARE` = 1.

## External hardware

- A low-noise differential signal source (or a single-ended source with a differential driver), with buffers.
- 0.9 V and 1.8 V reference sources.
- A 1 uA current sink (or a resistor) for IBIAS.
- A logic analyser or microcontroller to capture `uo[7:0]` on `EOC`.
