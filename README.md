# SaaS Subscription & Revenue Intelligence: Data Profiling, Modeling & BI Dashboard

An end-to-end analytics and data engineering project modeling 1,000,000+ subscription transactions across a SaaS enterprise. The project addresses real-world data quality anomalies, establishes canonical standardization, resolves many-to-many bridge fanouts, and delivers an executive reporting suite.

The workflow spans a dual-environment pipeline:
* **Data Profiling, Cleansing & Transformation (Tasks 1–7):** Executed in **Microsoft Excel (Power Query Editor)**.
* **Dimensional Modeling, DAX Analytics & Visual Reporting (Tasks 8–10):** Implemented in **Power BI Desktop**.

---

## 1. Project Architecture & Tooling

```
[ Raw CSV Files (7) ]
        │
        ▼
[ Microsoft Excel: Power Query Editor ]
  ├── 1. Data Profiling & Null Rate Audit (~10-15% missingness)
  ├── 2. Canonical ID Normalization (`ACCT-XXXX`) & Orphan FK Flagging (~4%)
  ├── 3. Multi-Locale Date Parsing (`en-US`) & Chronological Flagging
  ├── 4. Near-Duplicate Text Cleaning & Fact Timestamp Deduplication
  ├── 5. Measure Cleaning (Currency Stripping) & Anomaly Quarantine
  ├── 6. Plan Profit & Cycle Metric Derivation (`mrr_amount - cost`, `days_to_close`)
  └── 7. Account Age, Revenue, and Quantity Bucketing
        │
        ▼ (Metadata & Query Handoff via VertiPaq Engine)
[ Power BI Desktop ]
  ├── 8. Star Schema Modeling & Bridge Fanout Inflation Audit (M:M resolution)
  ├── 9. Automated Business Rules & Compliance Exceptions Engine
  └── 10. Wide Target Unpivoting, Dynamic Date Dimension & DAX Analytics
        │
        ▼
[ Multi-Page Executive Reporting Suite ]
  ├── Page 1: Subscription Overview (Executive KPIs, 24M Trends, Rolling Averages)
  ├── Page 2: Customer Retention & Cohorts (Cohort Heatmap, Retention Lifecycle)
  └── Page 3: Exceptions Audit (Operational Integrity & Anomaly Triage)
```

---

## 2. Dataset Inventory & Initial Dimensions

The project integrates 7 transactional and dimensional CSV files:

| Dataset / File | Grain / Role | Rows (Raw with Header) | Columns | Injected Data Quality Challenges Addressed |
| :--- | :--- | :---: | :---: | :--- |
| `fact_subscriptions` | Central Transaction Fact (1 row per subscription event) | 1,000,001 | 14 | Inconsistent prefixes, orphan FKs (~4%), timestamp-shifted duplicates, mixed dates, dirty currency strings, negative/spike amounts, ~12% regional mismatches, mixed booleans. |
| `dim_accounts` | Account Dimension (1 row per account) | 1,041 | 10 | Inconsistent ID prefixes, near-duplicate full names (casing/whitespace), missing emails on Guest segments, 999 age sentinels, underage (<18) profiles, stale churn statuses. |
| `dim_plans` | Product Catalog Dimension (1 row per plan) | 207 | 8 | Duplicate IDs, inconsistent ID prefixes, mixed text casing, dirty currency list prices, ~5% null costs, ~20% inactive plans generating active sales. |
| `dim_locations` | Geography Hierarchy (1 row per location) | 104 | 7 | Inconsistent ID prefixes, mixed city casing, ~10% state-to-region mismatches, non-standardized country strings (`USA`, `US`, `United States`, `U.S.A.`). |
| `dim_csms` | Customer Success Rep Lookup (1 row per CSM) | 76 | 6 | Inconsistent ID prefixes, mixed hire date formats, mixed boolean activity encodings (`Y/N`, `1/0`, `True/False`). |
| `bridge_account_manager` | Account-to-CSM Bridge (Many-to-Many resolver) | 1,683 | 2 | Links accounts to 1–3 Customer Success Managers, creating measure fanout inflation if joined directly to transactional facts. |
| `wide_monthly_targets` | Auxiliary Target Matrix (1 row per account-year) | 1,201 | 14 | Wide horizontal structure across 12 monthly columns (`<Mon>_target`); unpivoted to long tabular format for time-series evaluation. |

