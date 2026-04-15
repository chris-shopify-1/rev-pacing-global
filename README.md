# Revenue Pacing Dashboard — Global

**Live site:** [rev-pacing-global.quick.shopify.io](https://rev-pacing-global.quick.shopify.io)

All-segment, all-region pacing dashboard for Revenue Marketing. Queries BigQuery live for actuals, targets, velocity, pipeline coverage, and sales leader forecasts.

## Segments
D2C SMB Acq, Retail SMB Acq, Mid-Market, Large Accounts, Enterprise, Global Account, B2B

## Regions
AMER, EMEA, APAC

## How it works
Single-file Quick site (`index.html`) using `quick.dw` to query BigQuery directly from the browser. No backend. Auto-deploys on merge to main via GitHub Actions.

## Data sources
- **Actuals + Targets:** `sdp-for-analysts-platform.rev_ops_prod.temp_sales_performance`
- **Sales Leader Forecast:** `sdp-for-analysts-platform.rev_ops_prod.modelled_salesforce_forecast`
- **Pipeline Coverage:** `sdp-for-analysts-platform.rev_ops_prod.pipeline_coverage_weekly_archive_2026`

> **Note:** CW PBR targets are temporarily hardcoded as overrides (`CW_PBR_OVERRIDES` in index.html) while the REI Data team fixes a JOIN mismatch in `sales_performance_cohort_daily_summary`. Remove the override block once DW table targets are populated.

## Deploy
Merges to `main` auto-deploy via the Quick GitHub Action. Manual deploy:
```bash
quick deploy . rev-pacing-global
```
