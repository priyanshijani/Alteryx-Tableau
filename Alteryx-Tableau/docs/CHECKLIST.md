# Build Checklist — Alteryx + Tableau Sales Pipeline Project

Tick boxes as you go (`[ ]` → `[x]`). GitHub renders them as a progress list.
**Save evidence** where noted: a screenshot in `images/screenshots/` or a file in the repo.
Expected numbers for every check are in `docs/PLAN.md` → section 10.

Progress: Phase 0 ✅ · 1 ✅ · 2 ✅ · 3 ✅ · 4 🟡 · 5 🟡 · 6 ☐ · 6b ☐ · 7 ☐ · 8 ☐ · 9 ☐ · 10 ☐ · 11 ☐ · 12 ☐

> **Status (25 Sep 2026):** Phases 0–3 done. Phase 4 built: 5,012 → 4,800 deals, reject log (212), dates/amounts/probability/currency parsed and reconciled to CRM control totals, stages/departments/loss reasons/lead sources standardised, owners matched (4,800 with email), deals linked to master accounts (4,790 + 10 ACCOUNT_MISSING), blank currency filled (138 → 0). Only the owner screenshot + git commit left.
> **Next up:** Phase 5 parts 1–2 built by hand (Container D: 16,093 events → 12,017 stints, 206 regressed deals). Fix: Multi-Row `next_stage_no` → rows that don't exist = **Null**. Then part 3: per-deal Summarize (max stage, regression count, stage entered date, won_at → fills 48 missing close dates, lost_at_stage) and join back to the 4,800 deals.
> ⚠️ Alteryx licence ends 24 Oct 2026 — finish Phases 5–8 and 6b before then.

---

## Phase 0 — Setup (Day 1)

- [x] Create the folder `D:\Learnings\Alteryx-Tableau` and unzip the project files into it.
- [x] Apply for the free Alteryx **SparkED independent learner** licence (90 days) and install Alteryx Designer. Note the start date: licence ends 24 October 2026.
- [x] Install **Tableau Desktop Free Edition** (tableau.com → Products → Tableau Desktop → Free Edition).
- [x] Install **Git for Windows** (git-scm.com) and create a free **GitHub** account.
- [x] In Git Bash:
  ```bash
  cd /d/Learnings/Alteryx-Tableau
  git init
  git add .
  git commit -m "Project scaffold, synthetic raw data, plan and business definitions"
  ```
  Commit after every phase — a steady commit history shows recruiters how you work.
- [x] Open each raw file once (Excel is fine) just to *look* at it. Write 5 things that look wrong in `docs/profiling_notes.md`.

---

## Phase 1 — Profile the data in Alteryx (Day 1–2)

Goal: find the problems *before* fixing them. Profiling is the step most beginners skip.

- [x] New workflow `alteryx/00_profiling.yxmd`.
- [x] Input one v1 file (`opps_2025_03.csv`) and one v2 file (`opps_2026_03.csv`). In the Input tool set **Code Page = Unicode UTF-8** (otherwise "Müller" becomes "MÃ¼ller").
- [x] Add **Browse** tools and open the *Data Profile* view. Compare the column names of v1 vs v2.
- [x] For `Stage`, `Department`, `Currency`: **Summarize** → Group By the field + Count. You'll see every spelling.
- [x] Input `accounts_export.xlsx`. Notice the 3 junk rows and the footer. Fix with **Options → Start Data Import on Line 4**, then filter out the footer rows (`StartsWith([Account ID], "ACC-")`).
- [x] Fill `data/reference/value_aliases.csv` with every raw spelling you find (domains: stage, department, country, industry, company_size, loss_reason, lead_source). Compare with `docs/solutions/value_aliases_complete.csv` only when you're done.
- [x] 📸 Evidence: `images/screenshots/01_profiling_stage_spellings.png` (Summarize result showing messy stage names).

---

## Phase 2 — Batch macro: ingest 24 files with schema drift (Day 2–3)

