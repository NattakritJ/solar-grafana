# Project Research Summary

**Project:** Solar Monitoring Grafana Dashboard
**Domain:** Residential Solar PV Monitoring (single-site, Grafana JSON dashboard)
**Researched:** 2026-03-30
**Confidence:** HIGH

## Executive Summary

This project delivers a **single Grafana dashboard JSON file** for monitoring a residential 5.24 kWp solar PV system with two microinverters (East/West facing, 4 panels each) and a smart meter. The infrastructure is already running — Grafana 12.4.1 and InfluxDB 3 Core are deployed at 192.168.2.10:8181 with data flowing at ~10-second intervals. The deliverable is purely a dashboard JSON that can be imported. Experts build these as a single dashboard with collapsible row sections, using only built-in Grafana panel types (no plugins), with all data logic expressed in InfluxDB 3 SQL queries rather than Grafana-side transformations. The closest commercial analogues are SolarEdge Monitoring, Enphase App, and Fronius Solar.web.

The recommended approach is to build incrementally in 5 phases, starting with a dashboard skeleton and real-time stat panels to validate the InfluxDB 3 SQL connection (the highest-risk element), then layering on production charts, consumption/grid views, module-level detail, financial calculations, and finally the Canvas-based roof layout heatmap. **The single biggest risk is InfluxDB 3 SQL syntax** — nearly all community resources use Flux or InfluxQL, which are incompatible. Every query must use `DATE_BIN()` instead of `GROUP BY time()`, `LAG()` instead of `difference()`, and `AVG()` instead of `mean()`. The second major risk is TOU financial calculations requiring correct UTC-to-Bangkok timezone conversion — getting this wrong produces savings errors up to 2.2x.

No external plugins are needed. All 10 recommended panel types (`timeseries`, `stat`, `gauge`, `barchart`, `table`, `canvas`, `piechart`, `text`, `state-timeline`, `bargauge`) are built-in to Grafana 12.4.1. The Canvas panel is the key differentiator for the roof layout heatmap but is the most complex to build (manual element positioning with data binding). The dashboard JSON must use `__inputs` for datasource portability to work on any Grafana instance.

## Key Findings

### Recommended Stack

The entire stack is already deployed. No new infrastructure is needed — the deliverable is a single dashboard JSON file.

**Core technologies:**
- **Grafana OSS 12.4.1**: Dashboard platform — already running, latest stable with full panel support
- **InfluxDB 3 Core**: Time-series database — already running, uses **SQL only** (NOT Flux/InfluxQL)
- **Built-in Grafana panels only**: 10 panel types cover all use cases — zero plugin dependencies for maximum portability
- **InfluxDB SQL (DataFusion)**: Query language — `DATE_BIN()` for time bucketing, `LAG()` for deltas, `CASE WHEN` for TOU rates

**Critical version/config requirements:**
- Grafana schema version 39-40 (v12.x)
- InfluxDB datasource configured as "InfluxDB Enterprise 3.x" in SQL mode with Flight SQL (gRPC)
- Dashboard timezone: `Asia/Bangkok` (UTC+7, no DST)
- Datasource must use "Insecure Connection" if no TLS configured

### Expected Features

**Must have (table stakes):**
- Real-time total system power (kW) — the single most important number
- Today's energy production (kWh) — users check this daily
- Lifetime/total energy production — cumulative proof of system value
- Production time-series graph — the signature solar "bell curve"
- Grid import power — what the house draws from the grid
- House load / consumption — solar + grid import (zero-export system)
- Self-consumption ratio (%) — key metric for zero-export systems
- Per-inverter breakdown (East vs West) — natural comparison for dual-orientation
- Production history (daily/weekly/monthly bar charts)
- Inverter status, temperature, grid voltage/frequency — system health basics
- Today's savings (THB) — financial impact is the #1 homeowner motivator
- Power flow overview — at minimum stat panels showing Solar → House ← Grid relationship

**Should have (differentiators):**
- Roof layout heatmap via Canvas panel — 8 panels color-coded by production (SolarEdge-level feature, the visual wow factor)
- Module-level monitoring — individual PV power/voltage/current for all 8 panels
- TOU-aware financial calculations — peak (5.7982 THB/kWh) vs off-peak (2.6369 THB/kWh) accuracy
- East vs West production comparison over time — educational and diagnostic
- CO2 offset estimation — simple multiplication, high perceived value
- Cumulative savings tracker (month/year/lifetime)
- Peak power achieved today — gamification element
- Power losses tracking — unique data point most dashboards ignore
- Inverter alarm/fault history table

