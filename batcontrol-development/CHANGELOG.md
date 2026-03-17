# 🚀 Release 0.7.2 - on development

## What's Changed

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


