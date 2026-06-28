# data-analysis-project

[![lang: English](https://img.shields.io/badge/lang-English-2b6cb0.svg)](README.md)
[![lang: 한국어](https://img.shields.io/badge/lang-한국어-lightgrey.svg)](README.ko.md)
&nbsp;
[![methodology: CRISP-DM](https://img.shields.io/badge/methodology-CRISP--DM-2f855a.svg)](#crisp-dm)
[![license: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](#license)

> An end-to-end, gate-verified data-analysis workflow — from framing a real problem to shipping a shareable HTML report. A Claude Code **plugin** (one skill + 3 subagents + 3 scripts) that is an **opinionated implementation of [CRISP-DM](#crisp-dm)**.

🇰🇷 **한국어 README → [README.ko.md](README.ko.md)**

### What it is
`data-analysis-project` is a **Claude Code plugin that packages a single skill**. The "product" is not code but the **prompt content** — the instructions a future Claude reads when it runs a data-analysis project.

Core thesis: **weak analyses fail at framing and goal-setting, not at modeling.** So every stage ends in a **gate** that must genuinely pass before the next begins, and a *reframe* (the problem wasn't what you assumed) is treated as a strong result, not a failure.

### The 9 gated stages
1. **Frame & qualify** — 3 tests (specific · real · actionable) + user sign-off + a lightweight situation scan (data · legal/license · cost · risk)
2. **Set the goal** — one-sentence goal + the decision + a **business bar** + a **model-agnostic quantitative target** (baseline-relative by default)
3. **Source the data** — real sample fetched; schema · keys · access · license verified
4. **Profile the data** — `data_quality.py` profile (null-rate · outliers · dupes · distributions); measure size · grain · seasonality · imbalance
5. **Plan the method** — sub-question → method → verification, with a named metric + finalized threshold (predictive: baseline-first / descriptive: method-appropriate validity)
6. **Prepare & validate** — handle missing/outliers (judgment via the `data-quality` agent); gate = `data_quality.py --strict`; verify joins, block leakage, log transforms
7. **Build the notebook** — one cell = one action, markdown interpretation between cells; for predictive models: fit, tune, rank candidates, select a winner
8. **Conclude** — explicit **MET / PARTIAL / NOT-MET verdict** (achieved vs threshold) + no false success + user go/no-go on a shortfall
9. **Write the report** — a self-contained, **bilingual (ko/en) HTML data essay** that passes a screenshot visual-review gate

Progress is tracked with a **per-stage Claude TODO** (in-progress on entry, completed only when the gate passes).

### <a id="crisp-dm"></a>CRISP-DM
This plugin is an **opinionated implementation of CRISP-DM** (Cross-Industry Standard Process for Data Mining). The 9 gated stages map onto CRISP-DM's 6 phases:

| CRISP-DM phase | Stage(s) here |
|---|---|
| 1. Business Understanding | **1** Frame & qualify · **2** Set the goal |
| 2. Data Understanding | **3** Source the data · **4** Profile the data |
| 3. Data Preparation | **6** Prepare & validate |
| 4. Modeling | **5** Plan the method · **7** Build the notebook |
| 5. Evaluation | **8** Conclude & verdict (MET/NOT-MET + go/no-go) |
| 6. Deployment | **9** Write the report |

It is faithful to CRISP-DM's **intent**: Business Understanding and Evaluation are load-bearing, evaluation is judged against the **business goal** (not just a model metric), and a shortfall **loops back** (reframe / refine) instead of shipping a false success — CRISP-DM's defining non-linear move. Deployment is deliberately scoped to **producing the final report** (an analysis/research deliverable), not model serving/monitoring.

### Components
- **Skill** `skills/data-analysis-project/` — `SKILL.md` (always-loaded entry point) + `references/` (per-stage detail) + `assets/report-template.html` + `scripts/`
- **Personalization skill** `skills/report-personalization/` — a one-time setup utility: records your identity preset (portfolio · author · GitHub owner · footer) at `~/.claude/data-analysis-report-preset.json`, so generated reports carry *your* identity instead of the template author's. Run it before your first report.
- **Subagents** `agents/`
  - `data-source-scout` — Stage 3: explores/probes open data and returns a distilled candidate shortlist (keeps search noise out of the main context)
  - `data-quality` — Stage 6: reads the `data_quality.py` report and recommends per-column treatment (impute/drop/cap/keep), flags leakage
  - `analysis-critic` — Stages 5 & 8: adversarially reviews the method plan and the conclusions (catches false success / overclaiming)
- **Scripts** `skills/.../scripts/`
  - `data_quality.py` — strict profiler/gate (null-rate · IQR + robust-z outliers · duplicates · constant columns)
  - `execute_notebook.py` — runs the notebook end-to-end and embeds outputs (Stage 7 gate)
  - `screenshot_report.py` — renders the report to 8 PNGs (desktop/mobile × light/dark × ko/en, Stage 9 gate)

### Install
```bash
# Register the marketplace, then install
/plugin marketplace add sciencemj/data-analysis-project
/plugin install data-analysis-project
```

### Requirements (to run the scripts)
```bash
uv add pandas numpy                                            # Stage 4·6  data_quality.py
uv add --dev nbclient nbformat ipykernel                       # Stage 7    execute_notebook.py
uv add --dev playwright && uv run playwright install chromium  # Stage 9    screenshot_report.py
```

### Repo layout
```
.claude-plugin/{plugin,marketplace}.json   # manifest + marketplace entry
agents/                                     # 3 subagents
skills/data-analysis-project/              # the skill (SKILL.md · references · scripts · assets)
evals/                                      # dev-only behavioral tests (not shipped)
CLAUDE.md                                   # repo dev guidance (not shipped)
```

### Development & evals
`evals/evals.json` holds **behavioral test cases** consumed by an external eval harness (`skill-creator`) — there is no test runner in this repo, and assertions are LLM-judged. When you change a stage's behavior, update the matching assertion.

### <a id="license"></a>License
MIT
