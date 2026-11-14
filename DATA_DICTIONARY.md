# Data Dictionary -- SaaS / Subscriptions

Legend for **Injected issues**: the quality problems deliberately placed in each column.


## `fact_subscriptions`  (fact table)

| Column | Type | Description | Injected issues |
|---|---|---|---|
| `order_id` | string | Unique subscription event identifier | Inconsistent prefixes; some duplicated (exact, timestamp-shifted, or same id w/ different amount) |
| `account_id` | string (FK) | account reference | KEY-NAME MISMATCH vs dim (`acct_id`); ~4% orphaned; inconsistent prefixes |
| `plan_id` | string (FK) | plan reference | Some point to discontinued items |
| `location_id` | string (FK) | Location reference | Inconsistent prefixes |
| `event_date` | date | Primary subscription event date | Mixed formats (MM/DD/YYYY, YYYY-MM-DD, DD-Mon-YYYY); ~6% blank; ~1% future-dated |
| `renewal_date` | date | Follow-up date (ship/close/settle) | Mixed formats; a few earlier than the primary date (end < start) |
| `quantity` | integer | Units / count | ~3% null; some 0 on Completed status |
| `unit_price` | currency string | Price per unit | Mixed currency formats ($1,200.00 / 1200 / 1,200) |
| `mrr_amount` | currency/number | Subscription Event measure | ~3% null; negatives; 10x+ spikes; some 0 on Completed; mixed currency formats |
| `discount_pct` | float | Discount fraction 0-0.4 | - |
| `status` | string | Lifecycle status | Enables status-based validation rules |
| `order_region` | string | Region recorded on the subscription event | ~12% mismatch the account's region in the dimension |
| `is_flagged` | boolean | Review/fraud flag | Mixed encodings (True/False, YES/NO, 1/0, Y/N) |
| `satisfaction_score` | integer 1-10 | Experience score | ~4% null |

## `dim_accounts`  (account dimension)

| Column | Type | Description | Injected issues |
|---|---|---|---|
| `acct_id` | string (PK) | account id | Inconsistent prefixes; some IDs referenced in fact are MISSING here (orphans) |
| `full_name` | string | Full name | Near-duplicates: casing, double spaces, trailing space, initial+last |
| `email` | string | Email address | Structured missingness: usually blank when segment = Guest; ~5% null elsewhere |
| `phone` | string | Phone | Mixed formats; ~10% null |
| `region` | string | Region | ~4% null |
| `state` | string | State code | Some region/state combos mismatched |
| `segment` | string | account segment/type | - |
| `join_date` | date | First-seen date | Mixed formats (cohort anchor) |
| `status` | string | Lifecycle status | Some marked churned/closed yet active in recent fact rows (stale) |
| `age` | integer | Age | 999 sentinels; some < 18 (adult-verification violations) |

## `dim_plans`  (plan dimension)

| Column | Type | Description | Injected issues |
|---|---|---|---|
| `plan_id` | string (PK) | plan id | Inconsistent prefixes; exact duplicates |
| `plan_name` | string | plan name | - |
| `category` | string | Category | Mixed text case (Title / lower / UPPER) |
| `subcategory` | string | Subcategory | - |
| `unit_price` | currency string | List price | Mixed currency formats; ~5% null |
| `cost` | number | Unit cost | ~5% null (needed for profit = price - cost) |
| `active_flag` | boolean | Active/discontinued | Mixed encodings; ~20% inactive yet referenced in fact |
| `launch_date` | date | Launch date | Mixed formats; a few future-dated |

## `dim_locations`  (geography)

| Column | Type | Description | Injected issues |
|---|---|---|---|
| `location_id` | string (PK) | Location id | Inconsistent prefixes |
| `city` | string | City | Mixed text case |
| `state` | string | State code | - |
| `region` | string | Region | ~10% mismatched vs state |
| `country` | string | Country | Inconsistent values (USA / US / United States / U.S.A.) |
| `postal_code` | string | ZIP | - |
| `territory` | string | Sales territory | - |

## `dim_csms`  (Customer Success Manager lookup)

| Column | Type | Description | Injected issues |
|---|---|---|---|
| `csm_id` | string (PK) | Customer Success Manager id | Inconsistent prefixes |
| `csm_name` | string | Customer Success Manager name | - |
| `team` | string | Team | - |
| `manager` | string | Manager | - |
| `hire_date` | date | Hire date | Mixed formats |
| `active` | boolean | Active flag | Mixed encodings |

## `bridge_account_manager`  (many-to-many link)

| Column | Type | Description | Injected issues |
|---|---|---|---|
| `acct_id` | string (FK) | account id | Links to dim_accounts |
| `csm_id` | string (FK) | Customer Success Manager id | Links to dim_csms; fanout (1-3 per account) |

## `wide_monthly_targets`  (wide format -- unpivot me)

One row per account-year; twelve `<Mon>_target` columns. Unpivot to long format (one row per account-month) before analysis.

