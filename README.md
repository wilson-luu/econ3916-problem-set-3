# ECON 3916 — Problem Set 3: The Number Nobody Can Reproduce

## Overview
An applied SQL and causal-inference investigation in DuckDB and Python examining why a company dashboard reported **$1,240** for West-region average revenue per customer while Finance reported **$812** from the same warehouse, and whether the West region's higher revenue was actually caused by its new account-management programme.

## Key Findings
1. **The $1,240 vs. $812 Discrepancy (Phase 1 — Join Fan-Out):**
   - **Correct Figure:** **$812.17** (`$812`), computed at the **customer grain** (`2,000` West customers in `customers`).
   - **Why the Dashboard Reported $1,242.54 (`$1,240`):** Joining `customers` to `orders` (`13,440` West order rows) without re-aggregating shifted the unit of observation from **customer** to **order**, weighting each customer's revenue by their order count ($n_i \sim 1 + \text{NegBin}(0.8, 0.12)$). Because order count drives revenue, order-weighting inflated the average by **53.0% (+$430.37)**.
2. **Silent Data Loss from SQL Three-Valued Logic (Phase 2 — NULLs & Window Functions):**
   - With `15.17%` (`455` rows) of legacy customers missing a `region` value (`NULL`), running `WHERE region != 'West'` returns only `841` rows instead of the `1,296` rows not known to be in West—silently dropping **35.1% of the non-West comparison group** because `NULL != 'West'` evaluates to `UNKNOWN`.
   - Using `AVG(revenue) OVER (PARTITION BY region)` appends regional benchmarks directly to all `3,000` customer rows without collapsing grain or requiring a join.
3. **Confounder vs. Collider Bias in Programme Evaluation (Phase 3 — Causal Inference):**
   - **True Programme ATE:** **+$45.00** per customer.
   - **Naive Estimate (`revenue ~ programme`):** **+$284.39** ($p < 10^{-200}$), severely upward-biased by pre-existing `account_size` confounding both regional programme assignment and baseline revenue.
   - **Confounder-Adjusted Estimate (`revenue ~ programme + account_size`):** **+$38.21** ($95\%$ CI $[\$31.46, \$44.96]$), blocking the backdoor path and recovering the true modest effect.
   - **Collider Bias (`+ support_tickets`):** Controlling for `support_tickets` (which is caused by both proactive rep logging in the programme and higher revenue) opens a spurious path that flips the estimated effect to **-$67.30** ($p = 2.4 \times 10^{-57}$).
4. **Automated SQL Query Reviewer (Phase 4):**
   - Built `review_query(con, sql)` to lint DuckDB queries prior to dashboard deployment, automatically catching 1-to-many join fan-outs, aggregates over fanned-out tables, and unguarded `WHERE` filters on nullable columns.
