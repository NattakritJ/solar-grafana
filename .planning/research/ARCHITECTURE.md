# Architecture Patterns

**Domain:** Grafana Solar Monitoring Dashboard (single comprehensive dashboard)
**Researched:** 2026-03-30
**Confidence:** HIGH (Grafana official docs + InfluxDB 3 Core docs verified)

## Recommended Architecture

A single Grafana dashboard organized into **collapsible row sections**, each containing panels that share a logical domain. The dashboard uses the Grafana 12.4 JSON model with `panels` array, `gridPos` positioning on a 24-column grid, and `templating` for dashboard variables.

### High-Level Dashboard Layout (Top to Bottom)

```
┌─────────────────────────────────────────────────────┐
│  [Header Stats Row - always visible, no collapse]   │
│  Stat panels: Current Power | Today Energy |        │
│  Total Energy | Grid Import | House Load | Savings  │
├─────────────────────────────────────────────────────┤
│  ▼ Row: Solar Production                            │
│  Time series: Total + East/West power over time     │
│  Bar gauge: Today production per inverter            │
├─────────────────────────────────────────────────────┤
│  ▼ Row: Module-Level Monitoring                     │
│  Canvas: Roof layout heatmap (8 panels)             │
│  Table/Stats: Per-PV power, voltage, current        │
├─────────────────────────────────────────────────────┤
│  ▼ Row: Grid & Consumption                          │
│  Time series: Grid import vs solar vs house load    │
│  Stat panels: Grid voltage, frequency, PF           │
├─────────────────────────────────────────────────────┤
│  ▼ Row: Financial Savings                           │
│  Stat panels: Today/monthly savings (THB)           │
│  Time series: Savings over time                     │
├─────────────────────────────────────────────────────┤
│  ▼ Row: Inverter Health                             │
│  Table: Device state, alarms, faults                │
│  Time series: Temperature over time                 │
│  Stat: Power losses                                 │
└─────────────────────────────────────────────────────┘
```

### Component Boundaries

| Component (Row) | Responsibility | Data Sources (Measurements) | Panel Types |
|-----------------|---------------|----------------------------|-------------|
| **Header Stats** | At-a-glance KPIs, always visible | East/West Microinverter + Smart Meter | Stat, Gauge |
| **Solar Production** | Production trends & comparisons | East/West Microinverter (`power_sensor`, `pv_power_sensor`) | Time series, Bar gauge |
| **Module-Level** | Individual panel monitoring + roof visualization | East/West Microinverter (`pv{1-4}_power/voltage/current_sensor`) | Canvas, Table, Stat |
| **Grid & Consumption** | Grid import, house load, grid quality | Smart Meter + derived calculations | Time series, Stat |
| **Financial Savings** | Money saved via TOU rate calculation | Derived from production + TOU rates | Stat, Time series |
| **Inverter Health** | Equipment monitoring & diagnostics | East/West Microinverter (`temperature_sensor`, `device_state/alarm/fault_sensor`) | Table, Time series, State timeline |

### Data Flow

```
InfluxDB 3 Core (SQL)
    │
    ├── "East Microinverter" measurement ──┐
    ├── "West Microinverter" measurement ──┼── Grafana influxdb-solar datasource
    └── "Smart Meter" measurement ─────────┘
                                            │
                            ┌───────────────┤
                            │               │
                    Raw Queries      Calculated Fields
                    (SELECT ...)     (SQL arithmetic in query)
                            │               │
                            └───────┬───────┘
                                    │
                            Panel Visualizations
                            (stat, time series,
                             canvas, table, etc.)
```

**Key principle:** All calculations happen in SQL queries, not in Grafana transformations. This keeps the dashboard JSON portable and avoids Grafana-side processing overhead. The only exception is Canvas panel data binding, which maps query results to visual elements.

## Dashboard JSON Structure

### Top-Level Schema (Grafana 12.4)

