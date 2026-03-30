# Domain Pitfalls

**Domain:** Solar monitoring Grafana dashboard with InfluxDB 3 Core
**Researched:** 2026-03-30

## Critical Pitfalls

Mistakes that cause rewrites, broken dashboards, or fundamentally wrong data.

---

### Pitfall 1: Using Flux or InfluxQL Syntax in InfluxDB 3 Core SQL Queries

**What goes wrong:** InfluxDB 3 Core uses Apache Arrow DataFusion SQL — NOT Flux, NOT InfluxQL (when configured for SQL). The vast majority of InfluxDB tutorials, community dashboards, and Stack Overflow answers use Flux or InfluxQL syntax. Copy-pasting these will silently fail or produce errors. Specific traps:
- `GROUP BY time(1h)` is InfluxQL syntax. InfluxDB 3 SQL requires `DATE_BIN(INTERVAL '1 hour', time)` with `GROUP BY 1`.
- There is no `FILL()` clause. Use `date_bin_gapfill()` with `interpolate()` or `locf()`.
- There is no `derivative()` or `difference()` function. Must use `LAG()` window function: `value - LAG(value) OVER (ORDER BY time)`.
- `mean()` is not a function — use `avg()`.
- There is no `non_negative_difference()` — must manually handle with `CASE WHEN ... >= 0`.
- Selector functions like `selector_first()` return structs — you must use `selector_first(field, time)['value']` bracket notation.
- `GROUP BY time` in the GROUP BY clause always refers to the source table's `time` column, NOT a `DATE_BIN` alias named `time`. Use column ordinal `GROUP BY 1` or alias it as something other than `time` (e.g., `_time`).

**Why it happens:** InfluxDB 3 Core is brand new (2025). Almost all community resources target InfluxDB 1.x (InfluxQL) or 2.x (Flux). Even AI assistants are trained predominantly on Flux/InfluxQL examples.

**Consequences:** Queries fail entirely, or return no data. Hours wasted debugging syntax.

**Prevention:**
- Reference ONLY the InfluxDB 3 Core SQL docs: `https://docs.influxdata.com/influxdb3/core/reference/sql/`
- Use `DATE_BIN()` for all time-binned aggregations
- Use `LAG()` / `LEAD()` window functions for value differences
- Use `selector_first()['value']` with bracket notation for selector results
- Always test queries in the InfluxDB 3 CLI before putting them in Grafana

**Detection:** Query returns error messages like "function not found", "unexpected token", or returns empty results when data exists.

**Confidence:** HIGH — verified against official InfluxDB 3 Core SQL documentation (March 2026)

**Phase:** Phase 1 (initial dashboard queries) — this is the #1 risk from day one.

---

### Pitfall 2: Cumulative Energy Counter Reset Handling

**What goes wrong:** The `total_production_{1-4}_sensor` and `total_import_energy_sensor` fields are monotonically increasing cumulative counters (kWh). Calculating "energy produced today" requires computing the difference between the latest and earliest value of the day. But:
- **Counter resets:** If an inverter reboots, the counter may reset to zero. A naive `MAX - MIN` calculation produces a huge negative number or wrong result.
- **kWh vs W confusion:** `power_sensor` fields report instantaneous watts (W). `total_production` fields report cumulative kilowatt-hours (kWh). Mixing these up (e.g., summing power readings and calling it kWh) gives nonsensical results.
- **Integrating power to energy incorrectly:** To convert instantaneous power (W) to energy (Wh), you must integrate over time: `power * time_interval`. Simply averaging power and multiplying by hours is an approximation that degrades with variable production.
- **Per-channel vs total:** `power_sensor` on the inverter is the total output, while `pv1_power_sensor` through `pv4_power_sensor` are per-channel. The sum of pv1-pv4 may not exactly equal `power_sensor` due to `power_losses_sensor`.

**Why it happens:** Solar monitoring is fundamentally about energy (kWh), but sensors report power (W) at intervals. The conversion requires understanding of integration, and cumulative counters have edge cases.

**Consequences:** "Today's production" shows wrong numbers. Financial savings calculations are wildly off. User loses trust in dashboard.

**Prevention:**
- For "today's energy": Use `MAX(total_production) - MIN(total_production)` within the day's time window, then add logic to handle resets: detect when a subsequent value is less than a prior value within the same window and add the prior max to the calculation.
- Alternatively, use the `LAG()` window function to compute per-row differences and filter out negative differences (counter resets): `CASE WHEN val - LAG(val) OVER (ORDER BY time) < 0 THEN 0 ELSE val - LAG(val) OVER (ORDER BY time) END`.
- Always verify units. Label every panel with W vs Wh vs kWh.
- Test with simulated counter reset data before deploying.

