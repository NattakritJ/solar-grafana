# Phase 10: Temporal Alignment — Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-04
**Phase:** 10-temporal-alignment
**Areas discussed:** Fix strategy, Window size, Module Detail column handling

---

## Live Data Investigation

Before presenting gray areas, the user requested a live database query to characterize each device's logging behavior.

**InfluxDB 3 queries run against `solar` database at `192.168.2.10:8181`:**
- `ORDER BY time DESC LIMIT 10` on each table to determine log interval and phase offset

**Findings:**

| Device | Log Interval | Phase Offset | Max Data Age at Arbitrary Query |
|--------|-------------|-------------|--------------------------------|
| East Microinverter | ~10s | :08, :18, :28... | 0–10s |
| West Microinverter | ~10s | :07, :17, :27... | 0–10s |
| Smart Meter | ~10s | :09, :19, :29... | 0–10s (now ONLY used for frequency/PF) |
| grid CT | ~2s | every ~2s | 0–2s |
| 235_floor_1 | ~2s | every ~2s | 0–2s |
| 235_floor_2 | ~2s | every ~2s | 0–2s |

**Key insight:** East/West inverters and Smart Meter are all ~10s interval, staggered ~1s apart from each other (minimal skew). The CT meters (grid, floor_1, floor_2) log every 2 seconds — 5× faster. With `ORDER BY time DESC LIMIT 1`, a cross-device expression like Self-Consumption can combine an inverter reading up to 10s old with a CT reading that's only 1s old — a 9-second mismatch window.

Additional finding: East Inverter output was stable at 58.8W for 5+ consecutive minutes (inverters output stable quantized values), confirming that `AVG` over 2 minutes would produce essentially the same result as `LIMIT 1` for typical inverter operation.

---

## Fix Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| AVG over 2-minute window | Replace selector_last/LIMIT 1 with AVG(field) over existing 2-min window. Production counters use MAX. State/alarm/fault kept unchanged. | ✓ |
| AVG over 30-second window | Tighter window, more "current" feel. 3 inverter readings, 15 CT readings. | |
| DATE_BIN anchor approach | Align all devices to same completed time bucket. Adds latency. | |

**User's choice:** AVG over 2-minute window (recommended default)
**Notes:** The existing 2-minute window from Phase 7 is preserved — only the aggregation function changes.

---

## Window Size

| Option | Description | Selected |
|--------|-------------|----------|
| 2 minutes (keep as-is) | Matches Phase 7 decision. ~12 readings per 10s-device, ~60 readings per 2s CT. | ✓ |
| 30 seconds | Tighter, more responsive. | |
| 5 minutes | Broader coverage for low-irradiance hours. | |

**User's choice:** 2 minutes — keep as-is
**Notes:** Consistent with Phase 7 recency window decision. No change to window duration.

---

## Module Detail Column Handling (Panel 23)

| Option | Description | Selected |
|--------|-------------|----------|
| AVG for power/VC, MAX for counters | Power/Voltage/Current → AVG; Today/Total kWh → MAX. Same rules as all other panels. | ✓ |
| MAX for all columns | Use MAX for everything including power/voltage/current. | |
| Keep Module Detail unchanged | Module Detail stays on selector_last. | |

**User's choice:** AVG for power/voltage/current, MAX for production counters
**Notes:** Consistent rule applied uniformly — no special-casing for the detail table.

---

## Agent's Discretion

- Exact sequencing of replaceAll operations
- Whether to use single regex or per-field replacements for selector_last
- Handling of COALESCE fallback values in edge cases

## Deferred Ideas

- Today's Grid Import energy stat using `grid.energy_today` — future phase
- Air conditioner monitoring (`air_conditioner` table) — future phase
- 30-second AVG window option for "wall display" mode — deferred
