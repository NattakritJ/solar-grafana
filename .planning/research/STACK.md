# Technology Stack

**Project:** Solar Monitoring Grafana Dashboard
**Researched:** 2026-03-30

## Recommended Stack

### Core Platform (Already Running — No Changes Needed)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Grafana OSS | 12.4.1 | Dashboard visualization platform | Already deployed; latest stable version with full panel type support |
| InfluxDB 3 Core | Latest | Time-series data storage | Already deployed at 192.168.2.10:8181; SQL query language (NOT Flux/InfluxQL) |
| InfluxDB datasource | Built-in (Grafana core) | Connect Grafana to InfluxDB | `influxdb-solar` datasource already configured; supports SQL query mode for InfluxDB v3 |

### Grafana Panel Types — Recommended for Solar Monitoring

All panel types below are **built-in to Grafana 12.4.1** — no plugins needed.

| Panel Type (`type` in JSON) | Use Case in This Dashboard | Why This Panel | Confidence |
|---|---|---|---|
| `timeseries` | Production over time, grid import trends, inverter temperature history | The workhorse of time-series visualization. Supports multiple series overlay, fill below, thresholds, and gradient coloring. Perfect for showing East vs West production curves over a day. | HIGH |
| `stat` | Current total power (W), today's energy (kWh), total savings (THB), system capacity | Big single-number readout with optional sparkline. Ideal for "at a glance" KPIs like "2,450W right now" or "23.5 kWh today". Supports color thresholds (green=producing, gray=night). | HIGH |
| `gauge` | Current power as % of system capacity (5,240W), power factor, grid voltage deviation | Circular/bar gauge showing a value relative to min/max. Perfect for "how close to max capacity" and "is grid voltage in normal range". | HIGH |
| `barchart` | Daily/weekly/monthly energy production comparison, per-panel production ranking | Native bar chart for categorical or time-bucketed comparisons. Better than time series for "which panel produced most today" or "daily kWh this week". | HIGH |
| `table` | Module-level detail (all 8 panels: power, voltage, current), inverter fault/alarm status | Data table with color-coded cells. Shows all panel metrics side by side. Cell coloring highlights underperforming panels. | HIGH |
| `canvas` | **Roof layout visualization** — physical panel positions with data-driven colors | Free-form layout where elements (rectangles/text/icons) can be positioned absolutely and bound to query data. **This is the key panel for the roof layout heatmap.** Place 8 rectangle elements representing physical panel positions, color them by production. | HIGH |
| `piechart` | East vs West production split, peak vs off-peak energy distribution | Donut/pie chart for proportional breakdowns. Good for "65% East, 35% West" at a glance. | HIGH |
| `text` | Dashboard section headers, rate information, system description | Markdown/HTML text panel. Use for organizing dashboard sections ("Solar Production", "Grid Status", "Financials"). | HIGH |
| `state-timeline` | Inverter state history (producing/standby/fault/alarm), device state over time | Horizontal bars showing state transitions over time. Perfect for "when were inverters producing vs idle". | MEDIUM |
| `bargauge` | Per-panel current power comparison (8 horizontal bars) | Horizontal bar gauges stacked vertically. Compact visual comparing 8 individual panel outputs at a glance. | HIGH |

### Panel Types — NOT Recommended

| Panel Type | Why NOT to Use |
|---|---|
| `heatmap` | Designed for histogram/distribution data, not spatial layout. Canvas panel is the right choice for roof layout heatmap. |
| `histogram` | Statistical distribution — not useful for solar monitoring's continuous time series. |
| `geomap` | Requires geospatial coordinates. Roof panel layout is better served by Canvas. |
| `traces` / `logs` / `flamegraph` | Observability-specific panels. Not applicable to solar monitoring. |
| `news` | RSS feed panel. Not relevant. |
| `nodeGraph` | Network topology visualization. Not applicable. |
| `datagrid` | Editable spreadsheet. We need read-only display. |

