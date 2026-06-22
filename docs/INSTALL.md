# Yarbo Cadence EQ — Installation

Condition-aware autonomous mowing for a Yarbo + Home Assistant. This installs a
**package** (helpers, normalization layer, decision logic, scripts, automations)
plus a **dashboard**. All entities use the prefix `yarbocadenceeq_`.

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| **Home Assistant** | Any recent version. You need a way to edit YAML files — File Editor add-on, Studio Code Server, or Samba/SSH. |
| **YarboHA integration** | Install via HACS (uses `yarbo-data-sdk`). Your Yarbo must be paired and its entities present. YarboHA commonly prefixes entities with the robot serial, e.g. `sensor.25070102ovpie639_battery`. |
| **Weather integration** | Any `weather.*` entity with forecast support. `weather.openweathermap` is preferred when present; otherwise the package can auto-detect a single weather entity or you can set an override in the dashboard. |
| **Soil moisture sensor(s)** | Optional but recommended. Any moisture/soil sensors that report a %; configured by entity ID in a UI field (see step 4). The gate can also be switched off entirely. |
| **Mower head attached** | The logic refuses to dispatch unless the head type reads `Mower` or `Mower Pro`. |

Files you'll install:
- `yarbo_cadence_eq.yaml` — the package
- `yarbo_cadence_eq-dashboard.yaml` — the dashboard

---

## 2. Enable packages (one-time)

In `configuration.yaml`, make sure you have this under `homeassistant:`:

```yaml
homeassistant:
  packages: !include_dir_named packages/
```

Then create the folder `config/packages/` if it doesn't exist.

> ⚠️ **Do NOT paste the package body into `configuration.yaml`.** It defines its
> own top-level keys (`template:`, `sensor:`, `automation:`, `script:`, …). If
> those collide with keys already in `configuration.yaml`, that whole domain
> fails to load — the classic symptom is helpers showing up but every template
> sensor reading **"Entity not found."** It must live as its own file in
> `packages/`.

---

## 3. Drop in the package

1. Copy `yarbo_cadence_eq.yaml` into `config/packages/`.
2. Make sure there's **no second copy** in there (e.g. an older `yarbo_lawn.yaml`).
   Two copies = duplicate helpers and a half-loaded mess.

---

## 4. Point it at YOUR entities

The package creates a stable `yarbocadenceeq_*` adapter layer over YarboHA's
entity IDs. If there is exactly one YarboHA device, it auto-detects the slug from
the native `select.<slug>_plan_select` / `select.<slug>_working_state` entities.
If you have multiple Yarbo devices, or auto-detection does not find yours, set the
dashboard's **"Yarbo slug override"** field to the prefix before `_battery`.

Examples:

| Battery entity | Slug override |
|---|---|
| `sensor.25070102ovpie639_battery` | `25070102ovpie639` |
| `sensor.yarbo_battery` | `yarbo` |

The dashboard uses `select.yarbocadenceeq_plan_select`, so you should not edit the
dashboard for serial-specific YarboHA entity IDs.

- **Yarbo slug status.** On the dashboard's Tuning card, **Yarbo slug status**
  should read `OK`. If it says `Set Yarbo slug`, search Developer Tools → States
  for your Yarbo battery entity and fill in **Yarbo slug override**.