---

## 3. Data Cleansing & Transformation (Excel / Power Query)

Tasks 1 through 7 were performed within Excel's Power Query engine and loaded into the Data Model (`Only Create Connection`):

### Task 1: Data Profiling & Quality Assessment
* **Column Profiling:** Activated *Column quality*, *Column distribution*, and *Column profile* across the entire dataset.
* **Missingness Baseline:** Validated overall missingness between 10% and 15%.
* **Type Auditing:** Identified mixed data types (`ABC/123`), non-standard boolean encodings (`YES/NO`, `1/0`, `Y/N`, `True/False`), and freeform country naming variants.

### Task 2: Canonical ID Standardization & Orphan Identification
* **Standardization:** Resolved four conflicting ID patterns (`ACCT-0001`, `0001`, `A0001`, `ACCT0001`) by stripping non-numeric characters and zero-padding into a 4-digit canonical format:
  ```powerquery
  "ACCT-" & Text.PadStart(Text.Select(Text.From([account_id]), {"0".."9"}), 4, "0")
  ```
* Applied identical normalization routines across `order_id`, `plan_id`, `location_id`, and `csm_id`.
* **Foreign Key Alignment:** Resolved key name discrepancies (`fact_subscriptions.account_id` $\rightarrow$ `dim_accounts.acct_id`).
* **Orphan Analysis:** Identified ~40,000 fact records (~4%) lacking parent records in `dim_accounts`. Maintained records via Left Outer Join, labeling unmapped accounts as `"Unassigned Account"` to preserve transaction counts.

### Task 3: Multi-Locale Date Parsing & Flagging
* `event_date` mixed `MM/DD/YYYY`, `YYYY-MM-DD`, and `DD-Mon-YYYY`. Handled all variations simultaneously using `Date.FromText` under an explicit `en-US` culture:
  ```powerquery
  if [event_date] = null or Text.Trim(Text.From([event_date])) = "" then null 
  else try Date.FromText(Text.Trim(Text.From([event_date])), [Culture = "en-US"]) otherwise null
  ```
* Engineered independent boolean flags:
  * `is_blank_event_date`: Isolated ~6% missing event dates.
  * `is_future_event_date`: Isolated ~1% records exceeding current system dates.

### Task 4: Deduplication Framework
* **Dimension Cleaning:** Applied `Text.Trim`, `Text.Clean`, and casing normalization to `dim_accounts.full_name`, removing redundant whitespace and duplicates on `clean_acct_id`.
* **Timestamp-Shifted Fact Records:** Sorted `fact_subscriptions` by `order_id` ascending and `clean_event_date` ascending, retaining the initial occurrence (`MIN`) and removing lagged duplicates.
* **Conflicting Order Amounts:** Quarantined duplicate `order_id` occurrences carrying distinct `mrr_amount` values for audit analysis.

### Task 5: Measure Cleansing & Anomaly Quarantine
* **Currency Standardization:** Stripped currency symbols (`$`), commas, and whitespace, converting `unit_price` and `mrr_amount` to strict decimal numbers:
  ```powerquery
  Number.FromText(Text.Select(Text.From([mrr_amount]), {"0".."9", ".", "-"}))
  ```
* **Quarantine Pipeline:** Created an `is_quarantined` boolean column isolating records meeting any of the following:
  1. Negative MRR (`clean_mrr < 0`) or Nulls.
  2. Revenue spikes exceeding 10x the 99th percentile cutoff ($> \$4,500$).
  3. Zero revenue on completed orders (`status = "Completed"` and `clean_mrr = 0`).
  4. Zero quantity on completed orders (`status = "Completed"` and `quantity = 0`).

### Task 6: Analytical Column Calculations
* **Unit Profitability:** Merged `dim_plans` on `plan_id` to retrieve unit `cost`, calculating unit profit:
  $$\text{profit} = \text{clean\_mrr} - \text{cost}$$
