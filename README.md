# Context-Aware Smart Desk Fan Controller

Gesture-controlled, occupancy-aware, thermally-regulated fan controller built on a single ESP32, with no cloud dependency and no ML — just deterministic finite-state-machine logic.

## Overview

Most desk fans run on manual switches or single-threshold automation — both waste energy or cause "actuator hunting" (the fan flipping speeds repeatedly near a temperature boundary). This project combines **gesture control, occupancy sensing, thermal hysteresis, and RTC-based scheduling** into one embedded architecture with explicit conflict resolution between them.

Two core contributions:
- A **triple-layer gesture filter** (hardware validity flag → majority voting → lockout + reversal-detection timing) to kill false-positive swipes.
- A **hysteresis-based thermal state machine** with a ±1°C deadband to stop the fan from oscillating near threshold temperatures.

## Features

- Non-contact gesture control (swipe left/right/down)
- Occupancy detection — auto-off after 5 min of no motion
- Schedule-aware operation via RTC (active only 11:00–23:59)
- Thermal hysteresis with LOW / MEDIUM / HIGH fan states
- 4-level hierarchical priority resolution between all of the above
- Real-time status on a 16×2 I2C LCD

## Hardware

| Component | Role |
|---|---|
| ESP32 | Main MCU — sensing, decision-making, actuation |
| DHT11 | Temperature sensing |
| HC-SR501 (PIR) | Occupancy detection |
| APDS-9960 | Gesture sensing (directional swipes) |
| DS3231 (RTC) | Schedule-aware operation |
| L298N | Motor driver for PWM-driven DC fan |
| 16×2 I2C LCD | User feedback / status display |

Interfaces: I2C + single-wire, to keep GPIO usage and wiring low. All control decisions run locally on the ESP32 — no wireless, no cloud.

## System Architecture

Three layers:
1. **Sensing layer** — APDS-9960, DHT11, PIR, RTC
2. **Processing & control layer** — ESP32 running the priority-gated pipeline
3. **Actuation & display layer** — L298N → DC fan (PWM), I2C LCD

## Gesture Filtering Pipeline

1. Hardware-native validity flag rejects incomplete gesture events (no blocking delay)
2. 3-sample majority vote (accept if ≥2 of last 3 agree)
3. 500 ms lockout window + 800 ms reversal-detection window

## Thermal Hysteresis

| Parameter | Value |
|---|---|
| LOW_TEMP_THRESHOLD | 27°C (enter LOW) |
| LOW_EXIT_THRESHOLD | 28°C (exit LOW) |
| HIGH_TEMP_THRESHOLD | 34°C (enter HIGH) |
| HIGH_EXIT_THRESHOLD | 33°C (exit HIGH) |
| DHT_INTERVAL_MS | 2000 ms |

Gesture input is only enabled in MEDIUM mode (28–33°C) — it's locked out in the LOW ("Cold Lock") and HIGH ("Hot Lock") states.

## Priority Hierarchy

```
P1 RTC Schedule  >  P2 Occupancy Detection  >  P3 Thermal State Machine  >  P4 Gesture Control
```

| Priority | Layer | Condition | Behavior | Overrides |
|---|---|---|---|---|
| P1 | RTC Schedule | Outside 11:00–23:59 | Fan disabled, LCD backlight off | All lower layers |
| P2 | Occupancy | No motion ≥5 min | Forced fan OFF | Thermal + gesture |
| P3 | Thermal | LOW/HIGH mode | Fan forced OFF/HIGH; gesture ignored | Gesture |
| P4 | Gesture | MEDIUM mode | User-driven speed adjustment | — |

## Results

- **Gesture recognition accuracy:** 97.1% (68/70 trials across LEFT/RIGHT/DOWN swipes)
- **False-positive rate:** 0% over a 30-minute static-environment test
- **Thermal hysteresis:** verified stable, no boundary oscillation (T1–T6)
- **Functional validation:** 12/12 hardware-in-the-loop tests passed (occupancy, scheduling, gesture gating, priority resolution)

## Limitations & Future Scope

- Gesture-to-actuation response latency wasn't instrumented (planned)
- PIR can't detect stationary occupants — mmWave radar (e.g. HLK-LD2450) proposed as a fix
- Fixed thermal thresholds — no personalization yet
- Future: onboard data logging (EEPROM/flash) + TinyML-based adaptive thermal control

## Acknowledgement

Developed during an Embedded Systems & IoT internship at SURE ProED, Puttaparthi.

---

**Authors:** Hamsini T S, Mehak Majeed
