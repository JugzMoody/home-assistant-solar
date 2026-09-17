# Home Assistant — Solar / Battery Control

Home Assistant configuration for a dual-inverter solar site with a Tesla Powerwall,
covering live monitoring, forecast tracking, and price-aware curtailment.

## Hardware
- **Fronius Primo 5** (East array, ~10 yr old) — SunSpec Modbus TCP
- **Sungrow SG5.0RS** (West array) — Modbus TCP
- **Tesla Powerwall 2** — local TEDAPI (paired) + Fleet API via PowerSync
- Amber (dynamic pricing) + Solcast (PV forecast, 2 rooftop sites) via the
  PowerSync integration

## Layout
```
configuration.yaml              Main HA config: Modbus sensors, template
                                sensors/switches, helpers, recorder, statistics,
                                Riemann integration + utility meters.
automations/
  SolarProportionalCurtail.yaml Battery-aware proportional curtailment loop —
                                holds grid export near a target at negative
                                prices by modulating both inverters' power limit.
  SolarManualCurtail.yaml       Manual emergency override (force 0% / restore).
dashboards/
  SolarCurtailmentDashboard.yaml   Live curtailment status, gauges, diagnostics.
  SolcastDashboard.yaml            Forecast vs actual, tracking %, per-inverter health.
  LoggingDashboard.yaml            Activity log (HACS multiple-logbook-card).
  weather_dashboard.yaml           Kitchen weather display (EcoWitt station, portrait
                                   1080x1920, fits exactly with no scrolling).
www/
  bom-radar.html                   Animated BOM rain radar loop. Deploy to
                                   <config>/www/ and reference as
                                   /local/bom-radar.html from an iframe card.
```

