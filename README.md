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
- HACS frontend cards: `apexcharts-card`, `logbook-card`, `card-mod`

## Notes
- This is a partial config: `automations.yaml`, `scripts.yaml`, `scenes.yaml`,
  `secrets.yaml` and `themes/` are referenced by `configuration.yaml` but are not
  included here.
- Local LAN IPs are present in `configuration.yaml` (inverters at 192.168.0.x).
  Redact if that's a concern for a public repo.