* **Cycle Duration:** Calculated fulfillment/settlement duration:
  $$\text{days\_to\_close} = \text{clean\_renewal\_date} - \text{clean\_event\_date}$$
* **Calendar Partitions:** Extracted `Year`, `Month`, `Quarter`, and `Day_of_Week` from `clean_event_date`.

### Task 7: Segmentation & Binning
* **Age Groups (`dim_accounts`):** Converted 999 sentinels and underage (<18) records to `Invalid / Underage`, grouping valid profiles into `18-25`, `26-35`, `36-50`, and `51+`.
* **Revenue Tiers (`fact_subscriptions`):** Segmented into `Low Tier` ($\le \$100$), `Mid Tier` ($\$101 - \$500$), and `Enterprise Tier` ($> \$500$).
* **Quantity Buckets:** Grouped volumes into `Single (1)`, `Small Team (2-5)`, `Department (6-20)`, and `Enterprise (20+)`.

---

## 4. Modeling & Advanced Analytics (Power BI Desktop)

Applied queries were imported directly into Power BI Desktop via `File -> Import -> Power Query, Power Pivot, Power BI`, persisting all M steps.

### Task 8: Bridge Fanout & Many-to-Many Architecture
Because `bridge_account_manager` maps accounts to 1–3 Customer Success Managers, a standard relational join against transactional facts multiplies transaction rows, artificially inflating top-line revenue.

```
       dim_plans (1)                 dim_locations (1)
             │                              │
             └───┐                      ┌───┘
                 ▼                      ▼
             fact_subscriptions (*) ◄─── dim_accounts (1)
                                               │ (1)
                                               ▼
                                    bridge_account_manager (*)
                                               │ (*)
                                               ▼
                                         dim_csms (1)
```

* **Fanout Demonstration:** Calculated the baseline non-quarantined fact MRR against the inflated bridged sum:
  * **Correct Total MRR:** Direct aggregation on fact records.
  * **Inflated Bridge MRR:** Aggregation evaluated across the multi-CSM bridge, producing an approximate 15% to 25% artificial metric inflation.
  * **Reported Gap:** Quantified via `[Fanout_Dollar_Gap]` and `[Fanout_Inflation_Pct]`.
* **Primary CSM Resolution:** Built a deduplicated dimension link (`dim_primary_csm_link`) assigning a single primary CSM per account (`MIN(clean_csm_id)`), eliminating row duplication while retaining CSM slicing capabilities.

### Task 9: Business Rules Validation Engine
Engineered calculated exception columns in `fact_subscriptions` tracking 5 operational violations:
1. `Ex_Inverted_Dates`: Identifies rows where `clean_renewal_date < clean_event_date`.
2. `Ex_Zero_Dollar_Completed`: Identifies fulfilled events (`status = "Completed"`) logging zero or null MRR.
3. `Ex_Underage_Violation`: Flags customer accounts where `age < 18` or `age = 999`.
4. `Ex_Conflicting_Order_Amounts`: Flags duplicate `order_id` keys carrying conflicting revenue amounts.
5. `Ex_Regional_Mismatch`: Flags transactions where `order_region` diverges from the account's registered `region` (~12% discrepancy).

### Task 10: Unpivoting, Dynamic Dates & Core DAX Measures
* **Unpivot Targets:** Transformed `wide_monthly_targets` from 12 monthly columns into a normalized long table (`clean_account_id`, `Target_Date`, `Target_MRR`).
* **Continuous Date Dimension (`Dim_Date`):** Created a standard DAX calendar table linked to `fact_subscriptions[clean_event_date]` and `wide_monthly_targets[Target_Date]`.

