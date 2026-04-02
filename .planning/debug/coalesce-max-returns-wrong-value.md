---
status: awaiting_human_verify
trigger: "COALESCE(MAX(power_sensor), 0) returns 558, but the most recent value is 50. MAX without COALESCE returns 50 (the most recently seen value)."
created: 2026-04-02T18:00:00+07:00
updated: 2026-04-02T18:45:00+07:00
---

## Current Focus

hypothesis: CONFIRMED — SQL-level fix using `COALESCE(MAX(column), 0)` pattern. Aggregate functions (MAX/SUM) on zero rows return 1 row with NULL, then COALESCE converts NULL to 0. This guarantees InfluxDB always returns exactly 1 row, preventing Grafana's NoData propagation.
test: User needs to import updated dashboard JSON and verify panels show values (not "No data") when inverters are offline at night.
expecting: House Load = 0 + 0 + Grid = Grid value. Solar Power = 0. All panels show numeric values instead of "No data".
next_action: AWAITING USER VERIFICATION — user must import dashboard and confirm COALESCE(selector_last()) returns the current value (50, not the old peak 558) and panels still show 0 at night.

## Symptoms

expected: COALESCE(MAX(power_sensor), 0) should return the same value as MAX(power_sensor) when data exists — i.e., 50 (the most recent value in the last 2 minutes)
actual: COALESCE(MAX(power_sensor), 0) returns 558, which appears to be a stale/old maximum instead of the most recent value. The two queries tested by the user differed in ORDER BY as well, but that is a red herring — the core issue is MAX() vs LAST().
errors: No error — just wrong numeric result
reproduction: In the last 2 minutes Smart Meter power readings went from ~558W down to ~50W. COALESCE(MAX()) returns 558 (the peak in window). ORDER BY time DESC LIMIT 1 returns 50 (the latest).
started: Applied after house-load-null-when-solar-offline fix converted all inverter queries from ORDER BY time DESC LIMIT 1 to COALESCE(MAX(), 0)

## Eliminated

- hypothesis: ORDER BY '' (empty string literal) in Query A causes WHERE time filter to be ignored, resulting in full-table MAX scan
  evidence: The actual dashboard queries (Smart Meter in House Load, Self-Consumption) do NOT have ORDER BY at all — they are pure SELECT COALESCE(MAX(power_sensor), 0) FROM "Smart Meter" WHERE time >= now() - INTERVAL '2 minutes'. The ORDER BY difference in the user's reproduction queries was incidental to the real bug. Both queries execute within the time-filtered window. The WHERE clause is applied correctly.
  timestamp: 2026-04-02T18:15:00+07:00

- hypothesis: COALESCE changes query execution plan in InfluxDB 3 Core, causing the time filter to be bypassed
  evidence: COALESCE is a scalar NULL-handling function. It operates on the output of MAX(), not on the WHERE clause. The WHERE filter is applied first, then MAX() aggregates the filtered rows, then COALESCE handles the aggregate result. COALESCE cannot affect filter execution. Verified: if COALESCE bypassed the filter, we'd see values from days ago (much higher), but 558 is consistent with a reading from ~2 minutes ago in the window.
  timestamp: 2026-04-02T18:15:00+07:00

## Evidence

- timestamp: 2026-04-02T18:00:00+07:00
  checked: Prior debug session (house-load-null-when-solar-offline.md)
  found: The previous fix replaced all `ORDER BY time DESC LIMIT 1` queries with `COALESCE(MAX(column), 0)`. The logic was: aggregate functions return 1 row with NULL when no data exists → COALESCE converts NULL to 0 → NoData prevention. This is correct for the NoData problem but introduced a new semantic error.
  implication: MAX() was chosen as a vehicle for the aggregate behavior, NOT for its "maximum" semantics. The intent was to get "current value" but MAX is incorrect for that purpose.

