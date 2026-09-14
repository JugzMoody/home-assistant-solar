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

**MEASURED ANOMALY (2026-09-14): the timer clock appears to be ~1h20m late.**
With the timer believed set to 3 am, 20 days of `sensor.power_sync_home_load`
show the 3–4 am band flat at **0.65 kW** — identical to the 2–3 am baseline. The
compressor step actually appears at **~4:20 am** (0.61 -> 1.02 kW), on 13 of 20
days between 04:10 and 04:50. Either the unit's internal clock is wrong/drifted,
or it was not below threshold until then (less likely — a thermostat-driven
start would scatter more). **Verify the unit's displayed clock before trusting
any timer setting**: a "5 am" setting may really start ~6:20 am, too late for the
first shower. Two days (Aug 30, Sep 13) show no morning run at all, consistent
with the tank still being above 50 °C after light usage.
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
  - `windrose-card` — wind direction history rose
  - `sensor-bar-card-plus` — lightning storm-proximity bar
  - `multiple-logbook-card` — activity log dashboard
    (this is NOT the similarly-named `logbook-card`)

## Notes
- This is a partial config: `automations.yaml`, `scripts.yaml`, `scenes.yaml`,
  `secrets.yaml` and `themes/` are referenced by `configuration.yaml` but are not
  included here.
- Local LAN IPs are present in `configuration.yaml` (inverters at 192.168.0.x).
  Redact if that's a concern for a public repo.
