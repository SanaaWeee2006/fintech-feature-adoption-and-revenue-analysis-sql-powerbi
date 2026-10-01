# Fintech Feature Adoption & Revenue Analysis

The project focuses on answering questions related to a product-led fintech scenario:-
**which features actually drive retention and revenue, and what should the product team do about each one; invest, fix discovery, monetize differently, or sunset?**

Built end-to-end as a project using medallion pipeline (bronze/silver/gold) in SQL, performed data cleaning & EDA to answer business questions. Built a star-schema Power BI dashboard with DAX measures. 

---

## 1. Business Problem

A fintech app has 7 product features. Management doesn't know which ones justify their engineering cost. This project builds the data and analysis needed to classify each feature into one of four actions:

- **Invest / protect** — high adoption, high revenue
- **Fix discovery** — high value, low reach
- **Retention hook / monetize better** — high adoption, low direct revenue
- **Sunset candidate** — low adoption, low revenue, no retention benefit

The 7 features are: **UPI, BillPay, Investments, CreditScore, Loans, Cards, Budgeting**.

---

## 2. Project Architecture

```
   5 CSV datasets
        ↓
   BRONZE  (raw CSVs, loaded as it is)
        ↓
   SILVER  (cleaned, validated)
        ↓
   GOLD   (analysis-ready: user-level, feature-level, segment views)
        ↓
   EDA & Business-Question SQL
        ↓
   Power BI Dashboard (star schema & DAX measures)
```

**Medallion architecture** keeps every transformation auditable. Each layer can be rebuilt from the one before it, and a reconciliation check confirms no row silently disappears without a logged reason.

---

## 3. Dataset

5 source tables, ~3,000 users, ~18 months of activity (Apr 2024–Sep 2025)

| Table | Grain | Purpose |
|---|---|---|
| `users` | 1 row per user | Signup date, city tier, age band, acquisition channel |
| `feature_usage` | 1 row per user-feature-usage event | Behavioral engagement log |
| `transactions` | 1 row per monetizable action | Payments, investments, loan disbursals (negative = refund) |
| `revenue` | 1 row per user-month-feature-source | Fees, interest, subscriptions |
| `churn_status` | 1 row per user | Churn flag snapshot as of the data's end date |


---
## 4. Source to Bronze

Data is as it is loaded into the bronze layer without any cleaning or validation logic applied.

Script: `bronze folder`

## 5. Bronze to Silver: Cleaning Logic

Bronze is immutable ambiguous values are **flagged, not silently overwritten**; financial records are **quarantined, never deleted**; every row is accounted for in a reconciliation check.

| Table | Issue | Treatment | Why |
|---|---|---|---|
| `users` | Missing city_tier/age_band | Replaced with `'Unknown'` | An honest segment beats a guessed one |
| `feature_usage` | Dirty casing/whitespace | Mapped via explicit CASE lookup | Unrecognized values stay visible for review |
| `feature_usage` | Exact duplicates | Deduplicated on full-row match (not `usage_id` alone — duplicates share IDs) | |
| `transactions` | Negative amounts | **Kept unchanged** | Real refunds, not errors |
| `transactions` | Orphan `user_id`s | Quarantined to `transactions_rejected` with a reason | Financial rows shouldn't vanish without a trail |
| `transactions` | Unusual amounts | Flagged (`is_outlier_amount`) | Large amounts can be legitimate |
| `revenue` | Missing `revenue_source` | **Repaired** via deterministic mapping from `feature_name` | Unlike demographics, this value is derivable from a known business rule |
| `churn_status` | ~5% label disagrees with recency | **Not overwritten.** A second column, `recency_based_churn_flag`, plus `churn_flag_mismatch` sit alongside the original | The recorded flag may reflect real ops (e.g. a frozen account); the mismatch is a finding, not a bug to fix |

Script: `silcer folder`

---

## 5. Silver to Gold: Analysis-Ready Data Preparation

**Key modeling decisions:**

1. **Adopter definition spans two sources.** A user counts as adopting a feature if they have a usage row *or* revenue from it built as a `FULL OUTER JOIN` of usage and revenue aggregates. This matters because Budgeting subscribers can pay without ever appearing in the usage log; a usage-only definition would silently drop them.

2. **Bias-controlled retention design (the most important fix in the project).** Two problems had to be handled before any retention comparison was trustworthy:
   - *Immortal time bias:* users who stay longer mechanically have more time to adopt more features, making breadth look artificially protective. **Fix:** adoption is measured only in each user's first 30 days (`adopted_in_first_30d`); retention is judged only among users still active at day 30 (`survived_day30`).
   - *Right-censoring:* a user who signed up last week can't yet be labeled "retained" or "churned." **Fix:** retention analysis is restricted to users with ≥90 days of tenure (`retention_eligible`).

