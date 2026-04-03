---
status: resolved
trigger: "Some panels divide power values by 1000.0 and use kW unit instead of using watts directly. Need to check actual InfluxDB data to determine if division is necessary."
created: 2026-04-04T00:00:00Z
updated: 2026-04-04T00:00:00Z
---

## Current Focus

hypothesis: CONFIRMED — Power Profile panel was unnecessarily dividing watt values by 1000 and displaying as kW. Fix applied.
test: Verified fix in JSON — /1000.0 removed from 5 queries, unit changed kwatt→watt, savings panels untouched
expecting: Power Profile panel now shows raw watt values (e.g., 1896 W instead of 1.896 kW); savings panels continue to work correctly
next_action: Human verify in Grafana UI that Power Profile shows sensible watt values

## Symptoms

expected: Panel power values should be displayed in watts (W) without any /1000.0 division if the raw data is already in watts
actual: Some panels divide the raw value by 1000.0 and display as kW, which may be unnecessary if the raw data is already in watts
errors: No runtime errors — this is a data accuracy / unit correctness issue
reproduction: View the Grafana dashboard JSON and compare query transformations against actual InfluxDB data
started: Noticed during dashboard review — unknown if ever correct

## Eliminated

- hypothesis: Raw data might be in milliwatts (so /1000 converts to watts)
  evidence: East Microinverter power_sensor peaks at 1896W, West peaks at 2159W — clearly watts, not mW
  timestamp: 2026-04-04T00:00:00Z

- hypothesis: Raw data might already be in kW (so /1000 is wrong in the other direction)
  evidence: Values like 1896.1, 2159.3, 1439.91, 1062.4, 393.9, 1464.0 — all hundreds-to-thousands range = watts
  timestamp: 2026-04-04T00:00:00Z

- hypothesis: Savings /1000.0 divisions are also wrong
  evidence: Savings queries compute AVG(pv_power_watts) over 1 hour / 1000 = kWh, then * 3.5 THB/kWh — this is mathematically correct energy→savings conversion
  timestamp: 2026-04-04T00:00:00Z

## Evidence

- timestamp: 2026-04-04T00:00:00Z
  checked: East Microinverter "power_sensor" field (all-time)
  found: MAX=1896.1W, AVG=674.9W, MIN=0.9W
  implication: Data is in watts

- timestamp: 2026-04-04T00:00:00Z
  checked: West Microinverter "power_sensor" field (all-time)
  found: MAX=2159.3W, AVG=924.4W, MIN=1.8W
  implication: Data is in watts

- timestamp: 2026-04-04T00:00:00Z
  checked: grid "power" field (all-time)
  found: MAX=3953.16W, AVG=1740.65W, MIN=-1789.25W (negative = backfeed)
  implication: Data is in watts

- timestamp: 2026-04-04T00:00:00Z
  checked: 235_floor_1 "power" field
  found: Latest values ~393-394W
  implication: Data is in watts

- timestamp: 2026-04-04T00:00:00Z
  checked: 235_floor_2 "power" field
  found: Latest values ~1062-1068W
  implication: Data is in watts

- timestamp: 2026-04-04T00:00:00Z
  checked: Smart Meter "power_sensor" field
  found: Latest values ~1450-1468W
  implication: Data is in watts

- timestamp: 2026-04-04T00:00:00Z
  checked: East Microinverter per-panel pv1-pv4 power sensors
  found: MAX per panel ~469W (physically correct for ~460W rated panels)
  implication: Data is in watts; confirms savings /1000 is correct for W→kW→kWh conversion

- timestamp: 2026-04-04T00:00:00Z
  checked: Post-fix state of solar-pv-monitor.json
  found: Power Profile unit=watt, all 5 targets have /1000.0 removed; 6 savings panel targets still retain /1000.0 correctly
  implication: Fix applied correctly, no collateral damage

## Resolution

root_cause: Power Profile panel had all 5 SQL queries dividing watt values by 1000.0 and the panel unit was "kwatt". InfluxDB stores all power measurements in watts (verified: East inverter peaks at 1896W, West at 2159W, grid at 3953W, smart meter at ~1464W, floor clamps at ~394-1068W). The /1000.0 was producing technically correct kW values but was unnecessary and made small values harder to read (e.g., 18.9W displayed as 0.019 kW). Note: savings panel /1000.0 divisions are intentionally correct — they convert W→kW for kWh energy integration (AVG watts × 1 hour / 1000 = kWh).
fix: Removed /1000.0 from all 5 Power Profile SQL targets (Production, Grid, Load, 235 Floor 1, 235 Floor 2) and changed fieldConfig.defaults.unit from "kwatt" to "watt". Savings panel queries (Today/Month/Year Savings) left unchanged.
verification:
files_changed: [solar-pv-monitor.json]