**Detection:** "Today's production" suddenly shows 0 or an impossibly large number. Sum of panels doesn't roughly match inverter total.

**Confidence:** HIGH — fundamental domain knowledge for solar monitoring systems.

**Phase:** Phase 1-2 (energy calculations). Must get right before financial calculations.

---

### Pitfall 3: Time Zone Mishandling for TOU Rate Calculations

**What goes wrong:** TOU rates are defined by local wall-clock time (Bangkok, UTC+7): Peak is weekday 09:00-22:00 local, Off-Peak is weekday 22:00-09:00 + weekends. But:
- InfluxDB stores all timestamps in UTC. Thailand is UTC+7.
- A naive `EXTRACT(HOUR FROM time)` extracts the UTC hour, not local hour. Peak hours 09:00-22:00 ICT = 02:00-15:00 UTC. Getting this wrong means applying peak rates to off-peak hours and vice versa.
- `EXTRACT(DOW FROM time)` returns the UTC day-of-week, which differs from local DOW near midnight. A Saturday night at 23:00 ICT is Sunday 16:00 UTC — wrong DOW classification.
- Thailand does not observe DST, which simplifies things (fixed UTC+7 offset), but the code must still explicitly handle the conversion.

**Why it happens:** Developers test during daytime when UTC vs local differences may not be obvious. The "weekend" classification fails most visibly at time-zone boundary hours.

**Consequences:** Financial savings calculations can be off by up to 2.2x (the ratio between peak and off-peak rates: 5.7982 / 2.6369). Over a month, this compounds to significant misreporting.

**Prevention:**
- Use InfluxDB 3 SQL's `AT TIME ZONE` operator or the `tz()` function to convert UTC to `'Asia/Bangkok'` before extracting hour/DOW.
- For TOU classification in SQL:
  ```sql
  CASE
    WHEN EXTRACT(DOW FROM time AT TIME ZONE 'Asia/Bangkok') IN (0, 6) THEN 'off_peak'  -- Weekend
    WHEN EXTRACT(HOUR FROM time AT TIME ZONE 'Asia/Bangkok') >= 9
     AND EXTRACT(HOUR FROM time AT TIME ZONE 'Asia/Bangkok') < 22 THEN 'peak'
    ELSE 'off_peak'
  END
  ```
- Note: InfluxDB 3 has `date_bin_wallclock()` and `tz()` functions specifically for wall-clock time operations. Use these for daily aggregations.
- Thailand doesn't have holidays in the TOU definition from what's provided, but validate whether Thai public holidays count as off-peak.
- Test boundary cases: 08:59 ICT (off-peak), 09:00 ICT (peak), 21:59 ICT (peak), 22:00 ICT (off-peak), Saturday 12:00 ICT (off-peak).

**Detection:** Compare savings at known times. If a weekday noon hour is calculated at off-peak rate, the TZ conversion is wrong.

**Confidence:** HIGH — verified `AT TIME ZONE` and `tz()` are available in InfluxDB 3 Core SQL docs. Thailand's fixed UTC+7 simplifies but doesn't eliminate the issue.

**Phase:** Phase 2-3 (financial calculations). Must be validated with known timestamps before trusting financial numbers.

---

### Pitfall 4: Grafana Dashboard JSON Export/Import Datasource UID Mismatch

**What goes wrong:** When Grafana exports a dashboard JSON, every panel's `datasource` field contains a `uid` property that is a unique identifier specific to that Grafana instance. When importing the JSON into a different Grafana instance, the UID won't match, and all panels will show "Datasource not found." The dashboard appears completely broken.

Additionally:
- Grafana 12.x uses `schemaVersion` in the JSON. Importing into an older Grafana may fail silently or lose panel configurations.
- If the datasource `type` is set to `influxdb`, Grafana must be configured with the InfluxDB plugin in the correct mode (SQL for InfluxDB 3). If the import target has the InfluxDB datasource configured for InfluxQL or Flux, the SQL queries will fail.
- Dashboard variables (`__from`, `__to`, `${__interval}`) are Grafana macros. Within raw SQL queries for InfluxDB 3 datasource, Grafana variable substitution may behave differently from InfluxQL mode. The SQL mode uses `$__timeFrom`, `$__timeTo` macros that get translated to SQL timestamp literals.