## Key concepts
- **Fronius dynamic scale factors** — SunSpec `W_SF`/`DCW_SF` change with power
  range and are read as separate registers, so pairing a stale SF with a fresh
  mantissa gave ~10× spikes. Mitigations:
  - AC power = raw mantissa (40083) × 10^W_SF (40084), then **bounded by DC**
    (AC can't exceed DC input) to reject residual spikes.
  - Per-string DC power = **DC voltage × DC current** (both static-scaled),
    avoiding the dynamic `DCW_SF` entirely.
  - `nan_value: 65535` on the DC voltage/current sensors turns the SunSpec
    "not implemented" (0xFFFF → 655) dusk-dropout reads into `unavailable`.
- **Per-inverter daily energy** — Sungrow exposes a **native daily-yield
  register** (self-resetting, LCD-exact). The Fronius SunSpec map has only a
  *lifetime* counter, so its daily figure is derived as
  `lifetime_now − lifetime_at_midnight` (a 00:00 template snapshot), matching how
  the Fronius LCD itself computes "today".
- **Proportional curtailment** — targets grid power (default 0 kW) with a 0.05 kW
  deadband; battery *discharge* counts as "need more solar" so it never drains
  the battery to hold export at zero. **Asymmetric ramp**: aggressive down (kill
  export fast), gentle up (battery buffers transient load). **5% floor** so the
  Sungrow never hits 0% (which makes it shut down/restart and overshoot). The
  Fronius limiter `Ena` tracks `limit < 100` exactly — leaving it armed at 100%
  makes the inverter report a *phantom* St=5 "Throttled".
- **Autonomous throttle flag** — St=5 while our limiter is OFF (`Ena == 0`)
  distinguishes real Volt-Watt/Freq-Watt throttling from commanded curtailment.

## Solcast site configuration
Two rooftop sites (per-site daily forecast sensors sum to the combined total).
Solcast azimuth convention: **N = 0, E = −90, W = +90, S = ±180**.

| Site | Array | Azimuth | Tilt | DC / AC | Efficiency |
|------|-------|---------|------|---------|-----------|
| East | Fronius, 20 × 270 W | −60 (ENE) | 22° | 5.4 / 5.0 kW | 80% |
| West | Sungrow | +120 (WSW/SW) | 22° | 6.6 / 5.0 kW | 95% (tuned to match observed output) |

Location: Brisbane/Redlands (−27.3169, 153.0355). The East daily forecast is a
valid benchmark; the intraday model over-predicts the ENE **morning** ramp, so
morning-only comparisons read low even on a healthy array.

## Investigation notes (East / Fronius array)
- **Capacity is healthy** — midday actual has exceeded the combined forecast,
  which a degraded array can't do. Both parallel strings work (String 1 has been
  seen > a single string's current).
- **All 20 panels are on MPPT1** as two parallel strings (~17 A in full sun);
  MPPT2 is unused (0 V). The Primo's **max usable input current is 12 A per
  MPPT**, so in strong sun MPPT1 current-limits at ~11.5 A and clips the rest.
  **CONFIRMED (2026-08)** on a clear day: String 1 current holds dead-flat at
  ~11.5 A across midday while the **DC voltage rises from ~275 V to ~330 V** —
  the inverter running off-MPP to enforce the current cap (textbook clipping).
  Loss ~15–30% of the East array in strong sun. Fix = split the strings one per
  MPPT (electrician; DC re-termination) — engaged. Each string ~10 panels,
  ~8.6 A / ~310 V, within the Primo's MPPT current and voltage limits.
- **Independent pvlib clear-sky POA check** (Solcast-independent): on a clear
  moment East ran ~0.78 of physical potential while West ran ~0.99 (same sky,
  same model = West is the control). That points to a **real ~20% East deficit**,
  not just a Solcast artifact — consistent with the two-strings-on-one-MPPT
  mismatch or degradation. Confirm with a full clear-day live run
  (`poa-check.py`) once the daily-tracking fix has a clean day.
- Not shading (clear hilltop east horizon). The Solcast ENE morning-model
  over-prediction is a *separate* forecast-side issue on top of the real deficit.

## Backup reserve / battery control
- **Only ONE optimiser should control the Powerwall reserve.** We chose
  **PowerSync**. **Amber SmartShift** was also enabled and kept re-asserting a
  20% reserve via the Tesla Fleet API, fighting PowerSync's commanded value —
  the "mystery 20%". Disable SmartShift in the **Amber app**; if it re-enables
  itself, revoke Amber's **Tesla control authorization** (keeps the price API).
- PowerSync's reserve **number mirrors the last commanded value**, not a live
  Gateway read (`sync_now` does not refresh it), and the Tesla app display lags
  too — so neither is a trustworthy real-time reserve readout. The reliable
  signal is **battery behaviour** (held/not-discharging while SOC > commanded
  reserve and importing), which is what `automations/ReserveGuard.yaml` acts on.
- Stale PowerSync reserve reads are worsened by **resource contention** on the
  shared HA host — a dedicated host is the intended fix.

## Surplus diversion (soak loads instead of curtailing)
Context: curtailment is frequent (applied inverter limit drops to ~5% on most
sunny days) to avoid negative feed-in. Reconstructed *potential* generation vs
measured showed **~17% of generation lost to curtailment** over a 13-day
late-winter sample (up to 37–44% on the best days), i.e. ~7 kWh/day of surplus
that is currently wasted. Diverting it to a load is worth more than exporting at
a negative price. Priority order for surplus should be:
**battery charge → hot water → A/C pre-cool → curtail only the residual.**

### Heat pump hot water (Rheem, no external controls)
- Only control is the unit's **built-in timer**; there is no comms/PV input, so
  heating cannot be *commanded* — the tank only heats when it is below setpoint.
- Old schedule was **3am–4pm**: the 4pm cutoff existed to avoid peak pricing.
  That cutoff is the root of two problems:
  - **3am is the worst time to heat** — coldest ambient air = lowest COP, and it
    is paid grid import (~19.66c off-peak).
  - **Reheat hysteresis + a fixed cutoff can strand the tank part-charged**: if
    it sits just above the reheat threshold at 4pm it won't cycle, then evening
    showers draw it down and it is locked out until 3am. Rheem declined to add a
    "top up to full before cutoff" behaviour.
- Measured overnight load (00:00–07:00) averages **6.63 kWh/day** with a bump
  from 04:00 peaking 05:00–06:00 — **~2.2 kWh/day above baseline**, attributed
  to the HPWH (cannot be cleanly separated from morning household load without a
  dedicated CT on that circuit). At 19.66c that is roughly **$110–160/yr** of
  grid import that could instead come from curtailed surplus.
- **Change made: timer window shifted to 9am–4pm** so heating lands in the solar
  window at the best COP of the day. Deliberately **no evening reheat window** —
  the choice is to preserve battery capacity for overnight house load rather than
  spend it reheating water.
  - Accepted trade-off: a heavy shower night can leave a cooler tank by morning.
  - If a post-shower top-up is ever wanted, use a **post-peak** slot (after
    8–9pm), not during peak — ~2 kWh at 41.72c would be worse than the 3am run.
- Future option (needs a licensed electrician): **contactor/relay on the HPWH
  circuit** (or a Catch Power Relay) so HA can gate power on *actual* surplus,
  plus a **strap-on tank temp sensor** for visibility. Worth checking the Rheem
  model for a hidden PV/dry-contact input. Caution: do not gate power so
  aggressively that the unit cannot run its periodic **legionella/sanitisation**
  high-temperature cycle.

### HPWH control plan (recommended, not yet implemented)
Constraints confirmed with the vendor/unit: **280 L Rheem heat pump**, threshold
**50 °C**, max **60 °C**, both **locked in firmware** for the energy rating. No
comms/PV input. The built-in timer supports **only ONE start/stop window**, so
"solar window + morning top-up" cannot be expressed with the timer alone.

Physics for this tank: 1 °C ≈ **0.33 kWh thermal**; at ~3 kW thermal that is
**~6.5 min per °C**. A full 40→60 °C reheat ≈ 2.2 kWh electrical.
Standing loss is small (~2 kWh/day ≈ **0.25 °C/hour**), so an idle tank holds
overnight; big overnight drops are USAGE, not loss.

**Price reality (live Amber forecast + 13-day history) — evening reheat is NOT cheaper:**

| Window | Import price |
|---|---|
| 7:30–9 pm | ~31–35 c/kWh (worst) |
| price cliff | ~9:20 pm, drops to ~16 c |
| 1–3 am | ~15.9 c (overnight low) |
| 5 am | ~16.7 c |
| **midday** | **~9 c (cheapest of the day)** |

So: never reheat in the evening peak; a cloudy-day top-up is CHEAPEST at midday,
not overnight. Morning grid top-up ≈ 37c for a full reheat.

**Solar crossover (measured, 12 days mid-Sep):** generation overtakes house load
at **~7:30 am** (median), and surplus reliably exceeds **1 kW at ~8:10-8:25 am**
— i.e. the earliest the ~1 kW compressor can run entirely on solar. Both shift
earlier toward summer; re-measure with `solar-crossover.py`.

**The core insight — park the tank UNDER the 50 °C threshold.** If a morning
grid run takes the tank above 50 °C, the unit will NOT heat when solar arrives,
wasting the free window. So a morning top-up should stop at ~45–46 °C: enough
for the first shower, still below threshold, leaving the rest for free solar.

**Target design (needs a relay + sensor):**
1. Keep the built-in timer set WIDE (e.g. 5 am–4 pm) as a **fail-safe** so a
   HA/relay/network failure can never run the unit in the evening peak. Prefer a
   **normally-closed** relay so an HA outage fails to *powered* (hot water) not
   *off*.
2. **Morning:** time-gated (from ~4:45 am) AND temp < 45 °C -> power on; off at
   ~45–46 °C or a max runtime. Do NOT trigger on temperature alone or it fires
   at 10 pm after the showers.
3. **Solar:** re-enable on measured surplus (>= ~1 kW sustained ~10 min), not
   clock time; let it run to 60 °C for free.
4. **Cloudy fallback:** if still low by ~1 pm, allow grid heating — midday is the
   cheapest grid energy of the day (~9 c).
5. **Guards:** compressor anti-short-cycle lockout is typically 3–10 min after
   power-up (so a 30-min window may only deliver ~20–25 min of heating); add
   min on/off times ~15 min. Ensure periodic full 60 °C cycles for legionella.

**Sensing — outer casing is NOT usable.** The tank is inner steel / PU foam /
outer jacket, so the casing sits near ambient (confirmed by touch); a strap-on
there measures room temperature. Options:
- **Best practical:** DS18B20 cable-tied to the **hot outlet pipe hard against
  the tank, lagged over the top**. Doubles as (a) approximate top-of-tank temp
  and (b) a **draw detector** — the pipe goes hot whenever water flows, so
  summing "hot minutes after 4 pm" quantifies evening usage and can warn at
  9 pm that tomorrow morning will be short. Absolute at-rest reading is soft
  (drifts toward ambient); the draw detection is robust.
- Under-jacket probe: usually inaccessible on a heat pump (compressor on top).
- Do NOT tap the unit's own thermistor (warranty + live control circuit).
- Expect strong **stratification** (community example: 54/45/31 °C at three
  heights on one tank), so thresholds are sensor-position-specific — calibrate
  against the unit's display over several days before trusting automation.

**Recommended hardware:** Shelly Plus 1PM (switching + power monitoring) +
**Shelly Plus Add-On** (galvanically isolated 1-Wire, up to 5 DS18B20) + one
DS18B20 3 m probe. One device does control, energy metering and temperature; it
sits in the enclosure on permanent mains with a native HA integration and no
firmware to compile. ESP32/ESP8266 + ESPHome is the DIY alternative but needs
its own PSU and weatherproofing.
Payback on energy alone is modest (~1.5–2 kWh/day shifted from ~16.7 c grid to
free solar ≈ **$90–110/yr** vs ~$200–350 installed). The real wins are a
guaranteed first shower and using surplus that is currently curtailed.

**LIMITATION of power-monitoring alone:** it CANNOT detect the evening
drawdown — with the unit unpowered overnight there is no compressor signal, so a
heavy-shower night looks identical to an unused one. It also cannot implement a
"stop at 45 °C" rule (the unit only signals at 60 °C, when it stops). Power
metering gives verification, cost, and a reliable "tank now full" signal; the
temperature/draw sensor is what makes the design work.

**MEASURED (2026-09-14): the HPWH is INVISIBLE in whole-house load, and the
3 am start was doing almost nothing.**
20 days of `sensor.power_sync_home_load` show the 3–4 am band flat at
**0.65 kW** — identical to the 2–3 am baseline — despite the timer being set to
3 am. There is a clear step at ~4:20 am rising to ~1.6 kW by 5–6 am, but that is
**household activity, not the compressor**:

| band | weekday | weekend | diff |
|---|---|---|---|
| 3–4 am | 0.65 kW | 0.68 kW | +0.03 (identical) |
| 4–6 am | 1.31 kW | 0.95 kW | **−0.36 (weekends lower)** |

A timer-driven load cannot know it is the weekend, so the 4–6 am rise is people
(showers/kettle/coffee). The 3–4 am band — which *would* show a timer load — is
flat. (An earlier note here hypothesised a ~1h20m timer clock offset; that was
WRONG. The owner verified the clock is accurate, and the weekday/weekend split
explains the pattern. Still worth re-checking the clock occasionally.)

**Why 3 am did nothing:** standing loss is only ~0.25 °C/h, so a tank at 59 °C
at 4 pm is still ~56 °C at 3 am — **above the 50 °C threshold**, so the unit had
no reason to run. It would only fire once the MORNING showers pulled it below
50 °C, i.e. ~6–8 am, which coincides with the ~7:30 am solar crossover. So the
system was probably already getting some solar heating, and a cold-shower
morning is the exception (unusually heavy evening use) rather than the norm.

**Consequence:** a ~0.4–1 kW compressor cannot be separated from household noise
in whole-house data. Per-circuit metering (the Shelly 1PM) is required to know
when the unit actually runs — this is the strongest practical reason to fit it,
ahead of any control benefit.
Reproduce with `hpwh-spike.py` (in `c:\Users\john\Code`, outside this repo).

### Sensibo A/C (integrated — 4 units, all bedrooms)
Integration confirmed live in HA. Entity IDs are **named by room, not by
integration** (a `find sensibo` search returns nothing — search the `climate`
domain instead):

| Entity | Room |
|---|---|
| `climate.bedroom_bedroom` | Bedroom |
| `climate.brianna_brianna` | Brianna |
| `climate.tyler_tyler` | Tyler |
| `climate.cate_cate` | Cate |

- Capabilities: modes `cool` / `heat` / `dry` / `fan_only` / `heat_cool` / `off`;
  target **16–31°C**, 1° step; fan `quiet`…`strong`; swing + horizontal swing.
- Each unit reports **`current_temperature` and `current_humidity`**, so
  automations can close the loop on real room temperature, not just commanded
  state. Also exposed per unit: `switch.*_timer` (+ `sensor.*_timer_end_time`),
  `switch.*_climate_react`, `binary_sensor.*_filter_clean_required`,
  `button.*_reset_filter`, `update.*_firmware`.
- **No A/C in the living areas** (confirmed) — a lounge-room unit may be
  installed later. This matters: pre-cooling pays best in the space occupied
  *during* the 4–9pm peak. With bedroom-only coverage the useful variant is a
  **late-window bedroom pre-cool** (~1pm–4pm) so rooms start the night cooler and
  the overnight A/C draw on the battery is reduced.
- **All four filters currently report `filter_clean_required = on`** (last reset
  2026-01-08). Dirty filters cut airflow and COP — clean them before relying on
  A/C as a soak load, then clear via `button.*_reset_filter`.

Pros of A/C soak: COP ~3–4 (1 kW electrical ≈ 3–4 kW cooling); avoids the
negative-export charge *and* the curtailment loss; no new hardware; preserves
battery for the night by shifting cooling earlier.

Cons / design constraints:
- Thermal mass is small and leaky — holds cool for **hours, not days**; soak late
  in the window or the benefit is gone by evening. Modest setpoints (22–23°C).
- **Coarse dump load** (~0.5–2 kW per head, on/off) — it reduces curtailment but
  cannot finely track export to zero like the Modbus inverter control; the
  residual still needs curtailing.
- **Short-cycling risk**: needs entry hysteresis (only engage once export price is
  negative by a margin), a **15–30 min minimum run time**, and a wider turn-off
  threshold so price wobble doesn't stop/start the compressor.
- **Sensibo is cloud + IR**: command latency and occasional failures; no true
  compressor-state feedback (only what was commanded).
- **Sequencing**: gate the A/C on the same "battery can't absorb more" condition
  that `automations/SolarProportionalCurtail.yaml` already computes, so the two
  automations don't oscillate against each other.
- Snapshot/restore prior climate state when the soak window ends; stagger heads
  so load ramps in steps; set a room-temp floor (~21°C).

## Rain radar / nowcasting (investigated 2026-09-14, not yet implemented)
Goal: show "rain in X minutes" and/or an approaching-rain picture on the kitchen
weather dashboard.

### What is BROKEN
- **`custom:bom-radar-card` (HACS) cannot work.** It fetches
  `https://api.weather.bom.gov.au/v1/radar/capabilities`, which **BOM retired in
  Dec 2024**. Verified: that path returns **HTTP 404**, while other endpoints on
  the same host (e.g. `/v1/locations?search=4017`) return **200 with
  `Access-Control-Allow-Origin: *`**. The browser reports it as a CORS failure
  only because a 404 response carries no CORS headers, which masks the real
  cause. The card was installed and reverted; the windrose was restored.
  Re-confirmed still 404 on 2026-09-16. **Note:** BOM *does* now have a working
  replacement radar service (see below), but the card still cannot be pointed at
  it, because that service restricts CORS to BOM's own origin.
- **OpenWeatherMap minutely nowcast is unavailable to us.** HA's
  `openweathermap.get_minute_forecast` action requires integration mode `v3.0`
  (it fails on `current`/`forecast`). OWM now only sells **One Call API 4.0** to
  new accounts, 3.0 and 4.0 are separate products with different endpoints, and
  **both cannot be active on one account**. Tracked upstream:
  home-assistant/core issue #174333. Revisit if HA adds 4.0 support.

### What still WORKS (verified by direct probing)
BOM's **legacy** radar products are unaffected by the tiled-API removal:

| Resource | Status |
|---|---|
| `http://www.bom.gov.au/radar/IDR663.T.<ts>.png` (frames) | 200, live |
| `.../radar_transparencies/IDR663.background.png` | 200 |
| `.../IDR663.topography.png`, `...locations.png`, `...range.png` | 200 |
| `.../IDR.legend.0.png` | 200 |
| `api.weather.bom.gov.au/v1/radar/*` | **404** |

- **Cadence: every 5 minutes, at minutes ending in 4 or 9** (`:04, :09, :14 ...`),
  i.e. epoch seconds where `(t mod 300) == 240`. Confirmed 9 frames in 75 min.
  (An initial probe on a 6-minute grid found only 2 frames and looked like a
  30-minute cadence - that was **aliasing**, not reality.)
- All layers are **512x512 and pixel-aligned**, so they composite cleanly.
- Frames carry **no CORS header**, which does not matter if HA fetches them
  **server-side** (camera entity). CORS only broke the browser-based card.
- Brisbane (Mt Stapylton) products: `IDR664` 64 km, **`IDR663` 128 km**,
  `IDR662` 256 km, `IDR661` 512 km (earliest warning).

### BOM's NEW radar service (discovered 2026-09-16, WORKS server-side)
BOM's current map viewer
(`https://www.bom.gov.au/weather-and-climate/rain-radar-and-weather-maps`) does
**not** use the retired API. It is backed by a standard **WMTS** service, found by
capturing the page's network traffic:

```
https://api.bom.gov.au/apikey/v1/mapping/timeseries/wmts/1.0.0/
  {layer}/default/{time}/GoogleMapsCompatible_BoM/{TileMatrix}/{TileRow}/{TileCol}.png
```

Capabilities: `.../wmts/1.0.0/WMTSCapabilities.xml` (~86 kB, 41 layers).
Probe script: **`probe-bom-wmts.py`** (outside this repo, in `Code/`).

Four non-obvious gotchas, each of which costs an afternoon if unknown:

| Gotcha | Detail |
|---|---|
| Referer required | 404 without `Referer: https://www.bom.gov.au/`. No API key needed, despite the `/apikey/` path segment. |
| CORS locked to BOM | `Access-Control-Allow-Origin` is the **literal** `https://www.bom.gov.au`, not `*`. Browser-side cards are blocked. **Server-side fetch only.** |
| Time format lies | Capabilities advertise `2026-09-16T22:30:00Z`; the tile endpoint **404s on that** and requires minute precision, `2026-09-16T22:30Z`. |
| Not slippy tiles | Max zoom **8**. `GoogleMapsCompatible_BoM` is cropped to Australia with its own origin (z4 is **3x3**, not 16x16), so indices must come from `TopLeftCorner` + `ScaleDenominator`, not web-mercator slippy maths. |

- Radar frames: **9 frames, 5 min apart** (~40 min history).
- z8 = 156 km per 256 px tile, i.e. **~0.61 km/px** — finer than the legacy
  `IDR662` 256 km product.
- Brisbane (−27.33, 153.07) at z8 = **col 34, row 15**; a 3x3 block around it all
  returns 200.
- Radar tiles are **transparent overlays**. A readable picture also needs the
  basemap from a separate ArcGIS service:
  `.../v1/mapping/basemaps/basemap_default/MapServer/tile/{z}/{y}/{x}`.
- Useful sibling layers: `atm_surf_air_precip_rate_1hr_total_mm_h`,
  `atm_surf_air_precip_accumulation_1hr_total_mm`.
- **Still no nowcast.** All 41 layers were checked: the finest *future*
  precipitation is **3-hourly probability**
  (`atm_surf_air_precip_any_probability_percent_3hourly`, 7 days out). Nothing
  minute-level, so "rain in X minutes" still needs a third party.

Cost to use: a server-side script to set the Referer, stitch 2x2/3x3 tiles and
composite the basemap. Higher quality than anything else here, but the most work.

### Option A - generic camera (free, works today, STATIC)
```yaml
camera:
  - platform: generic
    name: BOM Radar Brisbane
    still_image_url: >-
      {% set ts = as_timestamp(utcnow()) - 120 %}
      {% set slot = (((ts - 240) / 300) | int) * 300 + 240 %}
      http://www.bom.gov.au/radar/IDR663.T.{{ slot | timestamp_custom('%Y%m%d%H%M', false) }}.png
    framerate: 0.2
    verify_ssl: false
```
The 120 s lag keeps us behind BOM's publish time (worst case ~7 min stale).
Verified: the template resolves to a live frame (200, ~3.6 kB).

Radar frames are **transparent overlays** (rain only, no map), so stack them with
a `picture-elements` card: `background` as the base image, then `topography`,
then `camera_image:` for the rain, then `locations` on top - all at
`width: 100%`, `top/left: 50%`.

**Main limitation: it is a STILL image, not a loop** - you cannot tell whether a
band is approaching or receding. Needs a `configuration.yaml` deploy + restart
(camera platforms are not reloadable).

### Option B - Tomorrow.io (free tier) - best for "rain in X minutes"
Core HA integration (`tomorrowio`, successor to ClimaCell), free tier ~500
req/day (25/hour), global so AU is covered; community reports alerts of the form
"precipitation expected within 10 minutes". Caveat: HA's integration
**hard-codes 100 requests/day** even though the free account allows 500, which
limits update frequency; and accuracy relies on their model, not BOM radar.
**Try this first for the numeric nowcast** - no containers, no cost.

### Option C - bom-local-service (free, restores the ANIMATED loop)
`github.com/alexhopeoconnor/bom-local-service` exists specifically to work around
the Dec-2024 BOM breakage by re-serving BOM radar locally. Runs as a container -
viable given Docker already runs on the Synology. This is the option that gets a
real animated radar loop back.

### Option D - paid, mostly NOT viable
- **Weatherzone** has exactly the wanted product ("Future radar": predicted
  precipitation over the next 30 min / 1 h / 2 h from past radar + satellite),
  but it is B2B/enterprise (DTN-owned) with no consumer API tier and no HA
  integration. Effectively unavailable.
- **WillyWeather** ~$1.20/month with an HA integration (safepay) - adds warnings,
  tides, swell, and is the *supported* route to BOM-derived data (BOM's own API
  is not intended for third parties). But it does **not** provide minute-level
  nowcasting, and — checked 2026-09-16 — **it serves no radar imagery at all**.
  The [API feature list](https://www.willyweather.com/info/api.html) is wind,
  tides, rainfall, swell, sunrise/sunset, moon phases and UV. Their *app* shows
  BOM radar, but that is not exposed through the API, and their free website
  widgets are warnings and forecast graphs. **Not a route to a radar picture.**

### Option E - Windy.com iframe (free, no key, IMPLEMENTED)
This is what the dashboard now uses, in place of the windrose.

```
https://embed.windy.com/embed2.html?lat=-27.33&lon=153.07&...&overlay=radar&product=radar
```

- Embeds in the **built-in** `iframe` card. No HACS card, no API key, no
  container.
- Verified: `embed.windy.com` returns 200 with **no `X-Frame-Options` and no CSP
  `frame-ancestors`**, so HA can frame it.
- Windy's Australian radar is the **BOM national composite** (confirmed by Windy
  staff on their community forum), so the source data is still BOM.
- **It self-refreshes.** Observed advancing 9:11 AM -> 9:31 AM unattended, with
  the echo pattern moving. No page reload needed, so it does not go stale.
- It does **not** animate unless play is pressed. On the non-touch kitchen
  display that is unreachable, so treat it as a live still, not a loop.
- `zoom=8` spans Gympie -> Warwick (~0.6 km/px equivalent view); `zoom=7` widens
  it. Tune `aspect_ratio` on the card to keep the 1920px page fit.
- Trade-offs: Windy branding, a national composite rather than the local Brisbane
  product, and a fairly heavy interactive map on an always-on display.

### Option F - DIY BOM loop (IMPLEMENTED — this is what runs now)
`www/bom-radar.html`, served as `/local/bom-radar.html` in an `iframe` card.

A single static HTML file. **No proxy, no container, no Python.** That works
because *displaying* a cross-origin image in an `<img>` needs no CORS — only
reading its pixels via canvas does. Verified from the HA origin that all four
composite layers and every frame load.

| | BOM loop | RainViewer card |
|---|---|---|
| frames | 9–10 | 13 |
| interval | **5 min** | 10 min |
| history | 40 min | **120 min** |
| resolution | **0.5 km/px native, downscaled to 362 px** | ~1.4 km/px, *upscaled* |
| basemap | BOM terrain + coast + dense labels + range rings | Esri grey, sparse |

BOM wins on sharpness and time resolution; RainViewer wins on history length.
The upscaling is why RainViewer echo reads as blocky squares — confirmed by
network trace: the card requests `/256/7/...` at *every* `zoom_level`, so its zoom
only changes magnification, never detail.

Implementation notes worth keeping:
- **Cadence** is 5 min at minutes ending 4 or 9, i.e. `(epoch mod 300) == 240`.
- BOM retains only ~9 frames then **404s cleanly**, so frame discovery is
  "try to load, skip failures" and missing slots are normal.
- **Sequential crossfade, not symmetric.** The incoming frame sits at a higher
  `z-index` and fades in *over* the outgoing one, which is held at full opacity
  until the fade completes. A symmetric crossfade leaves both partly transparent
  mid-transition and dips brightness against BOM's pale basemap.
- The **loop wrap is a hard cut** — frame 0 has the lowest `z-index` so fading it
  in would be invisible, and a snap reads better than time running backwards.
- **Home marker** is computed from BOM's published radar position (Mt Stapylton,
  27.718 °S 153.240 °E) and the HA home coords: 49.0 km NNW, landing at
  42.11 %, 32.56 % of the 128 km image. Stored as percentages so it survives
  scaling, and recomputed if the product/range changes.
