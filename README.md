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
- **Fronius dynamic scale factor** — SunSpec `W_SF` changes with power range, so
  AC power is read as raw mantissa (40083) × 10^W_SF (40084) in a template.
- **Proportional curtailment** — targets grid power (default 0 kW) with a 0.1 kW
  deadband; battery *discharge* counts as "need more solar" so it never drains
  the battery to hold export at zero.
- **Autonomous throttle flag** — distinguishes inverter Volt-Watt/Freq-Watt
  throttling from commanded curtailment.

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
