# Release 0.10.0 - Released on 28.10.2026

> **⚠️ Back up your configuration before updating.** This release renames options and removes
> one, and the add-on does not start while it holds an option it no longer knows. Before you
> update, open the add-on's *Configuration* tab, switch to *Edit in YAML* and save a copy of the
> whole text (or take a Home Assistant backup of the add-on). Then rename `apikey`, `apitoken`
> and `base_url` and remove `peak_shaving.mode` as described under *Breaking Changes*.

## Impact Sentences

What each change means for you, in one sentence each. Details follow below.

### Breaking Changes

- **One name per thing: `api_token` and `url`**: You must rename `apikey`, `apitoken` and `base_url` in your add-on configuration (before or right after updating), otherwise the add-on will not start; Docker, pip and hand-written configs keep working with a deprecation warning.
- **`peak_shaving.mode` removed**: If your configuration still contains `peak_shaving.mode`, batcontrol now refuses to start and you must switch to `time_active` / `price_active`; the related MQTT topic and HA select entity disappear on their own.

### New Features

- **Dashboard in the sidebar and read-only API**: You get a new Batcontrol panel in Home Assistant that explains the current decision, enabled by default, and you only need to act if you want to turn it off or expose port 8080 (set `auth_token` first).
- **Every MQTT value also via REST and MCP**: The API now delivers the full picture even with MQTT disabled, and two new sensors (`gross_energy_capacity`, `reserved_energy_capacity_effective`) appear automatically in Home Assistant, with no action needed.
- **New logo**: The add-on and home-screen icon change to the new batcontrol bat battery, which is purely cosmetic.
- **Forecasts and prices survive a restart**: After a restart or update, batcontrol can fall back on the last saved provider data if the first refresh fails, so you need no configuration change, but you should clear the cache if you change site geometry, forecast sensor or tariff account.
- **New "Decision" sensor**: With MQTT enabled, a new auto-discovered sensor shows why batcontrol chose its current mode, and you do not need to configure anything.

### Enhancements

- **Solar forecast uses the matching `pvinstallations` entry**: A correctly filled second `pvinstallations` entry is no longer ignored, and a missing `api_token` or `entity_id` now produces a clear error naming the key.
- **`cache_ttl_hours` works for the solar forecast**: If you set this option, it now actually takes effect, while the default of 12 hours stays the same for everyone else.
- **Peak shaving works with every logic type**: If your add-on still has a leftover `peak_shaving.enabled: true`, it becomes active now, so check the startup log after updating and set it to `false` if you do not want it.

### Bug Fixes

- **Inverter outages are bounded, Modbus connections recover**: Outage handling is more reliable and Fronius Modbus connections recover better after timeouts, without any configuration change.
- **Credentials no longer end up in the log**: Fronius digest headers and forecast.solar API keys are now masked in logs, so attaching `/data/batcontrol.log` to an issue is safer.
- **Decision sensor: current numbers and clearer texts**: The Decision sensor now shows values that are at most 15 minutes old and reads more clearly, with nothing for you to do.
- **15-minute resolution: four times too many HomeAssistant history requests**: If you use `homeassistant-api` with 15-minute resolution, startup is faster and Home Assistant is less loaded, while all other setups are unaffected.
- **Telegraf/InfluxDB: full decision text**: If you feed MQTT into InfluxDB with the upstream sample config, you should adopt the updated `telegraf.sample.conf` so the full decision sentence is stored instead of a single word.

### Technical Updates

- **Load profile generator in the documentation**: There is a new optional browser tool to build a `load_profile.csv` from a Fronius Solar.web export, and nothing changes in the add-on itself.
- **Python 3.14, minimum raised to 3.11**: The add-on moves to Python 3.14 under the hood, which requires no action from you.
- **Add-on image build restructured for the dashboard**: The build is faster and now fails if the dashboard is missing, and port 8080 is exposed but not mapped by default, so you do not need to do anything.
- **Upstream reference config slimmed down**: Only comments in the upstream example config were trimmed and no parameter changed, so the add-on is unaffected.
- **ASCII art startup banner**: The log now shows a banner with the version at each startup, which makes restarts easy to spot.

## What's Changed

### Breaking Changes

