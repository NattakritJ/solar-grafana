# Phase 8: CT Meters, Per-Floor Distribution & Reconciliation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-02
**Phase:** 08-update-house-load-measurement-to-use-ct-meters-on-both-consumer-units
**Areas discussed:** CT Data Source & Measurement Setup, House Load Replacement Strategy, Per-Floor Power Distribution Panels, Data Reconciliation Panel Design

---

## CT Data Source & Measurement Setup

| Option | Description | Selected |
|--------|-------------|----------|
| Already in InfluxDB | CT data is already flowing and queryable | ✓ |
| Installed but not in InfluxDB | Meters installed, no pipeline yet | |
| Not yet installed | Planned only | |

**User's choice:** Already in InfluxDB

| Option | Description | Selected |
|--------|-------------|----------|
| One measurement, multiple fields | Single table with floor1_power, floor2_power | |
| Separate measurements per unit/floor | Separate tables per consumer unit | ✓ |

**User's choice:** Separate measurements per unit/floor

**Agent discovered live data:** Queried InfluxDB directly (http://192.168.2.10:8181, database: solar). Found two measurements: `235_floor_1` and `235_floor_2`, each with fields: power (W), energy (Wh cumulative), current (A), voltage (V), frequency (Hz), power_factor. 10-second reporting. Data started 2026-04-02T11:34 UTC.

**CT Datasheet consulted:** `ct_datasheet.txt` (PZEM-014/016) — confirmed power field = active power W (0.5% accuracy), energy = cumulative Wh counter (can be reset via software command), PZEM-016 = 100A external CT transformer.

---

## House Load Panel Replacement Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Replace with CT data | Panels 5 and 10 use CT direct (floor1+floor2) | ✓ |
| Keep derived, add CT extra | Both methods visible, no replacement | |
| CT primary, derived fallback | CT main with derived as subtitle | |

**User's choice:** Replace with CT data (recommended)

**Follow-up — Self-Consumption panel:**
User asked about implications for Self-Consumption panel (panel 6). Agent explained current formula: `Solar ÷ (Solar + Grid)` — mathematically equivalent to `Solar ÷ CT_total` for zero-export systems.

| Option | Description | Selected |
|--------|-------------|----------|
| Keep current formula | Solar / (Solar + Grid) via smart meter | ✓ |
| Switch to CT-based | Solar / (floor1+floor2) | |

**User's choice:** Keep current formula — smart meter is more proven, equivalent result.

---

## Per-Floor Power Distribution Panels

| Option | Description | Selected |
|--------|-------------|----------|
| Stat panels (power + energy) | 4 stats: power + today energy per floor | |
| Stat panels (power only) | 2 stats: current power per floor | |
| Bar gauge | Both floors as horizontal bars | |
| Both stat + time-series | Separate current view and historical chart | (modified) |

**User's choice:** Power-only stat panels, AND update Power Profile time-series to include Floor 1 and Floor 2 series alongside Production, Grid, Load.

**User clarification on energy:** "CT doesn't have today energy. It's an accumulate energy since meter start. Use today's energy data from current smart meter only." — confirmed `energy` field is cumulative, not daily-reset.

**Panel location:**

| Option | Description | Selected |
|--------|-------------|----------|
| New 'Power Distribution' row | Between Overview and Power Profile | ✓ |
| Add to Overview row | Alongside Solar Power, Grid Import, etc. | |
| Add to Grid & Consumption | In existing grid section | |

**User's choice:** New 'Power Distribution' row (recommended)

---

## Data Reconciliation Panel Design

| Option | Description | Selected |
|--------|-------------|----------|
| Stat: current delta in watts | Single stat showing CT - Calculated | |
| Time-series: overlay | Historical chart of both methods | |
| Both stat + time-series | Combined view | |
| Two stats side by side | CT Load and Calculated Load as separate stats | ✓ |

**User's choice:** Two stat panels side by side — CT Load (floor1+floor2) and Calculated Load (solar+grid)

**Delta display format:**

| Option | Description | Selected |
|--------|-------------|----------|
| Absolute delta (W) | One stat showing difference with color thresholds | |
| Percentage deviation (%) | Relative difference | |
| Two stats side by side | CT total and Calculated total as separate panels | ✓ |

**User's choice:** Two stats side by side for direct comparison (no delta calculation needed)

**Location:**

| Option | Description | Selected |
|--------|-------------|----------|
| In Power Distribution row | Alongside Floor 1 and Floor 2 stats | ✓ |
| Separate Reconciliation row | Own dedicated section | |
| In Grid & Consumption row | Near grid measurement panels | |

**User's choice:** In Power Distribution row — all CT-related info in one section

---

## Agent's Discretion

- Exact gridPos values for new panels
- Color of CT Load / Calculated Load reconciliation panels
- Row ID number for new Power Distribution row (suggested: 900)
- Whether Power Distribution row description/label text is needed

## Deferred Ideas

None noted during discussion.