- **Soil moisture sensors** — configured in the UI, no file edit. After load,
  open the dashboard's **Tuning → Moisture sensors** card and put your sensor
  entity IDs in the **"Moisture sensor entity IDs"** field as a comma-separated
  list (any count). The field defaults to three example Ecowitt channels:
  ```
  sensor.gw3000b_wifi08ab_soil_ch1, sensor.gw3000b_wifi08ab_soil_ch2, sensor.gw3000b_wifi08ab_soil_ch3
  ```
  Replace them with yours, then turn **"Use moisture sensors"** on if you want
  soil moisture to gate mowing. The gate is **off by default**. When enabled, it
  uses the **wettest** listed zone. If the gate is on but every listed sensor is
  unavailable, dispatch is **blocked** and you get a notification (a
  wet-but-unknown lawn shouldn't be mowed).
  > The entity-ID field is capped at **255 characters** — enough for ~8–10 typical
  > IDs. If you have more sensors than fit, shorten their entity IDs or open an issue.
- **Weather.** Forecast timing uses the dashboard's **Weather entity override**
  only when you fill it in. Leave it blank to use `weather.openweathermap` when
  present, or the only `weather.*` entity if your Home Assistant has exactly one.
  **Weather entity status** should read `OK`. If it says `Set weather entity`,
  search Developer Tools → States for your weather entity ID, for example
  `weather.forecast_home`, and put that full entity ID in the override field.

### Enable the Yarbo rain sensor
The native **`sensor.<slug>_rain_sensor`** may be disabled by default in the
integration. The rain logic needs it. Go to **Settings → Devices & Services →
Yarbo → entities**, find *Rain Sensor*, and **enable** it. The package exposes it
as `sensor.yarbocadenceeq_rain_sensor` and reads a 30-second rolling average of
that value.

---

## 5. Check config & restart

1. **Developer Tools → YAML → Check Configuration.** Fix anything it flags.
2. **Settings → System → Restart** — do a **full restart**, not a YAML reload.
   Helpers + template entities defined in a package only come up together on a
   restart.

### Verify it loaded
**Developer Tools → States**, filter `yarbocadenceeq`. You should see the
sensors and binary_sensors (e.g. `sensor.yarbocadenceeq_status_reason`,
`binary_sensor.yarbocadenceeq_ok_to_cut_now`). If the **helpers** exist but the
**template sensors** are missing → check **Settings → System → Logs** for a
template error, and re-confirm step 2 (loaded as a package, not pasted into
`configuration.yaml`).

---

## 6. Add the dashboard

1. **Settings → Dashboards → + Add Dashboard → New dashboard from scratch.**
2. Open it → top-right **pencil (Edit)** → **⋮ → Raw configuration editor**.
3. Paste the entire contents of `yarbo_cadence_eq-dashboard.yaml` → **Save**.
4. (Optional) Rename the dashboard's sidebar title to **Yarbo Cadence EQ** in its
   settings — the in-file title is already set, but the sidebar label is chosen
   when you create the dashboard.

---

## 7. First-run setup

1. **Confirm the slug.** On the **Tuning** card, **Yarbo slug status** should say
   `OK`. If not, set **Yarbo slug override** to your YarboHA entity prefix.
2. **Pick your plan.** On the **Controls** card, open **"Plan to run (from
   Yarbo)"** and select your mowing plan. If the list is empty, press **"Refresh
   all Yarbo data"** once, then try again. Your choice is remembered across
   restarts.
3. **Tune the sliders.** In the **Tuning** card (all prefixed *YarboCadenceEQ:*):
   - **Use moisture sensors** — off by default; enable only if you have configured
     soil sensors.
   - **Max allowed soil moisture** — cut only when the wettest zone is below this.
   - **Min battery to dispatch** — defaults to 95.
   - **Hot temp threshold** + **Sun buffers** — heat pause only in direct sun.
   - **Onboard rain trigger value** + **Rain must persist** — spike protection.
   - **Target cut interval** / **Min hours between cuts** — these now go as low as
     **1 hour** for testing.
4. **Leave the master switch OFF** until you've reviewed everything, then flip
   **🟢 Mowing** on.

---

## 8. HomeKit (optional)

Expose only these to HomeKit (the dashboard's **"Show HomeKit setup help"** toggle
lists them too):

- `input_boolean.yarbocadenceeq_auto_enabled` — master on/off (shows as a switch)
- `script.yarbocadenceeq_cmd_start` — start a cut
- `script.yarbocadenceeq_cmd_stop` — pause / freeze
- `script.yarbocadenceeq_cmd_resume` — resume
- `script.yarbocadenceeq_cmd_go_home` — return to dock
- `script.yarbocadenceeq_refresh_all` — refresh all Yarbo data

