# Build Checklist — Alteryx + Tableau Sales Pipeline Project

Tick boxes as you go (`[ ]` → `[x]`). GitHub renders them as a progress list.
**Save evidence** where noted: a screenshot in `images/screenshots/` or a file in the repo.
Expected numbers for every check are in `docs/PLAN.md` → section 10.

Progress: Phase 0 ☐ · 1 ☐ · 2 ☐ · 3 ☐ · 4 ☐ · 5 ☐ · 6 ☐ · 7 ☐ · 8 ☐ · 9 ☐ · 10 ☐ · 11 ☐ · 12 ☐

---

## Phase 0 — Setup (Day 1)

- [ ] Create the folder `D:\Learnings\Alteryx-Tableau` and unzip the project files into it.
- [ ] Apply for the free Alteryx **SparkED independent learner** licence (90 days) and install Alteryx Designer. Note the start date: licence ends ____________.
- [ ] Install **Tableau Desktop Free Edition** (tableau.com → Products → Tableau Desktop → Free Edition).
- [ ] Install **Git for Windows** (git-scm.com) and create a free **GitHub** account.
- [ ] In Git Bash:
  ```bash
  cd /d/Learnings/Alteryx-Tableau
  git init
  git add .
  git commit -m "Project scaffold, synthetic raw data, plan and business definitions"
  ```
  Commit after every phase — a steady commit history shows recruiters how you work.
- [ ] Open each raw file once (Excel is fine) just to *look* at it. Write 5 things that look wrong in `docs/profiling_notes.md`.

---

## Phase 1 — Profile the data in Alteryx (Day 1–2)

Goal: find the problems *before* fixing them. Profiling is the step most beginners skip.

- [ ] New workflow `alteryx/00_profiling.yxmd`.
- [ ] Input one v1 file (`opps_2025_03.csv`) and one v2 file (`opps_2026_03.csv`). In the Input tool set **Code Page = Unicode UTF-8** (otherwise "Müller" becomes "MÃ¼ller").
- [ ] Add **Browse** tools and open the *Data Profile* view. Compare the column names of v1 vs v2.
- [ ] For `Stage`, `Department`, `Currency`: **Summarize** → Group By the field + Count. You'll see every spelling.
- [ ] Input `accounts_export.xlsx`. Notice the 3 junk rows and the footer. Fix with **Options → Start Data Import on Line 4**, then filter out the footer rows (`StartsWith([Account ID], "ACC-")`).
- [ ] Fill `data/reference/value_aliases.csv` with every raw spelling you find (domains: stage, department, country, industry, company_size, loss_reason, lead_source). Compare with `docs/solutions/value_aliases_complete.csv` only when you're done.
- [ ] 📸 Evidence: `images/screenshots/01_profiling_stage_spellings.png` (Summarize result showing messy stage names).

---

## Phase 2 — Batch macro: ingest 24 files with schema drift (Day 2–3)

Why a batch macro? A wildcard Input (`opps_*.csv`) fails or scrambles columns when files have different schemas. The macro reads **one file at a time**, renames its columns to a standard set, and then stacks the results.

**Build the macro `alteryx/macros/mc_ingest_crm_extract.yxmc`:**
- [ ] New workflow. Add **Input Data** pointing at any one opportunities CSV (UTF-8, *Output File Name as Field → File Name only*).
- [ ] Add **Input Data** for `data/reference/crm_field_mapping_v1_v2.csv`.
- [ ] Add **Dynamic Rename**: left = the opportunities file, right = the mapping file. Mode **Take Field Names from Right Input Rows**, Old = `source_field`, New = `standard_field`.
- [ ] Add a **Select** to rename `FileName` to `source_file`, and make every field `V_WString` (so all iterations stack cleanly).
- [ ] Add **Macro Output** (Interface palette).
- [ ] Add **Control Parameter** (Interface palette, label it `File Path`). Connect it to the file Input tool; an **Action** tool appears. Configure the Action: *Update Input Data value → replace the full file path*.
- [ ] Workflow Configuration → Workflow tab → type **Batch Macro**. Save as `.yxmc`.
- [ ] Interface Designer → Properties → *Output Mode*: **Auto Configure by Name (Wait Until All Iterations Run)**. This is what makes v1 and v2 stack even though some columns exist in only one of them.

**Build `alteryx/01_ingest_crm_extracts.yxmd`:**
- [ ] **Directory** tool → folder `data\raw\crm_exports\opportunities`, pattern `opps_*.csv`.
- [ ] Right-click canvas → Insert → Macro → your batch macro. Connect Directory to its control-parameter input (¿ icon), choose `FullPath`.
- [ ] **Formula**: `crm_version = IIF([source_file] >= "opps_2025_07.csv", "v2", "v1")`.
- [ ] Output to `data/output/staging/opportunities_unioned.yxdb` **and** `data/output/ai/raw_union_uncleaned.csv` (used in the AI experiment).
- [ ] ✅ Check: 5,012 rows, 24 distinct `source_file` values.
- [ ] 📸 `02_batch_macro_inside.png` (macro canvas) and `03_ingest_workflow.png` (Directory → macro).
- [ ] `git commit -m "Batch macro ingests 24 CRM extracts with schema drift"`