3. **Revenue-timing bug, found and fixed during gold-layer QA.** The generator assigned Budgeting subscription months independent of signup date, producing some revenue rows dated before signup or after the observation window — impossible in reality. Caught via a validation check, not visible in silver. Fixed with `gold.vw_valid_revenue`, which restricts revenue to each user's valid observation window. The excluded rows remain untouched in silver; an informational query reports how much was excluded.

4. **Confounder-check views.** `vw_segment_city_tier`, `vw_segment_channel`, and `vw_feature_lift_by_segment` test whether a feature's retention lift survives once you control for city tier or acquisition channel — this is what separates a real effect from a segment-mix artifact.

Scripts: `silver_to_gold.sql` (includes an 8-check validation block — every check should read PASS before trusting any downstream chart).

---

## 6. EDA & Business-Question SQL

`eda_and_business_answers.sql`, organized in 5 sections:

- **A — EDA:** row counts, signup trend, feature/revenue/churn distributions, raw vs. eligible-only churn rate comparison
- **B — Adoption:** ranking, cohort-over-time trend, breadth by segment
- **C — Retention:** breadth vs. churn, per-feature lift ranking, confounder check within segments
- **D — Revenue:** total vs. per-adopter ranking, adoption/profitability mismatch, cross-sell effect, loss-leader candidates
- **E — Synthesis:** the final feature-by-feature decision table, weakest-case candidates, and features that look weak overall but are protected within a specific segment

---

## 7. Power BI: Star Schema

Moved from static, pre-aggregated SQL tables to a dynamic model so **any** dimension can filter **any** metric (e.g. checking if a feature's retention lift holds within Tier1 alone, just by clicking a slicer — no new SQL needed).

**Dimensions:** `Dim_User` (signup_date, city_tier, age_band, channel, cohort_month), `Dim_Feature` (feature_name, feature_category — added in Power Query), `Dim_Date` (calendar table, marked as the model's date table).

**Facts:** `Fact_UserFeature` (user × feature grain — adoption, usage, revenue), `Fact_UserSummary` (user grain — churn, breadth, revenue), `Fact_CohortRetention` (cohort month × period).

**Key relationship rule:** all single-direction (dimension → fact); `Fact_UserFeature` and `Fact_UserSummary` are *not* directly related — they only connect through `Dim_User`, which is why several DAX measures use `CALCULATETABLE` + `TREATAS` to bridge user sets across the two fact tables explicitly.

DAX measures: `dax_measures.txt`, grouped by page (Core, Adoption, Revenue, Retention Lift, Matrix Quadrant, Breadth/Cross-sell), each validated against the SQL `gold.feature_summary` reference before being trusted.

---

## 8. Dashboard Structure (4 pages)

| Page | Purpose |
|---|---|
| **Overview** | Total users, revenue, overall vs. eligible-only churn rate, signup trend, segment composition |
| **Adoption** | Adoption ranking, cohort adoption trend over time, breadth by segment |
| **Retention & Revenue** | The 2×2 adoption-vs-revenue matrix, retention lift ranking with sample sizes, breadth-vs-churn, revenue concentration, cross-sell effect — with city tier / channel slicers for confounder testing |
| **Recommendation** | One decision table (adoption, revenue, lift, quadrant) plus written findings — no slicers, meant to read as a conclusion |

---

## 9. Key Findings

1. **Breadth matters more than any single feature.** Churn drops sharply between 1 and 2 features used, and revenue per user rises sharply from 2 to 4+ features — a bigger effect than any individual feature's retention lift. Framed as a hypothesis (correlation, not proven causation): onboarding that nudges a second feature within 30 days may be the single highest-leverage lever.
2. **UPI is a retention hook, not a revenue source** — highest adoption, near-zero revenue per adopter, solid positive lift. By design (regulated, mostly-free payment rail).
3. **BillPay is an under-monetized retention anchor**, not a loss-leader — it earns real (if modest) revenue and has the strongest retention lift of all 7 features. Flagged as a pricing/upsell test candidate, distinct from true zero-revenue features.
4. **CreditScore's high adoption doesn't translate into revenue or retention** — zero revenue and a negative lift. Unlike a true loss-leader, it isn't shown to help retention either; recommended to test whether it funnels users into Loans before deciding its fate.
5. **Loans drives the majority of revenue but shows the worst retention lift.** Flagged explicitly as needing further investigation (product friction vs. adverse selection / pre-existing financial distress) before recommending any product change — correlation alone can't distinguish the two explanations.
6. **Confounders matter:** Tier1 users over-index on both wealth-feature usage and revenue; several feature-level findings are checked against segment-level views before being stated as feature effects.

---

## 10. Tools Used

SQL Server (T-SQL), Power BI (Power Query & DAX) for the dashboard and data model.
