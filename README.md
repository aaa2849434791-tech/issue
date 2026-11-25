# VL53L0X: no valid reading below ~25 cm on STM32F103 (hardware I2C2), works fine on ESP32

## Summary

A VL53L0X module reports **0 mm with RangeStatus = 0** for every target between ~3 cm and ~25 cm, and only starts returning data at ~30 cm (where it reads 257 mm for a true 300 mm). Readings alternate between `0` and `257`.

The **same module works correctly** on an ESP32-S3 with the Pololu Arduino library (valid readings from 10 cm to 30+ cm).

I would like help determining whether this is a configuration issue in my ST API port, or a hardware characteristic of this particular unit.

## Hardware / software

- MCU: STM32F103C8T6, Keil MDK, STM32 HAL, FreeRTOS
- Sensor: VL53L0X breakout (GY-VL53L0XV2 style), 4 pins (VIN, GND, SDA, SCL), no XSHUT control used, XSHUT tied to 3.3 V
- Supply: 3.3 V (also tried 5 V on VIN, same result)
- Bus: **hardware I2C2** (PB10 = SCL, PB11 = SDA), 100 kHz, 7-bit address 0x29 → HAL DevAddress 0x52
- Driver: ST official VL53L0X API (`vl53l0x_api.c`, `vl53l0x_api_core.c`, `vl53l0x_api_calibration.c`, `vl53l0x_api_ranging.c`, `vl53l0x_api_strings.c`)
- Platform layer: `HAL_I2C_Master_Transmit` / `HAL_I2C_Master_Receive`, timeout = 100 + count ms, per-byte buffer

The platform layer is unmodified apart from the handle, and `VL53L0X_PollingDelay()` is `HAL_Delay(2)`.

## Initialisation sequence used

```c
My_VL.I2cHandle  = &hi2c2;
My_VL.I2cDevAddr = 0x52;

status = VL53L0X_DataInit(&My_VL);
status = VL53L0X_StaticInit(&My_VL);
status = VL53L0X_PerformRefSpadManagement(&My_VL, &spad_count, &aperture_mode);
status = VL53L0X_SetReferenceSpads(&My_VL, spad_count, aperture_mode);
status = VL53L0X_SetDeviceMode(&My_VL, VL53L0X_DEVICEMODE_SINGLE_RANGING);
status = VL53L0X_SetMeasurementTimingBudgetMicroSeconds(&My_VL, 33000);
```

All calls return `VL53L0X_ERROR_NONE`. `spad_count` and `aperture_mode` are then used as returned by `PerformRefSpadManagement`.

`PerformRefCalibration`, `PerformOffsetCalibration` and `PerformXTalkCalibration` are **not** called (no cover glass, no window).

## Measurement

Called in a FreeRTOS task, roughly every 120 ms:

```c
VL53L0X_RangingMeasurementData_t m;
status = VL53L0X_PerformSingleRangingMeasurement(&My_VL, &m);
Serial_Printf("st=%d d=%d s=%d\r\n", (int)status, m.RangeMilliMeter, m.RangeStatus);
```

## Observed output

Target is a white A4 sheet (also tried hand and a dark book — identical results):

```
st=0 d=7967 s=2     // nothing in front of the sensor (expected)
st=0 d=0    s=0     // target at 3–25 cm  <-- problem
st=0 d=257  s=0     // target at 30 cm   <-- 43 mm low
```

When the target sits at 30 cm, consecutive samples alternate:

```
st=0 d=0   s=0
st=0 d=0   s=0
st=0 d=0   s=0
st=0 d=257 s=0
st=0 d=0   s=0
...
```

## Cross-check on ESP32-S3 (same module, same wires)

Pololu VL53L0X Arduino library 1.0.0, `Single.ino` example, default settings (no SPAD management, no offset/XTalk calibration):

- 10 cm → ~100 mm
- 20 cm → ~200 mm
- 30 cm → ~300 mm
- no dead zone, no alternating values

So the optical path, supply and module are fine.

## What I have already ruled out

1. **Module hardware** — works on ESP32.
2. **Supply** — 3.3 V and 5 V both tested, decoupled, same behaviour.
3. **Soft vs hardware I2C confusion** — this sensor is on hardware I2C2; a PCA9685 on the same board uses a separate bit-banged bus on PB3/PB4, so there is no bus sharing.
4. **Interrupt interference** — stopping the 10 ms timer ISR completely (`HAL_TIM_Base_Stop_IT`) makes no difference.
5. **Timing budget** — tried 33000 and 200000 µs, no difference.
6. **SPAD calibration** — removing `PerformRefSpadManagement`/`SetReferenceSpads` makes it worse (constant `d=7967 s=2`, i.e. signal too weak), so the calibration is doing something, but the result still has a ~25 cm dead zone.


Any pointer to the correct initialisation sequence for a bare VL53L0X used at 10–40 cm would be very welcome.