```json
{
  "id": null,
  "uid": "solar-monitoring",
  "title": "Solar Monitoring",
  "tags": ["solar", "energy"],
  "timezone": "browser",
  "editable": true,
  "graphTooltip": 1,
  "panels": [],
  "time": { "from": "now-24h", "to": "now" },
  "timepicker": {
    "refresh_intervals": ["5s", "10s", "30s", "1m", "5m"]
  },
  "templating": { "list": [] },
  "annotations": { "list": [] },
  "refresh": "10s",
  "schemaVersion": 39,
  "version": 0,
  "links": []
}
```

**Key JSON fields for this project:**

| Field | Value | Rationale |
|-------|-------|-----------|
| `graphTooltip` | `1` (shared crosshair) | Correlate data across panels when hovering |
| `refresh` | `"10s"` | Match InfluxDB reporting interval (~10s) |
| `time.from` | `"now-24h"` | Default to today's solar production view |
| `timezone` | `"browser"` | Thailand local time (UTC+7) via browser |
| `schemaVersion` | `39` | Grafana 12.4 current schema version |

### Panel Grid Positioning System

Grafana uses a 24-column grid. Each grid height unit = 30 pixels.

**Recommended panel widths for this dashboard:**

| Layout Pattern | Columns | Use For |
|----------------|---------|---------|
| Full width | `w: 24` | Time series charts (production, grid power) |
| Half width | `w: 12` | Side-by-side comparisons (East vs West) |
| Third width | `w: 8` | Stat panels in groups of 3 |
| Quarter width | `w: 6` | Stat panels in groups of 4 |
| Sixth width | `w: 4` | Stat panels in groups of 6 (header KPIs) |

**Row panels** (type `"row"`) act as collapsible section headers:

```json
{
  "type": "row",
  "title": "Solar Production",
  "collapsed": false,
  "gridPos": { "x": 0, "y": 10, "w": 24, "h": 1 },
  "panels": []
}
```

When `collapsed: true`, child panels are nested inside the row's `panels` array. When `collapsed: false`, child panels are at the top level with `gridPos.y` values immediately after the row.

### Panel Target (Query) Structure

Each panel has a `targets` array. For InfluxDB 3 SQL:

```json
{
  "targets": [
    {
      "datasource": {
        "type": "influxdb",
        "uid": "influxdb-solar"
      },
      "rawSql": "SELECT time, power_sensor FROM \"East Microinverter\" WHERE time >= $__timeFrom AND time <= $__timeTo ORDER BY time",
      "refId": "A",
      "format": "time_series"
    }
  ]
}
```

**Critical:** The datasource UID in the JSON must match the configured InfluxDB datasource. Since the project specifies `influxdb-solar` as the datasource name, the UID needs to either match or be parameterized via a datasource variable.

## Query Architecture for InfluxDB 3 SQL

### Grafana Time Range Macros

InfluxDB 3 SQL datasource in Grafana supports these macros for time filtering:

| Macro | Expands To | Usage |
|-------|-----------|-------|
| `$__timeFrom` | Start of dashboard time range | `WHERE time >= $__timeFrom` |
| `$__timeTo` | End of dashboard time range | `WHERE time <= $__timeTo` |
| `$__timeFilter(time)` | Combined time range filter | `WHERE $__timeFilter(time)` |
| `$__timeGroup(time, interval)` | Time bucketing for aggregation | `GROUP BY $__timeGroup(time, '1h')` |
| `$__interval` | Auto-calculated interval | Used in `DATE_BIN` |

**Note (LOW confidence):** InfluxDB 3 Core datasource in Grafana 12.4 may use a SQL query mode that supports `$__timeFilter` and `$__timeGroup` macros similar to other SQL datasources (PostgreSQL, MySQL). However, the exact macro support depends on how the InfluxDB datasource plugin implements SQL mode. Fallback approach: use `$__timeFrom` and `$__timeTo` directly in WHERE clauses.

### Query Patterns

#### Pattern 1: Real-Time Latest Value (for Stat panels)

```sql
SELECT power_sensor
FROM "East Microinverter"
WHERE time >= now() - INTERVAL '5 minutes'
ORDER BY time DESC
LIMIT 1
```

**Use for:** Current power, current temperature, latest grid voltage. Simple, fast query.

#### Pattern 2: Time Series (for Time series panels)

```sql
SELECT
  time,
  power_sensor
FROM "East Microinverter"
WHERE $__timeFilter(time)
ORDER BY time
```

