# Phase 1: Foundation & Overview Stats - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-30
**Phase:** 01-foundation-overview-stats
**Areas discussed:** Stat panel layout & density, Power flow visualization, Dashboard scaffolding & rows, SQL query strategy

---

## Stat Panel Layout & Density

| Option | Description | Selected |
|--------|-------------|----------|
| Single compact KPI row | Dense 24-col row of 8 stat panels, transparent, tight spacing | ✓ |
| Multi-row arrangement | Larger panels spread over 2+ rows, more spacious | |
| Mixed sizes | Hero stat for total power, smaller stats for others | |

**User's choice:** "Let you decide all" — agent selected single compact KPI row
**Notes:** Agent chose the single-row approach to maximize screen real estate for charts in later phases.

---

## Power Flow Visualization

| Option | Description | Selected |
|--------|-------------|----------|
| Three labeled stat panels (Solar / House / Grid) | Simple, no custom elements, reuses stat panel type | ✓ |
| Canvas panel with arrows | Visual arrows showing flow direction, more complex | |
| Text panel with Markdown layout | Markdown table or layout with unicode arrows | |

**User's choice:** "Let you decide all" — agent selected three labeled stat panels
**Notes:** Simpler approach chosen for Phase 1; can be enhanced with Canvas in Phase 4 if desired.

---

## Dashboard Scaffolding & Rows

| Option | Description | Selected |
|--------|-------------|----------|
| Full skeleton upfront (all 7 rows) | Phase 1 creates all rows; later phases fill them in | ✓ |
| Incremental rows (add per phase) | Each phase adds its own rows to the JSON | |
| Single flat structure | No collapsible rows, just panels in order | |

**User's choice:** "Let you decide all" — agent selected full skeleton upfront
**Notes:** Avoids JSON restructuring in later phases; collapsed placeholder rows have zero visual impact.

---

## SQL Query Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Latest-value queries only (Phase 1) | `WHERE time >= now() - INTERVAL '1 minute' ORDER BY time DESC LIMIT 1` | ✓ |
| Time-range macros from Phase 1 | Use `$__timeFrom`/`$__timeTo` in all queries | |
| Hardcoded time windows | Explicitly hardcode last 24h or similar | |

**User's choice:** "Let you decide all" — agent selected latest-value queries for Phase 1
**Notes:** #1 risk flagged in STATE.md is macro syntax validation. Using latest-value queries first validates SQL connectivity without macro complexity. Macros deferred to Phase 2.

---

## the Agent's Discretion

All four gray areas were delegated to the agent ("Let you decide all"). Decisions made:
- Compact single KPI row for stats
- Three labeled stat panels for power flow (not canvas)
- Full dashboard skeleton with 7 placeholder rows created in Phase 1
- Latest-value SQL queries (`LIMIT 1`) for all Phase 1 stat panels; `$__timeFrom`/`$__timeTo` macros deferred to Phase 2

## Deferred Ideas

- Time-range macro validation → deferred to Phase 2 (required for production charts)
- Sparklines on stat panels → agent discretion (low priority)
- Canvas-based power flow diagram → consider for Phase 4 polish
- Grafana annotations (sunrise/sunset) → Phase 4
