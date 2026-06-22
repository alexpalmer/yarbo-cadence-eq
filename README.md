# Yarbo Cadence EQ

**Condition-aware autonomous mowing for a [Yarbo](https://www.yarbo.com/) robotic mower, on [Home Assistant](https://www.home-assistant.io/).**

A drop-in HA **package** + **dashboard** that turns "mow on a fixed schedule" into
"mow when it actually makes sense." It only dispatches the robot when the lawn is
dry, the robot is home and charged, GPS/RTK is **Strong**, it's on **Halow** with
a mower head attached, and it isn't dangerously hot in direct sun — and it
will pull a cut *earlier* to beat incoming rain, or push it *past* the rain when it
can't finish in time.

Built and tested against **YarboHA v0.3.3** (`yarbo-data-sdk 0.2.1`).

> Not affiliated with or endorsed by Yarbo. Community project; use at your own risk.

---

## What it does

- **One master switch** (`input_boolean.yarbocadenceeq_auto_enabled`, shown as *Mowing*).
  On = the scheduler may dispatch. Off = it stops the job, returns to dock, and
  cancels the plan — without resetting the last-cut clock, so it resumes cleanly
  when you turn it back on.
- **Serial-prefix tolerant by default.** YarboHA often creates entities from the
  robot serial (`sensor.25070102ovpie639_battery`, etc.). The package now creates
  stable `yarbocadenceeq_*` wrapper entities, auto-detects one YarboHA device, and
  lets you override the slug only when needed.
- **A single decision sensor.** Every dispatch condition is evaluated in one place
  (`sensor.yarbocadenceeq_cut_blocker`) that reads the YarboHA adapter state and
  returns either `OK` or a plain-English list of what's holding things up. The
  dashboard's **Status** line always tells you the reason.
- **Cadence.** Target interval + minimum gap between cuts (defaults 48 h / 24 h),
  with an estimated cut duration so pre-rain timing is realistic.
- **Rain handling.** Two independent signals — the robot's **onboard** rain reading
  (cleaned up with a threshold + persistence debounce) and the **forecast** from
  your configured `weather.*` entity. Either one blocks a cut.
- **Heat protection.** Pauses and docks above a temperature threshold, but only
  during the direct-sun window, and resumes when it cools off or the window ends.
- **Moisture gate.** Blocks mowing when your wettest soil zone is above a limit.
  Fully configurable by entity ID, and switchable off.

---

## Files & layout

```
yarbo-cadence-eq/
├── packages/
│   └── yarbo_cadence_eq.yaml          # the package: helpers, logic, scripts, automations
├── lovelace/
│   └── yarbo_cadence_eq-dashboard.yaml # the dashboard (desktop 3-column + phone view)
├── docs/
│   └── INSTALL.md                     # step-by-step install
├── ROADMAP.md                         # planned 2.0 features + design notes
├── CHANGELOG.md
└── LICENSE
```

Everything the package creates is prefixed **`yarbocadenceeq_`**.
Full setup is in **[docs/INSTALL.md](docs/INSTALL.md)**; the short version:

1. Enable packages and drop `yarbo_cadence_eq.yaml` into `config/packages/`.
2. Enable the integration's disabled-by-default **Rain Sensor** entity.
3. Restart HA (a package's helpers + template entities come up together only on a restart).
4. Paste `yarbo_cadence_eq-dashboard.yaml` into a new dashboard's raw editor.
5. Confirm **Yarbo slug status** and **Weather entity status** are `OK`, pick your
   plan, optionally enable/configure moisture sensors, tune the sliders, flip
   *Mowing* on.

---

## Dispatch gates

All of these must pass before the scheduler auto-starts a cut. Any failure shows
up by name on the Status line.

| Gate | Requirement |
|---|---|
| Master switch | *Mowing* is **on** |
| Online | Robot reachable |
| Home | **Docked** (tracked by a latch, not the spiky charging flag — see below) |
| Battery | ≥ *Min battery to dispatch* (default **95%**) |
| Plan | A plan is selected on the Cadence EQ plan dropdown |
| RTK / GPS | **Strong** (Medium allowed only if you opt in) |
| Network | **Halow** |
| Head | **Mower** or **Mower Pro** (won't run with the SAM/blower head) |
| Moisture | Wettest configured zone below the limit (default **52%**) |
| Rain | No onboard rain and no forecast rain |
| Heat | Not above the hot threshold while in the direct-sun window |
| Dock GPS | Robot within **0.5 m** of the saved dock coordinates (optional gate) |

**Charging is *not* a blocker.** Earlier builds refused to dispatch while charging;
now a cut may start as soon as battery ≥ your minimum, even mid-charge (matching
what the phone app allows). Set the minimum to 100 if you'd rather always wait for
a full battery.

**Why a dock latch?** The robot publishes no reliable "is docked" boolean. Its
`Recharging Status` reads *Charging* on the dock (good), but battery-care cycles can
clear that field while the robot is still physically docked (bad). So the package
keeps its own `input_boolean.yarbocadenceeq_docked` latch: set on a charge signal,
cleared only on a GPS-confirmed departure from the dock. This is the single most
important reliability fix in the project.

---

## Controls

**Master:** `input_boolean.yarbocadenceeq_auto_enabled` — persistent kill switch;
this is the one to expose to HomeKit.

**Four transient commands** (scripts — they run regardless of the master switch, so
they double as manual overrides):

| Script | Action |
|---|---|
| `script.yarbocadenceeq_cmd_start` | Start the selected plan now |
| `script.yarbocadenceeq_cmd_stop` | Pause / freeze in place (resumable) |
| `script.yarbocadenceeq_cmd_resume` | Resume from the pause point |
| `script.yarbocadenceeq_cmd_go_home` | Return to charge |
| `script.yarbocadenceeq_refresh_all` | Refresh all Yarbo data (plans, map, sensors) |

Rain-cancel and heat-pause still run even while the master switch is off, so a cut
you start by hand is still protected. Heat-*resume* only runs while enabled.

---

## Tuning

Every knob is a Home Assistant **helper**, persisted across reboots; move a slider
and the change takes effect immediately. The dashboard groups them:

- **Yarbo slug** — auto-detected for a single YarboHA device; override only if the
  status is not `OK` or you have multiple Yarbo devices.
- **Cadence** — target interval, min gap, active-plan average run time, cut buffer.
- **Heat / Sun** — hot threshold, morning/evening sun buffers, heat-resume mode.
- **Yarbo Rain sensor** — onboard rain trigger value (the raw reading that counts as
  "wet," default **500** so dry-sensor noise is rejected) and how long it must
  persist (default **60 s**).
- **Moisture sensors** — off by default; turn on only after entering your soil
  sensor entity IDs. Includes max allowed % and the comma-separated entity-ID list
  (capped at 255 chars).
- **Dock GPS** — require-match toggle and the saved dock latitude/longitude.

A **"Reset tuning to recommended"** toggle re-seeds every value to its default.
Defaults are also applied automatically on first install.

---

## How rain detection works

- **Onboard** (`binary_sensor.yarbocadenceeq_onboard_rain_detected`): the robot's
  rain bool **or** a 30-second rolling average of the raw rain reading crossing your
  trigger value, then required to persist for the debounce time. The persistence is
  what kills the spike the sensor throws at head startup.
- **Forecast** (folded into `binary_sensor.yarbocadenceeq_rain_detected`): the OWM
  condition (rainy/pouring/lightning) or current-hour precipitation in the hourly
  forecast. Forecast signals fire immediately — debouncing a prediction is pointless.

The forecast also drives **pre-rain timing**: the scheduler pulls a cut forward only
when the rain is far enough out that the cut can genuinely finish first; otherwise it
holds the normal schedule, or schedules for after the rain clears.

> The forecast is used for **timing**. Stopping an in-progress cut is the onboard
> sensor's job. Multi-source forecast cross-checking is on the roadmap (2.0).

---

## A note on fresh installs vs. renames

Home Assistant pins a template/automation entity_id from its name **once**, on first
creation, in the entity registry. A brand-new install gets clean IDs that match the
dashboard automatically. But if you ever **rename** an entity's name after the fact,
the friendly name changes while the entity_id stays frozen — which surfaces as
"Entity not found" on the dashboard even though the entity exists under an old ID.
The fix is to edit that one entity's ID back (Settings → Entities → the entity →
Settings → Entity ID), or do the clean registry reset in
[docs/INSTALL.md §10](docs/INSTALL.md). New users won't hit this.

---

## Roadmap

Planned for 2.0: a seasonal "graphic EQ" cadence model (per-month intervals),
post-rain drying delay (leaf-wetness + soil), multi-source weather cross-checking,
rain-interruption recovery, blackout windows, and a voice-armed ad-hoc "cut now"
plan. Details and design notes in **[ROADMAP.md](ROADMAP.md)**.

---

## License

MIT — see [LICENSE](LICENSE).
