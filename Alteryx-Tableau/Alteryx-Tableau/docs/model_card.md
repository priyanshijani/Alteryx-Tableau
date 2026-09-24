# Model Card — Win Probability `wp-v1-2026-09-15`

> Status: **Advisory** (not the forecast of record) · Owner: Sales Operations (you) · Built in: Alteryx Designer, `alteryx/03_win_probability_model.yxmd`

## 1. Purpose
Estimate the probability that an open opportunity will be **Closed Won**, so the CRO can see which deals are
stronger or weaker than their stage suggests. Used for review and prioritisation, **not** for the official forecast
until it has been validated for two consecutive quarters.

## 2. Data
- Source: `deals_for_model.yxdb` and `stints_for_model.yxdb` (outputs of the reconciled pipeline mart).
- Training unit: a "photo" of a closed deal on day 0/14/30/45/60/90/120 of each open stage it passed through.
- Label: `Won` / `Lost` (final outcome).
- Excluded rows: deals with no amount or an amount outlier; test records; deals still open (outcome unknown).

## 3. Features
| Used | Why it's allowed |
|---|---|
| | |

| Excluded | Why (how it would leak) |
|---|---|
| `rep_probability` | 0 or 100 on closed deals, so it reveals the answer |
| `last_activity_date` / `is_stale` | Equals the close date on closed deals |
| `account_status` | Uses wins that happened after the photo was taken |
| *(add the rest from the checklist, 6b.2)* | |

## 4. Validation design
- Time-based backtest: trained on deals closed before **1 Mar 2026**, tested on the pipeline as it stood on 1 Mar 2026.
- All photos of a deal stay on one side of the split.
- Final model retrained on all deals closed before 15 Sep 2026.

## 5. Results (backtest)
| Metric | Governed stage probability | Logistic regression | Forest model |
|---|---|---|---|
| Brier score (lower is better) | | | |
| AUC (optional) | | | |
| Forecast € vs actual won € | | | |

Chosen model: ________ because ________.
Top 3 drivers: 1. ________ 2. ________ 3. ________ — do these make business sense? ________

## 6. Limitations
- Synthetic data with designed patterns; real CRM data will be noisier.
- Deals still open at 15 Sep 2026 have no outcome yet, so the backtest is based on deals decided within ~6 months.
- The model learns historical team and region effects. It shows where results have come from, not why.
- Photos over-represent deals that lived longer.

## 7. Governance
- Review quarterly; retrain when new outcomes arrive.
- Promote from *Advisory* to *Forecast of record* only after two quarters of better accuracy than governed probabilities, with CRO sign-off.
- Any change is logged in `docs/business_definitions.md` → change log.