**Use for:** Power production over time. Returns raw data points (~10s intervals). For long time ranges, use aggregation (Pattern 3).

#### Pattern 3: Aggregated Time Series (for long time ranges)

```sql
SELECT
  DATE_BIN(INTERVAL '5 minutes', time, '2024-01-01T00:00:00Z'::TIMESTAMP) AS time,
  AVG(power_sensor) AS avg_power,
  MAX(power_sensor) AS max_power
FROM "East Microinverter"
WHERE $__timeFilter(time)
GROUP BY 1
ORDER BY 1
```

**Use for:** Production trends over days/weeks. `DATE_BIN` is the InfluxDB 3 SQL equivalent of time bucketing. The interval should scale with dashboard time range.

#### Pattern 4: Combined East + West Total Production

```sql
SELECT
  e.time,
  COALESCE(e.power_sensor, 0) + COALESCE(w.power_sensor, 0) AS total_power
FROM "East Microinverter" e
JOIN "West Microinverter" w ON e.time = w.time
WHERE e.time >= $__timeFrom AND e.time <= $__timeTo
ORDER BY e.time
```

**Caution:** JOIN on exact timestamps requires both measurements to report at the same time. If timestamps don't align perfectly, use `DATE_BIN` to bucket both before joining:

```sql
SELECT
  DATE_BIN(INTERVAL '30 seconds', e.time, '2024-01-01T00:00:00Z'::TIMESTAMP) AS time,
  AVG(e.power_sensor) + AVG(w.power_sensor) AS total_power
FROM "East Microinverter" e
JOIN "West Microinverter" w
  ON DATE_BIN(INTERVAL '30 seconds', e.time, '2024-01-01T00:00:00Z'::TIMESTAMP)
   = DATE_BIN(INTERVAL '30 seconds', w.time, '2024-01-01T00:00:00Z'::TIMESTAMP)
WHERE e.time >= $__timeFrom AND e.time <= $__timeTo
GROUP BY 1
ORDER BY 1
```

**Alternative (simpler, recommended):** Use UNION ALL with Grafana's "Merge" transformation, or use two separate queries (refId A and B) in the same panel and let Grafana render as separate series, then use a "Reduce + Merge" transformation to sum.

**Best approach for this project:** Use separate queries per inverter and let Grafana handle stacking/combining visually. This avoids complex JOINs and works reliably regardless of timestamp alignment.

#### Pattern 5: Today's Energy Production (cumulative)

```sql
SELECT
  MAX(today_production_1_sensor) + MAX(today_production_2_sensor) +
  MAX(today_production_3_sensor) + MAX(today_production_4_sensor) AS today_energy_kwh
FROM "East Microinverter"
WHERE time >= now() - INTERVAL '5 minutes'
```

**Use for:** Today's total kWh per inverter. The `today_production_{1-4}_sensor` fields are running counters that reset daily, so `MAX` over recent data gives the current day's total.

#### Pattern 6: Financial Savings with TOU Rates

```sql
SELECT
  DATE_BIN(INTERVAL '1 hour', time, '2024-01-01T00:00:00Z'::TIMESTAMP) AS time,
  AVG(e.power_sensor + w.power_sensor) / 1000.0 AS avg_power_kw,
  CASE
    WHEN EXTRACT(DOW FROM time) IN (0, 6) THEN 2.6369
    WHEN EXTRACT(HOUR FROM time) >= 9 AND EXTRACT(HOUR FROM time) < 22 THEN 5.7982
    ELSE 2.6369
  END AS rate_thb_kwh
FROM "East Microinverter" e
JOIN "West Microinverter" w ON e.time = w.time
WHERE e.time >= $__timeFrom AND e.time <= $__timeTo
GROUP BY 1
ORDER BY 1
```

**Complexity note:** TOU calculation is the most complex query because it requires:
1. Combining production from both inverters
2. Time-of-use rate determination based on hour and day-of-week
3. Converting power (W) to energy (kWh) over time intervals
4. Multiplying energy × rate for monetary value

**Recommended approach:** Break this into Grafana transformations or use a simpler approach — calculate total daily production per rate period, multiply by rate. This may need a Grafana expression or transformation rather than a single SQL query.