Scripts appear in the Home app as buttons.

---

## 9. (Optional) Group the entities

HA can't put YAML helpers/template entities into a "device," so use a **Label**:

1. **Settings → Labels → Create label** → `Yarbo Cadence EQ`.
2. **Settings → Devices & Services → Entities** → search `yarbocadenceeq`.
3. Select all → **Add label** → `Yarbo Cadence EQ`.

Now you can filter the entity list by that label and target the whole group.

---

## 10. Migrating from an earlier build? (clean entity-ID reset)

If you previously loaded a `lawn_*` or `cadence_*` version, you'll have **stale,
duplicated entity IDs** — e.g. an automation whose friendly name is correct
("YarboCadenceEQ: Main scheduler") but whose entity_id is `automation.lawn_main_scheduler_3`.

**Why:** automation and template-sensor entity_ids are generated from the
alias/name **once**, on first load, and pinned in the entity registry. Renaming
the prefix later updates the friendly name but **not** the entity_id, and each
prefix change registered a fresh copy while the old ones lingered — so HA tacked
on `_2`, `_3` to avoid collisions. (Helpers and scripts are keyed differently and
stay clean — they just leave orphaned old-prefix siblings.)

The only way to get clean entity_ids back is a one-time registry purge:

1. **Remove** `yarbo_cadence_eq.yaml` from `config/packages/` (move it elsewhere).
2. **Restart** HA. Every `yarbocadenceeq_*` / `cadence_*` / `lawn_*` entity now
   shows as **unavailable/restored**.
3. **Settings → Devices & Services → Entities.** Search `lawn` → select all →
   **Remove selected**. Repeat for `cadence`, then `yarbocadenceeq`. (Also check
   **Settings → Automations** for any leftover unavailable automations and delete
   them.) This wipes every generation from the registry.
4. **Put the package file back** in `config/packages/`.
5. **Restart** HA again.
6. Everything now registers **fresh** with clean IDs derived from the current
   names — `automation.yarbocadenceeq_main_scheduler`,
   `sensor.yarbocadenceeq_status_reason`, etc. — which is also what the dashboard
   references, so "Entity not found" clears at the same time.
7. **Re-point HomeKit** and **re-check the Tuning sliders** (helpers come back at
   their YAML defaults).

> A brand-new install never has this problem — IDs are clean from the first load.
> This only bites systems that were renamed in place.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Every template sensor = "Entity not found", but helpers exist | Either the package was pasted into `configuration.yaml` (key collision) / you reloaded instead of restarting **or** stale entity_ids from an earlier rename don't match the dashboard — see §10 clean reset. |
| Entity IDs look like `automation.lawn_..._3` though the names are right | Registry cruft from renaming a live system. Do the §10 clean reset. |
| Battery/sensors read 0 or unknown | Slug mismatch — set **Yarbo slug override**, or your soil/weather entity IDs don't match (step 4). |
| Status says "No plan selected" | Pick a plan on the dropdown; press **Refresh all Yarbo data** if it's empty. |
| Status says "Waiting for STRONG GPS/RTK" | Normal on cloudy/cold mornings — it wakes the robot and refreshes GPS, retrying every 5 min until RTK is Strong. |
| Rain cancels a cut the instant it starts | The onboard sensor spikes at head startup; raise **Rain must persist** (default 60s) or the trigger value. |
| Won't start while charging | The integration may reject Start while charging even at 100%. This build attempts it anyway once battery ≥ your minimum — watch whether your robot actually starts (see project notes). |
| Last-cut time didn't update after a finished cut | Should be fixed (it now catches both "Completed" and "Waypoint Complete" plus a docked fallback). If it still misses, check Logs and tell the maintainer. |

---

*Behavior reference is in `README.md`; the roadmap and design notes are in
`ROADMAP.md`.*
