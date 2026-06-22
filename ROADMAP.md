# Roadmap & design notes

This file tracks what's already shipped and what's planned, with enough design
detail to make each future item buildable. Items marked ⭐ are the substantial
ones.

---

## Shipped in 1.0

- **Single-source decision logic.** Every dispatch condition lives in one sensor
  (`sensor.yarbocadenceeq_cut_blocker`) that reads normalized YarboHA adapter state and
  returns `OK` or a human-readable reason list. The dashboard status and the
  `ok_to_cut_now` binary are thin wrappers over it.
- **Dock latch.** The robot exposes no reliable "is docked" boolean, and its
  charging flag clears during battery-care while still docked. A persistent latch
  (`input_boolean.yarbocadenceeq_docked`) is set on a charge signal and cleared only
  on a GPS-confirmed departure, which resolved the project's worst class of bugs.
- **Charging is not a blocker.** A cut may start as soon as battery ≥ the minimum,
  even mid-charge (matching the phone app). Set the minimum to 100 to always wait
  for full.
- **Configurable moisture gate.** Soil sensors are entered as a comma-separated
  entity-ID list in the UI (any count, any brand), with an on/off toggle and a max
  %. Gate-on-but-all-unavailable blocks and notifies rather than failing open.
- **Rain handling.** A cleaned-up onboard rain signal (threshold on a 30-second
  rolling average + persistence debounce to kill the head-startup spike) plus a
  forecast signal; either blocks a cut. Pre-rain timing pulls a cut forward only
  when it can finish before the rain, and otherwise schedules past it.
- **Dock-GPS gate.** Optional requirement that the robot be within 0.5 m of saved
  dock coordinates.
- **Reset to recommended.** A single defaults map seeds every tunable on first
  install and on demand via a toggle, excluding dock coords, slug, the last-cut
  clock, and the master switch.

---

## Planned for 2.0

### Event-driven dispatch
Today the scheduler is time-driven (every 5 minutes). Add state triggers so it acts
the instant the last blocker clears — RTK → Strong, battery → target, moisture
below threshold — with the periodic tick kept as a backstop.

### Per-plan cut times
Plans come from the robot's dropdown; next, attach an average cut duration to *each*
plan instead of one shared value, so each plan's pre-rain timing is accurate.

### Easy plan switching
Smooth switching between plans (e.g., to block off part of the yard), building on
the remembered-pick mechanism already in place.

### Seasonal cadence model — the "Graphic EQ" ⭐
The cut interval changes across the year, set and visualized like a stereo graphic
equalizer:

- **12 vertical sliders**, one per month, as the EQ "bands," each with a value
  readout, and a smooth curve drawn across the year.
- Think in **days** on the EQ (more intuitive), convert to hours internally.
- **Smooth interpolation between months is the whole point.** Don't step the
  interval at month boundaries — interpolate between adjacent monthly anchors by
  day-of-year so the effective interval ramps (moving from a 2-day to a 3-day
  cadence, mid-transition is ~2.5 days). That continuous ramp produces the
  sine-wave look.
- Directional intent (anchors user-tunable): winter months = **0 / off**, ramp in
  through spring, tightest in peak growth (~24 h), relaxing toward late summer
  (~48 h).
- Open questions: how to ramp in/out of an "off" (0) month, and whether to
  interpolate between month midpoints or starts.
- Implementation: the EQ + curve isn't a native Lovelace card (custom/HACS or a
  bespoke canvas card). The interval math (interpolated day-of-year → hours) lives
  in a template sensor the scheduler reads as the effective `cut_interval_hours`.

### Validate wettest-zone logic — max vs. average vs. per-zone ⭐
The moisture gate currently uses the **wettest** configured zone. Open question:
is max right, or average, or per-zone (each plan gated on its own zone)? Pull soil
history, compare how each method would have gated real cuts, then decide. Likely
pairs with per-plan/zone mapping.

### Rain-interruption recovery (resume vs. restart) ⭐
When rain stops a cut, later decide whether to **resume the remaining portion** or
**start over**, instead of just letting the next interval catch up.

- Re-dispatch only when onboard rain reads dry (debounced) **and** the forecast
  shows no rain for a configurable window (start 6 h).
- If more than ~6 h (tunable) passed since the stop, start over (grass grew).
- Completion-based choice: < ~50% done → start over; ≥ ~95% → finish the remainder;
  middle → resume.
- **No native progress %.** The SDK exposes no percent-complete read-back, so
  estimate from elapsed run time vs. the plan average
  (`current_job_started` is already recorded). The integration's
  `number.<slug>_plan_start_percent` can start a plan partway in to "finish the bit."

