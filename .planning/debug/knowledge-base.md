# GSD Debug Knowledge Base

Resolved debug sessions. Used by `gsd-debugger` to surface known-pattern hypotheses at the start of new investigations.

---

## panel-power-unit-division — Power Profile panel unnecessarily divides watt values by 1000 and displays as kW
- **Date:** 2026-04-04
- **Error patterns:** /1000.0, kwatt, power_sensor, kW, watt, division, unit, Power Profile, timeseries
- **Root cause:** Power Profile panel had all 5 SQL queries dividing watt values by 1000.0 and the panel unit was "kwatt". InfluxDB stores all power measurements in watts (East inverter peaks at 1896W, West at 2159W, grid at 3953W, smart meter at ~1464W, floor clamps at ~394-1068W). The /1000.0 was producing technically correct kW values but was unnecessary. Savings panel /1000.0 divisions are intentionally correct — they convert W→kW for kWh energy integration.
- **Fix:** Removed /1000.0 from all 5 Power Profile SQL targets (Production, Grid, Load, 235 Floor 1, 235 Floor 2) and changed fieldConfig.defaults.unit from "kwatt" to "watt". Savings panel queries (Today/Month/Year Savings) left unchanged.
- **Files changed:** solar-pv-monitor.json
---

