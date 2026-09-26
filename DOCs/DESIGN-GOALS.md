# Design Goals

What P2-VibrationSense should do and how its code is organized. Wiring is in [`../HARDWARE.md`](../HARDWARE.md).

## Purpose

Measure the vibration of a P2-driven BLDC motor platform with two sensors, and report what the vibration looks like:

- **Piezo sensor:** the vibration's frequency. It gives no direction.
- **6DOF IMU (LSM6DSL):** the vibration's size and frequency along each of X, Y and Z.

The focus is the vibration during **motor startup and spin-down**, when the motor's speed sweeps from rest to full speed and back, passing through the platform's resonances.

## Target motors and sample rates

| Motor | Top speed | Once per rev | Role |
| ----- | --------- | ------------ | ---- |
| 6.5" hub motor | ~450 RPM, 30 poles (15 pole pairs) | 7.5 Hz | **Primary target** |
| Faster motors | ~4,000 RPM and up | ~67 Hz and up | Secondary; same drivers, higher rates |

Vibration shows up at the once-per-rev rate and its multiples, at the platform's resonances (typically tens to a few hundred Hz), and at electrical ripple from the motor's poles. Sampling must be more than twice the highest frequency of interest.

6.5" hub motor at full speed (450 RPM):

| Source | Frequency |
| ------ | --------- |
| Once per rev (imbalance) and multiples | 7.5 Hz, 15, 22.5 … |
| Electrical frequency (7.5 Hz × 15 pole pairs) | 112.5 Hz |
| 2× electrical | 225 Hz |
| 6× electrical (six-step commutation ripple) | 675 Hz |

All of these sweep up from 0 during startup and back down during spin-down.

**Proposed starting rates** (the sample rate is a `start()` parameter, so these are defaults, not fixed):