#### Pattern 7: Module-Level Power (for Canvas/Table)

```sql
SELECT
  'East PV1' AS panel_name,
  pv1_power_sensor AS power,
  pv1_voltage_sensor AS voltage,
  pv1_current_sensor AS current
FROM "East Microinverter"
WHERE time >= now() - INTERVAL '5 minutes'
ORDER BY time DESC
LIMIT 1

UNION ALL

SELECT
  'East PV2' AS panel_name,
  pv2_power_sensor AS power,
  pv2_voltage_sensor AS voltage,
  pv2_current_sensor AS current
FROM "East Microinverter"
WHERE time >= now() - INTERVAL '5 minutes'
ORDER BY time DESC
LIMIT 1
-- ... repeat for all 8 panels
```

**Alternative:** Use 8 separate queries (one per PV) for the Canvas panel, each bound to a specific visual element. This is simpler and how Canvas data binding typically works — each data source query maps to a specific element.

## Canvas Panel Architecture (Roof Layout)

### Approach: Static Layout + Dynamic Data Binding

The Canvas panel in Grafana 12.4 is a free-form layout panel where you place elements at specific coordinates and bind data to them.

**Architecture for roof layout visualization:**

```
Canvas Panel (full width, ~400px height)
├── Background: Roof outline (rectangle elements or image)
├── East Row Label (text element)
│   ├── PV3 Rectangle (data-bound: color = power level)
│   ├── PV4 Rectangle (data-bound: color = power level)
│   ├── PV1 Rectangle (data-bound: color = power level)
│   └── PV2 Rectangle (data-bound: color = power level)
├── West Row Label (text element)
│   ├── PV3 Rectangle (data-bound: color = power level)
│   ├── PV4 Rectangle (data-bound: color = power level)
│   ├── PV1 Rectangle (data-bound: color = power level)
│   └── PV2 Rectangle (data-bound: color = power level)
└── Legend (text elements showing color scale)
```

**Data binding:** Each rectangle element is bound to a query result field. The Canvas panel supports:
- **Color** based on field value (thresholds: green=high production, yellow=medium, red=low/zero)
- **Text overlay** showing the power value on each panel rectangle
- **Fixed positioning** (x, y coordinates relative to canvas)

**Canvas JSON structure (simplified):**

```json
{
  "type": "canvas",
  "options": {
    "root": {
      "elements": [
        {
          "type": "rectangle",
          "name": "East PV1",
          "config": {
            "color": { "field": "east_pv1_power", "fixed": "" },
            "text": { "field": "east_pv1_power" }
          },
          "placement": { "left": 200, "top": 50, "width": 80, "height": 120 }
        }
      ]
    }
  }
}
```

**Build complexity:** HIGH. The Canvas panel requires precise manual positioning of elements in the UI or careful JSON construction. Each element needs field-to-data mapping. Recommend building this section last.

## Variable/Template Architecture

### Recommended Variables

For this single-dashboard project, variables serve two purposes: making queries reusable and enabling user control.

| Variable | Type | Purpose | Value |
|----------|------|---------|-------|
| `datasource` | datasource | Allow datasource switching | Default: `influxdb-solar` |

**Note:** For a single fixed installation, extensive templating is unnecessary. The dashboard hardcodes the two inverter names and smart meter since they won't change. Don't over-engineer with variables for a single-home system.

### What NOT to Templatize

- Inverter names (fixed: "East Microinverter", "West Microinverter")
- Panel count (fixed: 4 per inverter)
- TOU rates (fixed, change very rarely — hardcode in queries)
- Measurement names (fixed: 3 measurements)

## Patterns to Follow

### Pattern 1: Row-Based Section Organization

**What:** Group related panels under collapsible row headers.
**When:** Always — every logical section gets a row.
**Why:** Users can collapse sections they don't need, reducing scroll. Grafana's JSON model naturally supports this.

```json
{ "type": "row", "title": "Solar Production", "collapsed": false, "gridPos": { "x": 0, "y": 10, "w": 24, "h": 1 } }
```

### Pattern 2: Stat Panel Header Row

