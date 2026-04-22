# Automated Pharma Media Budget Tracker (Google, LinkedIn, other channels)

## 1) What to build (MVP first)

Build a **monthly pacing tracker** that shows, by **Brand** and **Audience** (HCP vs Patient):

- Total approved budget (month + campaign period)
- Actual spend to date (auto-fetched from platforms)
- Pace vs expected spend (linear or custom pacing curve)
- Forecasted month-end spend
- Variance (over/under)

Start with a single source of truth in a small database and a dashboard layer. Keep Excel only as a backup/export.

---

## 2) Data model (simple and scalable)

Use these core tables:

### `brands`
- `brand_id`
- `brand_name`

### `campaigns`
- `campaign_id` (internal)
- `platform` (google_ads, linkedin, etc.)
- `platform_campaign_id`
- `brand_id`
- `audience_type` (hcp, patient)
- `country`
- `start_date`, `end_date`
- `status`

### `budgets`
- `budget_id`
- `campaign_id`
- `month` (YYYY-MM)
- `planned_budget`
- `currency`

### `daily_spend`
- `date`
- `campaign_id`
- `spend`
- `currency`
- `impressions`, `clicks` (optional)

### `fx_rates` (optional if multi-currency)
- `date`
- `from_currency`
- `to_currency`
- `rate`

### `monthly_pacing` (materialized / derived)
- `month`
- `brand_id`
- `audience_type`
- `planned_budget`
- `actual_spend_to_date`
- `expected_spend_to_date`
- `pace_pct` (= actual / expected)
- `utilization_pct` (= actual / planned)
- `forecast_eom`
- `variance_to_plan`

---

## 3) Ingestion architecture

Recommended practical stack:

- **Orchestration:** Airflow, Prefect, or Dagster (or cron to start)
- **Compute:** Python
- **Warehouse:** BigQuery / Snowflake / PostgreSQL
- **BI Dashboard:** Looker Studio, Power BI, or Tableau

### API connectors
- Google Ads API: pull campaign/day spend, campaign metadata, account currency
- LinkedIn Marketing API: pull campaign spend and metadata
- Other channels (Meta, DV360, etc.): same normalized schema

### Load frequency
- Daily at 6am local + optional intraday refresh (e.g., noon)

### Incremental strategy
- Re-pull rolling window of last 7–14 days to handle attribution and late adjustments

---

## 4) Pacing logic (monthly)

For each brand + audience in month M:

1. `planned_budget_month = SUM(planned_budget)`
2. `actual_spend_to_date = SUM(daily_spend where date <= today)`
3. `elapsed_ratio = elapsed_days / total_days_in_month`
4. `expected_spend_to_date = planned_budget_month * elapsed_ratio`
5. `pace_pct = actual_spend_to_date / expected_spend_to_date`
6. `utilization_pct = actual_spend_to_date / planned_budget_month`
7. `forecast_eom = (actual_spend_to_date / elapsed_days) * total_days_in_month`
8. `variance_to_plan = forecast_eom - planned_budget_month`

You can later replace linear pacing with weekday-weighted curves.

---

## 5) Governance and pharma-specific controls

Given pharma requirements, add:

- Role-based access (brand teams only see allowed brands)
- Audit trail (who changed budget and when)
- Data retention policy aligned with internal compliance
- Separation of media metrics from any patient-level data
- Clear dictionary for "HCP" and "Patient" campaign labeling

---

## 6) Dashboard layout

### Page A: Executive summary
- Month-to-date spend vs plan
- Over/under by brand
- Forecasted month-end overburn/underspend

### Page B: Brand drill-down
Filters:
- Brand
- Audience type (HCP/Patient)
- Platform
- Country

Visuals:
- Cumulative spend vs cumulative expected pace
- Table of campaigns with variance and forecast
- Alert flags (e.g., >10% over pace)

### Page C: Data quality
- Last refresh time by platform
- Missing campaign mappings
- API failures and retries

---

## 7) Suggested implementation phases

### Phase 1 (2–4 weeks): MVP
- Build schema
- Connect Google + LinkedIn
- Ingest daily spend
- Load budget file manually (CSV template)
- Publish pacing dashboard

### Phase 2 (2–4 weeks)
- Add additional channels
- Add forecast improvements and anomaly flags
- Add Slack/Email alerts

### Phase 3
- Add budget change workflow (approval + versioning)
- Add scenario planner ("what if we shift $X to HCP?")

---

## 8) Minimal technical blueprint (Python)

1. Connector jobs:
   - `fetch_google_spend.py`
   - `fetch_linkedin_spend.py`
2. Normalize into staging tables
3. Transform SQL models to canonical tables (`campaigns`, `daily_spend`)
4. Build `monthly_pacing` model
5. Serve dashboard from `monthly_pacing`

---

## 9) First 10 decisions to lock before building

1. Source of truth for approved budgets (finance vs marketing ops)
2. Fiscal calendar definition
3. Time zone for daily close
4. Currency normalization policy
5. Brand taxonomy and naming conventions
6. Rule for campaign tagging to HCP/Patient
7. Data latency tolerance (same day vs next day)
8. Alert thresholds (e.g., ±10%)
9. Ownership for API credentials and rotation
10. Who signs off dashboard numbers each month

---

## 10) If you want the fastest path

If your team needs value quickly:

- Keep data warehouse + BI as above
- Use managed connectors where possible
- Start with **one brand**, then template for all brands
- Run old Excel in parallel for 1 month to validate

This gives a low-risk migration from manual spreadsheets to an automated, auditable pacing system.