Why a batch macro? A wildcard Input (`opps_*.csv`) fails or scrambles columns when files have different schemas. The macro reads **one file at a time**, renames its columns to a standard set, and then stacks the results.

**Build the macro `alteryx/macros/mc_ingest_crm_extract.yxmc`:**
- [x] New workflow. Add **Input Data** pointing at any one opportunities CSV (UTF-8, *Output File Name as Field → File Name only*).
- [x] Add **Input Data** for `data/reference/crm_field_mapping_v1_v2.csv`.
- [x] Add **Dynamic Rename**: left = the opportunities file, right = the mapping file. Mode **Take Field Names from Right Input Rows**, Old = `source_field`, New = `standard_field`.
- [x] Add a **Select** to rename `FileName` to `source_file`, and make every field `V_WString` (so all iterations stack cleanly).
- [x] Add **Macro Output** (Interface palette).
- [x] Add **Control Parameter** (Interface palette, label it `File Path`). Connect it to the file Input tool; an **Action** tool appears. Configure the Action: *Update Input Data value → replace the full file path*.
- [x] Workflow Configuration → Workflow tab → type **Batch Macro**. Save as `.yxmc`.
- [x] Interface Designer → Properties → *Output Mode*: **Auto Configure by Name (Wait Until All Iterations Run)**. This is what makes v1 and v2 stack even though some columns exist in only one of them.

**Build `alteryx/01_ingest_crm_extracts.yxmd`:**
- [x] **Directory** tool → folder `data\raw\crm_exports\opportunities`, pattern `opps_*.csv`.
- [x] Right-click canvas → Insert → Macro → your batch macro. Connect Directory to its control-parameter input (¿ icon), choose `FullPath`.
- [x] **Formula**: `crm_version = IIF([source_file] >= "opps_2025_07", "v2", "v1")` (FileName has no ".csv" extension).
- [x] Output to `data/output/staging/opportunities_unioned.yxdb` **and** `data/output/ai/raw_union_uncleaned.csv` (used in the AI experiment).
- [x] ✅ Check: 5,012 rows, 24 distinct `source_file` values.
- [x] 📸 `02_batch_macro_inside.png` (macro canvas) and `03_ingest_workflow.png` (Directory → macro).
- [x] `git commit -m "Batch macro ingests 24 CRM extracts with schema drift"`

---

## Phase 3 — Sales team & accounts (Day 3–4)

Start `alteryx/02_build_pipeline_mart.yxmd`. Add a **workflow constant**: Workflow Configuration → Workflow → Constants → `SnapshotDate = 2026-09-15`. Use it everywhere as `[User.SnapshotDate]`.

**Container A · Sales team**
- [x] Input roster (UTF-8) → **Data Cleansing** (trim leading/trailing whitespace).
- [x] **Find Replace** on `Department` against `value_aliases.csv` (filtered to domain = department, *entire field*, *case insensitive*, append `standard_value`).
- [x] Parse `Hire Date`, `Leave Date` (`DateTimeParse([Hire Date], "%d/%m/%Y")`).
- [x] `owner_is_active = IsNull([leave_date]) OR [leave_date] > [User.SnapshotDate]`.
- [x] Keep AEs (`Role = "Account Executive"`) as the owner lookup. Add a matching key `initial_key = Left([Full Name],1) + ". " + <surname>` for pass-2 owner matching.
- [x] Prorated quota for FY2026 (see business_definitions §4).

**Container B · Accounts**
- [x] Input xlsx (start on line 4) → Filter out footer rows.
- [x] Find Replace: country → `country_code`; industry; company size. Then Join to `country_region_mapping.csv` on `country_code`. ✅ Check: the Join's **L output is empty** (every country mapped).
- [x] **Formula** to build the dedupe key (or build the stretch **standard macro** `mc_normalize_company_name.yxmc` and reuse it):
  ```
  REGEX_Replace(
    REGEX_Replace(Trim(REGEX_Replace(Uppercase([Account Name]), "\s+", " ")), "\.+$", ""),
    "\s+(GMBH|AG|SE|SA|B\.V|N\.V|NV|S\.A|S\.À R\.L|LTD|PLC|DAC|SAS|AB|A/S|APS|AS|ASA|OY|OYJ|S\.L|S\.P\.A|S\.R\.L|LDA|SP\. Z O\.O|A\.S|S\.R\.O)$", "")
  ```