| Sensor | 6.5" hub motor | Faster motors |
| ------ | -------------- | ------------- |
| IMU accelerometer | **1.66 kHz** (833 Hz bandwidth, covers the 675 Hz commutation ripple), **±8 g** (±4 g clipped on desk thumps) | 1.66–3.33 kHz (up to the chip's 1.5 kHz analog bandwidth), ±8 g |
| IMU gyro | optional, same rate | optional |
| Piezo | **1–2 kHz** | 5–10 kHz |

Notes:

- **LSM6DSL limits (datasheet):** sample rates 12.5 Hz–6.66 kHz. At 1.66 kHz and above, an analog filter caps accelerometer bandwidth at 1.5 kHz, so 3.33 kHz already captures everything; 6.66 kHz adds data but no bandwidth. Noise density is 80 µg/√Hz at ±2/±4 g, 90 at ±8 g, 130 at ±16 g.
- **Piezo at low frequency:** piezo sensors usually respond poorly near DC, and this sensor's low-frequency cutoff is unknown. It may barely register the hub motor's 7.5 Hz rotation; it's better suited to faster content. First captures will show what it picks up.
- **Why not 833 Hz for the hub motor:** its 416 Hz bandwidth would lose the 675 Hz commutation ripple, or fold it down into lower frequencies. 833 Hz is enough only if the mechanical frequencies (up to ~225 Hz) are all that matter.
- **Low-frequency analysis needs long windows.** Resolving 7.5 Hz from 15 Hz needs windows around 0.5–1 s (e.g. 1024 samples at 1.66 kHz = 0.6 s, 1.6 Hz resolution). During a fast spin-up the frequency changes within a window, so window length trades time detail against frequency detail; tune it on real captures.
- **Use the measured rate, not the nominal one.** On our board, a nominal 1.66 kHz measured **1,732 Hz** accel-only and **1,702.5 Hz** with the gyro on (repeatable; the datasheet gives no rate tolerance). Spacing was very steady (576–578 µs, 587 µs) with no missed samples, so Spin2 keeps up at this rate. Analysis must derive the sample rate from the CT offsets; using 1,660 Hz would put every frequency ~4% off.
- **Measured noise at rest (±4 g, 1.66 kHz):** accel ~2 mg RMS per axis, close to the datasheet's 80 µg/√Hz prediction (~2.3 mg).
- **Piezo rates come from averaging.** The ADC's SINC2 sampling mode only offers power-of-two periods (8,192 clocks = 24.4 kHz). The piezo capture cog averages N readings per stored sample, so the rate is exactly `clkfreq / (8192 × N)` (e.g. 2,034.5 Hz for N = 12), with the averaging acting as a filter.
- **Piezo pin recovery.** After the ADC reads GIO or VIO, P0 starts near 0 V and ramps back to its resting level at about 150 mV/ms, taking ~7 ms. Calibration therefore happens only while idle, followed by a 50 ms settle, never during a capture. The cause isn't known.
- **Measured piezo characteristics (desk mount):** resting level ~1,134 mV; noise ~1.3 mV RMS, mostly low-frequency (averaging 12 readings didn't reduce it). A firm desk thump gave +28 / −37 mV, decaying to the noise floor within ~200 ms, with a rough ~48 Hz dominant frequency. The 1x ADC range fits; the 3.16x range is centered near 1.64 V and would clip at this resting level.
- **First capture:** run both sensors through one startup and spin-down to see where the energy actually is, then settle final rates.

## Structure

Two independent driver objects and one top-level demo.

| Object | Sensor | Interface to sensor |
| ------ | ------ | ------------------- |
| Piezo driver | Amplified piezo on P0 | P0 as an ADC smart pin, calibrated against its pin group's GND (`P_ADC_GIO`) and 3.3 V (`P_ADC_VIO`) |
| IMU driver | LSM6DSL Click on P32–P47 | **SPI** (CS P40, SCK P41, MISO P42, MOSI P43), with data-ready interrupt on P36 |

### Why SPI for the IMU

- Speed: a 12-byte sample (3 accel + 3 gyro axes) takes roughly 300–350 µs at 400 kHz I2C, which caps the rate near 3 kHz. At SPI's up to 10 MHz it takes a few µs, so the bus doesn't limit the sensor's highest rates (accelerometer up to 6.66 kHz).
- The Click's jumpers ship set to SPI, so no hardware changes are needed.
- The INT1 data-ready pulse gives each sample an accurate timestamp.

### IMU driver layers

```
Demo program
   └── IMU capture object      (capture cog: waits for data-ready on P36,
         │                       grabs GETCT, stores samples in the buffer)
         └── isp_lsm6dsl        (register read/write, WHO_AM_I check, rate and
               │                  range setup, data-ready routing, 12-byte burst read)
               └── isp_spi       (smart pin SPI, selectable mode 0–3; moves bytes only)
```

`isp_spi` is derived from `jm_ez_spi` (mode 0 only, kept as the reference). The LSM6DSL runs in **SPI mode 3**, the only mode its datasheet documents.

**One cog owns the IMU pins.** `isp_spi` configures smart pins and drives pin direction in the cog that calls it, and `isp_lsm6dsl` drives CS the same way. P2 cogs' output and direction registers are OR'ed at the pins, so `start()` and every later call must come from the same cog: the capture cog. Other cogs talk to that cog through shared variables, never by calling the SPI or LSM6DSL methods directly.

## Driver behavior

Each driver:

1. Runs in its own cog, which only paces samples, timestamps them and stores them.
2. Writes each sample, with its **CT clock offset**, into a buffer.
3. Captures at a steady rate while turned on, then stops. Processing happens afterward, not during capture.

### Common interface

Both objects expose roughly the same calls. This sketch isn't final:

```
start(basePin, sampleRateHz) : ok   ' launch cog, configure sensor, idle
startCapture(pBuf, maxSamples)      ' begin filling buffer
stopCapture()                       ' or it stops itself when full
isCapturing() : bool
sampleCount() : n
getSample(i) : ctOffset, value(s)   ' ctOffset relative to capture start
stop()                              ' release cog
```

The piezo driver returns one value per sample. The IMU driver returns per-axis values.

### Sampling rules

- **Even spacing.** Frequency analysis assumes a steady sample rate. The piezo cog paces itself with `WAITCT`; the IMU cog paces off the sensor's data-ready edge on P36. The CT offsets confirm the spacing and show any jitter.
- **Timestamps relative to capture start.** CT is a 32-bit counter and wraps about every 21 s at 200 MHz. Storing each offset from the capture start works for any capture shorter than that.
- **Sample rate over twice the highest frequency of interest.** See [Target motors and sample rates](#target-motors-and-sample-rates).
- **Buffer size.** Hub RAM is 512 KB. At the hub-motor rates, per-sample timestamps fit easily:

  | Stream | Per second | 20 s capture |
  | ------ | ---------- | ------------ |
  | IMU accel, 1.66 kHz (6 B data + 4 B CT) | 16.6 KB | 333 KB |
  | IMU accel + gyro, 1.66 kHz (12 B + 4 B) | 26.6 KB | 533 KB, too big; ~18 s max with nothing else in memory |
  | IMU accel, 833 Hz (6 B + 4 B) | 8.3 KB | 167 KB |
  | Piezo 2 kHz (4 B + 4 B) | 16 KB | 320 KB |
  | Piezo 1 kHz (4 B + 4 B) | 8 KB | 160 KB |

  At the faster-motor rates (piezo 5–10 kHz), per-sample timestamps use memory quickly; a timestamp per block of samples may be needed there.

## Demo program

The top-level demo:

1. Creates both driver objects.
2. Starts them.
3. Captures samples.
4. Stops them.
5. Displays the samples with an analysis of what was learned.

Implemented as `src/vibration_demo.spin2`, with the frequency analysis in `src/isp_fft.spin2`. It captures 8 s (piezo 2,034.5 Hz; accel 1.66 kHz nominal at **±8 g**) and prints a text report through `DEBUG`.

### Analysis

Each channel's mean is removed first, which also removes gravity on the IMU axes (about 1 g on one axis at rest). The report has three sections:

| Section | Contents |
| ------- | -------- |
| Level and vibration size | Per channel: resting level, RMS, peak swing and when it happened. Accel axes warn when samples **clip** (hit the range limit). |
| Strongest frequencies | Top 3 peaks (frequency and amplitude) in the **most active** 4,096 samples of each channel. |
| Dominant frequency over time | Per 1,024-sample segment (~0.5–0.6 s): dominant frequency and amplitude, or **quiet** when the segment's RMS is under 2× the channel's quietest segment. |

Piezo amplitudes are in mV and only relative (the sensor is uncalibrated); accel amplitudes are in mg.

`isp_fft` is a radix-2 FFT in Spin2 floats with a Hann window. Frequency and amplitude are refined between bins by parabolic interpolation. On synthetic tones (`src/fft_test.spin2`) it found frequencies within 0.05 Hz and amplitudes within ~2% (5.6% worst at n = 1,024). A 4,096-point analysis takes ~0.76 s.

### What the demo showed on a desk (2026-09-25)

- **Quiet desk:** 60 Hz in the piezo (~1.3 mV, likely mains pickup) and 120 Hz on all accel axes (~1 mg, twice mains frequency, likely a transformer or fan nearby).
- **Thumps:** the desk rings at ~34 Hz side to side (X/Y) and ~63 Hz vertically (Z). The piezo's dominant frequency during thumps (34–35 Hz) matched the accelerometer's.
- **Hand shaking:** 6–9 Hz at 25–48 mg on X/Y; the piezo didn't register it.
- **Range:** hard thumps swung Z by 5–7 g, clipping at ±4 g and briefly even at ±8 g. The default is now ±8 g; ±16 g would suit hard impacts at the cost of more noise (130 vs 90 µg/√Hz).

## Open questions

1. **Output:** plain `DEBUG` text is implemented. Graphical `DEBUG` windows (live waveform and spectrum plots in PNut-Term-TS) could be added.
2. **Capture length:** how long does one startup plus spin-down take? This sets buffer sizes; a single capture must stay under ~21 s (CT wrap).
3. **Units:** is vibration size in g enough, or is velocity (mm/s) also wanted?
4. **Gyro:** capture the gyro axes too, or accelerometer only?
