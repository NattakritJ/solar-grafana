# Feature Landscape

**Domain:** Residential Solar PV Monitoring Dashboard (Grafana)
**Researched:** 2026-03-30

## Table Stakes

Features users expect from any solar monitoring dashboard. Missing = product feels incomplete compared to SolarEdge Monitoring, Enphase App, or Fronius Solar.web.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Real-time total power (kW)** | Every solar app shows current production front-and-center. The single most important number. | Low | Stat panel with large font. Sum of both inverter `power_sensor` values. |
| **Today's energy production (kWh)** | Users check daily production every evening. SolarEdge/Enphase both prominently display this. | Low | Sum of `today_production_{1-4}_sensor` across both inverters. Stat panel. |
| **Lifetime/total energy production (kWh/MWh)** | Cumulative proof of system value. All commercial platforms show this. | Low | Sum of `total_production_{1-4}_sensor`. Stat panel. |
| **Production time-series graph** | The signature view: power curve showing production over time (bell curve shape on clear days). Every solar platform has this. | Low | Time series panel, both inverters stacked or overlaid. |
| **Grid import power (kW)** | Shows what the house is drawing from the grid right now. Smart meter `power_sensor`. | Low | Stat panel. Negative values might indicate export (zero-export system makes this unlikely). |
| **House load / consumption (kW)** | Users want to see total house consumption. Calculated as solar production + grid import. Essential for understanding self-consumption. | Med | Calculated field: `inverter_power_total + smart_meter_power`. Stat panel + time series overlay. |
| **Self-consumption ratio (%)** | "How much of my solar am I actually using?" Key metric for zero-export systems. | Med | `(solar_production - export) / solar_production * 100`. Since zero-export, this is effectively 100% when producing, but the ratio of solar-vs-grid for total consumption matters. Better framed as "Solar coverage %": `solar / (solar + grid_import) * 100`. |
| **Per-inverter production breakdown** | With East/West split, users naturally want to compare. SolarEdge shows per-inverter; Enphase shows per-microinverter. | Low | Two stat panels or grouped bar chart. East vs West comparison. |
| **Production history (daily/weekly/monthly bars)** | Bar charts showing production aggregated over time. Every commercial platform offers day/week/month/year views. | Med | Bar chart panel with time-grouped SQL aggregation. Grafana time range selector handles period switching. |
| **Grid voltage and frequency** | Basic grid quality indicators. Fronius and SolarEdge show these. Important for diagnosing grid issues. | Low | Stat panels or small gauges. From smart meter data. |
| **Inverter status/state** | Is the system running? SolarEdge shows device state (producing, sleeping, error). Critical for knowing if something is wrong. | Low | Stat panel with value mapping for `device_state_sensor` (e.g., 0=Off, 1=Producing, etc.). |
| **Inverter temperature** | Standard health metric shown by all commercial platforms. Overheating = performance throttling. | Low | Stat or gauge panel from `temperature_sensor`. |
| **Today's savings (THB)** | Financial impact is the #1 motivator for homeowners. All modern solar apps show monetary savings. | Med | Production kWh multiplied by applicable TOU rate. Requires time-of-day aware calculation in SQL. |
| **Power flow overview** | Visual showing energy flowing from panels to house to grid. Enphase and SolarEdge both have animated flow diagrams. At minimum, stat panels arranged to show the relationship. | Med | Row of stat panels showing Solar -> House <- Grid, or a simple text/canvas representation. Full animated flow is a differentiator (see below). |

## Differentiators

