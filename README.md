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