- **One name per thing: `api_token` and `url`** (#451): batcontrol used three spellings for a
  credential (`apikey`, `apitoken`, `api`) and two for an endpoint (`url`, `base_url`),
  depending on the section. A user who put a long-lived access token into `apikey` of a
  HomeAssistant solar installation got no error - just `expected str at 'access_token'. Got None`
  at startup, because the right value sat in the wrong key. From now on the credential is always
  `api_token` and the endpoint is always `url`.

  **What you have to do:** rename these options in your add-on configuration before updating,
  or right after the update - the add-on does not start while an option it no longer knows is
  stored:

  | old option | new option | section |
  |---|---|---|
  | `apikey` | `api_token` | `utility`, `pvinstallations` |
  | `apitoken` | `api_token` | `consumption_forecast` |
  | `base_url` | `url` | `pvinstallations`, `consumption_forecast` |

  batcontrol itself (Docker, pip, own `batcontrol_config.yaml`) still reads the old spellings and
  logs a deprecation warning instead, so a hand-written configuration keeps working.
  All `api_token` fields are now masked in the UI.

- **`peak_shaving.mode` removed** (#443, completes #418): the deprecated `peak_shaving.mode`
  parameter is no longer accepted - a configuration that still contains it is **rejected at
  startup** with a migration hint, instead of being mapped onto the rule switches with a
  deprecation warning as in 0.9.x. The option has been removed from the add-on's `options:` and
  `schema:`, so it can no longer be set from the HA UI. Migration, if you still had it set:
  `time` -> `time_active: true` + `price_active: false`, `price` -> `price_active: true` +
  `time_active: false`, `combined` -> both `true`. The `peak_shaving/mode` MQTT topic and its
  Home Assistant select entity are gone as well; the retained discovery config is cleared, so the
  stale entity disappears from HA on its own. Runtime control via MQTT keeps `enabled`,
  `allow_full_battery_after` and `price_limit`.

### New Features

- **A dashboard in the sidebar, and a read-only API behind it** (#457): the add-on now has a
  **Batcontrol** entry in the Home Assistant sidebar showing what batcontrol is doing and why -
  the current decision with its full trace, a hundred-cell state-of-charge grid in the style of
  the old defragmenter, and the price forecast with the cheapest and most expensive hours. It is
  served through ingress, so Home Assistant authenticates it and nothing has to be published.

  Behind it sits a read-only HTTP API (`/api/v1`) with its own OpenAPI documentation, and
  optionally an MCP server (`/mcp`) that lets an AI assistant answer questions like *"why is the
  battery not discharging right now?"*. Both are **read-only by design**: there is no way to set
  a mode, a charge rate or a parameter through them - overrides stay on MQTT, where Home
  Assistant can gate them behind its own automations.

  Configured under the new `api:` option block. It is **enabled by default** in this add-on,
  because the sidebar panel needs it; set `enabled: false` to turn the whole thing off. Leave
  `host` at `0.0.0.0` - ingress connects from the supervisor, and `127.0.0.1` makes the panel
  unreachable. To reach the API from outside Home Assistant, map port 8080 in the add-on's
  network settings and set `auth_token` first: a mapped port bypasses Home Assistant's
  authentication.

  See https://mastr.github.io/batcontrol-dev/features/dashboard/ and
  https://mastr.github.io/batcontrol-dev/integrations/rest-api/.

- **Every MQTT value is also available via REST and MCP** (#464): some values - the effective
  minimum grid charge SoC, the solar surplus, the peak shaving state and more - were only
  computed when MQTT was enabled. They are now computed once per cycle and read alike by MQTT,
  the REST API and the MCP server, so the API delivers the full picture with MQTT turned off.
  New endpoints `/price/status` (24 h price statistics), `/solar`, `/peak-shaving`, `/inverters`
  and `/inverter/{id}`, each with a matching MCP tool; `/battery` additionally reports the gross
  capacity, the effective reservation and the always-allow-discharge limit in Wh, and `/config`
  lists parameters changed at runtime (via MQTT, evcc or the grid charge lock) next to their
  configured value. **Two new MQTT topics** with Home Assistant auto-discovery:
  `gross_energy_capacity` and `reserved_energy_capacity_effective` - they show up as new
  sensors on their own. The inverter SoC is now read once per cycle instead of up to four times.
  See https://mastr.github.io/batcontrol-dev/integrations/rest-api/.

- **New logo, also on the home screen** (#466): the add-on icon and logo are replaced by the
  batcontrol bat battery - an upright battery with bat wings, neon green on black - the same
  logo the dashboard uses. Adding the dashboard to a phone's home screen now shows that icon
  instead of a plain "B" (apple-touch-icon and a web app manifest, both working under ingress).
  The stable add-on gets the new icon and logo as well.

- **Forecasts and prices survive a restart** (#423): all solar forecast providers
  and the API tariff providers (awattar, tibber, energyforecast, evcc) now save their latest raw
  response to a shared data directory. After a restart, or an add-on update, batcontrol loads
  that response and uses it as a fallback when the first refresh fails - for example because the
  internet is not up yet or the provider rate-limits you. The original fetch time is kept, so
  the normal 12-hour lifetime (`cache_ttl_hours` for the HomeAssistant solar provider) is never
  extended by a restart. In the add-on the directory is `/data/data` inside Home Assistant's
  persistent volume, passed to batcontrol via the new `--data-dir` option by the add-on
  entrypoint. **No configuration change is needed** - `options:` and `schema:` are unaffected.
  The cache contains raw provider responses, which may include installation or household data;
  files are written with owner-only permissions. If you change a site's geometry, a forecast
  sensor or your tariff account and the old response no longer fits, clear the cache: stop the
  add-on and delete the `cache` folder inside the add-on's persistent data volume (the add-on
  UI has no startup arguments, so batcontrol's `--clear-cache` flag is not reachable from it).
  Forecast.Solar already used a restart cache next to the configuration file in an earlier
  development build; it now uses the shared directory too. Local CSV load profiles and the
  dynamic network fee fetcher are not cached. See
  https://mastr.github.io/batcontrol-dev/getting-started/installation/#application-data-and-provider-caches.

- **New Home Assistant "Decision" sensor - why did batcontrol do this?**: batcontrol now records
  a structured *decision trace* for every control cycle - the ordered list of rules it evaluated,
  each with a stable reason code, the numbers behind it, and which one was decisive - and
  publishes it over MQTT as a new auto-discovered `Decision` sensor. The sensor state is the
  resulting mode with its plain-language reason; the full trace is attached as sensor attributes,
  so the HA UI or an automation can show *why* the battery is charging instead of only *that* it
  is. The last 200 traces are kept in memory, and a new event is published on every mode change
  or when the mode's value moves by 25% or more. Nothing to configure - the sensor appears
  automatically when `mqtt` is enabled. See
  https://mastr.github.io/batcontrol-dev/features/decision-sensor/ for how to read it.

### Enhancements

- **Solar forecast: the matching `pvinstallations` entry is used** (#451): batcontrol read the
  connection details of the HomeAssistant Solar Forecast ML provider only from the *first*
  `pvinstallations` entry, so a correctly filled second entry was silently ignored. It now picks
  the entry with a matching `type`, or the one carrying `entity_id`. A missing `api_token` or
  `entity_id` is reported as a configuration error naming the key, before any connection to
  HomeAssistant is attempted.
- **`cache_ttl_hours` works for the solar forecast** (#451): the option was documented for
  `homeassistant-solar-forecast-ml` since it was introduced, but nothing ever read it - the cache
  always ran with its default of 12 hours. It is now applied. The default is unchanged, so
  existing setups behave exactly as before; the add-on comment and docs said 24.0 and were wrong.

- **Peak shaving now works with every logic type** (#445): peak shaving and the solar-cap rule
  moved from the `next` logic into the default logic, so there is a single code path.
  `battery_control.type: next` keeps working unchanged - it is now a silent alias for `default`.
  **Behavior change, please read if you use peak shaving:** up to 0.10.0 the peak shaving options
  only took effect with `type: next` and were documented as having no effect otherwise. The add-on
  UI exposes the `peak_shaving` options independently of the logic type, so an add-on
  installation may well carry `peak_shaving.enabled: true` from a time when it did nothing - and
  that setting now becomes active. batcontrol logs a one-time `Peak shaving is ENABLED ...` line
  at startup so the activation is announced rather than silent; check the add-on log after
  updating and set `peak_shaving.enabled: false` if you do not want it. Details:
  https://mastr.github.io/batcontrol-dev/features/peak-shaving/

### Bug Fixes

- **Inverter outages are bounded, Modbus connections recover**: the outage window of the
  resilient inverter wrapper (`outage_tolerance_minutes`) is now measured with a monotonic clock,
  so clock synchronization or a manual change of the system time can no longer stretch or shorten
  it. It is only cleared when every failed operation succeeds again - previously an unrelated
  successful call (for example a local capacity lookup) could hide a persistent failed SoC read or
  a failing write command, and the outage was never escalated. An accepted mode command supersedes
  earlier failed mode commands, so a newly calculated decision recovers from a failed force-charge
  without replaying the old command. A failure exactly at the limit is still tolerated; beyond it
  batcontrol exits with an error. For `fronius-modbus`, an operation now makes at most one
  reconnect and one retry after a socket timeout (5 s), connection error or remote close; if that
  fails, the connection is closed and the next control cycle opens a new one. Modbus exception
  responses are not retried. Relevant if you use the resilient wrapper (enabled by default) -
  especially with a Fronius Modbus inverter. No configuration changes. See
  https://mastr.github.io/batcontrol-dev/configuration/inverter-configuration/.

- **Credentials no longer end up in the log** (#453, two code scanning findings of severity
  "high"): the Fronius inverter's digest `Authorization` header - derived from the inverter
  password and replayable for as long as the server nonce is valid - was written to the log in
  clear text by the auth debug logger, and again through the `RuntimeError` raised for a failed
  request. forecast.solar takes the API key as the first path segment of the request URL, so the
  rate-limit and error branches logged response bodies and header dumps that echoed the key back.
  Credential header values are now shortened to their first and last three characters (six
  characters or less are masked completely), and the API key is replaced by that masked form in
  everything logged from a forecast.solar response, which keeps the provider's own message
  readable. The nonce, cnonce and nc values needed to debug digest auth are still logged in full.
  Relevant for everyone who attaches `/data/batcontrol.log` to an issue; no configuration changes.

- **Decision sensor: current numbers and clearer texts** (#450): the `Decision` sensor kept the
  trace of the last *mode change*, so the numbers in its text (e.g. the usable energy behind
  `Discharge Allowed`) could be hours old, and a new reason for the same mode never appeared at
  all. The sensor is now also updated when the decisive reason changes, and otherwise refreshed
  every 15 minutes, so its text is at most 15 minutes old. A refresh does not count as a status
  change. The texts were reworked as well: `Avoid Discharge` now explains both why the battery is
  held (usable energy vs. reserve) and why it is not topped up from the grid; a charge rate capped
  by `max_grid_charge_rate` / `max_pv_charge_rate` / `min_pv_charge_rate` is appended in brackets
  instead of replacing the actual reason; and the explanations for the always-allow level, peak
  shaving (now naming the active rule), the solar feed-in limit, the safe-mode fallbacks, the
  external unblock and the grid charge lock were reworded.
  See https://mastr.github.io/batcontrol-dev/features/decision-sensor/

- **15-minute resolution: four times too many HomeAssistant history requests** (#447, fixes #446):
  with `time_resolution_minutes: 15` the consumption forecast horizon was handed to the providers
  as hours although it was counted in slots, so the horizon was 4x too long and the
  `homeassistant-api` consumption provider issued roughly four times the needed
  `recorder/statistics_during_period` requests on a cold cache. The forecast values themselves were
  correct - the only symptoms were avoidable load on HomeAssistant and a slower startup. The
  request is now resolution-neutral, the slots already elapsed in the current hour are included so
  the full horizon is really delivered, and the horizon is capped at 48 hours in all code paths.
  Only relevant for `consumption_forecast.type: homeassistant-api`; CSV profiles are unaffected.

- **Telegraf/InfluxDB: full decision text instead of a single word** (#449): for users who pipe the
  batcontrol MQTT topics into InfluxDB with the upstream `config/telegraf.sample.conf`, the new
  `/decision` topic was picked up by the catch-all `house/batcontrol/+` input, whose `value` parser
  keeps only the last whitespace-separated token - `"Charge from Grid 1250 W - usable energy is
  below the reserve"` arrived as `decision="reserve"`. The sample config now excludes the topic
  from the catch-all and adds a dedicated consumer that stores the whole sentence as measurement
  `batcontrol-decision`, plus a commented-out consumer for `/decision/attributes`. The MQTT payload
  itself was always correct, so the HA sensor and other consumers were never affected.

### Technical Updates

- **Load profile generator in the documentation**: upstream added a browser-based tool that
  turns a detailed Fronius Solar.web XLSX export into a `load_profile.csv` locally (optional
  wallbox removal and smoothing, gap handling, plausibility checks, suggested
  `annual_consumption`). Documentation only, no add-on change - the resulting file goes to
  `/addon_configs/HEXCODE_batcontrol/load_profile.csv` as before. See the consumption forecast
  page of the documentation.

- **Python 3.14, minimum raised to 3.11** (#444): the add-on base image moves from
  `base-python:3.13-alpine3.23` to `base-python:3.14-alpine3.23` (Python 3.14.7), matching
  upstream batcontrol, whose Docker image now ships Python 3.14. Upstream also raised its
  supported range to Python 3.11 - 3.14: Python 3.9 has been end of life since 2025-10-31 and
  3.10 reaches it on 2026-10-31. No configuration changes - `options:` and `schema:` are
  unaffected.

- **Add-on image build restructured for the dashboard**: the Dockerfile now fetches the upstream
  source once in a separate `alpine:3.24` stage and builds the dashboard in a dedicated
  `node:26-alpine` stage. Both run natively on the build host instead of once per architecture
  under emulation, which keeps the multi-arch build fast. The build fails if the wheel comes out
  without the dashboard, rather than shipping an image that shows a hint page in the sidebar.
  The image exposes port 8080 (unmapped by default) and ships the licenses of the fonts bundled
  with the dashboard. Dependabot now tracks the pinned base images of the development add-on.

- **Upstream reference config slimmed down** (#448): upstream trimmed the comments in
  `config/batcontrol_config_dummy.yaml` (359 -> 242 lines) and moved the explanations into the
  documentation. No parameter was added, removed or changed its value, so the add-on's `options:`
  and `schema:` are untouched - the add-on takes its configuration from the HA UI, not from that
  file. Listed only so the upstream change is traceable here.

- **ASCII art startup banner** (#442): batcontrol logs an eyecatcher once at startup -
  `BATCONTROL` in block letters, the catch phrase "riding your power prices" and the running
  package version. Handy when scrolling the add-on log: it marks exactly where a restart happened
  and which version came up.

# Release 0.9.1 - Released on 06.10.2026

## What's Changed

### Enhancements

- **evcc: Support `minpv` and `smart` PV-Surplus Charging Modes** (#430): Peak shaving is now
  correctly bypassed when an evcc loadpoint reports mode `minpv` or `smart`, not just `pv`.
  Previously only the `pv` mode was recognized as "EV absorbing PV surplus", so peak shaving
  could still cap solar production while the EV charger was already soaking up the surplus in
  `minpv`/`smart` mode.

- **Spread Grid Charging Over the Charging Window** (#431): New expert option
  `battery_control_expert.spread_grid_charge_over_charge_window` (default `true`). With
  `soften_price_difference_on_charging` enabled, the recharge energy is now spread over all
  directly following slots that have a price lower than or equal to the current price, instead
  of being crammed into the current slot only. **Behavior change:** with 15-minute price slots
  this previously requested a very high charge rate and filled the battery long before the
  cheap block ended; with the new default, charging is spread out and the battery reaches full
  charge only at the end of the cheap block. Set the option to `false` to restore the previous,
  single-slot charging behaviour.

- **SIGHUP: Provider Refresh Now Happens at the Next Evaluation Interval** (#434): Sending
  `SIGHUP` to a running batcontrol process still re-reads the consumption forecast load profile
  CSV and recalculates the scaling factor, without a restart. **Behavior change:** previously
  the signal woke the running sleep immediately and refreshed all providers (solar forecast,
  dynamic tariff, consumption forecast) right away; now the handler only logs the request and
  sets a flag, and the refresh happens at the start of the next control loop interval (every 3
  minutes) instead, so the running evaluation loop is never interrupted. This can delay the
  refresh by up to 3 minutes compared to before.

- **Several Inverters as One Aggregated Battery** (#436): upstream batcontrol can now drive
  several inverters/batteries as one large battery — capacities and energies are summed, the SoC
  is the capacity-weighted average, and a charge rate is split over the inverters proportional to
  their free capacity (capped by each `max_grid_charge_rate`, with a per-inverter
  `min_charge_rate` floor of 500 W so a small rate feeds few inverters properly instead of all of
  them uselessly). **Not available in this add-on:** it is configured as a *list* under
  `inverter:`, which the Home Assistant add-on options/schema UI cannot express, so the add-on
  keeps exactly one inverter. Use a YAML config (Docker/Compose/local Python) if you need it, see
  https://mastr.github.io/batcontrol/configuration/multiple-inverters/. For single-inverter
  setups — which is every add-on installation — behaviour is unchanged.

- **New option `mqtt.client_id`** (#438): the MQTT client identifier can now be set explicitly
  (optional, default `batcontrol`). Only needed when several batcontrol instances share one
  broker — two instances with the same client ID make the broker disconnect each other in a
  loop. The option is commented out in the configuration; leave it that way unless you run more
  than one instance. It must not be left empty, an empty value aborts the start with a clear
  error.

### Bug Fixes

- **MQTT Inverter: `cache_ttl` Was Silently Ignored** (#427): The inverter factory built the
  MQTT inverter's config dict key by key but never forwarded `cache_ttl`, so `MqttInverter`
  always fell back to its 120s default regardless of what was configured. Users whose MQTT
  bridge publishes SoC less often than every 120s were hitting a stale-cache timeout the option
  exists to prevent. An explicit `cache_ttl: null` still falls back to the 120s default, same as
  an absent key.

- **`battery_control_expert.charge_rate_multiplier` Now Actually Takes Effect** (#426, closes
  #424): The documented `battery_control_expert.charge_rate_multiplier` key was set on the logic
  instance but never read back — the charge rate calculation silently used the undocumented
  `battery_control.charge_rate_multiplier` instead. If you already set
  `battery_control_expert.charge_rate_multiplier` in your config expecting it to change the
  charge rate, it will now actually do so (default `1.1`, a 10% increase over the previous
  hard-coded behaviour). The old `battery_control.charge_rate_multiplier` location still works
  as a deprecated fallback with a one-time startup warning; `battery_control_expert` wins if
  both are set. Invalid or non-finite values now raise a clear error at startup instead of
  crashing later during charge-rate calculation.

- **MQTT Status Stuck on `offline` After a Restart** (#438, closes #437): stopping or restarting
  the add-on left the retained `<topic>/status` topic in a wrong state — dashboards and Node-RED
  flows showed batcontrol as permanently offline while it was running normally. Three causes,
  all fixed: `SIGTERM` (the signal Home Assistant/Docker sends when stopping an add-on) is now
  handled like `Ctrl+C`, so the shutdown path runs at all; the shutdown now publishes `offline`
  to `<topic>/status` (retained) and disconnects cleanly from the broker instead of letting the
  connection time out; and the MQTT client now uses the fixed client ID `batcontrol` instead of a
  random one per start, so the broker drops the old session immediately and a stale Last-Will can
  no longer overwrite the fresh `online` of the restarted instance about 100 s later.

### Internal Changes

- Value-parsing helpers (`parse_optional_ratio`, `parse_positive_number`, `parse_bool_flag`)
  moved out of `core.py` into a new `value_utils` module; no behaviour change. (#428)
- CI: `actions/checkout`, `actions/upload-artifact`, and `actions/download-artifact` GitHub
  Actions workflows bumped to their latest major versions.
- Pylint cleanups across the code base; no behaviour change. (#435)

**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.9.0...0.9.1

### 📌 Full Release Notes

👉 [GitHub Release v0.9.1](https://github.com/MaStr/batcontrol/releases/tag/0.9.1)

### 📚 Documentation

👉 [Project Documentation](https://mastr.github.io/batcontrol/)


# Release 0.9.0 - Released on 17.08.2026

## What's Changed

### New Features

- **Fronius GEN24: Optional Fixed Battery Capacity**: The `fronius_gen24` inverter type now
  accepts an optional `capacity` key (in Wh). When set, batcontrol uses this fixed value instead
  of querying `DesignedCapacity` from the inverter's Solar API at startup. Useful for setups where
  the API reports an incorrect capacity. Non-finite or non-numeric values are rejected at startup.

- **Energyforecast Total Price Provider** (#371): New `energyforecast_total_price` provider type
  uses the API-computed `total_ct_kwh` field directly, which already includes dynamic network fees,
  VAT, and markup as calculated by energyforecast.de. The `vat`, `fees`, and `markup` config fields
  are not required and `dynamic_network_fees` must not be enabled alongside this type (the API
  already includes network fees in its total).

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

- **Peak Shaving: Solar Feed-In Limit / Solarspitzengesetz Clip Absorption** (#405): New
  per-rule switches `time_active`, `price_active`, and `solar_cap_active` replace the single
  `peak_shaving.mode` setting (still accepted but **deprecated** — a startup warning is logged
  and it is mapped onto the switches: `time` -> `time_active` only, `price` -> `price_active`
  only, `combined` -> both). The new `solar_cap_active` rule absorbs PV power above a
  configurable `feed_in_limit_w` (the German 60% feed-in cap for uncontrolled plants, e.g.
  `0.6 * kWp * 1000`) into the battery instead of losing it to inverter curtailment.
  `feed_in_limit_headroom` (default `1.0`, must be `>= 1.0`) adds a safety margin to the
  forecast surplus for sites where clear-day production is underestimated. The solar floor
  takes priority over all other caps — including past `allow_full_battery_after` and at high
  SoC — because clipped energy is otherwise lost outright. Do not use
  `battery_control_expert.production_offset_percent` as a substitute for the headroom setting.

- **External Grid-Charge Lock Signal / Section 14a EnWG** (#416, closes #216): New optional
  `mqtt.grid_charge_lock_topic` config key (an absolute topic, not nested below the base MQTT
  `topic` — mirrors evcc's "External Limit" convention) lets an external HEMS or grid-operator
  system, such as a German section 14a EnWG controllable-consumption-device signal, tell
  batcontrol to stop charging the battery from the grid. A payload of `1`/`true`
  (case-insensitive) immediately blocks grid charging: the current
  `max_charging_from_grid_limit` is remembered, forced to 0, and any active forced grid charge
  is cancelled. `0`/`false` (the default) restores the remembered limit, clamped to
  `always_allow_discharge_limit` so a limit lowered while locked cannot get stuck at 0.
  batcontrol publishes the lock state as a new retained `house/batcontrol/grid_charge_locked`
  MQTT topic, with a matching Home Assistant MQTT auto-discovery `binary_sensor` entity
  ("Grid Charge Locked"). The key is commented out by default in the reference config and is
  fully backward compatible.

### Bug Fixes

- **MQTT and evcc TLS/SSL Support Fixed** (#397): TLS connections for both the MQTT API and the
  evcc plugin were completely non-functional since their introduction. The code checked
  `config['tls'] is True` but then tried to index the same boolean as a dictionary, causing a
  `TypeError` at connection time. The fix reads `cafile`, `certfile`, and `keyfile` from the
  flat config keys where they actually live. TLS config is now validated at startup: if
  `tls: true` is set without a `cafile`, or if `certfile`/`keyfile` are only partially
  provided, batcontrol raises a clear `ValueError` instead of failing silently at connect time.
  The defunct `tls_version` and `cert_reqs` config keys have been removed from the example config.

- **evcc: Missing 15-Minute Slots on Mixed-Granularity Price Entries** (#406): evcc sometimes
  returns hourly-width rate entries further out in the forecast horizon while near-term prices
  are 15-minute slots. batcontrol only filled the entry's start slot, leaving the remaining
  15-minute slots of an hourly entry missing from the price data — this crashed batcontrol with
  an uncaught `KeyError` when it hit the gap. Every 15-minute slot actually covered by a rate
  entry is now filled.

- **MQTT Forecast Topics 4x Too Low at 15-Minute Resolution** (#407): `FCST/production`,
  `FCST/consumption`, and `FCST/net_consumption` were documented as Watts but published the raw
  Wh-per-interval value, which only matches average power at 60-minute resolution — at
  15-minute resolution the published number was 4x too low. `value` now stays Wh-per-interval
  (unchanged regardless of resolution) and a new `power_w` field carries the average power in
  Watts.

### Removed

- **SolarPrognose provider removed** (#391): SolarPrognose (solarprognose.de) has been
  discontinued. The provider module, factory entry, and all documentation references have been
  removed. Users who relied on SolarPrognose should migrate to an alternative forecast provider
  such as FCSolar, Forecast.Solar, or the newly added Solcast.

### Configuration Changes

- **Energyforecast migrated to API v2** (#371, closes #365): The energyforecast provider now uses
  the `/api/v2/forecast` endpoint. Data is always quarter-hourly (15-min native resolution),
  multi-day forecasts are returned natively, and a minimum 4-hour refresh interval is enforced.
  A new optional `market_zone` config key (default: `DE`) selects the price zone; `DE` and `LU`
  are both normalized to `DE-LU` for the API (supported zones: DE, LU, AT, FR, NL, BE, PL, DK1,
  DK2). The `energyforecast_96` provider type has been **removed** from the add-on — migrate to
  `energyforecast`, which now provides equivalent multi-day coverage via API v2.

- **Resilient Inverter Wrapper enabled by default** (#399): The `enable_resilient_wrapper`
  option now defaults to `true`. The resilient wrapper retries failed inverter commands and
  suppresses transient communication errors so batcontrol keeps running through short inverter
  outages. Existing configs that explicitly set `enable_resilient_wrapper: false` are unaffected.


**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.8.1...0.9.0

### 📌 Full Release Notes

👉 [GitHub Release v0.9.0](https://github.com/MaStr/batcontrol/releases/tag/0.9.0)

### 📚 Documentation

👉 [Project Documentation](https://mastr.github.io/batcontrol/)


# Release 0.8.1 - Released on 21.07.2026

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


**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.8.0...0.8.1

### 📌 Full Release Notes

👉 [GitHub Release v0.8.1](https://github.com/MaStr/batcontrol/releases/tag/0.8.1)

### 📚 Documentation

👉 [Project Documentation](https://mastr.github.io/batcontrol/)


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

# 🚀 Release 0.6.0 - published on 02.02.2026

Default time resolution is 60 minutes.

## What's Changed

- Introducing the 15-minute calculation slots with dynamic up/downsample
- Configuration for 15 or 60 minutes
- Expected Consumption Wh per hour is now distributed evenly across the 4 slots.
- PV is scaled linear between the adjacent hours.

**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.5.7...0.6.0 _(repository moved)_

---

### 📌 Full Release Notes

👉 [GitHub Release v0.6.0](https://github.com/MaStr/batcontrol/releases/tag/0.6.0) _(repository moved)_

### 📚 Wiki

👉 [Project Wiki](https://github.com/MaStr/batcontrol/wiki) _(repository moved)_

# 🚀 Release 0.5.7 - published on 05.01.2026

## What's Changed

- HomeAssistant Addon - Add configuration option for sensor unit to avoid get_sensor call.
- HomeAssistant Addon - get_entity call, increase limit for websocket call to 4 MB

**Full Changelog**: https://github.com/MaStr/batcontrol/compare/0.5.6...0.5.7 _(repository moved)_

---

### 📌 Full Release Notes

👉 [GitHub Release v0.5.7](https://github.com/MaStr/batcontrol/releases/tag/0.5.7) _(repository moved)_

### 📚 Wiki

👉 [Project Wiki](https://github.com/MaStr/batcontrol/wiki) _(repository moved)_

# 🚀 Release 0.5.6 - published on 04.12.2025

## What's Changed

- Fix HomeAssistant forecast cache validation to check specific future hours by @Copilot in https://github.com/muexxl/batcontrol/pull/232
- Fix cache offset bug in HomeAssistant consumption forecast for non-contiguous missing hours by @Copilot in https://github.com/muexxl/batcontrol/pull/235
- Make Fronius inverter and controller IDs configurable by @Copilot in https://github.com/muexxl/batcontrol/pull/239

**Full Changelog**: https://github.com/muexxl/batcontrol/compare/0.5.5...0.5.6

---

### 📌 Full Release Notes

👉 [GitHub Release v0.5.6](https://github.com/muexxl/batcontrol/releases/tag/0.5.6)

### 📚 Wiki

👉 [Project Wiki](https://github.com/muexxl/batcontrol/wiki)

# 🚀 Release 0.5.5 - published on 16.11.2025

## What's Changed

- Add energyforecast.de provider with 48-hour forecast window by @Copilot in https://github.com/muexxl/batcontrol/pull/225
- Add custom gh-copilot instruction by @MaStr in https://github.com/muexxl/batcontrol/pull/226
- Fix Tibber class KeyError by fetching from cache like other tariff providers by @Copilot in https://github.com/muexxl/batcontrol/pull/228 Thanks for reporting & testing @magicofit
- energyforecast: Fix response handling by @MaStr in https://github.com/muexxl/batcontrol/pull/229

**Full Changelog**: https://github.com/muexxl/batcontrol/compare/0.5.4...0.5.5

---

### 📌 Full Release Notes

👉 [GitHub Release v0.5.5](https://github.com/muexxl/batcontrol/releases/tag/0.5.5)

### 📚 Wiki

👉 [Project Wiki](https://github.com/muexxl/batcontrol/wiki)

# 🚀 Release 0.5.4 - published on 12.11.2025

### What's Changed

- Overwork provider management by @MaStr in https://github.com/muexxl/batcontrol/pull/213
- (beta) Add generic MQTT inverter driver & restructure mqtt handling & homeassistant discovery by @Copilot in https://github.com/muexxl/batcontrol/pull/196
- Homeassistant sensor as consumption forecast by @MaStr & @greteOne in https://github.com/muexxl/batcontrol/pull/214
- HA consumption input conversion by @MaStr in https://github.com/muexxl/batcontrol/pull/221
- Ha addon reduce logging by @MaStr in https://github.com/muexxl/batcontrol/pull/222
- Asynchroneaus forecast fetcher via schedule() by @MaStr in https://github.com/muexxl/batcontrol/pull/223

**Full Changelog**: https://github.com/muexxl/batcontrol/compare/0.5.3...0.5.4

---

### 📌 Full Release Notes

👉 [GitHub Release v0.5.4](https://github.com/muexxl/batcontrol/releases/tag/0.5.4)

### 📚 Wiki

👉 [Project Wiki](https://github.com/muexxl/batcontrol/wiki)

# 🚀 Version 0.5.3 — published on 17.10.2025

### What's New

- This release calculates hourly values based on 15-minute interval provided by different providers. (evcc).
- Tibber is switched to hourly by API request.
- Evcc solar forecast provider
- feat: add minimum recharge amount and related logic for charging from grid

---

### 📌 Full Release Notes

👉 [GitHub Release v0.5.3](https://github.com/muexxl/batcontrol/releases/tag/0.5.3)

### 📚 Wiki

👉 [Project Wiki](https://github.com/muexxl/batcontrol/wiki)

# 🚀 Version 0.5.2 — published on 17.09.2025

### ✨ What’s New

- ✅ Support for **Fronius firmware 1.38.XX** (🔑 updated login routine)
- 📖 **Documentation refreshed** with more details and clarity
- 🔧 New configuration parameter: `solar_forecast_provider` — choose between different providers for your solar production forecast

---

### 📌 Full Release Notes

👉 [GitHub Release v0.5.2](https://github.com/muexxl/batcontrol/releases/tag/0.5.2)

### 📚 Wiki

👉 [Project Wiki](https://github.com/muexxl/batcontrol/wiki)

---

### 🙌 Kudos

Special thanks for contributions & support in this release to:
💡 @MaStr & ⚡ @Julfried

# Version 0.5.1 published on 22.05.2025

Support for Fronius firmware 1.36.XX
(Thanks for changing all the API routes, Fronius XD)

Improved add-on building process. Kudos to @julfried

Full release notes can be found at the batcontrol project
https://github.com/muexxl/batcontrol/releases/
Kudos for this version to @MaStr @Julfried

# Version 0.4.1-0 published on 15.02.2025

Support for Fronius firmware 1.35.XX

Introduction of expert parameters for fine tuning the core batcontrol algorithm

Further release notes can be found at
https://github.com/muexxl/batcontrol/releases

_Kudos for quick adoption to changed login behaviour of Fronius Inverters go to @MaStr_ 🎉

# Version 0.3.3-1 published on 05.02.2025

### 🚀 MQTT Autodiscovery available!!

simply add below options to the config section mqtt to register batcontrol as a device in you HA

mqtt:
...

- auto_discover_enable: true
- auto_discover_topic: homeassistant

_Kudos for this beautiful contribution go to @ohAnd_ 🎉

### ⚡ Enhancement of batcontrol algorithm: relative price difference

The parameter `min_price_difference_rel`is an enhancement to the internal calculation when it is worth to charge your battery.
The legacy parameter `min_price_difference` is used to determine if it is worth to charge the battery for later hours with STATIC offset on the current price.
`min_price_difference_rel` is used to determine if it is worth to charge the battery for later hours with RELATIVE offset on the current price.
The algorithm always takes the HIGHER value of both calculations.

📌 Example 1:

    min_price_difference = 0.05 EUR
    min_price_difference_rel = 0.1
    current price = 0.20 EUR

    min_price_difference would suggest to charge if the price later on that day is above 0.25 EUR (=0.20 + 0.05)
    min_price_difference_rel would suggest to charge if the price later on that day is above 0.22 EUR (=0.20 + 0.20 * 0.1)

    The algorithm takes the decision based on min_price_difference and will charge the battery if the price is above 0.25 EUR in the future

📌 Example 2:

    min_price_difference = 0.05 EUR
    min_price_difference_rel = 0.1
    current price = 0.80 EUR

    min_price_difference would suggest to charge if the price later on that day is above 0.85 EUR (=0.80 + 0.05)
    min_price_difference_rel would suggest to charge if the price later on that day is above 0.88 EUR (=0.80 + 0.80 * 0.1)

    The algorithm takes the decision based on min_price_difference and will charge the battery if the price is above 0.88 EUR in the future

*Kudos for this algorithm enhancement go to @MaStr*🎉

# Version 0.3.3-hotfix1 published on 24.01.2025

!!! WARNING !!!

There has been a breaking change in the configuration file.
YOU NEED TO UPDATE YOUR CONFIGURATION in the evcc section

change below lines in the config to make it work again:

loadpoint_topic: evcc/loadpoints/1/charging

to

loadpoint_topic:

- evcc/loadpoints/1/charging

# Version 0.3.3 published on 23.01.2025

HA Addon specific changes:

- move logfile to /data/batcontrol.log to make it persistant
- added entryscript for container startup
- mount persistant addon config folder in /app/addon_config
- use load_profile.csv from addon config folder if file exists
- use batcontrol_config.yaml from addon config folder if file exists instead of configuration provided via the addon configuration

implement version 0.3.3 of batcontrol:

- EVCC : Enhance loadpoint configuration to support multiple topics by @MaStr in #80
- MQTT & EVCC : Improve reconnect handling on broker restart by @MaStr in #82
- Update to Documentation by @johannesghd in #86
- Set Fronius Solar.API to active on initialisation by @johannesghd in #85
- Set allow grid charging at the end of initialisation by @johannesghd in #83
- Update README.MD by @johannesghd in #88
- Feature: handle FC solar rate limit errors by @hashtagKnorke in #93
- Major refactoring of modules by @MaStr in #71
- Add MQTT Connection retry by @hashtagKnorke in #94
- Refactor pylint workflow to analyze only changed Python files by @hashtagKnorke in #102
- introduce proper OS-level timezone handling in Docker setup by @hashtagKnorke in #103
- gracefully handle multiple changed files by @hashtagKnorke in #104
- Add query parameter to allow modification of horizon used to forecast PV production in fcsolar.py by @johannesghd in #110
- 113 invalid login 01.00.00 am after installing the latest ha version by @MaStr in #114

special thanks goes to the contributers:
@MaStr
@hashtagKnorke
@johannesghd

corresponding issues can be found in the main github repo:
https://github.com/muexxl/batcontrol

# Version 0.3.2 published on 08.12.2024

- add evcc options in configuration

# Version 0.3.1 published on 07.12.2024

- Update to batcontrol 0.3.1
- fix bugs
- add MQTT options in configuration
- advanced logging

# Version 0.2.11 published on 14.11.2024

- Update to 0.2.11.
- fix bugs
- integrate PR

# Version 0.2.10 published on 29.10.2024

- Roll Back to intermediate version 0.2.8+ of underlying source code due to https://github.com/muexxl/batcontrol/issues/32

# Version 0.2.9 published on 29.10.2024

- exclude energy that is reserved for backup power when calculating the avaiable energy

# Version 0.2.8 published on 29.10.2024

- fixed issue with timezones in docker #3 https://github.com/muexxl/batcontrol_addon/issues/3

# Version 0.2.7 published on 21.10.2024

- fixed bug in force_charge() introduced with variable renaming in 0.2.0
- fixed bug in Forecast SolarAPI when providing empty apikeys

# Version 0.2.5 published on 18.10.2024

- fixed bug / Bad request to inverter when setting time of use
- code refactoring: renamed fronius module to inverter

# Version 0.2.3 published on 17.10.2024

- added optional parameter `apikey` in pv installations. This parameter allows you to use your own API Key if you are a forecast solar customer
- fixed bug if parameter api is missing

# Version 0.2.0 published on 17.10.2024

- included MQTT capabilities (deactivated per default)
- updated to batcontrol 0.2.1
- new parameter :`max_pv_charge_rate`. Limits the rate at which your PV System charges your battery
- renamed parameter `max_charge_rate` to `max grid charge rate`. Limits the maximum charging rate of your battery when charging from the grid.
- remove unused parameter `max_grid_power`

# Version 0.1.5 published on 09.02.2024

- introduced limit for log file
- added hints in configuration via translations/en.yaml

# Version 0.1.3 published on 16.11.2023

- added support for Awattar (DE+AT)
- enhanced documentation

# Version 0.1.2 published on 8.11.2023

- enhanced documentation

# Version 0.1.1 published on 7.11.2023

- Updated logo.
- Updated icon.

# Version 0.1.0 published on 7.11.2023

starting point