### Dashboard Structural Elements

| Element | Purpose | Notes |
|---------|---------|-------|
| **Row panels** | Section headers: "Overview", "Per-Inverter", "Module Level", "Grid & Consumption", "Financial Savings", "System Health" | Collapsible rows organize the single-dashboard layout |
| **Dashboard variables** | Time range, auto-refresh interval | Use `$__from` and `$__to` for time-range-aware SQL queries; 10s auto-refresh matches data interval |
| **Annotations** | Mark sunrise/sunset, fault events | Optional enhancement using Grafana annotation queries |

---

## InfluxDB 3 Core SQL Query Patterns

**CRITICAL: InfluxDB 3 Core uses SQL (Apache DataFusion SQL). NOT Flux. NOT InfluxQL.**

The Grafana InfluxDB datasource supports a "SQL" query language mode for InfluxDB v3. All queries in the dashboard JSON use raw SQL strings.

### Confidence: HIGH — Verified from InfluxDB 3 Core official docs (docs.influxdata.com/influxdb3/core)

### Key SQL Functions for Solar Dashboard

| Function | Purpose | Example |
|----------|---------|---------|
| `DATE_BIN(interval, time, origin)` | Time-bucket aggregation | `DATE_BIN(INTERVAL '1 hour', time, '2026-01-01T00:00:00Z'::TIMESTAMP)` |
| `date_bin_gapfill(interval, time)` | Fill missing time buckets | For continuous chart lines even when inverters report no data at night |
| `AVG()`, `SUM()`, `MAX()`, `MIN()` | Standard aggregates | Average power, total energy, peak production |
| `LAG() OVER (PARTITION BY ... ORDER BY time)` | Window function for delta calculations | Calculate energy produced between two readings using total_production counter differences |
| `LAST_VALUE() OVER (...)` | Latest reading | Get current/most recent value from each sensor |
| `CASE WHEN ... THEN ... END` | TOU rate calculation | Map time-of-day to peak/off-peak rates |
| `selector_last(field)['value']` | InfluxDB selector for latest value | Optimized "give me latest reading" query |
| `locf()` / `interpolate()` | Gap-fill strategies | Last-observation-carried-forward or linear interpolation for missing data |

### Essential Query Patterns

#### 1. Current Power (Real-time Stat Panel)

```sql
SELECT
  power_sensor AS power
FROM "East Microinverter"
WHERE time >= now() - INTERVAL '1 minute'
ORDER BY time DESC
LIMIT 1
```

#### 2. Total System Power (Sum Both Inverters)

```sql
SELECT
  time,
  SUM(power_sensor) AS total_power
FROM (
  SELECT time, power_sensor FROM "East Microinverter"
  WHERE time >= $__timeFrom AND time <= $__timeTo
  UNION ALL
  SELECT time, power_sensor FROM "West Microinverter"
  WHERE time >= $__timeFrom AND time <= $__timeTo
)
GROUP BY DATE_BIN(INTERVAL '10 seconds', time, '2026-01-01T00:00:00Z'::TIMESTAMP)
ORDER BY 1
```

**Note:** Table names with spaces MUST be double-quoted in SQL: `"East Microinverter"`, `"West Microinverter"`, `"Smart Meter"`.

#### 3. Daily Energy Production (Bar Chart)

```sql
SELECT
  DATE_BIN(INTERVAL '1 day', time, '2026-01-01T00:00:00Z'::TIMESTAMP) AS day,
  (MAX(total_production_1_sensor) - MIN(total_production_1_sensor)) +
  (MAX(total_production_2_sensor) - MIN(total_production_2_sensor)) +
  (MAX(total_production_3_sensor) - MIN(total_production_3_sensor)) +
  (MAX(total_production_4_sensor) - MIN(total_production_4_sensor)) AS daily_energy_kwh
FROM "East Microinverter"
WHERE time >= now() - INTERVAL '30 days'
GROUP BY 1
ORDER BY 1
```