---

## Phase 3 — Sales team & accounts (Day 3–4)

Start `alteryx/02_build_pipeline_mart.yxmd`. Add a **workflow constant**: Workflow Configuration → Workflow → Constants → `SnapshotDate = 2026-09-15`. Use it everywhere as `[User.SnapshotDate]`.

**Container A · Sales team**
- [ ] Input roster (UTF-8) → **Data Cleansing** (trim leading/trailing whitespace).
- [ ] **Find Replace** on `Department` against `value_aliases.csv` (filtered to domain = department, *entire field*, *case insensitive*, append `standard_value`).
- [ ] Parse `Hire Date`, `Leave Date` (`DateTimeParse([Hire Date], "%d/%m/%Y")`).
- [ ] `owner_is_active = IsNull([leave_date]) OR [leave_date] > [User.SnapshotDate]`.
- [ ] Keep AEs (`Role = "Account Executive"`) as the owner lookup. Add a matching key `initial_key = Left([Full Name],1) + ". " + <surname>` for pass-2 owner matching.
- [ ] Prorated quota for FY2026 (see business_definitions §4).

**Container B · Accounts**
- [ ] Input xlsx (start on line 4) → Filter out footer rows.
- [ ] Find Replace: country → `country_code`; industry; company size. Then Join to `country_region_mapping.csv` on `country_code`. ✅ Check: the Join's **L output is empty** (every country mapped).
- [ ] **Formula** to build the dedupe key (or build the stretch **standard macro** `mc_normalize_company_name.yxmc` and reuse it):
  ```
  REGEX_Replace(
    REGEX_Replace(Trim(REGEX_Replace(Uppercase([Account Name]), "\s+", " ")), "\.+$", ""),
    "\s+(GMBH|AG|SE|SA|B\.V|N\.V|NV|S\.A|S\.À R\.L|LTD|PLC|DAC|SAS|AB|A/S|APS|AS|ASA|OY|OYJ|S\.L|S\.P\.A|S\.R\.L|LDA|SP\. Z O\.O|A\.S|S\.R\.O)$", "")
  ```