Features that set this dashboard apart from basic solar dashboards. Not expected but high-value.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Roof layout visualization with production heatmap** | Physical panel layout (4 East + 4 West) with color-coded power output per panel. SolarEdge has this; rare in DIY dashboards. Makes module-level issues visually obvious. | High | Grafana Canvas panel with positioned rectangles matching physical layout `[PV3][PV4][PV1][PV2]` for each row. Color thresholds based on individual panel power. Requires careful manual positioning in Canvas. |
| **Module-level monitoring (8 panels)** | Individual panel power, voltage, current for all 8 panels. Enables detection of shading, dirt, degradation, or panel faults. Enphase's key selling point. | Med | 8-panel grid of stat panels, or a table showing all panel metrics. Data available from `pv{1-4}_power/voltage/current_sensor` on each inverter. |
| **TOU-aware financial calculations** | Not just "kWh x flat rate" but actual savings based on peak (5.7982 THB) vs off-peak (2.6369 THB) rates. Most DIY dashboards use flat rates. This reflects real bill savings accurately. | High | SQL CASE expressions based on hour-of-day and day-of-week. Need to handle Thai holidays for full accuracy. Monthly cumulative savings stat. |
| **East vs West production comparison** | Side-by-side comparison showing how orientation affects production throughout the day. East produces more in morning, West in afternoon. Educational and diagnostic. | Med | Dual time-series overlay or split panel. Interesting pattern: East peaks before noon, West after. |
| **Animated energy flow diagram** | Full Sankey-style or animated flow showing solar -> house, grid -> house with proportional line thickness. Premium feature in SolarEdge ONE. | High | Would require Canvas panel with dynamic elements or third-party plugin (e.g., Flowcharting). Complex but visually impressive. Consider using simpler stat panel arrangement first. |
| **Daily production calendar heatmap** | GitHub-style contribution graph but for daily solar production. Instantly shows patterns, cloudy stretches, seasonal trends. | Med | Grafana native heatmap or Hourly Heatmap plugin. Aggregated daily kWh values color-mapped. |
| **Peak power achieved today/ever** | "Your system hit 4.2 kW today!" Gamification element. SolarEdge shows peak power. | Low | SQL MAX() on total power within time range. Stat panel with sparkline. |
| **Power losses tracking** | Both inverters report `power_losses_sensor`. Tracking conversion losses shows inverter efficiency and health trends over time. | Low | Time series or stat panel. Unique data point most dashboards ignore. |
| **CO2 offset estimation** | "You avoided X kg of CO2 today." Environmental impact stat. SolarEdge and Enphase both show this. | Low | Production kWh * grid emission factor (Thailand ~0.5 kg CO2/kWh). Simple multiplication. |
| **Cumulative savings tracker (monthly/yearly/lifetime)** | Running total of money saved over different periods. "This month you saved 2,450 THB." Makes the investment tangible. | Med | Aggregated TOU-aware savings over different time windows. Multiple stat panels. |
| **Inverter alarm/fault history** | Table showing when alarms or faults occurred. `device_alarm_sensor` and `device_fault_sensor` values logged over time. | Med | Table panel filtered to non-zero alarm/fault values. Enables troubleshooting without checking inverter physically. |
| **System efficiency gauge** | Actual production vs theoretical max (5.24 kWp * sun hours). Shows how well the system is performing relative to capacity. | Med | Requires irradiance data or estimated peak sun hours for the location. Can approximate with `actual_kWh / (5.24 * estimated_sun_hours) * 100%`. |
| **Smart meter detailed view** | Dedicated section for power factor, current, voltage, frequency, import/export energy from smart meter. Grid quality deep-dive. | Low | Group of stat panels and time series from Smart Meter measurement. |

## Anti-Features

Features to explicitly NOT build. These add complexity without value for this specific system.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| **Battery management/SOC** | No battery in system. Building placeholders adds clutter and confusion. | Simply omit. If battery is added later, create a new dashboard row. |
| **Export revenue calculations** | Zero-export system. No feed-in tariff. Calculating export revenue is misleading. | Show export energy from smart meter as informational only (should be ~0). |
| **Weather forecast integration** | Requires external API, adds infrastructure complexity, stale quickly. Not worth it for a single-home dashboard. | Focus on actual production data. Historical patterns reveal weather impact naturally. |
| **Inverter firmware management** | Out of scope per project definition. Data pipeline handles communication. | Show firmware version if available in data, but no update functionality. |
| **Multi-site management** | Single residential system. Fleet management UI adds unnecessary complexity. | Single dashboard, single site. |
| **User authentication/roles** | Grafana handles this natively. Don't build custom auth into dashboard logic. | Use Grafana's built-in user/org management. |
| **Complex alerting rules** | Dashboard JSON format doesn't include alerting provisioning. Alerts are a separate Grafana concern. | Document recommended alerts separately, but don't embed in dashboard JSON. Keep dashboard focused on visualization. |
| **Production forecast/prediction** | Requires ML models, weather data, and ongoing calibration. Massive scope creep for a Grafana dashboard. | Show historical averages instead ("average daily production this month"). |
| **Mobile-specific layout** | Grafana's responsive mode handles basic mobile viewing. Building separate mobile-optimized panels doubles maintenance. | Rely on Grafana's built-in responsive behavior. Design desktop-first with reasonable panel sizing. |

## Feature Dependencies

```
Real-time total power ──────────────────────────> Production time-series graph
                                                   (same data, different viz)

Per-inverter production ────────────────────────> East vs West comparison
                                                   (requires both inverter queries)

Per-inverter production ────────────────────────> Module-level monitoring
                                                   (drills down from inverter to panel level)

Module-level monitoring ────────────────────────> Roof layout heatmap
                                                   (individual panel data feeds canvas)

Grid import + Solar production ─────────────────> House load calculation
                                                   (derived metric)

House load + Solar production ──────────────────> Self-consumption ratio
                                                   (derived from both)

Solar production + TOU rates ───────────────────> Today's savings
                                                   (requires TOU logic)

Today's savings ────────────────────────────────> Cumulative savings (month/year/lifetime)
                                                   (extends same TOU logic over longer periods)

Inverter state + alarms + faults ───────────────> Inverter alarm/fault history
                                                   (health data in table form)

Today's energy ─────────────────────────────────> Production history bars
                                                   (aggregated version of same data)
```

## Dashboard Section Layout (Recommended)

Based on analysis of SolarEdge, Enphase, and Fronius monitoring platforms, solar dashboards consistently organize into these sections:

