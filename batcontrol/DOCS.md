## Rationale

In winter time there is usually not enough energy available from your residential pvinstallation to meet your demand.
So you need to get additional energy from the grid.
This software helps you to get the additional energy at the cheapest price by controlling your inverter in a smart way.

## Customizing the load profile. (HA ADDON)

Step 1
download the default profile from https://raw.githubusercontent.com/MaStr/batcontrol/0.10.1/config/load_profile.csv
or generate one from your own Fronius Solar.web export with the browser-based
[load profile generator](https://mastr.github.io/batcontrol/configuration/consumption-forecast/).

Step 2
edit the load profile according to your requirements.

Step 3
place the edited load profile in your home assistant instance using below path. You can use the vscode plugin to do so.
/addon_configs/HEXCODE_batcontrol/load_profile.csv

Step 4
Restart the addon

## Documentation

The documentation of the underlying project provides further and up-to-date information about the project and its configuration parameters:
https://mastr.github.io/batcontrol/

## Configuration Parameters

**Credential and URL keys (changed in 0.10.0):** every section uses the same two key names -
`api_token` for a credential and `url` for an endpoint. The former `apikey`, `apitoken` and
`base_url` options no longer exist in this add-on. If you used them, move your values to
`api_token` / `url` once; the add-on will not start while an unknown option is stored.

`timezone:` Europe/Berlin your time zone. not optional.
`time_resolution_minutes:` 60 # time resolution of the forecasts: `15` (quarter-hourly) or `60` (hourly, default).
`loglevel:` Verbosity of the add-on logs (`debug`, `info`, `warning`, `error`).
`log_everything:` false # if false, debug messages from `fronius.auth` and `urllib3.connectionpool` are suppressed.
`max_logfile_size:` 100 # in kB

### `battery_control:`

`type`: default # logic type. `default` is the recommended standard logic, `next` is a historical alias kept for existing configurations and behaves identically.

`min_price_difference:` 0.05 in EUR/kwH # minimum price difference to justify charging your battery. This reflects the wear and tear of your battery as well as the electrical losses in the charging/ discharging cycle. 0.05 to 0.10 is a good range.
`min_price_difference_rel:` 0.10 # (optional) minimum price difference as a fraction of the current price. Helps to avoid charging at high prices with less efficiency.

`always_allow_discharge_limit`: 0.90 # 0.00 to 1.00 above this SOC limit using energy from the battery is allowed regardless of other parameters. This value can be set lower to 1.00 if you want to ensure that always some capacity remains available in case the sun starts shining more than forecasted.
`max_charging_from_grid_limit`: 0.89 # 0.00 to 1.00 charging from the grid is only allowed until this SOC limit. Above this limit only Solar Power may be used to increase the charge level.
`min_grid_charge_soc`: 0.55 # (optional) 0.00 to 1.00 SOC to preserve/charge before expensive slots.
`grid_charge_target_strategy`: fixed # (optional) `fixed` uses `min_grid_charge_soc` unchanged, `forecast` keeps it as a reserve after expensive slots.
`min_recharge_amount`: 100 # (optional) in Wh, start and minimum amount of energy to recharge the battery.

### `peak_shaving:`

Peak shaving limits the rate at which your **PV system** charges the battery, so the battery is
not already full when the sun is at its strongest. The reserved capacity absorbs the midday peak
instead of having it curtailed or fed in at a low price. Three rules work independently and can
each be switched off:

- **time rule** (`time_active`): spread charging so the battery is full only by
  `allow_full_battery_after` instead of as early as possible.
- **price rule** (`price_active`): keep capacity free for upcoming cheap-price PV windows below
  `price_limit`.
- **solar rule** (`solar_cap_active`): absorb PV power above `feed_in_limit_w` to prevent
  clipping against a grid feed-in limit.

> **Note when updating to 0.10.0:** peak shaving used to require `battery_control.type: next` and
> had no effect otherwise, but it now works with every logic type. Because this add-on always
> showed the options below regardless of the logic type, your configuration may still carry
> `enabled: true` from a time when it did nothing - and it becomes active with this update.
> batcontrol logs a one-time `Peak shaving is ENABLED ...` line at startup, so check the add-on
> log after updating and set `enabled: false` if you do not want it.

`enabled`: false # master switch for all three rules. Leave at false if you are unsure.

`time_active`: true # target-time rule: spread charging until `allow_full_battery_after`
`price_active`: true # price rule: reserve capacity for cheap-price PV windows
`solar_cap_active`: false # solar feed-in limit rule: absorb PV power above `feed_in_limit_w`

`allow_full_battery_after`: 14 # Hour 0-23 by which the battery should be full (time rule). Set it to the hour your PV peak is over.
`price_limit`: 0.05 # EUR/kWh cheap-slot threshold for the price rule. -1 disables the price rule component.

`feed_in_limit_w`: 0 # Watt grid feed-in power limit for the solar rule. 0 = neutral/off. For the German Solarspitzengesetz use 60% of installed power, i.e. 0.6 * kWp * 1000 (6000 for a 10 kWp plant).
`feed_in_limit_headroom`: 1.0 # Safety factor >= 1.0 applied to the forecast surplus (solar rule). Raise it to about 1.1 if you still observe curtailment.

The `mode` parameter (`time`/`price`/`combined`) was **removed** in 0.10.0 and is now rejected at
startup. Use the three switches instead: `time` -> `time_active` only, `price` -> `price_active`
only, `combined` -> both.

Full description with worked example days:
https://mastr.github.io/batcontrol/features/peak-shaving/

### `battery_control_expert`:

`charge_rate_multiplier`: 1.1 # Increase (>1) calculated charge rate to compensate charge inefficencies.
`soften_price_difference_on_charging`: false # enable earlier charging based on a more relaxed calculation # future_price <= current_price-min_price_difference/soften_price_difference_on_charging_factor
`soften_price_difference_on_charging_factor`: 5
`spread_grid_charge_over_charge_window`: true # only with `soften_price_difference_on_charging` enabled: spread grid charging over all following slots with a price <= current price (the charging window) instead of the current slot only. Reduces the peak charge rate needed with short (e.g. 15-minute) price slots.
`round_price_digits`: 4 # round price to n digits after the comma
`production_offset_percent`: 1.0 # adjust the production forecast (1.0 = 100%, 0.8 = 80%). Useful in winter when the panels are covered with snow.
`preserve_min_grid_charge_soc`: false # if true, `min_grid_charge_soc` is also kept as reserved energy during cheap/pre-expensive slots
`market_price_refresh_time`: "12:30" # (optional) UTC time "HH:MM" of a daily hard price refresh, to pick up newly published next-day prices (default: 12:30)

### `inverter:`

`type`: fronius_gen24 # `fronius_gen24`, `fronius-modbus`, `mqtt` or `dummy` (no inverter, for testing). See https://mastr.github.io/batcontrol/configuration/inverter-configuration/
`address`: 192.168.0.XX # the local IP of your inverter. needs to be reachable from the machine that runs batcontrol. Required for fronius_gen24 and fronius-modbus
`user`: customer #customer or technician lowercase only!! (fronius_gen24)
`password`: YOUR-PASSWORD # password for your inverter (fronius_gen24)
`max_pv_charge_rate`: 0 # Watt. Limits the rate at which your PV System charges your battery. 0 = unlimited.
`min_pv_charge_rate`: 0 # Watt. Lower limit for the PV charge rate in Mode 8 (peak shaving). 0 = no floor.
`max_grid_charge_rate`: 5000 # Watt. Limits the maximum charging rate of your battery when charging from the grid.
`capacity`: 10000 # (optional) battery capacity in Wh. Required for `mqtt`, optional for `fronius_gen24` (overrides the DesignedCapacity)

Fronius Gen24 only (optional): `fronius_inverter_id` (default `"1"`), `fronius_controller_id` (default `"0"`).

`fronius-modbus` only: `port` (default 502), `unit_id` (default 1), `revert_seconds` (default 0, seconds before reverting to the default mode).

`mqtt` only: `min_soc` (default 5), `max_soc` (default 100), `cache_ttl` (default 120, seconds) and `base_topic`. See https://mastr.github.io/batcontrol/integrations/mqtt-inverter/

Outage handling (optional): `enable_resilient_wrapper` (default true), `outage_tolerance_minutes` (default 24) and `retry_backoff_seconds` (default 60) control how long batcontrol tolerates an unreachable inverter before terminating.

### `utility`:

`type`: your electricity provider. Currently supported: `tibber`, `awattar_at`, `awattar_de`, `evcc`, `energyforecast`, `energyforecast_total_price`, `tariff_zones`
`api_token`: Zz-YOURAPIKEYYOURAPIKEYXXXXX # only required for tibber, energyforecast and energyforecast_total_price, ignored for awattar
`market_zone`: DE # optional, energyforecast and energyforecast_total_price only: DE/LU (both -> DE-LU, default), AT, FR, NL, BE, PL, DK1, DK2
`url`: http://evcc.local:7070/api/tariff/grid # only required for evcc
`vat`: 0.20 # only required for awattar and energyforecast, ignored for tibber
`fees`: 0.015 # only required for awattar and energyforecast, ignored for tibber
`markup`: 0.03 # only required for awattar and energyforecast, ignored for tibber
`tariff_zone_1`, `zone_1_hours`, `tariff_zone_2`, `zone_2_hours`, `tariff_zone_3`, `zone_3_hours`: only for `tariff_zones`. Prices in EUR/kWh including VAT and fees, hours as ranges or singles, e.g. `"7-22"` or `"0-5,6,7"`. Zones 1 and 2 must cover all 24 hours, zone 3 is optional and must not overlap.
Tibber provides the personal prices directly via the API. So you need the api_token to get the price for YOUR home.
Awattar only provides the EPEX Spot prices via the API. So the final price is calculated in the software using this formula:
(Spotprice x (1 + markup) + fees) x (1 + VAT)

### `dynamic_network_fees:`

Optional. Adds time-dependent network fees (e.g. German section 14a EnWG tariffs) to the prices.

`enabled`: false # set to true to activate
`country`: de # `de`, `at` or `ch`
`operator`: syna # network operator ID, e.g. syna, westnetz, ggv, ewr-netz
`url`: https://dyn-net.batcontrol.software/api/ # optional, override for a self-hosted instance

### `solar_forecast_provider`:

choose between `fcsolarapi` (default), `homeassistant-solar-forecast-ml`, `evcc-solar` and `solcast`. Check https://mastr.github.io/batcontrol/configuration/solar-forecast/ for details.

### `pvinstallations`:

Only the entry belonging to the selected `solar_forecast_provider` is used: batcontrol looks for an entry with a matching `type`, then for one carrying `entity_id`, and otherwise takes the first entry. Keep the keys of one provider together in one entry.

Common keys:

`- name:` Haus #name of the installation. multiple installations can be added. Dont forget the hyphen in front of `name` to indicate that its a list
`type`: fcsolarapi # optional, makes explicit which provider this entry belongs to

For `fcsolarapi` (PVGIS based, needs the location):

`lat`: 48.4334480 Latitude in degree. Negative values indicate locations south of the equator.
`lon`: 8.7654968 Longitude. Negative Values indicate locations in the West.
`declination`: 32 #inclination toward horizon 0..90 degree 0=flat 90=vertical (e.g. wallmounted)
`azimuth`: -90 # -90:East, 0:South, 90:West Range: -180..180
`kWp`: 15.695 # power in kWp
`api_token`: optional, only if you are a forecast.solar customer and have a key
`horizon`: 30,30,30,0,0,0 # optional, only if the array is shaded

For `homeassistant-solar-forecast-ml` (HACS integration):

`url`: ws://homeassistant:8123 # HomeAssistant WebSocket URL
`api_token`: YOUR_LONG_LIVED_ACCESS_TOKEN # long-lived access token from your HA profile
`entity_id`: sensor.solar_forecast_ml_evcc_solar_prognose # the forecast sensor
`sensor_unit`: auto # `auto` (default), `Wh` or `kWh`. A fixed value skips detection
`cache_ttl_hours`: 12.0 # how long a fetched forecast stays usable as a fallback (default: 12.0)

For `evcc-solar`:

`url`: http://evcc.local:7070/api/tariff/solar # evcc REST endpoint

For `solcast`: the site geometry is configured in the Solcast web toolkit, not here.

`resource_id`: xxxx-xxxx-xxxx-xxxx # from your Solcast rooftop site
`api_token`: your-solcast-api-key # from your Solcast account settings
`percentile`: 50 # optional: 10 (pessimistic), 50 (default), 90 (optimistic)

### consumption_forecast:

`type`: csv or homeassistant-api

For `csv`:

`annual_consumption`: 4500 # total consumption in kWh p.a. . The load profile is scaled to meet this value. (optional)
`load_profile`: load_profile.csv #name of the load profile file within the config folder. To use your own profile, place it in `/addon_configs/HEXCODE_batcontrol/` (see "Customizing the load profile" above) and restart the add-on.

For `homeassistant-api`:

`url`: ws://homeassistant:8123 # HomeAssistant WebSocket URL
`api_token`: YOUR_LONG_LIVED_ACCESS_TOKEN # long-lived access token from your HA profile
`entity_id`: sensor.energy_consumption # entity with consumption data, needs long-term statistics
`sensor_unit`: auto # `auto`, `Wh` or `kWh`
`history_days`: "-7;-14;-21" # which days to look back
`history_weights`: "1;1;1" # weight per history period (1-10)
`cache_ttl_hours`: 48.0 # how long computed statistics are cached
`multiplier`: 1.0 # forecast adjustment (1.0 = unchanged)

### `mqtt`:

Enables publishing of battery status and control messages via MQTT and integrates with Home Assistant through MQTT discovery.

`enabled`: true / false # enable or disable MQTT support
`logger`: true / false # if true, batcontrol log messages are also sent via MQTT
`broker`: localhost # MQTT broker hostname or IP
`port`: 1883 # MQTT broker port
`topic`: house/batcontrol # base topic for published MQTT messages
`client_id`: batcontrol # unique MQTT client identifier of this instance (optional, default: `batcontrol`). Only needed when several batcontrol instances share one broker - a duplicate client ID makes the broker disconnect the other instance. Must not be left empty.
`username`: user # MQTT username
`password`: password # MQTT password
`retry_attempts`: 5 # number of reconnect attempts before failing (optional, default: 5)
`retry_delay`: 10 # seconds to wait between reconnect attempts (optional, default: 10)
`tls`: true / false # enable TLS encryption
`cafile`: /etc/ssl/certs/ca-certificates.crt # path to CA certificate
`certfile`: /etc/ssl/certs/client.crt # client certificate (if required)
`keyfile`: /etc/ssl/certs/client.key # client private key (if required)
`tls_version`: tlsv1.2 # TLS version (default: tlsv1.2)
`auto_discover_enable`: true / false # enable MQTT discovery for Home Assistant
`auto_discover_topic`: homeassistant # base topic for Home Assistant discovery messages
`grid_charge_lock_topic`: hems/batcontrol/grid_charge_lock # optional: absolute topic (not nested below `topic`) for an external HEMS/grid-operator grid-charge lock signal (section 14a EnWG), mirrors evcc's "External Limit". Payload `1`/`true` blocks charging from the grid immediately, `0`/`false` (default) restores the previous limit.

### `api`:

The read-only web dashboard, HTTP API and MCP server. In this add-on the dashboard is the
**Batcontrol** entry in the Home Assistant sidebar; Home Assistant authenticates every request
to it, so no port has to be published and no token is needed for normal use.

Everything here is read-only. Nothing in the dashboard or the API can change the battery's
behaviour - overrides stay on MQTT.

`enabled`: true / false # serve the API, dashboard and MCP server (default: true - the sidebar panel needs it)
`host`: 0.0.0.0 # bind address. **Leave this at 0.0.0.0**: ingress connects from the Home Assistant supervisor, so 127.0.0.1 makes the panel unreachable
`port`: 8080 # must match the add-on's ingress port - change only if you know why
`dashboard_enabled`: true / false # serve the dashboard itself (default: true)
`docs_enabled`: true / false # serve the API documentation (Swagger UI) at `/docs` and the schema at `/openapi.json` (default: false)
`mcp_enabled`: true / false # serve an MCP server at `/mcp` so an AI assistant can ask about the battery (default: false)
`auth_token`: "" # optional bearer token for `/api/v1`. Only needed if you map port 8080 in the add-on's network settings - ingress is already authenticated
`mcp_auth_token`: "" # optional separate token for `/mcp`, so an assistant can be given MCP access without the REST API. Empty means it inherits `auth_token`

**Reaching it from outside Home Assistant** (another dashboard, a script, an MCP client): map
port 8080 under the add-on's *Network* settings. That port bypasses Home Assistant's
authentication, so set `auth_token` before you do.

Full documentation: [REST API](https://mastr.github.io/batcontrol/integrations/rest-api/),
[MCP server](https://mastr.github.io/batcontrol/integrations/mcp-server/) and
[Dashboard](https://mastr.github.io/batcontrol/features/dashboard/).

### `evcc`:

Allows integration with [evcc](https://docs.evcc.io/) to coordinate battery usage when charging an electric vehicle.
Batcontrol can pause/disallow battery discharge while EV charging is active.

`enabled`: true / false # enable or disable evcc integration
`broker`: localhost # MQTT broker used by evcc
`port`: 1883 # MQTT broker port
`status_topic`: evcc/status # topic for general evcc status
`loadpoint_topic`: # list of MQTT topics to monitor for charging status per loadpoint

- evcc/loadpoints/1/charging
- evcc/loadpoints/2/charging

`block_battery_while_charging`: true / false # if true, disallow battery discharge while EV is charging
`username`: user # MQTT username
`password`: password # MQTT password
`tls`: true / false # enable TLS encryption
`cafile`: /etc/ssl/certs/ca-certificates.crt # path to CA certificate
`certfile`: /etc/ssl/certs/client.crt # client certificate (if required)
`keyfile`: /etc/ssl/certs/client.key # client private key (if required)
`tls_version`: tlsv1.2 # TLS version (default: tlsv1.2)
`battery_halt_topic`: evcc/site/bufferSoc # optional: topic that delivers SOC threshold below which the battery will be locked

## Persistent data and caches

Batcontrol saves the latest raw responses of the solar forecast and API tariff providers in
`/data/data/cache` (Home Assistant's persistent add-on volume). After a restart or an add-on update
they are used as a fallback when the first refresh fails. No configuration is needed. The cache may
contain installation or household data. To clear it, stop the add-on and delete the `cache` folder
in the add-on's persistent data volume.

## Algorithm

The add-on gets three forecasts

1. electricity price forecast - from your tariff provider (see `utility`)
1. your electricity production forecast - from your solar forecast provider (see `solar_forecast_provider` and `pvinstallations`)
1. your electricity consumption forecast - from a load profile scaled to your `annual_consumption`, or from your Home Assistant history (see `consumption_forecast`)

Based on the three forecasts AND the current state of charge (SOC) the software puts the inverter in one of the following modes:

**MODE 10 - DISCHARGE ALLOWED. (DEFAULT)**
This is the normal mode of the battery if there is sufficient energy available or energy is currently expensive. If the battery is full the surplus will be feed-in. If there is not enough energy coming from your pv installatation to meet your demand energy from the battery will be used.

**MODE 8 - LIMIT PV CHARGE RATE.**
Used by peak shaving: the rate at which the PV system charges the battery is limited so that capacity remains for the midday peak. Discharging stays allowed. See `peak_shaving`.

**MODE 0 - AVOID DISCHARGE.**
If your consumption exceeds your current production energy from the grid will be used and the battery will not be discharged. This mode is used if prices are increasing in the future and the energy from the battery can be more efficiently used in the future

**MODE -1 - CHARGE FROM GRID.**
The battery is charged from the grid at a certain charge rate. This mode calculates the estimated required energy for future high priced hours. The objective is to charge the battery enough so that you do not need to consume energy from the grid in hours with high prices.

The Home Assistant "Decision" sensor and the dashboard show why a mode was chosen.

## Fronius Inverter control (`fronius_gen24`)

Fronius inverters do not allow setting parameters via the solar web API.
So instead the software pretends to be a user of the web UI to modify the corresponding settings of your inverter.
This procedure has several disatvantages:

- usage of non documented APIs
- a firmware update of the inverter might cause disruptions
- unintended side effects (e.g. manual settings in the WebUI are overwritten to put the inverter in the defined Modes)

Do not use this software if you do not want those side effects. The `fronius-modbus` and `mqtt`
inverter types are alternatives, see the inverter configuration documentation.