#### Primary DAX Measures Implemented:
```dax
// Actual Clean MRR (Excluding Quarantined Anomalies)
Actual_MRR = 
CALCULATE(
    SUM(fact_subscriptions[clean_mrr]),
    fact_subscriptions[is_quarantined] = FALSE()
)

// Target Variance Metrics
Target_MRR_Total = SUM(wide_monthly_targets[Target_MRR])
MRR_Variance = [Actual_MRR] - [Target_MRR_Total]
MRR_Variance_Pct = DIVIDE([MRR_Variance], [Target_MRR_Total], 0)

// 7-Day & 30-Day Moving Averages
MRR_7D_Rolling = 
AVERAGEX(
    DATESINPERIOD(Dim_Date[Date], MAX(Dim_Date[Date]), -7, DAY),
    [Actual_MRR]
)

MRR_30D_Rolling = 
AVERAGEX(
    DATESINPERIOD(Dim_Date[Date], MAX(Dim_Date[Date]), -30, DAY),
    [Actual_MRR]
)

// Customer RFM Metrics
RFM_Recency_Days = 
VAR LastDate = MAX(fact_subscriptions[clean_event_date])
RETURN IF(NOT(ISBLANK(LastDate)), DATEDIFF(LastDate, TODAY(), DAY), BLANK())

RFM_Frequency = 
CALCULATE(
    DISTINCTCOUNT(fact_subscriptions[order_id]),
    fact_subscriptions[is_quarantined] = FALSE()
)

RFM_Monetary = [Actual_MRR]
```

---

## 5. Dashboard Visual Architecture

The reporting interface is organized across three dedicated report pages in Power BI:

### Page 1: Subscription Overview
* **Target Audience:** Executive Leadership & Finance.
* **Core Visuals:**
  * **Top KPI Scorecard:** `[Actual_MRR]`, `[Target_MRR_Total]`, `[MRR_Variance_Pct]`, and Active Subscriber Count.
  * **24-Month Performance Trend (Combo Chart):** Monthly actual MRR bars plotted against target lines with variance percentage callouts.
  * **Volatility Smoothing (Line Chart):** Actual daily MRR overlaid with `7-Day` and `30-Day` rolling averages.
  * **Geographic Revenue Distribution (Filled Map / Bubble Map):** Regional subscription performance across states and territories.
  * **Interactive Slicers:** `revenue_tier`, `quantity_bucket`, and Plan Category.

### Page 2: Customer Retention & Cohorts
* **Target Audience:** Growth, Marketing & Product Teams.
* **Core Visuals:**
  * **Cohort Retention Matrix:** Rows mapped to customer `join_date` (cohort month), columns mapped to elapsed lifecycle months ($M_0, M_1, M_2, \dots$), and cells populated with `DISTINCTCOUNT(account_id)`.
  * **Retention Heatmap:** Conditional background color gradients highlighting drop-off velocity and long-term customer survival.
  * **Customer RFM Segmentation Matrix:** Granular account breakdown displaying Recency (days since last purchase), Order Frequency, and Lifetime Monetary Value.

### Page 3: Exceptions Audit
* **Target Audience:** Data Governance, Operations & Risk Management.
* **Core Visuals:**
  * **Exception Scorecards:** Total counts for Inverted Dates, Zero-Dollar Completed Orders, Underage Violations, Conflicting Order Amounts, and Regional Mismatches.
  * **Triage Slicer (`Primary_Exception_Type`):** Button-style slicer allowing auditors to isolate specific violation categories.
  * **Granular Remediation Table:** Row-level audit grid containing order identifiers, customer IDs, flagged dates, conflicting values, and audit timestamps.

---

## 6. How to Reproduce & Run

1. **Clone the Repository:**
   ```bash
   git clone https://github.com/your-username/saas-subscription-bi-audit.git
   ```
2. **Review Excel ETL:**
   * Open `etl_pipeline.xlsx` in Microsoft Excel.
   * Go to **Data $\rightarrow$ Queries & Connections** to inspect the M code and applied steps across all 7 staging tables.
3. **Launch Power BI Dashboard:**
   * Open `subscription_analytics.pbix` in **Power BI Desktop**.
   * If local CSV file paths differ, select **Transform Data $\rightarrow$ Data source settings** and update the source directory path to your local data folder.
   * Click **Refresh** to process all transformations, relationships, and visual layers.

LIVE DASHBOARD - https://app.powerbi.com/view?r=eyJrIjoiOWE2MzI1M2UtNWUzMS00ZTAxLWI4MzctNjljMTA4N2Q5OTRmIiwidCI6ImM2ZTU0OWIzLTVmNDUtNDAzMi1hYWU5LWQ0MjQ0ZGM1YjJjNCJ9
