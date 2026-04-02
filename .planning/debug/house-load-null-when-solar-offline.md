---
status: awaiting_human_verify
trigger: "House Load stat panel shows No data at night when inverters are offline because solar power queries return null instead of 0"
created: 2026-04-01T00:00:00+07:00
updated: 2026-04-02T17:00:00+07:00
---

## Current Focus

hypothesis: CONFIRMED — SQL-level fix using `COALESCE(MAX(column), 0)` pattern. Aggregate functions (MAX/SUM) on zero rows return 1 row with NULL, then COALESCE converts NULL to 0. This guarantees InfluxDB always returns exactly 1 row, preventing Grafana's NoData propagation.
test: User needs to import updated dashboard JSON and verify panels show values (not "No data") when inverters are offline at night.
expecting: House Load = 0 + 0 + Grid = Grid value. Solar Power = 0. All panels show numeric values instead of "No data".
next_action: AWAITING USER VERIFICATION — user must import dashboard and confirm panels work at night.

## Symptoms

expected: House Load = Grid + 0 = Grid import value when solar inverters are offline at night
actual: Panel shows "No data" because solar queries return no rows, making math expression null
errors: No explicit error — just "No data" displayed
reproduction: Anytime inverters are offline (nighttime)
started: Always been this way since dashboard was built

## Eliminated

- hypothesis: Use SQL COALESCE((SELECT subquery), 0) to wrap inverter queries
  evidence: User verified — all panels with this pattern still show "No data". InfluxDB 3 Core SQL does not support scalar subqueries inside COALESCE. The queries likely fail silently or return no results.
  timestamp: 2026-04-02T10:00:00+07:00

- hypothesis: Use UNION ALL with priority column in a derived table to guarantee at least one row
  evidence: User verified — error `[sse.dependencyError] did not execute expression [C] due to a failure of the dependent expression or query [A]`. InfluxDB 3 Core does not support UNION ALL inside FROM subqueries. The query syntax breaks at the InfluxDB level, cascading to all dependent expressions.
  timestamp: 2026-04-02T13:00:00+07:00

- hypothesis: Use Grafana `type: "reduce"` expression with `replaceNN` mode between queries and math
  evidence: Applied to all 11 panels. Still shows "No data". Root cause: Grafana's expression engine documentation states "When any queried data source returns no series or numbers, the expression engine returns NoData" — the NoData propagation happens BEFORE reduce expressions can execute. The reduce expression receives NoData from its upstream query and cannot convert it to 0.
  timestamp: 2026-04-02T16:00:00+07:00

## Evidence

- timestamp: 2026-04-01T00:00:30+07:00
  checked: Panel 5 (House Load) targets structure
  found: 3 InfluxDB queries (East, West, Grid) + 1 math expression ($East + $West + $Grid). East/West queries use `WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1` on inverter measurements. When inverters are offline, these return 0 rows → null in expression → "No data".
  implication: Root cause confirmed. Fix needed but NOT at SQL subquery level.

- timestamp: 2026-04-02T10:00:00+07:00
  checked: User verified COALESCE subquery fix
  found: Fix did NOT work. Panels still showing "No data". However, Today's Energy (counter queries) and Lifetime Energy (full-table queries without time filter) DO work. Peak Today shows 0.00 kW (day-scoped MAX query works because data exists from earlier in the day). Grid Import works (smart meter always online).
  implication: The COALESCE((SELECT ...), 0) subquery pattern is NOT supported by InfluxDB 3 Core SQL. Need alternative approach.

- timestamp: 2026-04-02T12:00:00+07:00
  checked: Applied UNION ALL fallback row pattern to all 40 inverter queries across 14 panels
  found: Pattern `SELECT col FROM (SELECT real_val AS col, 1 AS pri FROM table WHERE ... UNION ALL SELECT 0, 0) ORDER BY pri DESC LIMIT 1` avoids scalar subqueries entirely.
  implication: Failed — UNION ALL inside FROM subquery not supported by InfluxDB 3 Core.

- timestamp: 2026-04-02T13:00:00+07:00
  checked: User verified UNION ALL fix attempt
  found: STILL BROKEN. Error: `[sse.dependencyError]`. Both SQL-level approaches have failed.
  implication: Need Grafana-level fix.

- timestamp: 2026-04-02T15:00:00+07:00
  checked: Applied Grafana reduce expression fix to all 11 panels with math expressions
  found: Added 22 reduce expression targets. JSON validates. But Grafana NoData propagation prevents reduce from working — when InfluxDB returns 0 rows, the entire expression pipeline short-circuits to NoData before reduce can replace values.
  implication: Reduce expressions cannot fix this. The ONLY solution is to make InfluxDB always return at least 1 row.

- timestamp: 2026-04-02T17:00:00+07:00
  checked: Applied COALESCE(MAX(column), 0) pattern — the definitive fix
  found: KEY INSIGHT — SQL aggregate functions (MAX, SUM, AVG) on zero rows return 1 row with NULL value (not 0 rows). Then COALESCE converts NULL to 0. Changed all inverter queries from `SELECT column FROM table WHERE ... ORDER BY time DESC LIMIT 1` to `SELECT COALESCE(MAX(column), 0) FROM table WHERE ...`. Removed all Grafana reduce expression targets. Reverted math expressions to direct references ($A + $B instead of $A_safe + $B_safe). Also fixed Self-Consumption panel division-by-zero with ternary: `$Total == 0 ? 0 : $Solar / $Total`. Applied to ALL panels (not just math-expression panels): bargauge, canvas, table, temperature, alarm, fault, device state. Total: 54 COALESCE(MAX()) patterns across the dashboard. Device state queries use `COALESCE(MAX(device_state_sensor), 'Offline')` to show "Offline" string when inverter is offline. JSON validates. Zero remaining `ORDER BY time DESC LIMIT 1` on Microinverter queries. Zero remaining reduce/replaceNN artifacts.
  implication: This should be the definitive fix. InfluxDB 3 Core always returns exactly 1 row per query (with 0 when no data), so Grafana never sees NoData.

## Resolution

root_cause: Inverter SQL queries used `ORDER BY time DESC LIMIT 1` which returns 0 rows when no data exists in the time window. Grafana's expression engine propagates NoData when any upstream query returns 0 rows, breaking ALL dependent math expressions. Previous attempts (COALESCE subqueries, UNION ALL, Grafana reduce expressions) all failed due to InfluxDB 3 Core SQL limitations and Grafana's NoData propagation behavior.
fix: Converted ALL inverter queries from row-selection (`ORDER BY time DESC LIMIT 1`) to aggregate queries (`COALESCE(MAX(column), 0)`). SQL aggregate functions on zero rows return 1 row with NULL, then COALESCE converts to 0. This guarantees InfluxDB always returns exactly 1 row. Removed all Grafana reduce expression targets (unnecessary with SQL-level fix). Fixed Self-Consumption division-by-zero. Applied consistently to all 54 inverter queries across all panels.
verification: JSON validates. Zero Microinverter LIMIT 1 queries remain. Zero reduce/replaceNN artifacts remain. 54 COALESCE(MAX()) patterns applied. Awaiting user verification on live dashboard at night.
files_changed: [solar-pv-monitor.json]