#### 4. Module-Level Power (Canvas/Table — All 8 Panels)

```sql
SELECT
  'East' AS inverter,
  pv1_power_sensor AS pv1,
  pv2_power_sensor AS pv2,
  pv3_power_sensor AS pv3,
  pv4_power_sensor AS pv4
FROM "East Microinverter"
WHERE time >= now() - INTERVAL '1 minute'
ORDER BY time DESC
LIMIT 1
```

#### 5. TOU Financial Savings Calculation

```sql
SELECT
  DATE_BIN(INTERVAL '1 day', time, '2026-01-01T00:00:00Z'::TIMESTAMP) AS day,
  SUM(
    CASE
      WHEN EXTRACT(DOW FROM time) BETWEEN 1 AND 5
           AND EXTRACT(HOUR FROM time) BETWEEN 9 AND 21
      THEN energy_delta * 5.7982
      ELSE energy_delta * 2.6369
    END
  ) AS savings_thb
FROM (
  -- Subquery calculating energy deltas from production counters
  SELECT
    time,
    (total_production - LAG(total_production) OVER (ORDER BY time)) AS energy_delta
  FROM combined_production
  WHERE time >= now() - INTERVAL '30 days'
)
WHERE energy_delta > 0 AND energy_delta < 10
GROUP BY 1
ORDER BY 1
```

#### 6. Time Range Variables in Grafana SQL

Grafana provides macros for time range filtering in InfluxDB SQL mode:
- `$__timeFrom` — start of selected time range (timestamp)
- `$__timeTo` — end of selected time range (timestamp)
- Use `WHERE time >= $__timeFrom AND time <= $__timeTo` in all time-filtered queries

---

## Dashboard JSON Structure

### Confidence: HIGH — Verified from Grafana official docs

### Key Structural Rules

```json
{
  "__inputs": [
    {
      "name": "DS_INFLUXDB_SOLAR",
      "label": "influxdb-solar",
      "type": "datasource",
      "pluginId": "influxdb"
    }
  ],
  "title": "Solar PV Monitor",
  "uid": null,
  "schemaVersion": 40,
  "version": 1,
  "timezone": "Asia/Bangkok",
  "refresh": "10s",
  "time": { "from": "now-24h", "to": "now" },
  "panels": [ ... ],
  "templating": { "list": [] },
  "annotations": { "list": [] }
}
```

### Panel JSON Structure Pattern

```json
{
  "id": 1,
  "type": "stat",
  "title": "Current Power",
  "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
  "datasource": { "type": "influxdb", "uid": "${DS_INFLUXDB_SOLAR}" },
  "targets": [
    {
      "rawSql": "SELECT power_sensor FROM \"East Microinverter\" WHERE time >= now() - INTERVAL '1 minute' ORDER BY time DESC LIMIT 1",
      "refId": "A",
      "format": "table"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "watt",
      "color": { "mode": "thresholds" },
      "thresholds": {
        "steps": [
          { "value": null, "color": "dark-gray" },
          { "value": 1, "color": "yellow" },
          { "value": 1000, "color": "green" },
          { "value": 4000, "color": "dark-green" }
        ]
      }
    }
  },
  "options": {
    "reduceOptions": { "calcs": ["lastNotNull"] },
    "graphMode": "area",
    "textMode": "auto"
  }
}
```

### Critical JSON Details