**Why it happens:** Dashboard JSON is inherently tied to the datasource configuration of the source Grafana instance. This is a well-known Grafana limitation.

**Consequences:** Dashboard is unusable on import. User has to manually fix every panel's datasource reference.

**Prevention:**
- In the exported JSON, use the datasource `name` reference pattern instead of UID. Set datasource to `{"type": "influxdb", "uid": "${DS_INFLUXDB_SOLAR}"}` where `DS_INFLUXDB_SOLAR` is a dashboard variable of type "datasource".
- Better: Use `__inputs` in the dashboard JSON to define datasource inputs that get mapped on import:
  ```json
  "__inputs": [
    {
      "name": "DS_INFLUXDB_SOLAR",
      "label": "influxdb-solar",
      "description": "InfluxDB Solar datasource",
      "type": "datasource",
      "pluginId": "influxdb",
      "pluginName": "InfluxDB"
    }
  ]
  ```
- Document the required datasource name (`influxdb-solar`) and configuration (SQL mode, InfluxDB 3 Core) in a README alongside the JSON.
- Test the import on a clean Grafana instance.

**Detection:** After import, all panels show "No data" or "Datasource not found" error.

**Confidence:** HIGH — this is a universally documented Grafana issue. The `__inputs` mechanism is in Grafana's export/import docs.

**Phase:** Phase 1 (dashboard structure setup) — must be designed from the start with portability in mind.

---

### Pitfall 5: House Load Calculation Energy Balance Errors

**What goes wrong:** The fundamental energy balance equation for a zero-export solar system is:
```
House Load = Solar Production + Grid Import - Grid Export
```
Since this is a zero-export system, export should be ~0, simplifying to:
```
House Load = Solar Production + Grid Import
```

But there are pitfalls:
- **Timing mismatch:** Smart meter and inverter data arrive at slightly different times (~10s intervals but not synchronized). If you query the latest value from each, the timestamps may be seconds apart, causing momentary calculation errors where house load appears negative or impossibly high.
- **Metering point confusion:** The smart meter measures at the grid connection point. If solar production exceeds house consumption, the excess would normally be exported — but with zero-export, the inverter throttles itself. The smart meter's `power_sensor` may briefly show negative values (export) during throttling transients.
- **Cumulative vs instantaneous mixing:** Using `total_import_energy_sensor` (cumulative kWh) with `power_sensor` (instantaneous W) in the same calculation without proper unit conversion.
- **NULL values during nighttime:** At night, solar production is 0 and inverters may stop reporting. If the query returns NULL instead of 0 for solar production, the house load calculation fails.

**Why it happens:** Two independent devices (inverters and smart meter) are not clock-synchronized. Zero-export systems have transient edge cases at the inverter throttle boundary.

**Consequences:** House load shows negative values or unrealistic spikes. Undermines confidence in all dashboard numbers.

**Prevention:**
- Use `COALESCE(solar_value, 0)` to handle NULL solar production.
- For real-time panels, use the Last Value Cache or latest-value queries with a small time window (e.g., last 30 seconds) rather than exact timestamp matching.
- For historical calculations, aggregate both sources to the same time bins (e.g., 1-minute averages) before combining.
- Clamp house load to a minimum of 0: `GREATEST(solar + grid_import, 0)`.
- Add data quality checks: if house load exceeds total inverter rated capacity (4500W) + some margin, flag it.

**Detection:** House load displays negative values or exceeds the total inverter capacity (4500W) for extended periods.

**Confidence:** HIGH — domain-specific knowledge from solar monitoring best practices.

**Phase:** Phase 2 (consumption calculations). Depends on Phase 1 production and grid panels working correctly.

---

## Moderate Pitfalls

---

### Pitfall 6: Measurement Names with Spaces Break SQL Queries

**What goes wrong:** The InfluxDB measurements are named `"East Microinverter"`, `"West Microinverter"`, and `"Smart Meter"` — all containing spaces. In InfluxDB 3 SQL, table names with spaces MUST be double-quoted:
```sql
SELECT * FROM "East Microinverter" WHERE time >= now() - INTERVAL '1 hour'
```
Forgetting the double-quotes causes a parse error. Similarly, field names with special characters need quoting.

**Why it happens:** Most SQL examples use simple table names. The InfluxDB schema was designed with line protocol naming conventions (spaces are valid in measurement names) but SQL syntax requires quoting.