- timestamp: 2026-04-02T18:05:00+07:00
  checked: Dashboard JSON — all Smart Meter queries
  found: House Load (refId=Grid), Self-Consumption (refId=Grid) use COALESCE(MAX(power_sensor), 0) with 2-minute window. Grid Import, 🏠 House Load, ⚡ Grid → House use ORDER BY time DESC LIMIT 1 (the old correct pattern, not yet migrated). This inconsistency confirms some panels work correctly (LIMIT 1) while others show stale MAX.
  implication: The fix was applied inconsistently — only some Smart Meter panels got the COALESCE(MAX) treatment.

- timestamp: 2026-04-02T18:10:00+07:00
  checked: All COALESCE(MAX()) queries on 2-minute window across entire dashboard
  found: 40+ queries use COALESCE(MAX(col), 0) on 2-minute windows across: Solar Power, House Load, Self-Consumption, ☀ Solar → House, 🏠 House Load, Inverter Power, Panel Power (8), Module Detail (8×6 columns), Roof Layout (8), East/West Temperature, East/West State/Alarm/Fault. All have the same semantic error: they return PEAK in window, not MOST RECENT.
  implication: Bug affects all "current snapshot" panels. When power is declining, ALL panels show stale elevated values. Critical for Smart Meter (power can drop 500W+ in 2 minutes) and inverters (afternoon ramp-down shows stale high values).

- timestamp: 2026-04-02T18:15:00+07:00
  checked: InfluxDB 3 Core SQL docs — selector_last function
  found: selector_last(expression, timestamp) returns the LAST value ordered by time ascending (i.e., the most recent). Returns a struct with ['time'] and ['value'] properties. Returns NULL when no rows in group → COALESCE can wrap it for NoData protection. Syntax: COALESCE(selector_last(col, time)['value'], 0). This is officially documented at docs.influxdata.com/influxdb3/core/reference/sql/functions/selector/
  implication: COALESCE(selector_last(col, time)['value'], 0) is the correct pattern: gets most recent value AND falls back to 0 when offline. Directly replaces COALESCE(MAX(col), 0) for current-snapshot queries.

- timestamp: 2026-04-02T18:20:00+07:00
  checked: Queries to NOT change (intentional MAX usage)
  found: Today's Energy uses MAX(today_production_N_sensor) over TODAY window — correct, these are monotonic day-counters where MAX = current total. Lifetime Energy uses MAX(total_production_N_sensor) over ALL time — correct, same reason. Peak Today uses MAX(power_sensor) over TODAY — correct, explicitly wants the day's peak. Daily Production Heatmap uses AVG() in time buckets — correct, not affected.
  implication: Only the 2-minute window queries need to change. Day/all-time MAX queries are semantically correct.

## Resolution

root_cause: MAX() is an aggregate function that returns the HIGHEST value in the time window, not the MOST RECENT value. When power decreases within the 2-minute window (e.g., a large load turns off), COALESCE(MAX()) returns the old high reading (peak) instead of the current reading (last). This is a semantic mismatch: the query intent is "what is the current power right now?" but MAX answers "what was the peak power in the last 2 minutes?". COALESCE itself is correct and not at fault. The prior fix (house-load-null-when-solar-offline) correctly identified that aggregates guarantee a row even with no data, but chose the wrong aggregate — MAX instead of selector_last.
fix: Replace MAX(col) with selector_last(col, time)['value'] in all current-snapshot (2-minute window) queries. Full pattern: COALESCE(selector_last(col, time)['value'], 0). This returns the most recent value (correct semantics) AND returns NULL when no data exists (which COALESCE converts to 0, preserving NoData protection). Applies to all ~40 affected queries. Day/all-time MAX queries are left unchanged as they are semantically correct.
verification: Applied to solar-pv-monitor.json. 48 queries transformed: COALESCE(MAX(col), fallback) → COALESCE(selector_last(col, time)['value'], fallback). 6 intentional MAX queries (Today's Energy, Lifetime Energy, Peak Today) preserved unchanged. JSON validates. Awaiting user verification in live Grafana.
files_changed: [solar-pv-monitor.json]