**What:** Top row of 4-6 Stat panels showing key metrics at a glance (no row header, always visible).
**When:** Dashboard loads — user immediately sees system status.
**Why:** Most important information first. Stat panels are compact and highly readable.

### Pattern 3: Separate Queries per Series

**What:** Use multiple queries (refId A, B, C) in a single time series panel rather than complex JOINs.
**When:** Comparing East vs West production, or overlaying solar + grid.
**Why:** Simpler queries, each series independently labeled, no timestamp alignment issues.

```json
{
  "targets": [
    { "refId": "A", "rawSql": "SELECT time, power_sensor AS \"East\" FROM \"East Microinverter\" WHERE $__timeFilter(time) ORDER BY time" },
    { "refId": "B", "rawSql": "SELECT time, power_sensor AS \"West\" FROM \"West Microinverter\" WHERE $__timeFilter(time) ORDER BY time" }
  ]
}
```

### Pattern 4: Shared Crosshair for Correlation

**What:** Set `graphTooltip: 1` at dashboard level.
**When:** Always.
**Why:** Hovering over one time series panel shows crosshair on all panels at the same timestamp, enabling visual correlation between production, grid import, and temperature.

### Pattern 5: Value Mappings for State Fields

**What:** Use Grafana value mappings to convert numeric `device_state_sensor` values to human-readable text.
**When:** Inverter health section.
**Why:** Raw integer state codes are meaningless to users.

```json
"mappings": [
  { "type": "value", "options": { "0": { "text": "Standby", "color": "blue" } } },
  { "type": "value", "options": { "1": { "text": "Normal Operation", "color": "green" } } },
  { "type": "value", "options": { "2": { "text": "Alarm", "color": "orange" } } },
  { "type": "value", "options": { "3": { "text": "Fault", "color": "red" } } }
]
```

## Anti-Patterns to Avoid

### Anti-Pattern 1: Complex JOINs for Combined Metrics

**What:** Joining East and West Microinverter measurements on exact timestamps.
**Why bad:** Timestamps from different devices may not align to the nanosecond. JOINs will produce NULL or missing rows.
**Instead:** Use separate queries per panel with multi-query panels, or use `DATE_BIN` aggregation to align timestamps before joining.

### Anti-Pattern 2: Using Flux or InfluxQL Syntax

**What:** Writing queries in Flux (`from(bucket:...)`) or InfluxQL (`SELECT mean("field") FROM...`).
**Why bad:** InfluxDB 3 Core only supports SQL. Flux and InfluxQL will fail.
**Instead:** Use standard SQL with InfluxDB 3 extensions (`DATE_BIN`, `$__timeFilter`).

### Anti-Pattern 3: Excessive Dashboard Variables

**What:** Creating variables for inverter names, field names, panel counts, etc.
**Why bad:** This is a single fixed installation. Variables add query complexity and confuse future maintenance for no benefit.
**Instead:** Hardcode known values. Only use variables where the user needs to select/filter.

### Anti-Pattern 4: Client-Side Calculations via Transformations for Everything

**What:** Using Grafana transformations to compute derived metrics (total production, house load, savings).
**Why bad:** Transformations break when exported/imported, add client-side processing, and are harder to debug than SQL.
**Instead:** Do calculations in SQL queries whenever possible. Use transformations only when SQL can't express the logic (e.g., Canvas data binding).

### Anti-Pattern 5: Single Monolithic Query per Section

**What:** One massive query that returns all fields for an entire section.
**Why bad:** Poor performance, difficult to debug, panel-specific formatting becomes impossible.
**Instead:** One focused query per panel or per series within a panel.

### Anti-Pattern 6: No Row Organization

**What:** Dumping all panels flat without row separators.
**Why bad:** 20+ panels without sections creates an overwhelming wall of charts. Users can't find what they need.
**Instead:** Every logical section gets a collapsible row header.

## Build Order Recommendations

Based on dependencies and complexity:

### Phase 1: Foundation (build first)
1. **Dashboard skeleton** — JSON with metadata, time settings, refresh, `graphTooltip`
2. **Header stat panels** — Current power (East+West), today energy, grid import
3. **Basic production time series** — East and West power over time