- [ ] **Summarize**: group by `name_key` + `country_code`, `Min(account_id)` = `master_account_id`. Join back → an `account_id → master_account_id` map (765 rows). Keep the master rows as the account dimension (720 rows) with a clean display name (Trim + the master's name).
- [ ] Coordinates: swap if `[Latitude] < 34 OR [Latitude] > 72` and longitude is in 34–72. For blanks, Summarize average lat/long by city + country and fill.
- [ ] ✅ Check: 765 → 720 accounts; 0 accounts without coordinates.
- [ ] 📸 `04_account_dedup.png` (Browse showing two records collapsing into one master).

---

## Phase 4 — Clean the opportunities (Day 4–6)

**Container C · Opportunities** (input: `opportunities_unioned.yxdb`)
- [ ] Standard ID: `"OPP-" + PadLeft(REGEX_Replace([opportunity_id], "[^0-9]", ""), 6, "0")`.
- [ ] **Unique** on all fields → removes exact duplicates. Send the D output to the rejected log with reason `EXACT_DUPLICATE`. ✅ 75.
- [ ] Filter test records (`Contains([opportunity_name], "TEST")` or owner "CRM Admin" or account `ACC-00000`) → rejected, reason `TEST_RECORD`. ✅ 6.
- [ ] Parse `last_modified_raw` (two formats), **Sort** by id + last_modified descending, **Unique** on id → keeps the latest version. Duplicates → rejected, reason `OLDER_VERSION`. ✅ 131 → **4,800 deals**.
- [ ] **Multi-Field Formula** on all `*_raw` date fields (created, expected close, actual close, last activity), output type Date:
  ```
  IF REGEX_Match([_CurrentField_], "\d{4}-\d{2}-\d{2}") THEN DateTimeParse([_CurrentField_], "%Y-%m-%d")
  ELSEIF REGEX_Match([_CurrentField_], "\d{2}/\d{2}/\d{4}") THEN DateTimeParse([_CurrentField_], "%d/%m/%Y")
  ELSEIF REGEX_Match([_CurrentField_], "\d{2}\.\d{2}\.\d{4}") THEN DateTimeParse([_CurrentField_], "%d.%m.%Y")
  ELSE Null() ENDIF
  ```
  (Assumption to document: all slash dates are European DD/MM/YYYY.)
- [ ] Amount: `IIF(IsEmpty(Trim([amount_raw])), Null(), ToNumber(REGEX_Replace([amount_raw], "[^0-9.\-]", "")))`.
- [ ] Probability: strip `%`; if the raw value contains "." and the number ≤ 1, multiply by 100.
- [ ] Currency: `Uppercase(Trim())`, `€` → `EUR`. Keep blanks for now.
- [ ] Find Replace (value_aliases): stage, department, loss reason, lead source. Blank service line → "Unspecified".
- [ ] ✅ **Reconcile parsing:** Summarize `amount_local` by *raw* currency (after upper-casing and € → EUR) and compare with `crm_control_totals.csv`. Every currency must match to the cent — if not, your amount parser missed a format.
- [ ] Owners (the classic Join lesson):
  - v2: `owner_email = Lowercase(Trim([owner_email]))`.
  - v1 pass 1: reorder "Last, First", trim, collapse double spaces → Join to roster on name (make both sides lowercase).
  - Pass 2: take the **L output** (unmatched) → Join on `initial_key`.
  - Pass 3: remaining L output → Find Replace with `value_aliases` domain `owner` (e.g. "Anna Mueller").
  - ✅ Final unmatched = 0. 📸 `05_join_unmatched_owners.png` (the L output before your fix — this screenshot tells the story).
- [ ] Accounts: Join `account_id` to the id→master map. The **L output** = 10 deals pointing to deleted accounts → keep them with `ACCOUNT_MISSING`. Union back.
- [ ] Blank currency → currency of the account's country.

---

## Phase 5 — Stage history → stage stints (Day 6–7)

**Container D**
- [ ] Input `stage_history_log.csv` → Filter `Field = "Stage"` (16,289 rows).
- [ ] Standard ID (same formula as Phase 4). Map Old/New values with the stage aliases.
- [ ] Timestamp: first `Replace(Replace([Changed On], "T", " "), "Z", "")`, then parse `%Y-%m-%d %H:%M:%S` or `%d/%m/%Y %H:%M`.
- [ ] **Unique** on id + new stage + parsed timestamp → 16,129. 💡 Note in your README: duplicates only become visible *after* parsing, because the same event was logged with different timestamp formats.
- [ ] Join to the 4,800 clean deals; the **L output** (36 rows: deleted + test deals) → rejected, reason `ORPHAN_HISTORY`. ✅ 16,093.
- [ ] Join `stage_rules.csv` to get `stage_order`. **Sort** by id, timestamp.
- [ ] **Multi-Row Formula** (Group By `opportunity_id`) to create `exited_at` = the next row's timestamp:
  `[Row+1:changed_at]` — for the last row of each deal it is Null.
- [ ] For open-stage rows: `exited_at` Null → use `[User.SnapshotDate]` and `is_current = True`. `days_in_stage = DateTimeDiff([exited_at], [changed_at], "days")`.
- [ ] `exit_type`: Advanced / Regressed / Won / Lost / Still open (compare next row's stage order).
- [ ] Keep open-stage rows → `pipeline_stage_stints`. ✅ 12,017 stints.
- [ ] **Summarize** per deal: `max_stage_reached`, `regression_count` (✅ 206 deals > 0), current-stage entered date, Closed Won timestamp (to fill the 48 missing close dates), stage before Closed Lost (`lost_at_stage`).
- [ ] 📸 `06_multirow_stints.png` (Multi-Row Formula config + Browse).

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

## Phase 7 — Reconciliation & tests (Day 8)

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

## Phase 8 — Outputs (Day 8)

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
- [ ] Tableau Desktop → Connect → *More…* → open `pipeline_deals.hyper`. Add `pipeline_stage_stints` and `sales_team` as **related** tables (drag onto canvas; relationships on `opportunity_id` and `owner_email`). Add `dept_quarter_summary` as a second data source.
- [ ] Dashboard size: Fixed 1400 × 900. Create a colour palette (Won green, Lost grey, stage blues, risk amber/red) and reuse it.
- [ ] Create calculated fields listed in business_definitions §5 (Win Rate, Coverage, etc.) — put them in a folder called "Governed metrics".

Pages (specs in PLAN §6):
- [ ] **Page 1 Pipeline Health & Forecast** — KPIs, funnel with conversion %, forecast by quarter vs target, bookings trend. 📸 `10_tableau_pipeline_health.png`
- [ ] **Page 2 Deal Velocity & Stuck Deals** — days in stage vs threshold, heatmap, risk scatter, fix-this-week table. 📸 `11_tableau_velocity.png`
- [ ] **Page 3 Manager & Department Performance** — drill-down, quota bullets, pipeline vs win-rate scatter, forecast honesty. 📸 `12_tableau_performance.png`
- [ ] **Page 4 Europe Market Opportunity** — dual-axis map, industry × region heatmap, service mix, top accounts. 📸 `13_tableau_europe_map.png`
- [ ] **Page 5 Data Trust** — row bridge + checks. 📸 `14_tableau_data_trust.png`
- [ ] Navigation buttons between pages; one filter action per page; tooltips checked.
- [ ] Write a one-line "So what" on each page (e.g. "Data & AI Proposal deals wait 47 days — escalate budget approvals").
- [ ] File → Save As → **Packaged Workbook** `tableau/Nexora_Pipeline_Intelligence.twbx`.
- [ ] (Optional) Recreate in Tableau Public for a shareable link.
- [ ] `git commit -m "Tableau executive dashboards"`

---

## Phase 10 — AI experiment (Week 6)

- [ ] Follow `ai_experiment/README.md` (Rounds A, B, C). Fill in the results table.
- [ ] 📸 `15_ai_ungrounded.png`, `16_ai_grounded.png` (chat screenshots, same question, different answers).
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
