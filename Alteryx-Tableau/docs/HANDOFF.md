# Handoff — Alteryx + Tableau + ML Sales Pipeline Project
*Summary of the planning conversation (claude.ai chat, 24 Sep 2026) so Claude in Cowork can continue.*

## Who I am and how to work with me
- Beginner in data analytics, learning Alteryx (data prep) and Tableau, aiming to stand out in the 2026 job market.
- Explain in simple language, step by step, like a teacher. One phase at a time; check my work against the answer keys.
- Everything must be **free**: Alteryx SparkED independent-learner licence (90 days), Tableau Desktop Free Edition (saves locally, can't publish; Tableau Public optional for a link), Git + GitHub, a free AI chat. **Tableau Next is paid, so we don't use it.**
- Project folder: `D:\Learnings\Alteryx-Tableau` (becomes a public GitHub repo).

## Project in one sentence
A fictional European IT consultancy ("Nexora Consulting") has a messy CRM. I clean and reconcile it in Alteryx,
write governed business definitions, build a leakage-safe ML win-probability model, and deliver executive
Tableau dashboards. Then I run an AI experiment showing an AI assistant answers wrongly on raw data but correctly
when grounded in my definitions. **Core message:** the analyst's value is the business logic AI can't infer.

## Locked decisions
- **Use case:** sales pipeline for senior executives (CRO). 18 European countries, 7 sales regions, 4 departments (Cloud & Infrastructure, Data & AI, Cybersecurity, ERP & Business Apps), 8 sales managers, 29 account executives, about 720 accounts, 4,800 opportunities. Snapshot date **15 Sep 2026**.
- **Funnel:** Discovery (0–50%) → Solution Design (50–75%) → Proposal (75–90%) → Negotiation (90–100%) → Closed Won (100%), plus **Closed Lost with loss reasons**.
- **Size:** about 4,000 deals created over 24 months (Oct 2024 – Sep 2026) plus a go-live import (Apr–Sep 2024) inside `opps_2024_10.csv`.
- **Dashboard pages, in priority order:** 1 Pipeline Health & Forecast · 2 Deal Velocity & Stuck Deals · 3 Manager & Department Performance · 4 Europe Market Opportunity (map) · 5 Win Probability (ML) · 6 Data Trust.
- **Phase 6b (ML win probability) is mandatory**, not optional.
- **Design principles:** row-level logic in Alteryx, Tableau only aggregates · business rules stored as data (`stage_rules.csv`, `value_aliases.csv`) · inspect every Join's L/R output · nothing silently deleted (`rejected_records.csv`) · reconcile to `crm_control_totals.csv` with the Test tool · one workflow constant `SnapshotDate`.
- **ML design:** "photos" of closed deals at day 0/14/30/45/60/90/120 of each stage · strict leakage exclusions · time-based backtest at 1 Mar 2026 · Logistic Regression vs Forest Model · Brier score, calibration, business backtest · retrain on all history and score open deals · status **Advisory** (documented in the model card). Prototype results on this data: AUC ≈ 0.74 vs governed ≈ 0.67; Brier ≈ 0.20 vs 0.26.

## Files already in the folder (read these first)
| File | Purpose |
|---|---|
| `docs/PLAN.md` | Full plan: problem, sources, data model, Alteryx containers, Tableau page specs, AI + ML design, milestones, **answer key (section 10)**, planted stories (spoiler, section 9) |
| `docs/CHECKLIST.md` | Step-by-step build guide, Phases 0–12 including **6b**, with tools, formulas, checks, screenshot names, git commits |
| `docs/business_definitions.md` | The governed "mini semantic layer": every metric, rule and reason (includes section 5b, ML advisory) |
| `docs/model_card.md` | ML model card template to fill in during Phase 6b |
| `docs/profiling_notes.md` | Template for Phase 1 findings |
| `docs/solutions/value_aliases_complete.csv` | Spoiler answer key for the alias table |
| `ai_experiment/README.md` | Grounded vs ungrounded AI protocol, prompts, results table |
| `README.md` | GitHub README skeleton with TODOs, to finalise from screenshots |
| `data/raw/crm_exports/…` | Messy synthetic CRM data (24 monthly CSVs, stage log, accounts xlsx, team roster) |
| `data/reference/…` | FX rates, targets, field mapping v1→v2, stage rules, territories, control totals, alias starter |
| `scripts/generate_raw_data.py` | Seeded generator (re-running it gives identical data) |

## Current status
- Folder created. Project files downloaded (zip). **Next step: Phase 0** (unzip into the folder, install tools, `git init` and first commit), then Phase 1 profiling.
- Nothing built in Alteryx or Tableau yet.

## Still to do later
1. Guide me through Phases 0–12 and keep `docs/CHECKLIST.md` ticked and up to date.
2. Review my Alteryx outputs against the answer key; help debug errors.
3. After Phase 11: finalise `README.md` with my screenshots, reconciliation results, ML results and AI experiment results.
4. Design a LinkedIn post image (data, insights, decision-making, AI) and write the LinkedIn post about bridging business and tech with AI.
5. Give me the exact git commands to publish the repo.