### Row 1: System Overview (at-a-glance)
- Current total power (kW) — **large stat**
- Today's production (kWh) — stat
- Today's savings (THB) — stat
- Self-consumption ratio (%) — stat/gauge
- Grid import (kW) — stat
- House load (kW) — stat
- System status (Producing/Idle/Error) — stat with value mapping

### Row 2: Production Charts
- Power production time series (main graph, full width or 2/3)
- East vs West stacked/overlaid production
- Grid import overlaid or separate

### Row 3: Energy Summary
- Daily/weekly/monthly production bar chart
- Cumulative savings chart or table
- Peak power today

### Row 4: Module-Level Monitoring
- Roof layout visualization (Canvas) — if differentiator is pursued
- OR: 8-panel grid of individual power stats
- Per-panel power/voltage/current table

### Row 5: Financial
- Today's savings breakdown (peak vs off-peak kWh and value)
- Month-to-date savings
- Savings history bar chart

### Row 6: Inverter Health
- Temperature gauges (East + West)
- Device state indicators
- Alarm/fault log table
- Power losses time series

### Row 7: Grid Quality
- Voltage, frequency, current, power factor from smart meter
- Time series showing grid quality trends

## Time Range Views Expected

All commercial platforms offer these standard views:

| View | What Users See | Grafana Implementation |
|------|----------------|----------------------|
| **Live / Real-time** | Current power, updating every 10s | Auto-refresh at 10s, "Last 15 minutes" range |
| **Today** | Production curve, energy total, savings | "Today so far" time range |
| **Yesterday** | Compare to today, review full day | Relative time picker |
| **This Week** | Daily production bars | "This week so far" time range |
| **This Month** | Daily production bars, monthly savings total | "This month so far" time range |
| **Custom Range** | Any user-selected period | Grafana's native time picker |
| **Year-to-date** | Monthly production bars, annual savings | "This year so far" time range |

Grafana's built-in time range picker handles all these natively. Dashboard queries should use `$__timeFilter` or equivalent to be time-range-aware.

## MVP Recommendation

Prioritize for first working dashboard:

1. **System Overview row** — Current power, today's production, grid import, house load, system status (table stakes, Low complexity)
2. **Production time series** — The signature solar graph (table stakes, Low complexity)
3. **Per-inverter breakdown** — East vs West stats (table stakes, Low complexity)
4. **Today's energy + basic savings** — Even with flat-rate initially, show THB saved (table stakes, Med complexity)
5. **Module-level stats** — 8 panel power/voltage/current (differentiator, Med complexity, but data is readily available)
6. **Inverter health** — Temperature, state, alarms (table stakes, Low complexity)

**Defer:**
- **Roof layout heatmap (Canvas)**: High complexity, requires careful manual positioning. Add after core dashboard works. Worth doing — it's the visual wow factor — but not blocking.
- **TOU-aware calculations**: Med-High complexity. Start with flat peak rate, then refine with time-of-day logic.
- **Daily/monthly bar charts**: Require aggregation queries. Add after real-time views work.
- **Animated energy flow**: High complexity, limited ROI vs simpler stat panel arrangement.
- **Calendar heatmap**: Needs accumulated historical data (system is new). Add after weeks of data exist.

## Alerting (Documented Separately, Not in Dashboard)

Standard solar monitoring alerts (configure in Grafana Alerting, not dashboard JSON):

| Alert | Condition | Severity |
|-------|-----------|----------|
| **Zero production during daylight** | Total power = 0 between 08:00-16:00 for >30 min | High |
| **Inverter offline** | No data from inverter for >5 min | High |
| **Inverter overtemperature** | Temperature > 65C (typical threshold) | Medium |
| **Inverter fault/alarm** | `device_fault_sensor` or `device_alarm_sensor` != 0 | High |
| **Grid voltage out of range** | Voltage < 210V or > 250V | Medium |
| **Single panel underperformance** | One panel producing <50% of peers for >1 hour during production | Low |
| **Grid frequency anomaly** | Frequency outside 49.5-50.5 Hz | Medium |

## Sources

- SolarEdge Monitoring Platform features: https://www.solaredge.com/en/products/software-tools/monitoring-platform (verified 2026-03-30) — module-level monitoring, fleet management, automated alerts, physical layout view. **MEDIUM confidence** (marketing page, feature details inferred from descriptions).
- Enphase App features: https://enphase.com/homeowners/enphase-app (verified 2026-03-30) — panel-level production, grid tracking, system health, day/week/month/year reports, array layout view. **MEDIUM confidence** (marketing page with screenshot descriptions).
- Fronius Solar.web: Known features from training data — system overview, energy flow, yield tracking, component monitoring, notification system. **LOW confidence** (training data only, site returned 404).
- Grafana panel types: https://grafana.com/docs/grafana/latest/panels-visualizations/ (verified 2026-03-30) — stat, gauge, time series, bar chart, canvas, heatmap, table, pie chart available in v12.4. **HIGH confidence** (official docs).
- Community patterns: GitHub solar-monitoring topic, Home Assistant community patterns. **MEDIUM confidence** (multiple community sources align on standard features).
- InfluxDB data structure: From PROJECT.md — confirmed available fields for all features listed. **HIGH confidence** (project documentation).
