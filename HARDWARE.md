# Hardware

Sensor wiring for P2-VibrationSense on a Propeller 2 board.

## Pin summary

| P2 pins | Pin group | Device |
| ------- | --------- | ------ |
| P0 | P0–P7 header | Amplified piezo vibration sensor (signal) |
| P32–P47 | P32–P39 + P40–P47 headers | LSM6DSL Click in a P2-to-mikroBUS Click Adapter |

## Amplified piezo vibration sensor

| Sensor lead | Connection |
| ----------- | ---------- |
| Signal | **P0** |
| 3.3 V | 3.3 V pin of the P0–P7 accessory header |
| GND | GND pin of the P0–P7 accessory header |

The sensor is powered from the same header, and so the same pin group, as its signal pin.

The sensor has an on-board amplifier and puts out an analog voltage. There is no usable datasheet, so its output bias, gain and bandwidth are unknown and must be measured on the bench.

### Reading it: P0 in ADC mode

- P0 is configured as an ADC smart pin (`P_ADC`, internally clocked; smart-pin mode %11000). See p2kb `p2kbArchSmartPin11000AdcInternalClock`.
- P0's ADC measures against the supply and ground of its own **silicon power group, P0–P3**. That group is part of the P0–P7 header that also powers the sensor. So the sensor output and the ADC reference share one 3.3 V rail, and supply drift affects both the same way.
- Since the sensor runs on 3.3 V, its output should stay within the pin's 0–3.3 V range.
- **Calibration method: use P0's own ground and 3.3 V readings.** The ADC can be switched to read its power group's ground (`P_ADC_GIO`) or 3.3 V supply (`P_ADC_VIO`) instead of the pin. Readings of those two known levels give the 0 V and 3.3 V endpoints, and the pin reading (`P_ADC_1X`) is scaled between them:

  ```
  µV = (pin − GIO) / (VIO − GIO) × 3_300_000
  ```

  1. Set P0 to read GIO, throw away the first 3 samples, then average the next N. This is the ground count.
  2. Set P0 to read VIO, throw away 3 samples, then average N. This is the 3.3 V count.
  3. Set P0 to read the pin and convert each sample with the formula above.
  4. Repeat steps 1–2 now and then (for example between capture runs) to track drift from temperature and supply.

  After each source switch, the first 3 samples are unsettled (2 from the SINC2 filter, 1 from the analog front end). Use a 64-bit intermediate for the divide, for example Spin2 `MULDIV64`. The method comes from p2kb app note P2AN001 (`p2kbAppNoteP2an001SinglePinInstrumentationAdc`), which measured a fixed offset of up to about 9 mV that remains even after this calibration. That doesn't matter for vibration amplitude, which is AC. It only matters when reading the sensor's absolute DC bias.
- Noise on the P0–P7 3.3 V rail shows up in every reading, because it feeds both the sensor and the ADC reference.

## LSM6DSL Click (6DOF IMU)

**Module:** MikroElektronika LSM6DSL Click (MIKROE-2731), with an ST LSM6DSL 3-axis accelerometer (±2/4/8/16 g) and 3-axis gyroscope (±125/250/500/1000/2000 dps; the Click sheet says 245, the ST datasheet says 250). It runs on 3.3 V and talks SPI or I2C, with an interrupt line. References: [`DOCs/hardware/LSM6DSL_Click.pdf`](DOCs/hardware/LSM6DSL_Click.pdf) (Click board) and [`DOCs/hardware/DS_lsm6dsl.pdf`](DOCs/hardware/DS_lsm6dsl.pdf) (ST datasheet, DocID028475 Rev 7, with the register map). The ST datasheet describes SPI in mode 3 only, at up to 10 MHz.

**Adapter:** Parallax P2-to-mikroBUS Click Adapter (#64008), plugged into the 16-pin group **P32–P47** (two adjacent accessory headers). It is passive: no level shifting and no components. Actual pin = 32 + adapter offset.

### Pinout

| P2 pin | Offset | mikroBUS pin | LSM6DSL Click use | Direction (P2 view) |
| ------ | ------ | ------------ | ----------------- | ------------------- |
| P32 | +0  | 11 SDA  | I2C data (I2C mode only) | bidir |
| P33 | +1  | 12 SCL  | I2C clock (I2C mode only) | out |
| P34 | +2  | 13 TX   | not connected on Click | — |
| P35 | +3  | 14 RX   | not connected on Click | — |
| P36 | +4  | 15 INT  | IMU interrupt, INT1 or INT2 per JP4 (default INT1) | in |
| P37 | +5  | 16 PWM  | not connected on Click | — |
| P38 | +6  | 1 AN    | not connected on Click | — |
| P39 | +7  | 2 RST   | ID SEL (ClickID); **not** an IMU reset | out |
| P40 | +8  | 3 CS    | SPI chip select / ID COMM (active low) | out |
| P41 | +9  | 4 SCK   | SPI clock | out |
| P42 | +10 | 5 MISO  | IMU SDO → P2 | in |
| P43 | +11 | 6 MOSI  | P2 → IMU SDI | out |
| P44–P47 | +12..+15 | — | no-connect on the adapter; don't assume they're usable while it's seated | — |

### Jumper defaults (Click board)

| Jumper | Function | Default |
| ------ | -------- | ------- |
| JP1, JP2, JP3, JP5 | COMM SEL (SPI / I2C) | Left = **SPI** |
| JP4 | INT SEL (INT1 / INT2) | Left = **INT1** |
| JP6, JP7 | MODE SEL (1 / 2) | Left = **Mode 1** |
| JP8 | I2C address select (0 / 1) | Left = **0** |

With the defaults, use SPI on P40–P43 and the interrupt on P36. If the COMM SEL jumpers are moved to I2C, use P32 (SDA) and P33 (SCL) instead.

### Power

- The Click's 3.3 V comes from header A's 3.3 V rail (the P32–P39 header). Ground is shared by both headers.
- The Click does not use 5 V (mikroBUS pin 10 is NC on this board).
- The SPI signals (P40–P43) are on header B while everything else is on header A, so the socket spans two pin-power groups.

### Driver idiom

Spin2 Click drivers use `CLICK_OFST_*` constants plus a base pin:

```spin2
CON
  CLICK_BASE      = 32
  CLICK_OFST_SDA  = 0
  CLICK_OFST_SCL  = 1
  CLICK_OFST_INT  = 4
  CLICK_OFST_RST  = 7
  CLICK_OFST_CS   = 8
  CLICK_OFST_SCK  = 9
  CLICK_OFST_MISO = 10
  CLICK_OFST_MOSI = 11
```

## Sources

- Adapter offsets, power and header split: p2kb `p2kbHwAddonClickAdapterAddonClickAdapter` and `p2kbArchClickModuleIntegration`. The p2kb notes say these were checked against the 64008 Rev A schematic.
- Click-side pin meanings and jumpers: `DOCs/hardware/LSM6DSL_Click.pdf`.
- Piezo sensor wiring and use of ADC mode: as described by the project owner. No usable datasheet exists.
- P2 ADC behavior and power groups: p2kb `p2kbAppNoteP2an001SinglePinInstrumentationAdc` and `p2kbArchPinPowerDomains`.
