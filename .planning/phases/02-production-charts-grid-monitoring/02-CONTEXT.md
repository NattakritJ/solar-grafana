# Phase 2: Production Charts & Grid Monitoring - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning

## Phase Boundary

Phase 2 adds time-series production charts, per-inverter comparison, daily/weekly energy bars, and grid quality monitoring to the existing dashboard.

**In scope:** PROD-01, PROD-02, PROD-03, PROD-04, GRID-01, GRID-02
**Not in scope:** PROD-05 (calendar heatmap → Phase 4), module-level, financial, canvas

## Key Design Decisions

### D-16: Time-range macro usage
Phase 2 introduces `$__timeFrom` / `$__timeTo` macros for time-range-aware queries. All time-series and bar chart panels MUST respond to Grafana's time range picker. Pattern:
```sql
WHERE time >= $__timeFrom AND time <= $__timeTo
```

### D-17: Multi-measurement time-series
Since InfluxDB 3 does NOT support `SELECT FROM (subquery with UNION ALL)`, all multi-measurement time-series use **separate Grafana queries** (one per measurement) within the same panel. For `timeseries` panels, Grafana natively overlays multiple query results as separate series — no transformations needed.

### D-18: Time binning for production charts
Use `DATE_BIN(INTERVAL '5 minutes', time)` for time-series production graphs. This provides smooth curves without overwhelming data volume. For daily energy bars, use `DATE_BIN(INTERVAL '1 day', time)` with `date_trunc('day', time)`.

### D-19: Per-inverter comparison approach
- `timeseries` panel with 2 queries (East query A, West query B) overlaid on same chart — fulfills PROD-04
- `bargauge` panel showing current East vs West power side by side — fulfills PROD-02
- This means PROD-01 and PROD-04 share the same panel (total production = sum of overlaid East+West series)

### D-20: Daily energy calculation
Use `MAX(today_production_1_sensor + ... + today_production_4_sensor)` for each day bucket from each inverter. This uses the inverter's built-in daily counter rather than trying to integrate power readings.

For historical daily bars over multiple days, use:
```sql
SELECT DATE_BIN(INTERVAL '1 day', time) AS day,
       MAX(today_production_1_sensor) + MAX(today_production_2_sensor) +
       MAX(today_production_3_sensor) + MAX(today_production_4_sensor) AS daily_kwh
FROM "East Microinverter"
WHERE time >= $__timeFrom AND time <= $__timeTo
GROUP BY 1
ORDER BY 1
```

### D-21: Grid monitoring panel types
- GRID-01: 4 stat panels (Voltage, Frequency, Power Factor, Current) — latest values
- GRID-02: 1 timeseries panel with 4 series (voltage, frequency, power factor, current) OR separate panels for readability. Using separate panels for clarity since units differ.

### D-22: Row placement
- Production panels go under the "Production" row (id=200, y=20) — uncollapse it
- Grid panels go under the "Grid & Consumption" row (id=400, y=22) — uncollapse it

### D-23: Color scheme
- East production: `#FF9830` (orange/amber) — morning sun
- West production: `#5794F2` (blue) — afternoon sun  
- Total production: `#73BF69` (green) — combined
- Grid voltage: `#8AB8FF` (light blue)
- Grid frequency: `#B877D9` (purple)
- Power factor: `#F2495C` (red/pink)
- Grid current: `#FF9830` (orange)

## Canonical References

- `.planning/PROJECT.md` — InfluxDB schema, hardware specs
- `.planning/research/STACK.md` — Panel types, SQL patterns, units
- `.planning/research/PITFALLS.md` — Pitfall 1 (SQL syntax), 8 (performance), 10 (macros)
- `.planning/phases/01-foundation-overview-stats/01-CONTEXT.md` — Phase 1 patterns to follow
- `solar-pv-monitor.json` — Current dashboard with Phase 1 panels

## Existing Code Patterns

From Phase 1 bug fix, all multi-measurement panels use:
- Separate queries per measurement (no UNION ALL)
- `merge` + `reduce` transformations for stat panels combining values
- For `timeseries` panels, no transformations needed — Grafana overlays series natively