**Defer (v2+):**
- Animated energy flow diagram — high complexity, limited ROI vs stat panel arrangement
- Daily production calendar heatmap — needs weeks of accumulated historical data
- Weather forecast integration — requires external APIs, massive scope creep
- Production forecasting — requires ML, not a dashboard concern
- Mobile-specific layout — rely on Grafana's built-in responsive behavior

### Architecture Approach

Single dashboard JSON with **6 collapsible row sections** organized top-to-bottom by priority: Header Stats (always visible, no row header), Solar Production, Module-Level Monitoring, Grid & Consumption, Financial Savings, Inverter Health. All calculations live in SQL queries — no Grafana transformations except Canvas data binding. Each panel gets focused queries (one per series via refId A/B/C), avoiding complex JOINs by using separate queries and Grafana's visual stacking. Dashboard uses 24-column grid, 25-35 panels total, 10s auto-refresh, shared crosshair for correlation.

**Major components (row sections):**
1. **Header Stats** — 6-7 stat panels always visible: current power, today energy, total energy, savings, grid import, house load, system status
2. **Solar Production** — time series (East+West overlay), bar gauge per-inverter, daily energy bars
3. **Module-Level Monitoring** — Canvas roof layout (8 data-bound rectangles), per-PV stats table
4. **Grid & Consumption** — grid import time series, house load overlay, voltage/frequency/PF stats
5. **Financial Savings** — TOU-aware savings stats (today/month), savings over time chart
6. **Inverter Health** — temperature gauges, device state timeline, alarm/fault log, power losses

**Key patterns to follow:**
- `__inputs` pattern for portable datasource references (`${DS_INFLUXDB_SOLAR}`)
- Separate queries per series in multi-series panels (avoid JOINs on misaligned timestamps)
- `DATE_BIN()` aggregation scaled to dashboard time range for performance
- `COALESCE(field, 0)` for nighttime NULL handling across all power fields
- `graphTooltip: 1` (shared crosshair) for cross-panel correlation
- Value mappings for `device_state_sensor` to convert numeric codes to human-readable text
- SQL-based calculations over Grafana transformations for portability

### Critical Pitfalls

1. **InfluxDB 3 SQL syntax traps** — All community examples use Flux/InfluxQL which won't work. Must use `DATE_BIN()` not `GROUP BY time()`, `AVG()` not `mean()`, `LAG()` not `difference()`, `selector_first()['value']` with bracket notation. Reference ONLY official InfluxDB 3 Core SQL docs. **Phase 1 risk.**

2. **Cumulative energy counter resets** — `total_production` fields are monotonically increasing counters. Inverter reboots reset them. Use `LAG()` with negative-difference filtering: `CASE WHEN val - LAG(val) < 0 THEN 0 ELSE val - LAG(val) END`. **Phase 1-2 risk.**

3. **UTC vs Bangkok timezone for TOU rates** — InfluxDB stores UTC. Peak/off-peak is defined in local time (ICT, UTC+7). Must use `AT TIME ZONE 'Asia/Bangkok'` before `EXTRACT(HOUR/DOW)`. Getting this wrong produces savings errors up to 2.2x (ratio of peak to off-peak rate). **Phase 4 risk.**

4. **Datasource UID portability** — Dashboard JSON must use `__inputs` + `${DS_INFLUXDB_SOLAR}` pattern, not hardcoded UIDs. Otherwise dashboard is unusable on import to any other Grafana instance. **Design from Phase 1.**

5. **House load timing mismatches** — Smart meter and inverter data arrive at slightly different times (~10s intervals, not synchronized). Use `COALESCE()`, aggregate to same time bins before combining, and clamp to `GREATEST(value, 0)` to prevent negative display values. **Phase 2 risk.**

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Dashboard Foundation & Real-Time Stats
**Rationale:** Validates the most critical risk — InfluxDB 3 SQL query syntax and Grafana datasource connectivity via Flight SQL. Everything else depends on queries working correctly. This is the "prove it works" phase.
**Delivers:** Dashboard JSON skeleton (metadata, time settings, refresh, `__inputs`), header stat panels showing current power, today's energy, grid import, system status. Basic production time series chart (East + West).
**Addresses:** Real-time total power, today's energy, production time series, inverter status, grid import (5 table stakes features)
**Avoids:** Pitfall 1 (SQL syntax), Pitfall 4 (datasource portability), Pitfall 6 (spaces in measurement names), Pitfall 7 (Flight SQL/gRPC setup), Pitfall 10 (macro validation)

