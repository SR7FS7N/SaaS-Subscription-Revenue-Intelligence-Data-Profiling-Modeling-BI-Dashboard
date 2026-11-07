# Entity Relationship Summary -- SaaS / Subscriptions

Star schema: one fact table surrounded by dimension and lookup tables.

## Tables
| Table | Grain | Rows (approx) |
|-------|-------|---------------|
| `fact_subscriptions` | one row per subscription event | 1,000,000 |
| `dim_accounts` | one row per account | 1,040 |
| `dim_plans` | one row per plan | 206 |
| `dim_locations` | one row per location | 103 |
| `dim_csms` | one row per Customer Success Manager | 75 |
| `bridge_account_manager` | account-to-Customer Success Manager link (many-to-many) | 1,682 |
| `wide_monthly_targets` | wide format for unpivot practice | 1,200 |

## Correct join keys
- `fact_subscriptions.account_id`  ->  `dim_accounts.acct_id`   (**KEY-NAME MISMATCH on purpose**: fact calls it `account_id`, the dimension calls it `acct_id`)
- `fact_subscriptions.plan_id`  ->  `dim_plans.plan_id`
- `fact_subscriptions.location_id`  ->  `dim_locations.location_id`
- `dim_accounts.acct_id`  ->  `bridge_account_manager.acct_id`  ->  `dim_csms.csm_id`  (**two-step enrichment** to reach the Customer Success Manager)

## Known traps (do NOT treat these as clean data)
1. **Key mismatch** -- the account key is named differently in fact vs dimension (see above).
2. **Orphaned FKs** -- ~4% of `fact_subscriptions.account_id` values have no match in `dim_accounts`.
3. **Inconsistent ID formatting** -- IDs appear as `ACCT-0001`, `0001`, `A0001`, `ACCT0001`. Normalize before joining.
4. **Fanout** -- the bridge maps some accounts to multiple Customer Success Managers; joining through it inflates measure sums unless you aggregate carefully.
5. **Stale dimensions** -- ~20% of plans are inactive/discontinued and some accounts are churned, yet both still appear in recent `fact_subscriptions` rows.