**Rationale:** Validates InfluxDB connection, query syntax, and basic panel rendering. Every subsequent section depends on these patterns working.

### Phase 2: Core Production
4. **Solar Production row** — Expanded production charts, East vs West comparison, daily production bars
5. **Grid & Consumption row** — Grid import time series, house load calculation, grid quality stats

**Rationale:** Core monitoring functionality. House load = solar + grid import (for zero-export system), so it depends on production queries working.

### Phase 3: Detail Sections
6. **Module-Level row** — Per-PV stats table, individual panel power/voltage/current
7. **Inverter Health row** — Temperature, state, alarms, faults, power losses

**Rationale:** More panels, more queries, but patterns are already established from Phase 1-2.

### Phase 4: Advanced Features
8. **Financial Savings row** — TOU rate calculations, daily/monthly savings
9. **Canvas roof layout** — Visual panel layout with production heatmap

**Rationale:** Financial calculations require the most complex SQL (TOU logic). Canvas panel requires the most JSON complexity (element positioning, data binding). Both build on patterns from earlier phases but add significant new complexity.

### Phase 5: Polish
10. **Thresholds & color coding** — Green/yellow/red across all panels
11. **Tooltips, descriptions, units** — Panel descriptions, unit formatting (W, kWh, THB, °C, V, Hz)
12. **Final testing & JSON export** — Verify import on clean Grafana instance

### Dependency Graph

```
Dashboard Skeleton
    │
    ├── Header Stats ──────────┐
    │       │                  │
    │   Production Row    Grid & Consumption Row
    │       │                  │
    │   Module-Level Row  Inverter Health Row
    │       │                  │
    │   Canvas Layout     Financial Savings Row
    │                          │
    └──── Polish & Export ─────┘
```

## Panel Type Selection Guide

| Visualization Need | Panel Type | When to Use |
|-------------------|-----------|-------------|
| Single current value | `stat` | Current power, today energy, grid voltage |
| Gauge with min/max | `gauge` | Power as % of capacity (5.24 kW max) |
| Value over time | `timeseries` | Production curves, temperature trends |
| East vs West bars | `barchart` | Daily production comparison |
| Compact value bar | `bargauge` | Per-inverter production comparison |
| Tabular data | `table` | Module-level details (8 PVs with power/voltage/current) |
| Custom spatial layout | `canvas` | Roof panel layout with heatmap |
| State over time | `state-timeline` | Inverter operating state history |
| Categorized state | `statushistory` | Alarm/fault history across inverters |

## Scalability Considerations

| Concern | Current (single home) | Future consideration |
|---------|----------------------|---------------------|
| Query volume | ~20 queries per dashboard refresh | No concern at 10s refresh |
| Data volume | 3 measurements, ~10s interval | ~8.6K points/day per measurement, InfluxDB handles easily |
| Dashboard size | 25-35 panels | Within Grafana's comfortable range |
| JSON file size | ~50-100 KB estimated | Well within Grafana import limits |
| Time range queries | Raw data fine for 24h-7d | For 30d+, use `DATE_BIN` aggregation |

## Sources

- Grafana v12.4 Dashboard JSON Model: https://grafana.com/docs/grafana/v12.4/dashboards/build-dashboards/view-dashboard-json-model/ (HIGH confidence)
- Grafana v12.4 Canvas Panel: https://grafana.com/docs/grafana/v12.4/panels-visualizations/visualizations/canvas/ (HIGH confidence)
- Grafana v12.4 Variables: https://grafana.com/docs/grafana/v12.4/dashboards/variables/ (HIGH confidence)
- InfluxDB 3 Core SQL Query: https://docs.influxdata.com/influxdb3/core/query-data/sql/ (HIGH confidence)
- InfluxDB 3 Core SQL Reference (DATE_BIN, aggregates): https://docs.influxdata.com/influxdb3/core/reference/sql/ (HIGH confidence)
- Grafana v12.4 Dashboard Creation: https://grafana.com/docs/grafana/v12.4/dashboards/build-dashboards/create-dashboard/ (HIGH confidence)
- Grafana InfluxDB Datasource Macros — based on training data, macro support in SQL mode needs validation (LOW confidence for exact macro syntax)