### Phase 2: Core Production & Consumption
**Rationale:** Builds on validated query patterns to add the core monitoring views. House load depends on both solar and grid queries working. Energy calculations introduce counter arithmetic.
**Delivers:** Solar Production row (East vs West overlay, daily production bars, per-inverter bar gauge), Grid & Consumption row (house load calculation, grid import time series, grid voltage/frequency/PF stats).
**Addresses:** Per-inverter breakdown, East vs West comparison, house load, self-consumption ratio, production history, grid quality (6 table stakes + 1 differentiator)
**Avoids:** Pitfall 2 (counter resets for energy calcs), Pitfall 5 (house load timing mismatch), Pitfall 8 (performance via DATE_BIN), Pitfall 9 (nighttime NULLs)

### Phase 3: Module-Level & Inverter Health
**Rationale:** Extends established query patterns to per-PV granularity. More panels but no new query complexity — uses same `SELECT ... FROM "East Microinverter"` patterns. Health monitoring completes the "basic full dashboard."
**Delivers:** Module-Level row (per-PV power/voltage/current table, 8-panel bar gauge comparison), Inverter Health row (temperature gauges, device state timeline, alarm/fault table, power losses chart).
**Addresses:** Module-level monitoring, inverter temperature, alarm/fault history, power losses, peak power (5 differentiators)
**Avoids:** Pitfall 12 (PV aggregation — use `power_sensor` for totals, PV channels only for module-level)

### Phase 4: Financial Calculations & Canvas Roof Layout
**Rationale:** The two highest-complexity features deferred until all simpler patterns are proven. TOU calculations require timezone-correct SQL with CASE expressions. Canvas requires precise JSON element positioning with data binding. Both are the "premium" differentiators.
**Delivers:** Financial Savings row (TOU-aware daily/monthly/lifetime savings in THB, peak vs off-peak breakdown), Canvas roof layout heatmap (8 panel rectangles positioned to match physical layout, color-coded by real-time production).
**Addresses:** TOU-aware savings, cumulative savings, roof layout heatmap, CO2 offset (4 key differentiators)
**Avoids:** Pitfall 3 (UTC vs Asia/Bangkok timezone), Pitfall 13 (THB currency formatting), Pitfall 12 (PV-to-physical-position mapping: PV3, PV4, PV1, PV2 left-to-right)

### Phase 5: Polish & Export
**Rationale:** Final pass ensuring professional quality and portability. All functionality exists; this phase adds consistency, documentation, and validates the complete deliverable.
**Delivers:** Consistent color thresholds across all panels (solar=yellow/green, grid=blue, temp=orange/red, faults=red), unit formatting (W/kWh/THB/°C/V/Hz), panel descriptions and tooltips, transparent stat panel backgrounds for header row, verified JSON import on clean Grafana instance.
**Addresses:** Color scheme consistency, remaining polish items, export portability validation
**Avoids:** Pitfall 4 (final import verification), Pitfall 8 (DATE_BIN scaling for long time ranges), Pitfall 11 (panel type compatibility)

### Phase Ordering Rationale

- **Phase 1 first** because InfluxDB 3 SQL syntax is the #1 risk. If queries don't work, nothing else matters. Validating `DATE_BIN()`, `$__timeFrom`, table name quoting, and Flight SQL connectivity eliminates the biggest uncertainty before building 30+ panels.
- **Phase 2 before 3** because house load (solar + grid) is a derived metric needing both data sources proven. Production charts establish the core daily-use value.
- **Phase 3 before 4** because module-level queries establish per-PV patterns needed for Canvas data binding. Completing health monitoring means the dashboard is fully usable even if Phase 4 takes longer.
- **Phase 4 is the complexity peak** — TOU timezone calculations and Canvas element positioning are the two hardest technical challenges. The dashboard is already a complete monitoring tool after Phase 3; Phase 4 adds premium differentiators.
- **Phase 5 last** because polish requires all panels to exist. Import testing is the final acceptance gate.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1:** Must validate exact Grafana macro syntax (`$__timeFrom` vs `$__timeFilter`) in InfluxDB 3 SQL mode — documentation is sparse, needs runtime testing with query inspector
- **Phase 4 (Canvas):** Canvas panel JSON structure for data-bound elements needs exploration — docs describe UI-based construction, may need to build in Grafana UI first then extract JSON
- **Phase 4 (TOU):** Validate `AT TIME ZONE 'Asia/Bangkok'` syntax in InfluxDB 3 Core SQL — confirmed in docs but untested with Grafana macro interaction

