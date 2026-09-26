# P2-VibrationSense

Sensing the vibration of a Propeller 2 (P2) BLDC motor platform with a piezo sensor and a 6-degrees-of-freedom (6DOF) IMU.

The project provides two independent Spin2 driver objects, one per sensor. Each captures a burst of timestamped samples into a buffer you supply, so you can analyze the vibration afterward. A demo program uses both to report:

- **Piezo sensor:** the vibration's frequency. The piezo gives no direction.
- **6DOF IMU:** the vibration's size (in g) and frequency along each of X, Y and Z.

> **Status:** early development. Both capture objects and the demo work on hardware: the IMU at 1.66 kHz and the piezo at 2 kHz with no missed samples, and the demo reports levels, frequencies and a timeline from an 8 s capture. The interface may still change.

## The two driver objects

Both objects work the same way:

1. **Start** the object. It launches its own cog and sets up the sensor.
2. **Capture.** The cog samples the sensor at a steady rate. It stores each sample with its **CT clock offset** (P2 system-clock ticks since the first sample of the capture) into your buffer.
3. **Stop.** Capture ends when you stop it or when the buffer is full.
4. **Read** the samples back and process them. Nothing is analyzed during capture, so the sampling cog does nothing but sample.

Because each sample carries its own timestamp, you can check the actual sample spacing and jitter, not just assume the nominal rate.

### Common interface

Both objects share these calls, and can be called from any cog:

| Call | What it does |
| ---- | ------------ |
| `start(...) : ok` | Launch the capture cog, set up the sensor, return once ready (parameters differ; see below) |
| `startCapture(pBuf, maxSamples) : ok` | Begin filling your buffer; stops by itself when full |
| `stopCapture()` | End a capture early |
| `isCapturing() : bool` | True while samples are being written |
| `sampleCount() : n` | Samples written in the current or last capture |
| `actualRateHz() : hz` | The rate in use (nominal for the IMU; exact for the piezo) |
| `getSample(i) : ctOffset, value(s)` | One sample: clocks since the first sample, then the value(s) |
| `stop()` | Release the sensor, pins and cog |

Differences:

| | Piezo (`isp_piezo_capture`) | IMU (`isp_imu_capture`) |
| - | --------------------------- | ----------------------- |
| `start` | `start(pin, sampleRateHz)` | `start(clickBasePin, sampleRateHz, accelFullScale, withGyro)` |
| `getSample(i)` returns | `ctOffset, microvolts` | `ctOffset, ax, ay, az` (raw counts) |
| Bytes per sample | `SAMPLE_BYTES` (8) | `SAMPLE_BYTES_ACCEL` (10) or `SAMPLE_BYTES_ACCEL_GYRO` (16); `sampleBytes()` |
| Extras | `recalibrate()`, `calibration()`, `samplePeriodClocks()` | `getGyro(i)`, `accelMicroG(raw)`, `gyroMicroDps(raw)`, `ACCEL_FS_*` |

### Piezo capture object

- **Sensor:** an amplified piezo vibration sensor, analog output.
- **Connection:** signal on **P0**, powered from the 3.3 V and GND of the P0–P7 header.
- **Object:** [`src/isp_piezo_capture.spin2`](src/isp_piezo_capture.spin2).
- **How it samples:** P0 runs as a P2 ADC smart pin taking a reading every 8,192 clocks (24.4 kHz at 200 MHz). The capture cog averages N readings into each stored sample, so the rate is exactly `clkfreq / (8192 × N)`; for example, 2,034.5 Hz for N = 12. The averaging also filters out higher frequencies.
- **Calibration:** the object reads its pin group's own ground (`P_ADC_GIO`) and 3.3 V (`P_ADC_VIO`) and scales pin readings between them, so supply and temperature drift cancel out. The piezo's output needs several ms to recover after the ADC switches back to the pin, so calibration only happens while idle (at `start()` or `recalibrate()`), followed by a 50 ms settle.
- **Sample value:** one reading per sample, in microvolts.

Usage:

```spin2
OBJ
  piezo : "isp_piezo_capture"

VAR
  byte  buf[2048 * piezo.SAMPLE_BYTES]

PUB main() | i, ct, uv
  piezo.start(0, 2000)                                          ' pin, rate (-> 2,034.5 Hz)
  piezo.startCapture(@buf, 2048)
  repeat while piezo.isCapturing()
  repeat i from 0 to piezo.sampleCount() - 1
    ct, uv := piezo.getSample(i)                                ' ct: clocks since first sample
  piezo.stop()
```

### IMU capture object

