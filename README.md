# Revenue Pacing Dashboard — Global

**Live site:** [rev-pacing-global.quick.shopify.io](https://rev-pacing-global.quick.shopify.io)

All-segment, all-region pacing dashboard for Revenue Marketing. Queries BigQuery live for actuals, targets, velocity, pipeline coverage, and sales leader forecasts.

## Segments & Regions

| Segment | `owner_team` filter | Region |
|---|---|---|
| Mid-Market | `D2C Retail Mid-Mkt` | AMER, EMEA, APAC |
| Large Accounts | `D2C Retail Large` | AMER, EMEA, APAC |
| B2B | `LIKE '%B2B%'` | AMER, EMEA |
| D2C SMB Acq | `D2C SMB Acquisition` | AMER, EMEA, APAC |
| Retail SMB Acq | `Retail SMB Acquisition` | AMER, EMEA, APAC |
| SMB Cross-Sell | `D2C Retail SMB Cross-Sell` | AMER, EMEA, APAC |
| Enterprise | `Enterprise` | AMER, EMEA |
| Global Account | `Global Account` | AMER, EMEA |

## Shared Filters

All queries apply these base filters:

```sql
owner_team IN ('D2C SMB Acquisition','Retail SMB Acquisition','D2C Retail Mid-Mkt',
               'D2C Retail Large','Enterprise','Global Account',
               'B2B Large Mid-Mkt Acquisition','B2B Large Mid-Mkt Cross-Sell')
AND COALESCE(owner_line_of_business,'Unknown') NOT IN ('Ads','Lending')
AND source_channel IN ('Inbound','Outbound','Partner')
```

---

## Data Sources

| Table | What it provides |
|---|---|
| `sdp-for-analysts-platform.rev_ops_prod.temp_sales_performance` | Actuals (SQL/SAL/CW counts + PBR), Targets (count + PBR by stage), Open Pipeline |
| `sdp-for-analysts-platform.rev_ops_prod.modelled_salesforce_forecast` | Sales Leader CW PBR Forecast (the number sales leadership commits to landing) |
| `sdp-for-analysts-platform.rev_ops_prod.pipeline_coverage_weekly_archive_2026` | Weekly pipeline snapshots for coverage calculation |

### Migration Note

REI Data is migrating to `shopify-dw.mart_revenue_data.sales_performance_cohort_daily_summary`. Actuals are at parity. Targets are blocked on a `owner_motion` JOIN mismatch (Cheok, REI Data, is fixing). CW PBR targets are temporarily hardcoded in `CW_PBR_OVERRIDES` in `index.html`. Remove that block once the DW table targets are populated.

---

## Pacing Methodology

### Funnel Stages

Three pipeline stages, each anchored to its own date field:

| Stage | Date Field | Count Column | PBR Column |
|---|---|---|---|
| **SQL** (Created) | `created_date` | `created_opportunity_count` | `created_opportunity_projected_billed_revenue` |
| **SAL** (Qualified) | `qualified_date` | `sal_count` | `qualified_opportunity_projected_billed_revenue` |
| **CW** (Closed Won) | `close_date` + `is_won = TRUE` | `cw_count` | `closed_won_projected_billed_revenue` |

Date fields are never mixed across stages.

### Best Estimate Algorithm

Projected end-of-quarter performance for each segment × source channel × stage:

**Step 1: YTD Actuals**
- Sum counts and PBR from quarter start through today.

**Step 2: L4W Velocity**
- Calculate average weekly counts and PBR for the **last 4 completed ISO weeks** (excludes the current partial week).
- Computed per segment × source channel × stage.

**Step 3: Outlier Removal**
- For each segment × source × stage combination, identify individual deals where PBR exceeds `mean + 3σ` of that group.
- Remove outlier PBR from the YTD total → "scrubbed YTD."
- Remove outlier PBR from the L4W weekly averages → "scrubbed velocity."
- Outlier removal applies to PBR only, not counts.

**Step 4: Projection**
```
best_est_count = YTD_count + (L4W_avg_weekly_count × remaining_weeks × 0.90)
best_est_pbr   = scrubbed_YTD_pbr + (scrubbed_L4W_avg_weekly_pbr × remaining_weeks × 0.85)
```

| Parameter | Value | Rationale |
|---|---|---|
| Count damper | 0.90 | 10% buffer — assumes velocity slightly declines toward EoQ |
| PBR damper | 0.85 | 15% buffer — PBR is more volatile than counts |
| Velocity window | 4 weeks | Recent enough to capture trends, long enough to smooth noise |
| Outlier threshold | 3σ | Standard statistical threshold for extreme values |
| Remaining weeks | `(quarter_end - today) / 7` | Fractional weeks included |

**Step 5: CW PBR Exception**
- CW PBR best estimate uses the **Sales Leader Forecast** from `modelled_salesforce_forecast`, not the velocity formula.
- This is the number Sales leadership commits to landing. The latest forecast snapshot for the current quarter is used.
- If no Sales Leader Forecast is available, falls back to the velocity-based Best Estimate.

### Pace Status

Two pace indicators are used:

**YTD Pace (dot in table rows)** — compares current % to target against linear pace:

| Condition | Status |
|---|---|
| `% to target ≥ linear_pace − 5pp` | 🟢 Green |
| `% to target ≥ linear_pace − 15pp` | 🟡 Yellow |
| `% to target < linear_pace − 15pp` | 🔴 Red |

Where `linear_pace = days_elapsed / total_days_in_quarter`.

**Best Estimate Pace (hero cards)** — compares projected EoQ to full target:

| Condition | Status |
|---|---|
| `≥ 95%` of target | 🟢 Green |
| `80–94%` of target | 🟡 Yellow |
| `< 80%` of target | 🔴 Red |

### Pipeline Coverage

```
Coverage = Open Pipe PBR / (CW PBR Target − CW PBR Actuals)
```

- **Open pipe PBR** is scoped by close date — Q2 pipe = deals with close dates in Q2.
- **CW PBR Target** comes from the target table (or CW_PBR_OVERRIDES when the target query returns NULL).
- If remaining ≤ 0 (target exceeded), shows `--`. Capped at 10x.
- Channels: Inbound + Outbound + Partner (all three, no exclusions).
- Next-quarter coverage uses Sales Leader Forecast as the pipe estimate.

Key rule: Open pipe is sourced from `temp_sales_performance`, NOT from the archive table or forecast table. Scoped by close date to match how RevOps calculates it.

---

## Architecture

Single-file Quick site (`index.html`). No backend, no build step. Uses:
- `quick.dw.querySync()` — BigQuery queries from the browser via Quick's Data Warehouse API
- `quick.auth.requestScopes()` — OAuth for BigQuery access
- Chart.js — velocity charts (stacked bar by source channel)

All 8 queries run in parallel per region. Results are cached client-side per region × quarter to avoid redundant queries when switching segments.

## Deploy

Merges to `main` auto-deploy via the Quick GitHub Action. Manual deploy:

```bash
quick deploy . rev-pacing-global
```
