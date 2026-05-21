# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an embedded firmware project for the **Raspberry Pi Pico 2** (RP2350) that turns an **MPU6050 IMU** into a wireless mouse. The Pico reads gyroscope/accelerometer data, computes orientation via AHRS sensor fusion, and streams position packets over USB serial to a host Python app (`python.py`) that drives the OS cursor via `pyautogui`.

## Build

This project uses the [Raspberry Pi Pico VS Code Extension](https://marketplace.visualstudio.com/items?itemName=raspberry-pi.raspberry-pi-pico) with CMake. The extension manages the Pico SDK (v2.2.0), toolchain (arm-none-eabi-gcc 14.2), and picotool.

To build manually:
```bash
cmake -B build -DPICO_BOARD=pico2
cmake --build build
```

The output UF2 binary will be at `build/pico_emb.uf2`. Flash it by holding BOOTSEL, connecting the Pico, then copying the UF2 to the mass storage device.

## Host-Side Python App

Install dependencies (ideally in the `venv/` already present):
```bash
pip install -r requirements.txt
```

Run the mouse controller GUI:
```bash
python python.py
```

Select the correct COM port in the dropdown and click "Conectar e Iniciar Leitura". Set `DEBUG = True` in `python.py` to log each decoded packet.

## Architecture: FreeRTOS Task Pipeline

All logic lives in `main/main.c` as four FreeRTOS tasks communicating via queues:

```
mpu6050_task → xQueueMPU → fusion_task → xQueuePos   → uart_task  → USB serial → python.py
                                        → xQueueColor → pwm_task   → RGB LED
                                        → xSemaphoreBtn (click events)
```

| Task | Stack | Priority | Role |
|------|-------|----------|------|
| `mpu6050_task` | 8192 | 1 | Reads raw accel/gyro over I2C at 100 Hz |
| `fusion_task` | 8192 | 1 | AHRS filter → Euler angles → cursor velocity + LED color |
| `pwm_task` | 1024 | 1 | Drives RGB LED (GPIO 7/8/9) via PWM |
| `uart_task` | 2048 | 1 | Encodes and sends binary packets over USB UART |

**Startup**: `fusion_task` discards the first `WARMUP_SAMPLES` (200) frames to calibrate gyro bias before outputting data.

## Serial Protocol

3-byte packets over USB UART at 115200 baud:
```
[0xFF] [axis] [value + 128]
```
- `axis`: 0 = X, 1 = Y, 2 = CLICK
- `value`: int8 in [-94, +94]; encoded as `value + 128` (0xFF is reserved as sync byte, so 0xFE is used instead when encoding lands on it)

## Hardware Pinout

| Signal | GPIO |
|--------|------|
| I2C SDA (MPU6050) | 16 |
| I2C SCL (MPU6050) | 17 |
| LED Red | 7 |
| LED Green | 8 |
| LED Blue | 9 |
| Instrumentation (MPU) | 10 |
| Instrumentation (Fusion) | 11 |
| Instrumentation (PWM) | 12 |
| Instrumentation (UART) | 13 |

GPIOs 10–13 are toggled high/low around each task's active work for oscilloscope profiling.

## Fusion / Click Detection

- **AHRS**: `FusionAhrsUpdateNoMagnetometer` from the `Fusion/` library (Madgwick-based, no magnetometer)
- **Tilt-to-velocity**: Roll → X axis, pitch → Y axis. Dead zone = ±8°, max speed at ±25°.
- **Click detection**: A spike on accelerometer Y axis > 1.5 g triggers a click. A 30-sample cooldown suppresses AHRS corruption; during cooldown the accelerometer vector is zeroed before being passed to the filter.
- **LED color**: Smoothed with exponential filter (α = 0.06). Roll right → red, roll left → blue, pitch forward → green.

## CI

Three GitHub Actions workflows run on every push:
- **build** (`build.yml`): Compiles the firmware via `insper-embarcados/pico-build@v0`
- **cppcheck** (`cppcheck.yml`): Static analysis on `main/main.c`
- **embedded-check** (`embedded-check.yml`): Insper-specific RTOS compliance checks on `main/main.c`