Phases with standard patterns (skip research-phase):
- **Phase 2:** Time series and bar chart panels are well-documented Grafana patterns with abundant examples
- **Phase 3:** Table panels and stat grids are straightforward Grafana JSON construction
- **Phase 5:** Color/threshold/unit configuration is well-documented in Grafana panel options

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | **HIGH** | All infrastructure deployed and running. Grafana + InfluxDB docs verified. No technology decisions needed — stack is fixed. |
| Features | **MEDIUM-HIGH** | Table stakes well-understood from commercial platform analysis (SolarEdge, Enphase). Feature prioritization clear. TOU rates from project spec. Some features (calendar heatmap) based on community patterns. |
| Architecture | **HIGH** | Single-dashboard row-based layout is standard Grafana. JSON structure verified against Grafana 12.4 docs. Query patterns verified against InfluxDB 3 SQL docs. Build order follows clear dependency chain. |
| Pitfalls | **HIGH** | 13 pitfalls identified across critical/moderate/minor severity. Critical pitfalls (SQL syntax, timezone, counter resets, portability) verified against official docs. Only macro behavior is MEDIUM confidence. |

**Overall confidence:** HIGH

### Gaps to Address

- **Grafana macro substitution in InfluxDB 3 SQL mode:** MEDIUM confidence on exact syntax of `$__timeFrom`, `$__timeTo`, `$__timeFilter(time)`, `$__interval`. Must validate with Grafana query inspector in Phase 1 before building all queries on assumptions.
- **Canvas panel JSON construction:** Documentation focuses on UI-based element placement. Writing Canvas JSON directly (for a git-managed dashboard file) may require building in Grafana UI first and extracting the JSON. Affects Phase 4 workflow.
- **InfluxDB 3 `AT TIME ZONE 'Asia/Bangkok'`:** Confirmed available in SQL docs but interaction with Grafana time range macros needs runtime validation. Critical for TOU accuracy in Phase 4.
- **THB currency unit availability:** Grafana may or may not have built-in Thai Baht formatting. Documented fallback: `"unit": "none"` with `" THB"` suffix. Verify at runtime.
- **`date_bin_gapfill()` month/year limitation:** Does NOT support month or year intervals. Long-range monthly views must use `date_trunc('month', time)` instead. Affects Phase 2 bar charts for monthly views.
- **Device state value mappings:** Numeric values for `device_state_sensor` (0=Off?, 1=Producing?) need confirmation against actual data — not documented in project spec.
- **Smart Meter `power_sensor` sign convention:** Does positive = import and negative = export, or vice versa? Affects house load calculation in Phase 2.
- **Thai public holidays in TOU classification:** PROJECT.md defines weekday/weekend rates only. Unclear if Thai public holidays count as off-peak. Business rules question for Phase 4.

## Sources

### Primary (HIGH confidence)
- Grafana 12.4 Visualization Docs — panel types, JSON model, Canvas panel, import/export, variables
- InfluxDB 3 Core SQL Reference — `DATE_BIN`, `LAG/LEAD`, `AT TIME ZONE`, aggregate/selector functions, `date_bin_gapfill`
- InfluxDB 3 Core + Grafana Integration — datasource setup, Flight SQL, SQL query mode, macros
- Grafana Dashboard JSON Model — schema version, `__inputs`, `gridPos`, field config, value mappings
- PROJECT.md — system specs (5.24 kWp, 8 panels, 2 inverters), InfluxDB schema, TOU rates, physical layout

### Secondary (MEDIUM confidence)
- SolarEdge Monitoring Platform — feature expectations, layout patterns (marketing page analysis)
- Enphase App — feature expectations, module-level monitoring patterns (marketing page analysis)
- Grafana community solar dashboard patterns — layout conventions, color schemes, section organization
- Solar monitoring domain knowledge — energy balance equations, counter resets, nighttime behavior, self-consumption

### Tertiary (LOW confidence)
- Grafana InfluxDB SQL macro behavior — `$__timeFilter`, `$__interval` exact syntax needs runtime validation
- Fronius Solar.web features — training data only (site returned 404)
- Canvas panel JSON direct authoring — documentation focuses on UI construction, not raw JSON

---
*Research completed: 2026-03-30*
*Ready for roadmap: yes*