| Aspect | Value | Notes |
|--------|-------|-------|
| `schemaVersion` | 40 | Grafana 12.x schema version |
| `datasource.uid` | `"${DS_INFLUXDB_SOLAR}"` | Use `__inputs` pattern for importable dashboards |
| `targets[].rawSql` | SQL string | Raw SQL mode, NOT visual query builder |
| `targets[].format` | `"table"` or `"time_series"` | `table` for stat/gauge/table, `time_series` for time series chart |
| `gridPos` | `{h, w, x, y}` | 24-column grid; height in "units" (1 unit = ~30px) |
| `fieldConfig.defaults.unit` | Grafana unit string | `"watt"`, `"watth"`, `"kwatth"`, `"volt"`, `"amp"`, `"celsius"`, `"hertz"`, `"currencyTHB"` |
| Time zone | `"Asia/Bangkok"` | Thailand timezone for correct TOU rate calculations |

---

## Color Scheme & Theme for Solar/Energy Dashboards

### Confidence: MEDIUM — Based on industry patterns and Grafana community dashboards

### Recommended Color Palette

| Metric Type | Color(s) | Grafana Color Name | Rationale |
|---|---|---|---|
| Solar production (active) | Amber/Yellow/Green gradient | `"yellow"`, `"green"`, `"dark-green"` | Industry standard: sun/solar = yellow/amber; higher production = greener |
| Solar production (zero/night) | Dark gray | `"dark-gray"` | Clearly distinguishes non-producing state |
| Grid import | Blue | `"blue"`, `"light-blue"` | Industry convention: grid power = blue, contrasts with solar yellow |
| House consumption | Purple/Orange | `"purple"` or `"orange"` | Distinct from both solar (yellow) and grid (blue) |
| Voltage | Teal/Cyan | `"super-light-blue"` | Common for electrical measurements |
| Temperature | Orange → Red gradient | `"orange"`, `"red"` | Intuitive heat mapping |
| Financial savings | Green | `"green"` | Universal "money saved" = green |
| Faults/Alarms | Red | `"red"`, `"dark-red"` | Universal danger/alert |
| Off-peak period | Light blue background | Annotation color | Visual TOU period indicator |
| Peak period | Light orange background | Annotation color | Visual TOU period indicator |

### Panel Roof Layout Color Scale (Canvas Panel)

For the 8-panel roof layout visualization:
- **0W** (night/offline): `#1a1a2e` (dark navy)
- **1-100W** (low): `#614a19` (dark amber)
- **100-400W** (moderate): `#c7a035` (amber)
- **400-600W** (good): `#73BF69` (green)
- **600W+** (excellent): `#1a7c11` (dark green)

### Dashboard-Level Theming

- **Background**: Use Grafana's dark theme (default) — superior contrast for colored data
- **Row section headers**: Use `text` panel type with dark background, contrasting text
- **Panel spacing**: Tight (0-1px gap) for overview stats row; standard for chart rows
- **Transparent backgrounds**: Use `"transparent": true` for stat panels in the header row to create a clean KPI bar

---

## Grafana Plugins Assessment

### Confidence: MEDIUM — No solar-specific Grafana plugins verified as maintained for Grafana 12

### Verdict: No Plugins Needed — Use Built-in Panels Only

After investigating the Grafana plugin ecosystem:

1. **No dedicated "solar panel layout" plugin exists** that's actively maintained and compatible with Grafana 12. The Canvas panel (built-in since Grafana 10) fully covers this use case.

2. **No solar/energy-specific panel plugins** are needed because:
   - `canvas` handles spatial layout (roof visualization)
   - `stat` + `gauge` + `timeseries` + `barchart` cover all standard energy dashboard patterns
   - `bargauge` provides compact per-panel comparison
   - `state-timeline` covers inverter state history

3. **Why avoiding plugins is the right call:**
   - Zero dependency risk (plugins break across Grafana upgrades)
   - Dashboard JSON remains portable (no plugin install requirement)
   - All built-in panels are actively maintained by Grafana Labs
   - Canvas panel is powerful enough for custom spatial layouts

### If Future Enhancement Needed