- Timestamp badge sits **top-right** because BOM burns its own caption along the
  bottom edge of every frame. Ours shows local time + frame age; BOM's is UTC.
- Tunable by query string, no file edit needed:
  `?prod=IDR664|IDR663|IDR662` (64 / 128 / 256 km), `frames`, `delay`, `fade`,
  `pause`, `marker`, `range`.
- HA sets **`Referrer-Policy: no-referrer`** on `/local/` files. This is another
  reason BOM's new WMTS cannot be used from a `/local/` page even ignoring CORS —
  it *requires* a `Referer: https://www.bom.gov.au/` header and HA strips it.
- No CSP header on `/local/` assets, so the cross-origin BOM images are not
  blocked inside the iframe.

### Recommendation
**Implemented: the DIY BOM loop** (Option F) — sharpest imagery, 5-minute steps,
BOM's own basemap, and no external dependency beyond bom.gov.au itself.
Still outstanding: **"rain in X minutes"** needs a nowcast, which no free source
here provides (BOM's finest future product is 3-hourly probability; RainViewer's
free API returns 0 nowcast frames). **Tomorrow.io** remains the option for that.
**WillyWeather is the wrong product** — it serves no radar imagery at all.

## Dependencies
- Custom integration: **PowerSync** (Tesla/Amber/Solcast orchestration)
- Solcast HA integration (`ha-solcast-solar`)
- HACS frontend cards (all REQUIRED — dashboards render "Custom element doesn't
  exist" without them):
  - `card-mod` — used heavily for readability styling (kitchen weather display).
    NOTE: glance-card internals sit in a nested shadow root, so those styles use
    card-mod's `$` piercing syntax (see comments in `weather_dashboard.yaml`).
  - `apexcharts-card` — Solcast forecast-vs-actual charts
  - `mini-graph-card` — weather trend graphs (temp, lightning activity)
  - `sensor-bar-card-plus` — lightning storm-proximity bar
  - `multiple-logbook-card` — activity log dashboard
    (this is NOT the similarly-named `logbook-card`)
- No longer required (safe to uninstall from HACS):
  - `windrose-card` — was the wind direction rose in row 2b, replaced by the
    Windy radar iframe. Wind speed/direction/gust now live in the Right Now and
    Wind panels instead.
  - `bom-radar-card` — trialled and reverted, cannot work (see Rain radar above).

## Notes
- This is a partial config: `automations.yaml`, `scripts.yaml`, `scenes.yaml`,
  `secrets.yaml` and `themes/` are referenced by `configuration.yaml` but are not
  included here.
- Local LAN IPs are present in `configuration.yaml` (inverters at 192.168.0.x).
  Redact if that's a concern for a public repo.