**Prevention:**
- ALWAYS double-quote measurement names in SQL: `"East Microinverter"`, `"West Microinverter"`, `"Smart Meter"`.
- Consider creating a query template/snippet library at the start of the project.
- Test every query with these exact table names.

**Detection:** SQL parse errors like "table not found" or "unexpected identifier."

**Confidence:** HIGH — verified in InfluxDB 3 Core SQL reference documentation.

**Phase:** Phase 1 (every single query).

---

### Pitfall 7: Grafana 12 SQL Mode Requires Flight SQL (gRPC / HTTP/2)

**What goes wrong:** The Grafana InfluxDB plugin in SQL mode uses Flight SQL protocol (gRPC) to query InfluxDB 3, which requires HTTP/2. Key issues:
- If Grafana connects through any proxy (nginx, HAProxy, reverse proxy) that doesn't support HTTP/2, SQL queries will fail with connection errors.
- The datasource configuration in Grafana 12.2+ requires selecting "InfluxDB Enterprise 3.x" as the product (there's currently no separate "Core" option).
- Without TLS/SSL, you must explicitly enable the "Insecure Connection" option in Advanced Database Settings.
- The `newInfluxDSConfigPageDesign` feature flag may need to be enabled for the latest InfluxDB data source plugin UI.

**Why it happens:** InfluxDB 3 is a new architecture. The Grafana plugin is still catching up with naming conventions and configuration options.

**Consequences:** Dashboard fails to load any data. Confusing error messages about connection failures.

**Prevention:**
- Verify the InfluxDB datasource is configured as "InfluxDB Enterprise 3.x" with SQL query language.
- If not using TLS, enable "Insecure Connection" under Advanced Database Settings.
- If using a reverse proxy, ensure HTTP/2 is enabled.
- Include datasource setup instructions alongside the dashboard JSON.

**Detection:** "Query error" on all panels, connection timeout errors in Grafana logs.

**Confidence:** HIGH — verified in official Grafana InfluxDB docs for v12.4 and InfluxDB 3 Core docs.

**Phase:** Phase 0 (datasource setup validation before any dashboard work).

---

### Pitfall 8: High-Resolution Data Performance with Unbounded Queries

**What goes wrong:** At ~10-second reporting intervals with 3 measurements, the system generates ~26,000 rows/day or ~780,000 rows/month. Querying large time ranges without aggregation:
- A "Last 30 days" panel querying raw data will attempt to load ~780K rows PER measurement — millions of data points for a single panel.
- Multiple panels on one dashboard compound this — the dashboard could attempt to load 10+ million rows simultaneously.
- `date_bin_gapfill()` in InfluxDB 3 does NOT support month or year intervals — only up to weeks. This limits some long-range aggregation approaches.

**Why it happens:** Developers test with small datasets (hours/days) and everything is fast. Performance degrades as data accumulates over weeks/months.

**Consequences:** Dashboard load times exceed 30 seconds. Grafana times out. Browser tab crashes.

**Prevention:**
- Use `DATE_BIN()` aggregation for ALL time-series panels. Scale the bin size with the time range:
  - Last 1 hour: raw data or 1-minute bins
  - Last 24 hours: 5-minute bins
  - Last 7 days: 30-minute bins
  - Last 30 days: 1-hour bins
  - Last year: 1-day bins
- Use Grafana's `$__interval` variable to dynamically adjust bin size, or use `INTERVAL '$__interval'` if supported by the SQL datasource.
- Add `ORDER BY time` and `LIMIT` clauses as safety nets.
- Consider setting up InfluxDB 3's Last Value Cache for real-time "current value" stat panels — this avoids full table scans for latest values.
- For monthly/yearly aggregations, use `date_trunc('month', time)` since `date_bin()` supports months but `date_bin_gapfill()` does not.

**Detection:** Dashboard load time > 5 seconds. Grafana query inspector shows row counts > 10K per panel.

**Confidence:** MEDIUM-HIGH — performance characteristics inferred from data volume calculations and InfluxDB 3 docs on date_bin_gapfill limitations.

**Phase:** Phase 1-2 (query design). Must be designed correctly from the start, painful to retrofit.

---

### Pitfall 9: Nighttime NULL Handling and Zero Solar Production

**What goes wrong:** Micro inverters typically shut down at night when there's no sunlight. They may:
- Stop reporting entirely (no rows in InfluxDB for nighttime hours)
- Report zeros for all power fields
- Report NULLs for some fields while others retain last values

This causes multiple issues:
- Time-series graphs show gaps or drop to zero
- Daily production calculations may miss early morning or late evening data
- Average calculations are skewed — `AVG()` excludes NULLs, but including zero-value nighttime readings pulls averages down incorrectly
- Panel thresholds and color-coding designed for daytime values display misleadingly at night

**Why it happens:** Solar inverters are designed to sleep when there's no power to convert. The data pipeline may not insert zero-value rows.

**Prevention:**
- Use `COALESCE(field, 0)` for power fields in calculations.
- For time-series visualizations, use `date_bin_gapfill()` with `locf()` to fill gaps (last value carried forward shows the inverter went to sleep and stayed there).
- Design panels to gracefully handle null/zero states — show "0 W" at night, not "No Data."
- For daily averages of production, filter to only sunlight hours (e.g., 06:00-18:00 local) or use `AVG(NULLIF(power, 0))` to only average non-zero values where appropriate.

**Detection:** Gaps in time-series charts. "No Data" errors on stat panels during early morning/late evening.

**Confidence:** HIGH — standard solar monitoring behavior.

**Phase:** Phase 1 (basic panels) — handle from the start.

---

### Pitfall 10: Dashboard Variable Macros in Raw SQL for InfluxDB 3

**What goes wrong:** Grafana provides time range macros for query filtering, but their syntax differs between InfluxQL and SQL mode for the InfluxDB datasource:
- In SQL mode, use `$__timeFrom` and `$__timeTo` which resolve to SQL timestamp strings.
- The older `$timeFilter` or `$__timeFilter` macros from InfluxQL mode may not work in raw SQL mode.
- Template variables using `${variable}` syntax may not interpolate correctly in raw SQL strings depending on the query type.
- The `$__interval` macro for dynamic time binning may need special handling — it produces a duration string like `'10s'` that must be wrapped in `INTERVAL` keyword.

**Why it happens:** The Grafana InfluxDB SQL plugin is relatively new. Documentation is sparse and community examples mostly cover InfluxQL.

**Prevention:**
- Test macro substitution using Grafana's query inspector (shows the actual SQL sent).
- Use explicit time filters in WHERE clauses: `WHERE time >= $__timeFrom AND time <= $__timeTo`.
- For dynamic binning, test whether `DATE_BIN(INTERVAL '$__interval', time)` works or if you need to use a fixed interval.
- Avoid over-reliance on macros initially — start with hardcoded time ranges, then convert to macros once the base query works.

**Detection:** Grafana query inspector shows unsubstituted macro strings like `$__timeFrom` in the SQL. Panel returns error or no data.

**Confidence:** MEDIUM — specific macro behavior in Grafana 12 SQL mode for InfluxDB 3 may need runtime validation. Training data may be stale.

**Phase:** Phase 1 (query construction). Test early with query inspector.

---

## Minor Pitfalls

---

### Pitfall 11: Panel Type Compatibility in Grafana 12

**What goes wrong:** Grafana 12 has evolved panel types from earlier versions:
- The "Gauge" panel replaces older gauge visualizations
- "Stat" panel has different options than v8/v9 era
- Canvas panel (needed for roof layout visualization) may have different element types
- Some community dashboard JSON files reference deprecated panel types or plugins that aren't available in v12

**Prevention:**
- Use only core panel types available in Grafana 12.4.1: Time Series, Stat, Gauge, Bar Chart, Table, Canvas, Heatmap, State Timeline.
- For the roof layout visualization, the Canvas panel or a custom SVG-based approach will be needed — verify Canvas panel capabilities in 12.4.
- Don't reference external/community plugins unless absolutely necessary.

**Detection:** Panel shows "Unknown panel type" or a generic fallback rendering.

**Confidence:** MEDIUM — panel types are generally stable, but Canvas panel capabilities need runtime validation.

**Phase:** Phase 1-3 (varies by panel).

---

### Pitfall 12: Incorrect Aggregation of Per-Panel Production Data

**What goes wrong:** With 8 panels across 2 inverters (4 channels each), aggregating production data requires care:
- Summing `pv1_power + pv2_power + pv3_power + pv4_power` from BOTH inverters gives total panel-level power, but this ignores inverter conversion losses (`power_losses_sensor`).
- Total system power should come from `power_sensor` on each inverter (which is after losses), not from summing individual PV channels.
- When creating "East vs West" comparisons, make sure you're querying the correct measurement — `"East Microinverter"` vs `"West Microinverter"`.
- The physical layout mapping (`PV3, PV4, PV1, PV2` left-to-right) doesn't match the logical numbering — mixing these up in the roof layout visualization means the heatmap shows the wrong panel positions.

**Prevention:**
- Use `power_sensor` from each inverter for total production, NOT sum of PV channels.
- Use PV channel data only for module-level comparisons and fault detection.
- Document the PV-to-physical-position mapping explicitly in the code.
- Create a clear mapping constant: `East: [PV3=LeftMost, PV4, PV1, PV2=RightMost]`

**Detection:** Sum of PV channels doesn't match `power_sensor` value (off by `power_losses_sensor` amount).

**Confidence:** HIGH — clearly stated in PROJECT.md panel layout.

**Phase:** Phase 1-2 (production panels and roof layout).

---

### Pitfall 13: Thai Baht Currency Formatting

**What goes wrong:** Financial savings should display in Thai Baht (THB / ฿). Grafana's unit system supports many currencies but:
- The THB symbol may not render correctly in all Grafana themes.
- If using Grafana's built-in unit system, select "currency: THB" not "currency: USD" or similar.
- Custom value formats with `฿` prefix may have encoding issues in JSON export.
- Savings calculations must multiply energy (kWh) by the correct rate, not power (W). A common mistake: `power_W * rate_per_kWh` gives the wrong unit (should be `energy_kWh * rate`).

**Prevention:**
- Use Grafana's native currency formatting with THB.
- Verify the `฿` symbol appears correctly in the exported JSON (UTF-8 encoding).
- Add unit labels to every financial panel.

**Detection:** Currency shows as `$` instead of `฿`, or financial values seem ~30x off (1 USD ≈ 34 THB).

**Confidence:** MEDIUM — currency support depends on Grafana version/theme.

**Phase:** Phase 3 (financial calculations).

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Datasource setup | Pitfall 7: Flight SQL / HTTP/2 requirement | Validate connection before starting dashboard work |
| Basic production queries | Pitfall 1: Flux/InfluxQL syntax traps | Use only InfluxDB 3 SQL docs as reference |
| Basic production queries | Pitfall 6: Spaces in measurement names | Always double-quote table names |
| Energy calculations | Pitfall 2: Counter reset handling | Use LAG() with negative-difference filtering |
| Energy calculations | Pitfall 9: Nighttime NULL handling | COALESCE all power fields |
| Consumption/grid panels | Pitfall 5: Energy balance timing | Aggregate to same time bins before combining |
| TOU financial calculations | Pitfall 3: UTC vs local time | AT TIME ZONE 'Asia/Bangkok' on every time extraction |
| Financial calculations | Pitfall 13: Currency formatting | Use native THB unit, verify in JSON |
| Roof layout visualization | Pitfall 12: PV position mapping | Document physical-to-logical mapping |
| Long-range historical views | Pitfall 8: Performance with raw data | DATE_BIN aggregation scaled to time range |
| Dashboard export | Pitfall 4: UID mismatch | Use __inputs and datasource variables |
| All queries | Pitfall 10: Macro substitution | Test with query inspector immediately |

## Sources

- InfluxDB 3 Core SQL Reference (official docs): https://docs.influxdata.com/influxdb3/core/reference/sql/ [HIGH confidence]
- InfluxDB 3 Core SQL Functions — Time and Date: https://docs.influxdata.com/influxdb3/core/reference/sql/functions/time-and-date/ [HIGH confidence]
- InfluxDB 3 Core SQL Aggregate Queries: https://docs.influxdata.com/influxdb3/core/query-data/sql/aggregate-select/ [HIGH confidence]
- InfluxDB 3 Core + Grafana Integration: https://docs.influxdata.com/influxdb3/core/visualize-data/grafana/ [HIGH confidence]
- InfluxDB 3 Core SQL Compare Values (LAG/LEAD): https://docs.influxdata.com/influxdb3/core/query-data/sql/compare-values/ [HIGH confidence]
- Grafana 12.4 Dashboard Import Docs: https://grafana.com/docs/grafana/v12.4/dashboards/build-dashboards/import-dashboards/ [HIGH confidence]
- Grafana 12.0 What's New: https://grafana.com/docs/grafana/v12.4/whatsnew/whats-new-in-v12-0/ [HIGH confidence]
- PROJECT.md (project context with schema, TOU rates, hardware specs) [HIGH confidence]
- Solar monitoring domain knowledge (cumulative counters, energy balance) [HIGH confidence — domain fundamentals]
- Grafana SQL macro behavior in InfluxDB 3 mode [MEDIUM confidence — may need runtime validation]