| Plugin | Use Case | Status |
|--------|----------|--------|
| `grafana-image-renderer` | Screenshot/PDF export of dashboard | Official plugin, already available. Install only if email reports needed. |
| `volkovlabs-variable-panel` | Enhanced variable/filter UI | Community. Only if needing complex filtering. |

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Roof layout visualization | Canvas panel (built-in) | Custom HTML panel / iframe | Canvas is native, data-bound, and exportable in JSON. HTML panels are a security concern and harder to maintain. |
| Query language | InfluxDB SQL | InfluxQL (also supported by v3) | SQL is the primary language for InfluxDB 3 Core, has richer features (window functions, CTEs), and is the future direction. InfluxQL is legacy compatibility. |
| Financial calculations | In-query SQL (CASE/WHEN) | Grafana transformations | SQL is more testable, version-controllable, and doesn't break when Grafana transformation API changes. Keep logic in queries. |
| Per-panel comparison | `bargauge` panel | 8 separate `stat` panels | `bargauge` is more space-efficient and provides instant visual comparison. Stat panels waste grid space for 8 items. |
| Time-bucketed energy | `barchart` panel | `timeseries` with bar style | `barchart` is purpose-built for categorical/bucketed data. Time series bar mode works but is less flexible for labeling. |

---

## Datasource Configuration (Reference)

The InfluxDB datasource `influxdb-solar` is already configured. For reference, the key settings in the dashboard JSON `__inputs`:

```json
{
  "__inputs": [
    {
      "name": "DS_INFLUXDB_SOLAR",
      "label": "influxdb-solar",
      "description": "InfluxDB 3 Core - Solar monitoring bucket",
      "type": "datasource",
      "pluginId": "influxdb",
      "pluginName": "InfluxDB"
    }
  ]
}
```

When importing, Grafana will prompt the user to map `DS_INFLUXDB_SOLAR` to their configured `influxdb-solar` datasource. The datasource must be configured with:
- **Query Language**: SQL
- **URL**: `http://192.168.2.10:8181`
- **Database**: `solar`

---

## Grafana Units Reference (Solar-Specific)

| Unit String | Display | Use For |
|---|---|---|
| `"watt"` | W | Real-time power |
| `"kwatt"` | kW | System-level power |
| `"watth"` | Wh | Energy (small scale) |
| `"kwatth"` | kWh | Energy (daily/total) |
| `"volt"` | V | Grid/panel voltage |
| `"amp"` | A | Current |
| `"celsius"` | °C | Inverter temperature |
| `"hertz"` | Hz | Grid frequency |
| `"percentunit"` | % | Power factor, efficiency |
| `"currencyTHB"` | N/A — use custom | Thai Baht — **Note**: Grafana may not have THB built-in. Use `"none"` with `suffix: " THB"` or check available currency units. |
| `"none"` + suffix | Custom | Fallback for THB if no native unit |

**THB Currency Handling:** Grafana's built-in currency units may not include Thai Baht. Verify at runtime. If missing, use `"unit": "none"` with `"fieldConfig.defaults.custom.suffix": " THB"` or override the unit display with a value mapping.

---

## Sources

- **Grafana 12.4 Visualization Docs**: https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/ (verified 2026-03-30)
- **Grafana Canvas Panel Docs**: https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/canvas/ (verified 2026-03-30)
- **Grafana InfluxDB Datasource Docs**: https://grafana.com/docs/grafana/latest/datasources/influxdb/ (verified 2026-03-30)
- **Grafana Dashboard Import Docs**: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/import-dashboards/ (verified 2026-03-30)
- **InfluxDB 3 Core SQL Query Docs**: https://docs.influxdata.com/influxdb3/core/query-data/sql/ (verified 2026-03-30)
- **InfluxDB 3 Core SQL Reference**: https://docs.influxdata.com/influxdb3/core/reference/sql/ (verified 2026-03-30)
- **InfluxDB 3 Core + Grafana Setup**: https://docs.influxdata.com/influxdb3/core/visualize-data/grafana/ (verified 2026-03-30)