- **Sensor:** MikroElektronika LSM6DSL Click, MIKROE-2731 (ST LSM6DSL 3-axis accelerometer + 3-axis gyroscope).
- **Connection:** a Parallax P2-to-mikroBUS Click Adapter (#64008) on pin group **P32–P47**, talking **SPI** (CS P40, SCK P41, MISO P42, MOSI P43). The sensor's data-ready interrupt is on P36.
- **How it samples:** the capture cog waits for data-ready, reads the timestamp, then burst-reads all six axes. Accelerometer sample rates go up to 6.66 kHz.
- **Sample value:** accelerometer X, Y and Z per sample (gyro X, Y and Z optional).

Usage:

```spin2
OBJ
  imu : "isp_imu_capture"

VAR
  byte  buf[1660 * imu.SAMPLE_BYTES_ACCEL]                     ' ~1 s at 1.66 kHz, accel only

PUB main() | i, ct, ax, ay, az
  imu.start(32, 1660, imu.ACCEL_FS_8G, false)                  ' base pin, rate, range, gyro?
  imu.startCapture(@buf, 1660)
  repeat while imu.isCapturing()
  repeat i from 0 to imu.sampleCount() - 1
    ct, ax, ay, az := imu.getSample(i)                         ' ct: clocks since first sample
    ' imu.accelMicroG(ax) converts counts to micro-g
  imu.stop()
```

The rate you ask for is rounded up to the next rate the LSM6DSL supports (`actualRateHz()`). The chip's real rate differs from nominal by several percent, so compute the rate from the `ct` offsets for any frequency analysis. With the gyro on, use `imu.SAMPLE_BYTES_ACCEL_GYRO` per sample and read it with `getGyro(i)`.

It's built in layers. You'd normally use only the capture object.

| Layer | File | Role |
| ----- | ---- | ---- |
| IMU capture object | [`src/isp_imu_capture.spin2`](src/isp_imu_capture.spin2) | Capture cog, timestamps, buffer |
| LSM6DSL driver | [`src/isp_lsm6dsl.spin2`](src/isp_lsm6dsl.spin2) | Chip registers: setup, rate and range, data-ready, burst reads, unit conversion |
| SPI | [`src/isp_spi.spin2`](src/isp_spi.spin2) | SPI on P2 smart pins, selectable mode 0–3; the LSM6DSL uses mode 3 |

`isp_spi.spin2` is derived from Jon McPhalen's `jm_ez_spi.spin2` (mode 0 only), which stays in `src/` as the reference.

**Note:** the SPI and LSM6DSL layers drive pins from whichever cog calls them. Make every call to them from **one cog**; in the finished design that's the capture cog.

## Hardware

| P2 pins | Device |
| ------- | ------ |
| P0 | Amplified piezo vibration sensor |
| P32–P47 | LSM6DSL Click in a P2-to-mikroBUS Click Adapter |

Full pinout, jumper settings and power notes: [`HARDWARE.md`](HARDWARE.md).

## Building and running

The code is Spin2 for the P2 and builds with [PNut-TS](https://github.com/ironsheep/PNut-TS):

```
cd src
pnut-ts -d vibration_demo.spin2
pnut-term-ts --headless -r vibration_demo.bin --end-marker --timeout 120
```

### The demo

`vibration_demo.spin2` starts both capture objects, captures 8 s (thump or shake the platform during it), and prints a report:

- **Level and vibration size** per channel (piezo, accel X/Y/Z): resting level, RMS, peak swing and when it happened, with a warning if the accelerometer clipped.
- **Strongest frequencies:** the top 3 frequencies and amplitudes in the most active ~2 s of each channel.
- **Dominant frequency over time:** per ~0.5 s segment, the dominant frequency and its amplitude, or "quiet".

Frequencies are computed from each sensor's measured sample rate. The analysis uses [`src/isp_fft.spin2`](src/isp_fft.spin2), a Spin2 FFT object with a Hann window and between-bin refinement of frequency and amplitude.

### Test programs in `src/`

| Program | What it checks |
| ------- | -------------- |
| `lsm6dsl_test.spin2` | Bring-up: SPI mode probe, chip ID (`WHO_AM_I` = $6A), ten accel/gyro samples, temperature |
| `imu_capture_test.spin2` | 1 s captures at 1.66 kHz, accel-only then accel + gyro: measured rate, sample-interval jitter, missed samples, per-axis mean / RMS / peak |
| `piezo_test.spin2` | Piezo bring-up: GIO/VIO calibration, resting level and noise, and the pin's recovery after a source switch |
| `piezo_thump_test.spin2` | Waits up to 30 s for a desk thump, captures 1 s at 24.4 kHz: peak swing, decay per 50 ms, frequency estimate |
| `piezo_capture_test.spin2` | ~1 s capture at 2 kHz: measured rate, sample-interval jitter, missed samples, level and noise |
| `fft_test.spin2` | `isp_fft` on synthetic tones (50 Hz and 123.4 Hz plus a DC offset): found frequencies and amplitudes, and analysis time for 1,024 / 2,048 / 4,096 points |

## Documentation

- [`HARDWARE.md`](HARDWARE.md): wiring and pinout
- [`DOCs/DESIGN-GOALS.md`](DOCs/DESIGN-GOALS.md): design goals, driver structure, sample-rate choices, measured sensor behavior and the demo's analysis
- [`DOCs/hardware/`](DOCs/hardware/): LSM6DSL Click sheet and the ST LSM6DSL datasheet

## License

MIT; see [`LICENSE`](LICENSE). `jm_ez_spi.spin2`, and the parts of `isp_spi.spin2` derived from it, are © Jon McPhalen, also MIT.
