# Changelog

All notable changes to this project are documented here. This project aims to
follow [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [1.2.0] — 2026-06-24

### Added
- YarboHA adapter layer: package and dashboard now use stable `yarbocadenceeq_*`
  entities instead of assuming native `yarbo_*` entity IDs.
- Auto-detection for single-device YarboHA installs, plus a slug-override helper
  and status row for serial-prefixed or multi-device setups.
- Weather entity auto-detection/override and dashboard status rows — installs no
  longer fail when the weather entity is not `weather.openweathermap`.
- Per-slot hours-since-cut sensors (`sensor.yarbocadenceeq_plan_N_hours_since_last_cut`)
  surfaced on both desktop and phone dashboards.
- Latest-start-time field per slot; multi-area slot-scheduling documentation.
- `sensor.yarbocadenceeq_next_queued_slot` drives the "due now" pending-slot
  display on both dashboard views, replacing a verbose inline Jinja loop.

### Changed
- Replaced hard-coded native plan dropdown with `select.yarbocadenceeq_plan_select`.
- Fixed per-slot sensor slugs: `planN_*` corrected to `plan_N_*` throughout the
  dashboard and template sensors (affects `hours_since_last_cut` and
  `next_cut_prediction` references).
- Fixed `sensor.yarbocadenceeq_next_cut_prediction` slug in two package template
  sensors (`next_scheduled_cut` and `next_queued_slot`).
- Added blank-line spacers between slot entries in plan-schedule summary cards so
  entries no longer run together visually.
- Soil moisture gate now defaults to off — installs without soil sensors are
  unblocked by default.
- Head gate now accepts YarboHA's `Mower Pro` value as a valid mower head.
- Numeric sensors with missing inputs now become unavailable cleanly instead of
  emitting invalid `unknown`/`unavailable` numeric states.

### Fixed
- Forecast cache: last-good value is now retained so a failed weather poll does
  not erase the cached forecast.
- Dock latch: GPS proximity is now a primary factor alongside charging state,
  preventing false-docked conditions.
- Active-mow status display: big card battery line, go/no-go mowing state, and
  Yarbo nominal flag are now correct while a cut is running.
- Rain probability: 0–100 slider added; current rain probability shown live on
  the dashboard.

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
