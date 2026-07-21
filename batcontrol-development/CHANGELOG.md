# Release 0.9.0 - in Development

## What's Changed

### New Features

- **Solcast Rooftop API as Solar Forecast Provider**: Solcast (https://solcast.com) is now
  supported as a solar forecast provider with native 30-minute resolution. The free hobbyist
  account grants 10 API calls per day; batcontrol scales the refresh interval automatically
  with the number of configured installations so the quota is never exceeded. Rate-limit
  responses (HTTP 429) are handled via the `Retry-After` header. Configuration requires only
  `resource_id` and `apikey` per site — the panel geometry (tilt, azimuth, capacity) is
  managed in the Solcast web toolkit. Optional `percentile` selection (10/50/90, default 50)
  allows choosing between optimistic, median, and conservative forecasts.

- **Forecast Grid Charge Target Strategy**: A new `battery_control.grid_charge_target_strategy`
  option controls how `min_grid_charge_soc` is applied during expensive price slots. Set to
  `forecast` to let batcontrol compute the effective target SoC dynamically based on the
  expected overnight battery reserve (using the solar forecast), rather than always charging
  to the fixed configured value. The effective target is published to MQTT as
  `effective_min_grid_charge_soc`. The default value `fixed` retains the existing behaviour.
  Thanks to @filiplajszczak

### Bug Fixes

- **MQTT and evcc TLS/SSL Support Fixed** (#397): TLS connections for both the MQTT API and the
  evcc plugin were completely non-functional since their introduction. The code checked
  `config['tls'] is True` but then tried to index the same boolean as a dictionary, causing a
  `TypeError` at connection time. The fix reads `cafile`, `certfile`, and `keyfile` from the
  flat config keys where they actually live. TLS config is now validated at startup: if
  `tls: true` is set without a `cafile`, or if `certfile`/`keyfile` are only partially
  provided, batcontrol raises a clear `ValueError` instead of failing silently at connect time.
  The defunct `tls_version` and `cert_reqs` config keys have been removed from the example config.

### Removed

- **SolarPrognose provider removed** (#391): SolarPrognose (solarprognose.de) has been
  discontinued. The provider module, factory entry, and all documentation references have been
  removed. Users who relied on SolarPrognose should migrate to an alternative forecast provider
  such as FCSolar, Forecast.Solar, or the newly added Solcast.

### Configuration Changes

- **Resilient Inverter Wrapper enabled by default** (#399): The `enable_resilient_wrapper`
  option now defaults to `true`. The resilient wrapper retries failed inverter commands and
  suppresses transient communication errors so batcontrol keeps running through short inverter
  outages. Existing configs that explicitly set `enable_resilient_wrapper: false` are unaffected.

### Internal Changes

- UTC-aware clock handling in the Solcast provider ensures correct hour-alignment independent
  of the host timezone.
- Network-level exceptions (timeouts, DNS failures) in the Solcast provider are wrapped as
  `ProviderError` so the refresh scheduler falls back to cached data instead of crashing.
- Documentation deployment workflow now triggers on tag pushes in addition to branch pushes.
- CLAUDE.md expanded with CLI reference (`--one-shot` flag), release skill documentation, and
  HA add-on sync notes.
- Fronius GEN24 inverter integration documented: required access level (customer/technician),
  Solar.API activation on each startup, and battery/time-of-use config backup/restore behaviour
  on clean shutdown. (#400, closes #74)

# Release 0.8.1 - Released on 2026-07-21

## What's Changed

### New Features

- **Dynamic Network Fees - Section 14a EnWG** (#364): Adds time-variable network fee surcharges
  to energy prices for providers that calculate fees locally (Awattar, Energyforecast). Fees are
  sourced from `dyn-net.batcontrol.software`, cached for 12 hours, and applied net before VAT.
  Activate via the new top-level `dynamic_network_fees` config block (see config example).
  Providers that already include fees (Tibber, evcc, tariff_zones) are unaffected.

- **PV Start Battery & Forecast Min Battery** (#372, updated): Two MQTT sensors replace the
  former `night_surplus_wh` metric (legacy discovery entry is cleaned up automatically):
  - `pv_start_battery_wh`: battery level in Wh above MIN_SOC at the next net-charging
    crossover (when PV first exceeds household consumption). 0 means the battery hits MIN_SOC
    before solar restarts.
  - `forecast_min_battery_wh`: minimum battery level in Wh above MIN_SOC over the entire
    forecast horizon (slot-by-slot simulation). 0 means a shortage is expected at some point.
  Both sensors support Home Assistant MQTT auto-discovery. Useful for driving flexible loads
  (heat pump, EV, evcc) based on expected overnight battery availability.

- **Solar Surplus and Solar Active MQTT Sensors** (#370): Two new MQTT sensors published after
  each control cycle:
  - `solar_surplus_wh`: expected usable solar surplus in Wh for the current/next production window
  - `solar_active`: binary sensor (`true`/`false`) indicating whether solar production is currently active
  Both are available via Home Assistant MQTT auto-discovery.

### Enhancements

- **Solar Forecast Padded to Midnight** (#373): Solar forecast providers (e.g. FCSolar,
  SolarPrognose) often stop returning data at the last production interval (e.g. 21:00), leaving
  the rest of the day uncovered. batcontrol now pads the forecast with zero-production entries
  up to midnight, extending the effective planning horizon by up to 3 hours. DST transitions
  are handled correctly.

- **Configurable Daily Market Price Hard Refresh** (#367): EPEX Spot next-day prices are
  typically published around 12:00-13:00 UTC. The normal hourly polling could miss these for
  an extended time. A dedicated daily hard refresh now bypasses the cache and fetches fresh
  price data at a configurable UTC time. Configure via `battery_control_expert.market_price_refresh_time`
  (default: `"12:30"`). Applies to all dynamic tariff providers.

- **Solar API Rate Limit Protection** (#369): The solar forecast refresh interval is now set
  independently from the general API call interval. Providers such as FCSolar and SolarPrognose
  allow only 20 API calls per day. The new dedicated interval of ~80 minutes (18 calls/day)
  prevents quota exhaustion while leaving 2 requests in reserve for restarts or transient errors.

- **Inverter Resilient Wrapper Simplified** (#381): The fail-fast/retry logic in the inverter
  resilient wrapper has been restructured into a cleaner two-tier model: transient
  communication errors vs. full outage errors. Reduces noise in logs and improves recovery
  behaviour after short connection interruptions.

- **Docker Image Address Updated** (#375): The Docker image has moved from `muexx/batcontrol`
  to `mastr950/batcontrol`. Update your `docker run` or `docker-compose.yml` accordingly.

- **Documentation migrated to MkDocs** (#377): The GitHub Wiki has been replaced by a
  versioned MkDocs documentation site. All configuration references and integration guides
  are now maintained at https://mastr.github.io/batcontrol/.

### Documentation

- **Documentation inaccuracies corrected** (#378): Several wrong defaults and missing topics in
  the docs have been fixed based on a systematic code review. Key corrections:
  - `enable_resilient_wrapper` default is `false`, not `true`
  - `max_charging_from_grid_limit` default is `80%`, not `89%`
  - `history_weights` uses semicolons (`"1;1;1"`), not commas — a comma-separated value would
    crash parsing
  - MQTT/evcc TLS support is marked as non-functional (contradictory code paths) rather than
    merely untested
  - A number of MQTT topics that existed in the code but were absent from the docs are now
    documented: `min_grid_charge_soc`, `limit_battery_charge_rate`, `production_offset`,
    `api_override_active`, `control_source`, `solar_surplus_wh`, `solar_active`,
    `night_surplus_wh`, `peak_shaving/*`, and their `/set` counterparts

- **README rewritten and installation guide added** (#386): The README is now a concise project
  overview. Detailed installation steps, a pre-flight checklist, and next-steps guidance live in
  the new `docs/getting-started/installation.md` page on the docs site.

- **LLM-friendly docs** (#387): The docs site now generates `/llms.txt` and `/llms-full.txt`
  at build time, making the full documentation accessible to AI assistants and LLM tools.

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

### 📚 Docs

👉 [Documentation](https://mastr.github.io/batcontrol/)


# 🚀 Release 0.7.2 - Released on 07.04.2026

## What's Changed

- Support for the homeassistant evcc-sensor fomr ML solar forecastprediction

**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.7.1...0.7.2

### 📌 Full Release Notes

👉 [GitHub Release v0.7.2](https://github.com/MaStr/batcontrol/releases/tag/0.7.2) 

### 📚 Docs

👉 [Documentation](https://mastr.github.io/batcontrol/) 


# 🚀 Release 0.7.1 - Released on 17.03.2026

## What's Changed

- Fix type conversion error in interval_minutes


**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.7.0...0.7.1

### 📌 Full Release Notes

👉 [GitHub Release v0.7.1](https://github.com/MaStr/batcontrol/releases/tag/0.7.1) 

### 📚 Docs

👉 [Documentation](https://mastr.github.io/batcontrol/) 

