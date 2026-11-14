# Analyst Exercise List -- SaaS / Subscriptions

Ten tasks to complete **before** building a dashboard. They move through the full
workflow: profile -> clean -> join -> transform -> validate -> aggregate. File names
below refer to the CSVs in this folder.

## 1. Profile & assess quality
Load every table. For each, report row count, null rate per column, and spot the
columns with mixed formats. Expect ~10-15% missing values overall.

## 2. Standardize IDs, then join fact to account
Normalize the inconsistent ID prefixes (`ACCT-0001` / `0001` / `A0001` / `ACCT0001`)
to one canonical form. Join `fact_subscriptions.account_id` to `dim_accounts.acct_id`
(**note the different key names**). Count how many fact rows have no matching
account (orphans) -- you should find ~4%.

## 3. Parse the mixed date formats
`event_date` mixes MM/DD/YYYY, YYYY-MM-DD, and DD-Mon-YYYY in one column. Parse all
three into real dates. Flag the blanks and the future-dated rows separately.

## 4. De-duplicate
Find exact duplicate rows and near-duplicates (names differing only by case,
spacing, or punctuation in `dim_accounts`). In `fact_subscriptions`, find the same subscription event recorded
twice with a shifted timestamp, and the same `order_id` carrying two different
amounts. Decide a dedupe rule and document it.

## 5. Clean the measures
`unit_price` and `mrr_amount` mix currency formats ($1,200.00 / 1200 / 1,200).
Strip them to numbers. Then quarantine the anomalies: negative amounts, 10x+ spikes
(above the 99th percentile), quantity = 0 on Completed subscription events, and amount = 0 on
Completed subscription events.

## 6. Build calculated columns
After joining `dim_plans` for `cost`: compute `profit = mrr_amount - cost`.
Compute `days_to_close = renewal_date - event_date`. Extract `year`, `month`,
`quarter`, and `weekday` from `event_date`.

## 7. Bin and bucket
Create `age_group` from the account age (handle the 999 sentinels and <18 cases
first), a `revenue_tier` from `mrr_amount`, and a `quantity_bucket`.

## 8. Two-step enrichment via the bridge (watch for fanout)
Attach the Customer Success Manager to each subscription event by joining through `bridge_account_manager`.
Because some accounts map to multiple Customer Success Managers, a naive join inflates
`mrr_amount` sums -- show the inflated total vs the correct total and explain the gap.

## 9. Validate business rules
Produce an exceptions report for: (a) `renewal_date` earlier than `event_date`,
(b) amount = 0 on Completed status, (c) account age < 18 where adult verification
is required, (d) duplicate `order_id`s with different amounts, and (e) `order_region`
in `fact_subscriptions` disagreeing with the account's region in `dim_accounts`.

## 10. Unpivot, then aggregate for the dashboard
Unpivot `wide_monthly_targets` to long format. Then build the dashboard-ready
aggregates: a 24-month time-series trend, a 7-day and 30-day rolling average, a
cohort-retention grid from `join_date` + first subscription event date, and an RFM table
(recency, frequency, monetary) per account. These feed the time series, KPI
cards, geographic heatmap, scatter/correlation, cohort, and ranking charts.
