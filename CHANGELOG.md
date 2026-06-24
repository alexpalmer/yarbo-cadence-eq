# Changelog

All notable changes to this project are documented here. This project aims to
follow [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [1.2.0] — 2026-06-24

### Added
- **4-slot multi-area scheduling**: schedule up to four independent mowing areas
  (front yard, back yard, side strip, etc.) each on its own cadence, time window,
  and Yarbo plan assignment.
- `input_datetime.yarbocadenceeq_plan{1–4}_latest_start`: per-slot latest-start
  time; dispatch is skipped for the day once the window closes and retried from
  the next day's earliest-start.
- Plan schedule cards now show From / Until window per slot and display a
  "held by latest-start window" status hint when a slot is waiting for tomorrow.
- `sensor.yarbocadenceeq_current_rain_probability` and
  `sensor.yarbocadenceeq_current_rain_probability_display` — live rain-probability
  readout on both dashboard views.
- YarboHA adapter layer: package and dashboard use stable `yarbocadenceeq_*`
  entities instead of assuming native `yarbo_*` entity IDs. Works with
  serial-prefixed installs (e.g. `sensor.25070102ovpie639_battery`).
- Serial-prefix auto-detection for single-device YarboHA installs, plus a
  slug-override helper and status row for multi-device or custom-slug setups.
- Weather entity auto-detection/override with dashboard status rows — installs
  no longer require `weather.openweathermap`.
- Per-slot `sensor.yarbocadenceeq_plan_N_hours_since_last_cut` exposed on both
  desktop and phone dashboards (all 4 slots).
- Global last-cut datetime row on phone Controls card.
- `sensor.yarbocadenceeq_next_queued_slot` drives the "due now" pending-slot
  display on both dashboard views (replaces verbose inline Jinja loop).
- Forecast cache now fires on `yarbocadenceeq_refresh_weather` event, so
  "Refresh all Yarbo data" also refreshes the weather forecast.

### Changed
- Hot temp threshold entity renamed `_f` → `_c`; range updated to 27–43°C;
  default 33°C.
- Replaced hard-coded native plan dropdown with `select.yarbocadenceeq_plan_select`.
- Dashboard fully restored to slug-agnostic `yarbocadenceeq_*` adapter entities
  throughout — upstream v1.1.1 had re-introduced raw `yarbo_*` hardcoded
  references which broke custom-slug installs.
- Rain statistics source changed to `sensor.yarbocadenceeq_rain_sensor` for
  slug-agnostic installs (adapter; brief Unknown at startup during slug resolution,
  recovers within the 30-second window).
- Go/No-Go card: distinct headers for Mowing / Heat-paused / OK / Blocked states.
- Tuning card simplified to weather entity + rain probability + battery minimum +
  RTK toggle.
- Plan schedule markdown uses `{%- -%}` trim blocks, eliminating ~18 spurious
  blank lines per slot in folded YAML scalars.
- Plan schedule slot detail line consolidated: Last … (Xh ago) · Next … ·
  Window HH:MM–HH:MM.
- Fixed per-slot sensor slugs: `planN_*` → `plan_N_*` in dashboard and package
  template sensors (`hours_since_last_cut`, `next_cut_prediction`).
- Added blank-line spacers between slot entries in plan-schedule summary cards.
- Soil moisture gate defaults to off — installs without soil sensors unblocked.
- Head gate accepts `Mower Pro` as a valid mower head (alongside `Mower`).
- Numeric sensors with missing inputs now become unavailable cleanly instead of
  emitting invalid `unknown`/`unavailable` numeric states.
- Removed `initial:` from `use_moisture` and `moisture_sensors` helpers (was
  resetting configured values on HA restart).
- Removed `default_entity_id:` from template sensors (deprecated HA API).

### Fixed
- Forecast cache: last-good guard so a failed or empty weather fetch no longer
  wipes the cached forecast; `continue_on_error` per fetch type; self-healing
  attributes restore the last-good value automatically.
- Dock latch: GPS proximity (RTK Strong + within 0.5 m + not mid-plan) added as
  a second latch signal — fixes "100% battery, idle, not docked" false-negative.
- Yarbo Nominal sensor: now ON while actively mowing OR ready on dock; previously
  only fired for ready-on-dock, flipping it off during a cut.
- Active-mow status display: big-card battery line, go/no-go mowing state, and
  Yarbo nominal flag are now correct while a cut is in progress.
- Rain probability slider: minimum changed from 30 → 0, step from 5 → 1.

## [1.0.0] — 2026-06-21

First public release. Condition-aware autonomous mowing for Yarbo on Home
Assistant. Built against YarboHA v0.3.3 / `yarbo-data-sdk 0.2.1`.

### Added
- Single-source decision logic (`sensor.yarbocadenceeq_cut_blocker`) returning a
  plain-English reason list, with thin `ok_to_cut_now` / `status_reason` wrappers.
- Persistent **dock latch** (`input_boolean.yarbocadenceeq_docked`) set on charge and
  cleared only on GPS-confirmed departure — robust against the absence of any native
  "is docked" boolean and against battery-care clearing the charging flag.
- Configurable **moisture gate**: comma-separated sensor entity-ID list (UI field,
  255-char cap), on/off toggle, max-% limit, wettest-zone evaluation, and a
  block-and-notify path when the gate is on but all sensors are unavailable.
- **Onboard rain** detection as a standalone signal: factory rain bool OR a 30-second
  rolling-average threshold, persisted by a debounce to reject the head-startup spike.
- **Forecast-aware scheduling**: pre-rain pull-ahead only when a cut can finish before
  the rain; otherwise hold or schedule past rain-clear.
- Optional **dock-GPS gate** (within 0.5 m of saved coordinates).
- **Reset-to-recommended** defaults, applied on first install and on demand.
- Heat protection (pause/dock above threshold only within the direct-sun window;
  configurable resume mode).
- Four transient command scripts (start / stop / resume / go-home) plus refresh-all,
  HomeKit-friendly.
- Desktop (3-column) and phone Lovelace dashboards.

### Notes
- Charging is no longer a dispatch blocker; a cut may start at the battery minimum
  even mid-charge. Set the minimum to 100 to require a full battery.
- Forecast is used for **timing**; the onboard sensor is the authority for stopping a
  cut. Multi-source forecast cross-checking and a full forecast/onboard role split are
  planned for 2.0 (see ROADMAP.md).