- [x] **Summarize**: group by `name_key` + `country_code`, `Min(account_id)` = `master_account_id`. Join back → an `account_id → master_account_id` map (765 rows). Keep the master rows as the account dimension (720 rows) with a clean display name (Trim + the master's name).
- [x] Coordinates: swap if `[Latitude] < 34 OR [Latitude] > 72` and longitude is in 34–72. For blanks, Summarize average lat/long by city + country and fill.
- [x] ✅ Check: 765 → 720 accounts; 0 accounts without coordinates.
- [x] 📸 `04_account_dedup.png` (Browse showing two records collapsing into one master).

---

## Phase 4 — Clean the opportunities (Day 4–6)

**Container C · Opportunities** (input: `opportunities_unioned.yxdb`)
- [x] Standard ID: `"OPP-" + PadLeft(REGEX_Replace([opportunity_id], "[^0-9]", ""), 6, "0")`.
- [x] **Unique** on all fields → removes exact duplicates. Send the D output to the rejected log with reason `EXACT_DUPLICATE`. ✅ 75.
- [x] Filter test records (`Contains([opportunity_name], "TEST")` or owner "CRM Admin" or account `ACC-00000`) → rejected, reason `TEST_RECORD`. ✅ 6.
- [x] Parse `last_modified_raw` (two formats), **Sort** by id + last_modified descending, **Unique** on id → keeps the latest version. Duplicates → rejected, reason `OLDER_VERSION`. ✅ 131 → **4,800 deals**.
- [x] **Multi-Field Formula** on all `*_raw` date fields (created, expected close, actual close, last activity), output type Date:
  ```
  IF REGEX_Match([_CurrentField_], "\d{4}-\d{2}-\d{2}") THEN DateTimeParse([_CurrentField_], "%Y-%m-%d")
  ELSEIF REGEX_Match([_CurrentField_], "\d{2}/\d{2}/\d{4}") THEN DateTimeParse([_CurrentField_], "%d/%m/%Y")
  ELSEIF REGEX_Match([_CurrentField_], "\d{2}\.\d{2}\.\d{4}") THEN DateTimeParse([_CurrentField_], "%d.%m.%Y")
  ELSE Null() ENDIF
  ```
  (Assumption to document: all slash dates are European DD/MM/YYYY.)
- [x] Amount: `IIF(IsEmpty(Trim([amount_raw])), Null(), ToNumber(REGEX_Replace([amount_raw], "[^0-9.\-]", "")))`.
- [x] Probability: strip `%`; if the raw value contains "." and the number ≤ 1, multiply by 100.
- [x] Currency: `Uppercase(Trim())`, `€` → `EUR`. Keep blanks for now.
- [x] Find Replace (value_aliases): stage, department, loss reason, lead source. Blank service line → "Unspecified".
- [x] ✅ **Reconcile parsing:** Summarize `amount_local` by *raw* currency (after upper-casing and € → EUR) and compare with `crm_control_totals.csv`. Every currency must match to the cent — if not, your amount parser missed a format.
- [x] Owners (the classic Join lesson):
  - v2: `owner_email = Lowercase(Trim([owner_email]))`.
  - v1 pass 1: reorder "Last, First", trim, collapse double spaces → Join to roster on name (make both sides lowercase).
  - Pass 2: take the **L output** (unmatched) → Join on `initial_key`.
  - Pass 3: remaining L output → Find Replace with `value_aliases` domain `owner` (e.g. "Anna Mueller").
  - ✅ Final unmatched = 0. 📸 `05_join_unmatched_owners.png` (the L output before your fix — this screenshot tells the story).
- [x] Accounts: Join `account_id` to the id→master map. The **L output** = 10 deals pointing to deleted accounts → keep them with `ACCOUNT_MISSING`. Union back.
- [x] Blank currency → currency of the account's country.

- [x] 📸 `05_join_unmatched_owners.png` (owner pass 1 **L** output, 68 rows — the problem before the fix)
- [ ] `git commit -m "Phase 4: deals cleaned, owners matched, accounts linked"`
---

## Phase 5 — Stage history → stage stints (Day 6–7)

**Container D**
- [x] Input `stage_history_log.csv` → Filter `Field = "Stage"` (16,289 rows).
- [x] Standard ID (same formula as Phase 4). Map Old/New values with the stage aliases.
- [x] Timestamp: first `Replace(Replace([Changed On], "T", " "), "Z", "")`, then parse `%Y-%m-%d %H:%M:%S` or `%d/%m/%Y %H:%M`.
- [x] **Unique** on id + new stage + timestamp **truncated to the minute** (`DateTimeTrim([changed_at], "minute")`) → 16,129. (The DD/MM/YYYY HH:MM format has no seconds, so exact timestamps don't match.) 💡 Note in your README: duplicates only become visible *after* parsing, because the same event was logged with different timestamp formats.
- [x] Join to the 4,800 clean deals; the **L output** (36 rows: deleted + test deals) → rejected, reason `ORPHAN_HISTORY`. ✅ 16,093.
- [x] Join `stage_rules.csv` to get `stage_order`, `governed_probability` and `stuck_after_days` *for the stage of that stint* (Phase 6b needs them). **Sort** by id, timestamp.
- [x] **Multi-Row Formula** (Group By `opportunity_id`) to create `exited_at` = the next row's timestamp:
  `[Row+1:changed_at]` — for the last row of each deal it is Null.
- [x] For open-stage rows: `exited_at` Null → use `[User.SnapshotDate]` and `is_current = True`. `days_in_stage = DateTimeDiff([exited_at], [changed_at], "days")`.
- [x] `exit_type`: Advanced / Regressed / Won / Lost / Still open (compare next row's stage order).
- [x] Keep open-stage rows → `pipeline_stage_stints`. ✅ 12,017 stints.
- [ ] **Summarize** per deal: `max_stage_reached`, `regression_count` (✅ 206 deals > 0), current-stage entered date, Closed Won timestamp (to fill the 48 missing close dates), stage before Closed Lost (`lost_at_stage`).
- [x] 📸 `06_multirow_stints.png` (Multi-Row Formula config + Browse).

---

## Phase 6 — Business rules (Day 7–8)

**Container E** (all definitions in `docs/business_definitions.md` — keep them in sync)
- [ ] Join `stage_rules.csv` → governed probability, forecast category, stuck threshold, is_open/is_won/is_lost.
- [ ] FX: `rate_month = IIF([is_open], "2026-09", DateTimeFormat([actual_close_date], "%Y-%m"))` → Join `fx_rates_monthly.csv` on month + currency. ✅ L output empty.
- [ ] `amount_eur`, outlier flag (> 5,000,000), negative flag, `weighted_amount_eur`.
- [ ] `days_in_current_stage`, `is_stuck`, `days_since_activity`, `is_stale` (> 30), `is_past_due`, `OWNER_LEFT`, `is_at_risk`, `risk_reasons` (text list).
- [ ] Deal type: Summarize first Closed Won date per master account → Join → Expansion if it's before `created_date`.
- [ ] Account status (Active Customer / Lapsed / Prospect) per master account → Join to deals.
- [ ] Deal size band, close quarter, expected close quarter, `sales_cycle_days`, `probability_gap_pts`.
- [ ] `dq_flags`: concatenate every flag that applies (`AMOUNT_MISSING;OWNER_LEFT;...`).
- [ ] Build `dept_quarter_summary`: targets + bookings by close quarter + open/weighted pipeline by expected close quarter and forecast category.

---

## Phase 6b — Predictive win probability (ML) (Day 9–11)

**Goal:** give every open deal a data-driven win probability, prove it beats the governed stage probabilities
on a backtest, and publish it as an *advisory* signal with a model card.
Put this in its own workflow: `alteryx/03_win_probability_model.yxmd`.

### 6b.1 Install and prepare
- [ ] Install **Alteryx Predictive Tools** (a separate free installer that adds the R-based tools). Download it from the same Alteryx downloads/licence portal as Designer. **The installer version must match your Designer version.**
- [ ] Restart Designer. ✅ The **Predictive** tool palette now shows Logistic Regression, Forest Model, Score, and more.
- [ ] In `02_build_pipeline_mart.yxmd`, at the end of Container E, add two Output Data tools:
  `data/output/staging/deals_for_model.yxdb` (all 4,800 deals with business-rule fields) and
  `data/output/staging/stints_for_model.yxdb` (the 12,017 stints). Run 02.
- [ ] Create `03_win_probability_model.yxmd`. Add two workflow constants: `SnapshotDate = 2026-09-15` and `BacktestDate = 2026-03-01`.
- [ ] 💡 If a Predictive tool shows an engine-compatibility error, open Workflow Configuration → Runtime and switch this one workflow off the AMP engine.

### 6b.2 Understand the leakage rule before touching data
A model may only use information that was **known at the moment of prediction**. Copy this table into `docs/model_card.md` (the template is already there):

| Allowed features (known while the deal is open) | Excluded features (leak the outcome) |
|---|---|
| `stage`, `service_line`, `department`, `sales_region`, `industry`, `company_size_band`, `deal_type`, `lead_source`, `sales_manager`, `amount_eur` (as log), days in stage so far, deal age, regressions so far, over stuck threshold (yes/no) | `rep_probability` (is 0 or 100 on closed deals), rep `forecast_category`, `loss_reason`, `actual_close_date`, `sales_cycle_days`, `last_activity_date` / `days_since_activity` / `is_stale` (equal the close date on closed deals), `max_stage_reached`, `lost_at_stage`, `expected_close_date` (only today's value is stored), `account_status` (uses wins that happened later) |

### 6b.3 Container H1 · Build "photos" of deals (training observations)
The trick: take a "photo" of each closed deal at day 0, 14, 30, 45, 60, 90 and 120 of every stage it passed through. Each photo only shows what was known on that day, plus the final outcome.
- [ ] Input both staging files. **Join** stints to deals on `opportunity_id` to bring in the features and outcome.
  ⚠️ In the Join's field list, keep `stage`, `stage_order`, `governed_probability` and `stuck_after_days` from the **stint** side and untick the deal's versions. A photo describes the stage the deal was in *at that time*, not its final stage. Getting this wrong is a subtle form of leakage.
- [ ] **Sort** by `opportunity_id`, `entered_at`. **Multi-Row Formula** (group by `opportunity_id`), new field `regressions_so_far` (Int16), rows that don't exist = 0:
  `IIF([Row-1:exit_type] = "Regressed", [Row-1:regressions_so_far] + 1, [Row-1:regressions_so_far])`
- [ ] **Text Input** with one column `checkpoint_day`: 0, 14, 30, 45, 60, 90, 120.
- [ ] **Append Fields**: target = stints, source = checkpoints → each stint appears 7 times.
- [ ] **Filter** `[days_in_stage] >= [checkpoint_day]` (the deal must still have been in that stage on that day).
- [ ] **Formula** tool:
  ```
  days_in_stage_so_far = [checkpoint_day]
  observation_date     = DateTimeAdd([entered_at], [checkpoint_day], "days")
  deal_age_days        = DateTimeDiff([observation_date], [created_date], "days")
  over_stuck_threshold = IIF([checkpoint_day] > [stuck_after_days], "Yes", "No")
  log_amount           = Log10([amount_eur])
  outcome              = IIF([is_won], "Won", "Lost")
  ```
- [ ] **Filter** out rows with null `amount_eur` or `AMOUNT_OUTLIER` in `dq_flags`.

### 6b.4 Container H2 · Backtest split (by time, never random)
We pretend it is **1 March 2026**, train only on what was known then, and check the model against what actually happened. All photos of one deal stay on the same side of the split. Splitting randomly would let the model "see the future".
- [ ] **TRAIN**: photos of deals that are closed (`is_won` or `is_lost`) with `actual_close_date < [User.BacktestDate]`. ✅ About 20,000 rows from about 2,900 deals. The win rate per row (~43%) is higher than per deal, because deals that reach later stages produce more photos. That's expected.
- [ ] **TEST** ("the pipeline as it looked on 1 March 2026"): from the *stints* (not photos), keep rows where `entered_at <= BacktestDate` AND (`exited_at > BacktestDate` OR `exited_at` is Null). Then:
  ```
  days_in_stage_so_far = Min(DateTimeDiff([User.BacktestDate], [entered_at], "days"), 120)
  deal_age_days        = DateTimeDiff([User.BacktestDate], [created_date], "days")
  over_stuck_threshold = IIF([days_in_stage_so_far] > [stuck_after_days], "Yes", "No")
  outcome_by_snapshot  = IIF([is_won], "Won", IIF([is_lost], "Lost", "Still open"))
  ```
  Use `regressions_so_far` from the Multi-Row step. Also apply the same `log_amount` and amount filters. ✅ About 635 deals open on 1 March 2026; about 550 of them were decided by 15 September.
- [ ] Cap `days_in_stage_so_far` at 120 in TRAIN too, so both sides use identical feature definitions.

### 6b.5 Container H3 · Train two models on TRAIN
- [ ] **Logistic Regression** tool: model name `wp_logit`, target `outcome`, predictors = the allowed features **except `department`**. Service line and sales manager already imply the department. Giving a regression model the same information twice (multicollinearity) makes its coefficients unstable.
- [ ] **Forest Model** tool: model name `wp_forest`, same target, all allowed features (trees handle overlapping features). Default settings are fine; set a seed for reproducibility.
- [ ] Browse each tool's **R (report)** output. 📸 `17_logit_coefficients.png`, `18_forest_variable_importance.png`. Write in the model card which three features matter most and whether that makes business sense.

### 6b.6 Container H4 · Evaluate on the backtest
- [ ] **Score** tool × 2 (model → M input, TEST → D input). Each adds `Score_Won` = predicted probability of winning.
- [ ] Keep decided deals only (`outcome_by_snapshot` ≠ "Still open"), create `y = IIF([outcome_by_snapshot] = "Won", 1, 0)`.
- [ ] **Brier score** (lower = better; the average squared distance between prediction and reality): Formula `(Score_Won - y)^2` → Summarize Avg. Do the same with `governed_probability` in place of `Score_Won`.
- [ ] **Calibration**: `prob_bin = Floor([Score_Won] * 10) / 10` → Summarize by bin: Count, Avg(`Score_Won`), Avg(`y`). A good model's predicted and actual averages sit close together. Add a `method` field ("Model" / "Governed") and **Union** both results → this becomes a Tableau chart.
- [ ] (Optional) **Model Comparison** tool for AUC, a 0.5–1.0 score of how well the model ranks winners above losers. If it isn't in your palette, skip it; Brier and calibration are enough.
- [ ] **Business backtest** on all TEST deals (Summarize): `Sum(amount_eur × governed_probability)`, `Sum(amount_eur × Score_Won)`, actual won € by 15 Sep, € still open. Error % = forecast ÷ actual − 1.
- [ ] Pick the model with the lower Brier score and record the decision in the model card.
- [ ] ✅ Expected ballpark: model AUC ≈ 0.72–0.76 vs governed ≈ 0.67; Brier ≈ 0.20 vs ≈ 0.26; the governed forecast overstates actual won € far more than the model does. 💡 That last point is a business finding in itself: the stage probabilities in `stage_rules.csv` look too optimistic and Sales Ops should review them. This is the governance loop in action.

### 6b.7 Container H5 · Retrain on everything and score today's pipeline
- [ ] **TRAIN_FULL**: photos of all deals closed before `SnapshotDate` (✅ about 29,000 rows from about 4,070 deals). Copy your chosen model tool, same settings, name it `wp_final`.
- [ ] **SCORE set** = today's open deals from `deals_for_model`:
  `days_in_stage_so_far = Min([days_in_current_stage], 120)`, `deal_age_days = DateTimeDiff([User.SnapshotDate], [created_date], "days")`, `regressions_so_far = [regression_count]`, `over_stuck_threshold = IIF([is_stuck], "Yes", "No")`, `log_amount = Log10([amount_eur])`.
  Deals without an amount can't be scored. Keep them with `model_status = "NOT_SCORED_NO_AMOUNT"`.
- [ ] **Score** → Formula:
  ```
  model_win_probability = [Score_Won]
  model_weighted_eur    = [amount_eur] * [Score_Won]
  model_vs_governed_pts = ([Score_Won] - [governed_probability]) * 100
  model_review_flag     = IIF(Abs([model_vs_governed_pts]) >= 25, "Review", "OK")
  model_version         = "wp-v1-2026-09-15"
  ```
- [ ] **Test** tool guardrails: every score between 0 and 1; scored rows + not-scored rows = 724 open deals.
- [ ] 📸 `19_ml_workflow.png` (the whole of `03_win_probability_model.yxmd`).

### 6b.8 Outputs and documentation
- [ ] **Output Data** → `data/output/model_scores.hyper` (opportunity_id + the fields above), `data/output/model_calibration.hyper`, `data/output/model_backtest.hyper`.
- [ ] Fill in `docs/model_card.md` (purpose, data, features, excluded features, split, metrics, limitations, status = *Advisory*).
- [ ] `git commit -m "Leakage-safe win probability model with time-based backtest and model card"`

---

## Phase 7 — Reconciliation & tests (Day 12)

**Container F** — this is what makes the numbers trustworthy.
- [ ] Build a table of checks: `check_name`, `expected`, `actual`, `status` (PASS/FAIL). Include at least:
  1. Unique opportunities = CRM control total − test records (4,800)
  2. Deal counts by stage = control totals (6 checks)
  3. Sum of amount by raw currency = control totals
  4. Master accounts = distinct name keys (720)
  5. Every deal has an owner (0 unmatched)
  6. Every deal has a stage_rules match
  7. Every closed deal has a close date
  8. Row bridge: raw rows − exact dups − tests − older versions = final deals
- [ ] Add the **Test** tool: *Expression is True for all records* → `[status] = "PASS"`. If any check fails, the workflow shows an error. Add a **Message** tool that writes a friendly summary.
- [ ] 📸 `07_reconciliation_report.png` (Browse of the checks table, all PASS).

---

## Phase 8 — Outputs (Day 12)

**Container G**
- [ ] A final **Select** per table: rename to the names in PLAN §4.1, set types (Date, Double, Bool, V_WString), drop helper fields.
- [ ] **Output Data** → `data/output/*.hyper` (file format *Tableau Hyper Extract*). One file per table from PLAN §4.
- [ ] Also write `quality/dq_reconciliation.csv`, `quality/rejected_records.csv`, `ai/pipeline_deals_for_ai.csv`.
- [ ] Use Tool Containers with colours + a Comment in each container. Tidy the canvas (Ctrl+A → align).
- [ ] 📸 `08_alteryx_full_workflow.png` (entire canvas, zoomed to fit).
- [ ] `git commit -m "Pipeline mart: cleaning, business rules, reconciliation, Hyper outputs"`

---

## Phase 9 — Tableau dashboards (Week 4–5)

Setup:
- [ ] Tableau Desktop → Connect → *More…* → open `pipeline_deals.hyper`. Add `pipeline_stage_stints`, `model_scores` and `sales_team` as **related** tables (drag onto canvas; relationships on `opportunity_id`, `opportunity_id` and `owner_email`). Add `dept_quarter_summary`, `model_calibration` and `model_backtest` as separate data sources.
- [ ] Dashboard size: Fixed 1400 × 900. Create a colour palette (Won green, Lost grey, stage blues, risk amber/red) and reuse it.
- [ ] Create calculated fields listed in business_definitions §5 (Win Rate, Coverage, etc.) — put them in a folder called "Governed metrics".

Pages (specs in PLAN §6):
- [ ] **Page 1 Pipeline Health & Forecast** — KPIs, funnel with conversion %, forecast by quarter vs target, bookings trend. 📸 `10_tableau_pipeline_health.png`
- [ ] **Page 2 Deal Velocity & Stuck Deals** — days in stage vs threshold, heatmap, risk scatter, fix-this-week table. 📸 `11_tableau_velocity.png`
- [ ] **Page 3 Manager & Department Performance** — drill-down, quota bullets, pipeline vs win-rate scatter, forecast honesty. 📸 `12_tableau_performance.png`
- [ ] **Page 4 Europe Market Opportunity** — dual-axis map, industry × region heatmap, service mix, top accounts. 📸 `13_tableau_europe_map.png`
- [ ] **Page 5 Win Probability (ML)** — calibration chart (predicted vs actual, model vs governed, 45° reference line); backtest bars (governed forecast vs model forecast vs actual won); "gut vs rules vs data" by manager (avg rep vs governed vs model probability); open-deal table sorted by `model_review_flag`. 📸 `14_tableau_win_probability.png`
- [ ] Add a **Model-weighted Pipeline €** KPI tile to Page 1, labelled *advisory*.
- [ ] **Page 6 Data Trust** — row bridge + checks. 📸 `15_tableau_data_trust.png`
- [ ] Navigation buttons between pages; one filter action per page; tooltips checked.
- [ ] Write a one-line "So what" on each page (e.g. "Data & AI Proposal deals wait 47 days — escalate budget approvals").
- [ ] File → Save As → **Packaged Workbook** `tableau/Nexora_Pipeline_Intelligence.twbx`.
- [ ] (Optional) Recreate in Tableau Public for a shareable link.
- [ ] `git commit -m "Tableau executive dashboards"`

---

## Phase 10 — AI experiment (Week 6)

- [ ] Follow `ai_experiment/README.md` (Rounds A, B, C). Fill in the results table.
- [ ] 📸 `20_ai_ungrounded.png`, `21_ai_grounded.png` (chat screenshots, same question, different answers).
- [ ] `git commit -m "AI grounded vs ungrounded experiment"`

---

## Phase 11 — Screenshots & README (Week 6)

- [ ] All screenshots saved in `images/screenshots/` with the names above (PNG, full dashboard, no personal info visible).
- [ ] Send the screenshots + your AI results + your final numbers to Claude → finalise `README.md` and create the LinkedIn image.
- [ ] Proof-read the README on GitHub (images show? links work?).

---

## Phase 12 — Publish (Week 6)

- [ ] On github.com → New repository `alteryx-tableau-sales-pipeline` (Public, no README — you have one).
- [ ] In Git Bash:
  ```bash
  git branch -M main
  git remote add origin https://github.com/<your-username>/alteryx-tableau-sales-pipeline.git
  git push -u origin main
  ```
- [ ] Add repo topics: `alteryx`, `tableau`, `data-analytics`, `etl`, `sales-analytics`, `data-quality`, `ai`.
- [ ] Pin the repo on your GitHub profile.
- [ ] Publish the LinkedIn post (draft will be in `images/linkedin/`).