### Dew-point leaf-wetness model ⭐
Pause when there's too much **dew on the grass tops** — distinct from the soil gate
(this is water on the blades, which clumps/clogs/spreads disease). Blades
radiatively cool below air temp on clear, calm nights, so dew can form even when air
temp is above the dew point; it burns off after sunrise.

- Inputs: dew point, RH, wind, cloud cover, sun elevation — preferably from a local
  Ecowitt station; compute dew point via Magnus if not provided. An Ecowitt
  leaf-wetness sensor (WN035) would replace the whole model with a direct reading.
- Model: estimate leaf-surface temp, **accumulate** wetness while it's at/below dew
  point (faster when the gap is wide and wind is low), **dry** it down with sun/heat/
  falling RH. Maintain a persistent leaf-wetness estimate; gate above a threshold
  with resume hysteresis.
- Start simple (a dew-likely on/off heuristic), add the time-integration once the
  basic gate behaves.

### Blackout windows (no-run times) ⭐
User-defined times the robot must not run (overnight, quiet hours, guests over).
Hard gate on auto-dispatch; optionally pause + dock a running job when a window
begins (configurable hard-stop vs. let-finish). Cadence decides *whether* a cut is
due; blackout decides *when in the day* it's allowed.

### Forecast for timing, onboard sensor for the stop ⭐
Today onboard and forecast are OR'd into one block/cancel gate, so a wrong "rainy"
forecast can keep the robot from starting on a dry day. Split the roles: forecast
drives only the predicted start time and the pre-rain pull-ahead; the signal that
blocks a start or stops a running cut requires the **actual onboard sensor**. Add a
short confirm window before a mid-cut stop so a single spurious droplet can't abort
a job.

### Post-rain drying delay — leaf-surface dryness gate ⭐⭐
Soil moisture is a root-zone signal and structurally misses **surface wetness** from
light or late rain — e.g. a dusk drizzle that barely moves the soil sensors but
leaves the blades wet all night. Surface wetness is governed less by rainfall inches
than by **time-of-day + sun + heat** (¼" at 9am/90°F is gone by 11am; the same at
dusk sits until mid-morning).

- Keep both gates: soil (existing) for saturating rain, and a new leaf-surface
  gate requiring N **effective drying hours** after a rain event.
- Accumulator: `needed = base_hours × (amount / ref_amount)`; each hour after
  rain-end contributes `sun_factor × temp_factor × humidity_factor`, where night
  accrues ≈ 0 and overnight dew can re-wet. Gate opens when the running sum clears
  `needed`.
- Scheduling reduces to `next_cut = max(last_cut + interval, rain_end + drying_delay)`,
  with the real-time leaf-dry gate as the backstop.
- Rain amount + end-time are best from a **local rain gauge** (hyperlocal,
  real-time, no API cost); OWM History is a coarse fallback, weakest on exactly the
  light-local-rain case this targets. Shares inputs with the dew model above.

### Multiple weather sources / cross-check ⭐
One provider is a single point of failure for the rain gate (OWM has reported 100%
rain in a dry hour while every other source disagreed, which yanked a predicted cut
36 h early). Add 2–3 selectable sources (NWS, Open-Meteo, AccuWeather all feed
`weather.get_forecasts`) and combine them — consensus/median of hourly
precipitation probability, a veto rule requiring ≥2 to agree, or primary-plus-sanity
-check that suppresses a lone spike. Surface which source currently sees rain so a
bad one is obvious. Even a consensus forecast shouldn't stop a running cut without
the onboard sensor (ties to the split above).

### Ad-hoc "cut now" plan — voice-armed ⭐⭐
A single voice command ("cut the X area now") that runs a specific plan on demand,
obeying every rule of engagement **except** cadence. Deferred deliberately — the
happy path is easy (it rides `cut_blocker`, which already excludes cadence) but the
failure modes need care:

- Pin down exactly which gates an ad-hoc cut must respect (likely all of
  cut_blocker, possibly a different battery floor, and a decision on whether
  master-off overrides or strictly blocks).
- **A pending request must time out.** An armed "cut now" must not silently fire at
  3am on day 3 of rain — needs a configurable max-wait and a clear pending/expired
  state on the dashboard.
- Interaction with the scheduled cut, dock latch, and completion stamping all need
  thought so an ad-hoc fire can't corrupt the cadence clock or fight the scheduler.

---

## Hardware under evaluation

- **Local rain gauge (Ecowitt).** Gold-standard rain amount + end-time for the
  drying-delay model. Candidates pair with an existing Ecowitt gateway: WH40
  (tipping-bucket, most accurate rain), WS85 (piezo, maintenance-free, adds wind),
  WS90 (all-in-one, solar, feeds a full drying model).
- **Leaf-wetness sensor (Ecowitt WN035).** Would replace the dew/leaf-wetness
  estimation with a direct measurement.
