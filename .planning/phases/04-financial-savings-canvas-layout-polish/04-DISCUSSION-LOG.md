# Phase 4: Financial Savings, Canvas Layout & Polish - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-30
**Phase:** 04-financial-savings-canvas-layout-polish
**Areas discussed:** TOU savings display, Canvas roof layout, Calendar heatmap

---

## TOU Savings Display

### Top-level savings presentation

| Option | Description | Selected |
|--------|-------------|----------|
| Stats row | 3 stat panels: Today's Savings, MTD, YTD. Simple, scannable. | ✓ |
| Stats + savings trend chart | Today's stat + cumulative savings line chart. More visual but more space. | |
| Stats + daily stacked bars | Stats + stacked bar chart (peak vs off-peak THB). Rich but complex. | |

**User's choice:** Stats row (Recommended)
**Notes:** Consistent with Phase 1 overview stats style.

### Peak vs off-peak breakdown format

| Option | Description | Selected |
|--------|-------------|----------|
| Stat panel row | 4 stat panels: Peak kWh, Peak THB, Off-Peak kWh, Off-Peak THB. | ✓ |
| Table panel | Table with rows: Period, kWh, Rate, THB. Supports future rate tiers. | |
| Pie chart split | Donut chart showing proportion. Visual but less precise. | |

**User's choice:** Stat panel row (Recommended)
**Notes:** None.

### Thai public holidays in TOU

| Option | Description | Selected |
|--------|-------------|----------|
| Weekends only | Only Sat/Sun as off-peak. Simple, no maintenance. | |
| Include Thai holidays | Hard-code 15-20 holiday dates/year. More accurate, requires annual update. | ✓ |
| Note only | No logic, just a note about holiday adjustment. | |

**User's choice:** Include Thai holidays
**Notes:** User chose accuracy over simplicity. Researcher needs to compile 2026 Thai public holiday list.

### THB currency formatting

| Option | Description | Selected |
|--------|-------------|----------|
| Suffix ' THB' | Grafana 'none' unit with suffix. Guaranteed to work everywhere. | ✓ |
| ฿ symbol prefix | Built-in currency unit if available. May not render correctly. | |
| You decide | Agent picks. | |

**User's choice:** Suffix ' THB' (Recommended)
**Notes:** None.

---

## Canvas Roof Layout

### Panel representation style

| Option | Description | Selected |
|--------|-------------|----------|
| Rectangles + text overlay | 8 colored rectangles with W value as text. Color = production level. | ✓ |
| Rectangles + full metrics | Same but with voltage and current as smaller text too. Denser. | |
| Color-only heatmap | Rectangles with color only, no text. Requires hover for values. | |

**User's choice:** Rectangles + text overlay (Recommended)
**Notes:** None.

### Panel labeling

| Option | Description | Selected |
|--------|-------------|----------|
| Logical names | "East PV1", "East PV2", etc. with "East Slope"/"West Slope" row labels. | ✓ |
| Physical position labels | "E1 (leftmost)", etc. More intuitive for roof-looking. | |
| Both logical + position | Combined. Most complete but busier. | |

**User's choice:** Logical names (Recommended)
**Notes:** Matches Phase 3 table panel naming convention (D-26).

### Canvas orientation

| Option | Description | Selected |
|--------|-------------|----------|
| East top, West bottom | Natural top-down view. | |
| West top, East bottom | User-specified orientation. | ✓ |
| Side by side | East left, West right. | |

**User's choice:** West top, East bottom
**Notes:** User explicitly chose this over the recommended option.

### Panel position ordering (custom)

User specified a custom left-to-right ordering after multiple clarification rounds:

```
West (top):    [PV2] [PV1] [PV4] [PV3]
East (bottom): [PV3] [PV4] [PV1] [PV2]
```

**Notes:** East row matches PROJECT.md physical layout. West row is mirrored. This is an intentional user choice — the Canvas must implement this exact ordering.

---

## Calendar Heatmap

### Panel type

| Option | Description | Selected |
|--------|-------------|----------|
| Native heatmap panel | Built-in Grafana heatmap with daily buckets and color gradient. | ✓ |
| Table as calendar grid | Table with week rows, day columns, colored cells. GitHub-style. | |
| Color-gradient bar chart | Daily bars with color gradient. Simpler but less calendar-like. | |

**User's choice:** Native heatmap panel (Recommended)
**Notes:** None.

### Color scheme

| Option | Description | Selected |
|--------|-------------|----------|
| Solar production scale | Navy to amber to dark-green. Consistent with D-27/D-47. | ✓ |
| Grafana green default | Standard heatmap green scheme. | |
| You decide | Agent picks. | |

**User's choice:** Solar production scale (Recommended)
**Notes:** Maintains color consistency across all production visualizations.

### Default time range

| Option | Description | Selected |
|--------|-------------|----------|
| 90 days default | ~3 months of daily patterns. Good for trend visibility. | ✓ |
| 30 days default | More focused, less trend data. | |
| Follow time picker | Same as other panels. May look sparse or overwhelming. | |

**User's choice:** 90 days default (Recommended)
**Notes:** None.

### Panel placement

| Option | Description | Selected |
|--------|-------------|----------|
| Production row | Under row 200 with existing production charts. | ✓ |
| Financial Savings row | Under row 600 as complement to savings. | |
| New Analytics row | Separate row for long-range views. | |

**User's choice:** Production row (Recommended)
**Notes:** None.

---

## Agent's Discretion

- Dashboard polish scope (not selected for discussion — agent handles)
- Canvas element sizing, styling, and exact positioning
- Sparklines on savings stat panels
- Panel IDs and gridPos coordinates
- SQL implementation details for TOU calculations
- Heatmap 90-day default implementation approach

## Deferred Ideas

- Sunrise/sunset annotations — low priority polish item
- v2 requirements (animated flow, CO2 offset, efficiency gauge, alerting)
