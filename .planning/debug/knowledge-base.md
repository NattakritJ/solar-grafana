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

## today-savings-nonzero-at-3am — Daily/monthly/yearly aggregate panels show previous day's data at night due to InfluxDB 3 AT TIME ZONE wall-clock bug
- **Date:** 2026-04-04
- **Error patterns:** Today's Savings, THB, 3AM, night, non-zero, savings, date_trunc, AT TIME ZONE, Bangkok, timezone, now(), aggregate, daily, monthly, yearly
- **Root cause:** InfluxDB 3 SQL's `AT TIME ZONE` operator on a no-TZ timestamp (like `now()`) is a wall-clock label operation, not a UTC conversion. `now() AT TIME ZONE 'Asia/Bangkok'` at 20:15 UTC attaches `+07:00` without converting the hour, making the system think Bangkok time is 20:15 instead of 03:15. `date_trunc('day', ...)` then anchors to the wrong midnight — yesterday Bangkok instead of today Bangkok — causing all 14 daily/monthly/yearly aggregate queries to include the prior day's full solar production data even at 3AM.
- **Fix:** Replace all `now() AT TIME ZONE 'Asia/Bangkok'` with `tz(now(), 'Asia/Bangkok')` in every affected query. The `tz()` function treats a no-TZ timestamp as UTC and correctly converts it to the target timezone. Applies to 14 queries across Today's Savings, Month Savings, Year Savings, Today's Energy, Peak Today, Daily Bar Chart, Grid Export Count, Grid Max Export.
- **Files changed:** solar-pv-monitor.json
---

