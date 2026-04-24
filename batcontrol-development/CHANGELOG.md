# Release 0.8.0 - in Development

## What's Changed

### 🌟 Major New Features

- **Peak Shaving** (#328): Limits PV-to-battery charge rate so the battery fills gradually instead of reaching 100% by midday. Preserves free capacity to absorb more solar energy later in the day and reduces unnecessary grid feed-in.
  - Three modes: `time` (counter-linear ramp), `price` (cheap-slot reservation), `combined` (both, stricter wins)
  - Configurable target hour (`allow_full_battery_after`) and price threshold (`price_limit`)
  - MQTT runtime control for enable/disable and target hour
  - Home Assistant auto-discovery (switch, number, sensor)
  - Requires logic type `next` in `battery_control.type`
  - See [Peak Shaving wiki page](https://github.com/MaStr/batcontrol/wiki/Peak-Shaving) for details

- **Single-Zone Static Price Mode** (#324): The `tariff_zones` provider now supports 1, 2, or 3 zones. When only `tariff_zone_1` is configured, it acts as a static flat-price mode for the whole day.

### ✨ Enhancements

- **New Logic Type `next`**: Selectable via `battery_control.type: next`. Front-runs upcoming changes and includes peak shaving support. The `default` logic remains unchanged.
- **Scheduler Isolation** (#326): Scheduler state is isolated from the schedule library's global singleton, improving testability and reliability.
- **Removed Legacy HomeAssistant Sensor Formats** (#327): Legacy `hours_list` and `hour_N` sensor formats have been removed. Only the evcc-compatible format remains.
- **Peak Shaving Config Validation** (#336): Added explicit `PeakShavingConfig` validation and improved fallback behavior for `combined` mode.
- **MQTT Control/Discovery Improvements** (#338, #340, #342, #343, #344, #345, #347):
  - Exposes `limit-battery-charge` mode and charge-rate limit in Home Assistant auto-discovery
  - Publishes API override state to MQTT
  - Refreshes and publishes control state updates immediately (including price difference and charge-limit changes)
  - Refactors shared MQTT control topic helpers for consistency

### 🔧 Technical Updates

- Build: switched from pip to uv for Docker image builds and test setup (dev image)
- Tests: improved coverage for inverter factory, core dispatch, and pytest configuration
- Tests: added/expanded characterization tests for inverter battery math and mode contracts (#333, #334)
- Removed depricated architectures armhf & armv7

# 🚀 Release 0.7.2 - Released on 07.04.2026

## What's Changed

- Support for the homeassistant evcc-sensor fomr ML solar forecastprediction

# 🚀 Release 0.7.1 - Released on 17.03.2026

## What's Changed

- Fix type conversion error in interval_minutes


**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.7.0...0.7.1

### 📌 Full Release Notes

👉 [GitHub Release v0.7.1](https://github.com/MaStr/batcontrol/releases/tag/0.7.1) 

### 📚 Wiki

👉 [Project Wiki](https://github.com/MaStr/batcontrol/wiki) 


# 🚀 Release 0.7.0 - Released on 17.03.2026

## What's Changed

### 🌟 Major New Features

- **Limited PV Charging Mode** : Introduces a new operation mode (8), which limits the load of the battery with PV. This is in preparation to implement peak shaving.

- **Multi-Zone Tariff Provider** (#289): Introducing `tariff_zones` provider supporting up to 3 different price zones for flexible tariff configurations
  - Replaces the previous two-tariff implementation
  - Supports flexible hour format: ranges (`7-22`), comma-separated (`0,1,2`), or mixed (`0-5,23`)
  - All 24 hours must be assigned to exactly one zone
  - Optional third zone for complex tariff structures (e.g., day/night/peak pricing)

- **HomeAssistant Solar Forecast ML Integration** (#257): New solar forecast provider using ML-based predictions from HomeAssistant
  - Integrates with HomeAssistant Solar Forecast ML (HACS)
  - Provides ML-based hourly solar production forecasts
  - Configurable via `pvinstallations` with `type: homeassistant-solar-forecast-ml`
  - Supports automatic unit detection or manual configuration
  - Includes smart caching to reduce HomeAssistant load

- **Resilient Inverter Wrapper** (#254): Enhanced reliability for inverter communication
  - Gracefully handles temporary inverter outages (firmware updates, network issues)
  - Configurable outage tolerance (default: 24 minutes)
  - Uses cached values during outages to maintain operation
  - Automatic recovery when connection is restored
  - Enable via `enable_resilient_wrapper: true` in inverter config

### ✨ Enhancements

- **Production Offset Multiplier** (#253): Fine-tune solar forecast results
  - New parameter `production_offset_percent` (default: 1.0 = 100%)
  - Available via config, MQTT + HA Sensor
  - Useful for winter mode when solar panels are covered with snow or other seasonal adjustments
  - Changes via API will be reset after a reboot

- **Improved PV Forecast Failsafe** (#284): Reduced minimum forecast requirement from 18 to 12 hours
  - Better handling of partial forecast data
  - More resilient to provider outages

- **Enhanced Logging** (#278): Reworked default logic log messages for better clarity and debugging

### 🔧 Technical Updates

- **Python 3.13 Support** (#268): Updated base image from Python 3.11 to Python 3.13
- **Repository Migration** (#263): Project moved from `muexxl/batcontrol` to `MaStr/batcontrol` organization
- **Dependency Updates**: Various GitHub Actions and dependency updates via Dependabot (#287, #288, #276)

**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.6.1...0.7.0 

### 📌 Full Release Notes

👉 [GitHub Release v0.7.0](https://github.com/MaStr/batcontrol/releases/tag/0.7.0) 

### 📚 Wiki

👉 [Project Wiki](https://github.com/MaStr/batcontrol/wiki) 

# 🚀 Release 0.6.1 - published on 03.02.2026

- Fix Tibber API call


