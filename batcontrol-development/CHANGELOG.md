# Release 0.8.1 - in Development

"Make it real" - Cypecore

# 🚀 Release 0.8.0 - Released on 11.05.2026

## What's Changed

### 🌟 Major New Features

- **Peak Shaving** (#315): New logic type `next` with peak shaving support in three modes: `time` (counter-linear ramp), `price` (cheap-slot reservation), `combined` (both, stricter wins)
  - Configurable target hour (`allow_full_battery_after`) and price threshold (`price_limit`)
  - MQTT runtime control for enable/disable and target hour
  - Home Assistant auto-discovery (switch, number, sensor)
  - Requires logic type `next` in `battery_control.type`

- **Single-Zone Static Price Mode** (#324): The `tariff_zones` provider now supports 1, 2, or 3 zones. A single `tariff_zone_1` acts as a static flat-price mode for the whole day.

- **Minimum Grid Charge SoC** (#356): New "charge to min. SoC%" feature — pushes batcontrol to fill the battery up to a defined SoC% during a recharge event. Thanks to @filiplajszczak

- **Fronius Modbus Support** (#337, #350, #351, #352): New `fronius-modbus` inverter backend with backup safety and charge limit support. Thanks to @filiplajszczak

### ✨ Enhancements

- **New Logic Type `next`**: Selectable via `battery_control.type: next`. Front-runs upcoming changes and includes peak shaving support. The `default` logic remains unchanged.
- **Scheduler Isolation** (#326): Scheduler state is isolated from the schedule library's global singleton, improving testability and reliability.
- **Peak Shaving Config Validation** (#336): Added explicit `PeakShavingConfig` validation and improved fallback behavior for `combined` mode.
- **MQTT Improvements** (#338, #340, #342, #343, #344, #345, #347, #353, #354):
  - Exposes `limit-battery-charge` mode and charge-rate limit in Home Assistant auto-discovery
  - Publishes API override state and control source state to MQTT
  - Refreshes and publishes control state updates immediately (including price difference and charge-limit changes)
  - Refactors shared MQTT control topic helpers for consistency
  - Polishes published state values
- **Recharge Decision Logging** (#355): Logic now logs a recharge decision summary for better observability.

### 🔧 Technical Updates

- **Build**: switched from pip to uv for Docker image builds and test setup
- **Tests**: improved coverage for inverter factory, core dispatch, and pytest configuration (#319, #320, #321, #322, #330, #333, #334, #358, #359, #360)
- **Removed Legacy HomeAssistant Sensor Formats** (#327): Legacy `hours_list` and `hour_N` sensor formats have been removed. Only the evcc-compatible format remains.
- **Dropped armv6, armv7 and armhf** (#348): Removed support for deprecated Docker image architectures.

### ⚠️ Breaking Changes

- HA Solar ML Forecast only supports the evcc-typed forecast sensor format. Other variants were removed.
- Dropped armv6, armv7 and armhf Support on Docker images.

### 🙌 New Contributors

- @filiplajszczak made their first contribution

**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.7.2...0.8.0

### 📌 Full Release Notes

👉 [GitHub Release v0.8.0](https://github.com/MaStr/batcontrol/releases/tag/0.8.0)

### 📚 Wiki

👉 [Project Wiki](https://github.com/MaStr/batcontrol/wiki)


# 🚀 Release 0.7.2 - Released on 07.04.2026

## What's Changed

- Support for the homeassistant evcc-sensor fomr ML solar forecastprediction

**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.7.1...0.7.2

### 📌 Full Release Notes

👉 [GitHub Release v0.7.2](https://github.com/MaStr/batcontrol/releases/tag/0.7.2) 

### 📚 Wiki

👉 [Project Wiki](https://github.com/MaStr/batcontrol/wiki) 


# 🚀 Release 0.7.1 - Released on 17.03.2026

## What's Changed

- Fix type conversion error in interval_minutes


**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.7.0...0.7.1

### 📌 Full Release Notes

👉 [GitHub Release v0.7.1](https://github.com/MaStr/batcontrol/releases/tag/0.7.1) 

### 📚 Wiki

👉 [Project Wiki](https://github.com/MaStr/batcontrol/wiki) 

