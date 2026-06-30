---
name: data-analysis-project
description: >-
  End-to-end data analysis research workflow — from framing a real-world problem
  to shipping a polished HTML report. Run this whenever the user wants to do a data
  analysis project, explore or investigate a dataset, run EDA, build an analysis or
  research notebook, check whether a real problem actually shows up in data, analyze
  public/open data, forecast or model a metric, or produce a data report or dashboard
  — even if they never say the word "skill". It triggers on requests like
  "이 데이터로 분석해줘", "~에 대해 분석 프로젝트 하자", "analyze this dataset",
  "build a report from this data", "do EDA on X", "investigate X with data",
  "예측 모델 만들어줘", "공공데이터로 분석". Use it for the whole arc, not just one step.
---

# Data Analysis Project

A disciplined, end-to-end workflow for turning a vague "let's analyze X" into a
verified analysis and a shareable report. The value is in the **sequence and the
gates between stages** — most weak analyses fail because the goal was fuzzy or the
problem was never confirmed to exist, not because the modeling was wrong.

## Core principles (apply throughout)

- **Confirm before you build.** Each stage has a gate. Don't move to the next stage
  until the current one is real — the problem is verified, the goal is measurable,
  the data is actually accessible. Skipping a gate is how you spend a day analyzing
  a problem that doesn't exist.
- **Probe, don't assume.** API service names, CSV schemas, encodings, column meanings,
  join keys — verify them against the *real* source with a tiny live test before
  writing analysis code. Guessed field names and encodings are the #1 time sink.
- **Be honest with the result.** Report what the data actually shows, including when
  it contradicts the original premise. A finding that "the problem isn't what we
  thought" is a strong result, not a failure — surface it loudly.
- **Verify as you go.** Run each notebook cell as you write it; execute the whole
  notebook before claiming it works. Evidence over assertion.

## The workflow

Work through these nine stages in order. Treat the **→ gate** as a checkpoint you
must pass before continuing.

**Track progress with your todo list.** At the start, create **one todo per stage** (the nine
below) in your environment's task/todo list. Mark a stage **in-progress** when you enter it and
**completed only when its gate passes**, so you and the user always see the current stage at a
glance (확인) and the run keeps a record of what's done (기록). Add a todo for any Stage-8
loop-back or reframe so the detour stays visible.

**Project layout.** Build this structure in the user's project — create each part when its stage
needs it (scaffold `data/` + `scripts/` + `.env` at Stage 3; the rest appear at their stage):

```
<project>/
├── .env                    # API keys, gitignored — Stage 3
├── scripts/preprocess.py   # heavy download + aggregation (keeps the notebook light) — Stage 3
├── data/processed/         # compact aggregates + the Stage-6 validated table (committed); notebook reads here
├── notebooks/
│   ├── analysis.py         # jupytext percent source — EDIT THIS — Stage 7
│   └── analysis.ipynb      # generated + executed (jupytext → nbclient) — Stage 7
├── report.html             # the report, copied from the template — Stage 9
├── report_assets/          # figures referenced by report.html — Stage 9
└── _review/                # 8 screenshot PNGs from the visual gate (transient) — Stage 9
```

Keep raw downloads out of the repo (aggregate → `data/processed/`, delete bulky raw). The
personalization preset lives at `~/.claude/data-analysis-report-preset.json` (or a project
`./report-preset.json`).

### 1. 문제 제기 · Frame & qualify the problem (loop with the user)
Turn the hunch into a problem statement, then run it through a **qualification loop**
*before any data work*. The problem must clear three tests; if it fails any, ask the
user targeted questions, refine, and re-check — **loop until all three pass.**

1. **구체성 (Specific)** — named subject + a measurable phenomenon + time/scope bounds.
   "매출이 안 좋다" fails; "최근 12개월 재방문 고객 비중이 하락" passes.
2. **실재성 (Real)** — backed by evidence (an article/statistic or a quick data sniff),
   not just intuition. If you can't find any signal it's real, say so and reframe.
3. **실행가능성 → 비즈니스 인사이트 (Actionable)** — apply the **"so what?" test**: if you
   answered this perfectly, *what decision or action changes?* If the honest answer is
   "nothing", the problem won't yield a business insight — reshape it until it ties to a
   concrete decision (budget, placement, pricing, retention, staffing…).

Run the loop with `AskUserQuestion` or direct dialogue: for each failing test ask the
user the specific thing you need — *what decision would this inform? what does "good"
look like? which segment/period?* — fold in the answer and re-assess. Don't rush past a
vague or un-actionable problem; that is the single most common way an analysis ends up
useless. Show the user the current problem statement + which tests pass/fail each round
so they can see convergence.

**상황 점검 (quick feasibility scan):** alongside qualifying the problem, scan for blockers —
**데이터 가용성/접근 비용**, **법·프라이버시·라이선스** 제약, the **1–2 biggest risks + a
fallback**, and rough **비용 대비 효과**. Keep it to a few lines; surface blockers now, not after
modeling. (License/access detail is nailed down in Stage 3.) See `references/stage-playbook.md`.
→ **gate:** all three tests pass **and the user has confirmed** the problem statement, and the
quick 상황 점검 surfaced no unaddressed blocker (data access, legal/privacy, cost, key risks).
Only then move on. If evidence points to a different real problem, surface that reframe now rather than
after modeling.

### 2. 분석 목표 설정 · Set the analysis goal  ← most important
This is where projects are won or lost. Brainstorm the goal **in a loop until it is
concrete and measurable** — invoke the `superpowers:brainstorming` skill and keep iterating
with the user until you have: a specific question, the **decision** it informs, and a
**two-part success criterion**:
- **비즈니스 합격선** — the decision-level outcome that makes it useful ("재고 발주를 바꿀 만큼
  정확", "증설 우선순위 상위 N개를 신뢰").
- **정량 목표** — a number **derived from the decision, not the model** (the model is undecided
  until Stage 5). The problem *type* fixes the metric *family* — pick what fits, don't anchor on
  one: forecast → MAPE/sMAPE/RMSE/MAE/R²; classification → precision/recall@operating-point,
  F1, AUC, lift@k; ranking/targeting → precision@k, lift; spatial → % demand covered within Nm;
  relationship → correlation r / effect size / % variance explained. **Prefer a baseline-relative
  bar** ("naive/seasonal-naive/mean/majority/현행규칙 대비 ≥X 개선") since the model is unknown;
  use an absolute bar ("오차 ±10% 이내") only when the decision dictates it. If neither is settable
  yet: "Stage 5에서 확정, 최소 ≥ baseline".
→ **gate:** one-sentence goal + the decision + both criteria (business 합격선 & a model-agnostic
정량 목표), and the user has confirmed it. Do not start touching data before this.

### 3. 데이터 선정 · Source the data
**Scaffold the project first** (data work starts here): create `data/processed/` and `scripts/`,
and put any secrets in a **gitignored `.env`**. Then confirm relevant open/available data exists, then **investigate how to actually
access it** — exact API service names, endpoints, auth/keys, file formats, encoding,
schema, update cadence, and the **usage license / terms** (does it permit the intended use +
redistribution). Probe the live source to confirm before relying on it. When you pull the data,
acquire it **in bulk** (bulk download / largest page size / server-side filters), **not
record-by-record** — a slow per-record (N+1) loop is a last resort, only when the source offers no
other access.
→ **gate:** you have fetched a real sample and know the schema, keys, access method, and that
the license permits the intended use. See `references/data-sourcing.md`.

### 4. 데이터 이해 · Profile the data
Before planning method, **profile the data you actually sourced** — never carry *assumed*
characteristics into the plan. Run `scripts/data_quality.py <file>` and read it: per-column
dtype, **null-rate**, cardinality, distributions, and flagged **outliers / duplicates /
constant columns**. Then measure what drives the method choice: size, grain,
**seasonality/trend, gaps, class imbalance**. Keep it goal-targeted — profile to inform the
plan, not a generic EDA dump.
→ **gate:** a written profile of the *real* data (not the docs): every key field's null-rate
+ distribution seen, outliers/dupes/constant columns flagged, and the
size/grain/seasonality/imbalance facts the method needs are **measured, not assumed**. See
`references/stage-playbook.md`.

### 5. 분석 방법 · Plan the method
Choose methods that fit **both the goal and the data's characteristics** (size,
granularity, seasonality, gaps — **as measured in Stage 4, not assumed**). For **predictive**
work, plan model comparison baseline-first (a simple baseline a complex model must beat); for
**descriptive/unsupervised** endpoints (clustering/EDA/correlation/mapping) plan the
**method-appropriate validity** instead (silhouette / k-selection, CI / effect size, coverage).
Write the plan down. Now that the method is chosen, **finalize the Stage-2 정량 목표 into an exact metric + threshold** (e.g. "backtest
MAPE ≤ 15%", "precision@10% ≥ 2× random", "r ≥ 0.5 on holdout") — this is the bar Stage 8
judges against.
→ **gate:** a written plan mapping each goal sub-question to a concrete method and a
verification (holdout/backtest scheme + the **named metric + finalized threshold**, sanity
check). See `references/stage-playbook.md`.

### 6. 데이터 준비 · Prepare & validate the modeling table
Build the **final analysis-ready table** the notebook will model — CRISP-DM's most
labor-intensive phase, so do it deliberately, not as a stray cell. Clean (handle the
missing values + outliers the Stage-4 profile flagged), engineer the features the Stage-5
method needs, and select rows/columns with intent. Treatment is **judgment, not a fixed
rule** — delegate to the `data-quality` subagent (it reads the profile and recommends
per-column impute/drop/cap/keep tuned to the data shape + goal + method, and flags leakage),
then apply and **log every drop and impute**. The strict checker is the gate.
→ **gate:** `scripts/data_quality.py <table> --strict` exits 0 on the prepared table
(null-rate within bound; no full-row or key duplicates; no constant/all-null columns); join
row-counts / cardinality verified (no silent fan-out); no target/future leakage; every
transformation logged. See `references/stage-playbook.md`.

### 7. 코드 작성 · Build the notebook
Author the notebook as a **jupytext percent source** `notebooks/analysis.py` — a `# %%` code cell
= **one action** emitting **at most one kind of output**, with `# %% [markdown]` cells between that
**interpret the result you just saw**; test each cell as you write it. Then **generate and execute**:
`jupytext --to notebook notebooks/analysis.py` → `notebooks/analysis.ipynb`, then
`scripts/execute_notebook.py` to run clean and embed outputs. The `.py` is the editable source; the
`.ipynb` is the generated, executed artifact — commit both.

**If the project fits a predictive model** (forecast / classification / regression that
**predicts a value**, scored against a target metric — *not* regression used only to estimate an
effect/relationship, which is N/A): don't run it once — **fit and lightly tune** (try a few
settings; pick the winner on a **validation split or CV**, keeping the **holdout/backtest
untouched** for the final reported metric only — never tune on the number you report), **rank
baseline vs candidates by the Stage-5 named metric and select a winner**, and describe the chosen
model + its assumptions. **N/A for descriptive/unsupervised endpoints** (EDA, clustering e.g.
K-means, correlation, mapping) — give those method-appropriate validity (silhouette / k-selection
rationale, CI / effect size) instead of this backtest-and-tune loop.
→ **gate:** notebook executes top-to-bottom with zero errors; outputs match the markdown
commentary; and **if a predictive model was fit**, candidates were ranked by the named metric, a
winner selected, and a small tuning search logged (not a single un-compared, untuned run). See
`references/notebook-and-report.md`.

### 8. 결론 · 비즈니스 인사이트 · Conclude & critique
First, **render an explicit verdict against the Stage-2 criteria**: restate the finalized 정량
목표 (metric + threshold from Stage 5) and the 비즈니스 합격선, put the **achieved number** next
to each, and declare **MET / PARTIAL / NOT MET** — a number-to-threshold comparison, not a
qualitative "answered the goal". **Never declare success on an unmet threshold** (no false
success); a fine surface metric ≠ the business 합격선 met, so check both. Then derive business
insights tied to specific numbers, and **turn the lens on the study itself**: framed right?
what's missing? limitations + next steps.

If the verdict is **PARTIAL / NOT MET**, this is a **go/no-go decision the user makes** —
present the shortfall and **ask the user**: loop back (refine method/data at Stage 5/6, or
reframe the problem at Stage 1/2) **or** proceed and publish an honest "목표 미달 — 사유 + 다음
수" report. A reframe or an honest negative is a strong result; a faked success is not.
→ **gate:** an explicit MET/PARTIAL/NOT-MET verdict (achieved vs threshold, numeric) for both
criteria; insights tied to numbers; limitations name what would flip the conclusion; and on a
shortfall, the user has chosen loop-back vs honest-publish.

### 9. 리포트 작성 · Write the report
Produce a single self-contained **HTML report** as a **prose-driven data essay (blog ×
paper hybrid)**: flowing prose (줄글) carries the argument, and cards/figures/graphs/
pull-quotes are visual accents between prose blocks — **not** a dashboard of cards. It
covers **목표 · 과정 · 결과 · 보완할 점** as numbered chapters (질문 → 데이터 →
데이터 품질·준비 → 배경 → 핵심 검증 → 해석 주의 → 재프레임 → 방법 → 결론 → 한계). Start from the bundled template
(`assets/report-template.html`); it is **bilingual (ko/en)** with a language + theme
toggle. Title the report with the project's core question. Ground every number in the
notebook output; never invent figures.

**Personalize the report's identity.** Before filling the template's author slots (the nav
back-link, the GitHub repo link, the footer), apply the running user's **personalization preset**
(project `./report-preset.json`, else `~/.claude/data-analysis-report-preset.json`): use their
portfolio URL (omit the back-link if none), GitHub owner, footer choice, and author name in place
of the template's defaults. If no preset exists and you are **not** the template author, run the
**`report-personalization`** skill first (or ask for these inline) — never ship someone else's identity.
→ **gate:** the report opens standalone and every asset resolves; the limitations section
is present; i18n parity holds (`.i18n` == `data-ko` == `data-en`) with zero `TODO`; and
the **visual-review gate passes** — `scripts/screenshot_report.py` renders 8 PNGs
(desktop/mobile × light/dark × ko/en) and you confirmed no overlap, content blocks aligned
to one left/right edge, mobile single-column, and EN not breaking layout. See
`references/notebook-and-report.md`.

**Publish (optional, with the user's go-ahead).** Publishing is outward-facing — confirm first.
Commit `report.html` + `report_assets/`, enable **GitHub Pages** on the repo (branch `main`, `/`),
and share the live URL (`https://<owner>.github.io/<repo>/report.html`) — Pages renders the HTML,
while a raw `.html` on github.com shows only source. Verify it loads and assets resolve, link it
from the README / portfolio, and **add repo topics** (`gh repo edit --add-topic`) so the published
project is discoverable. See `references/notebook-and-report.md`.

## Bundled resources

Read these when you reach the relevant stage — don't load everything upfront.

- `references/stage-playbook.md` — detailed per-stage tactics: the brainstorming-loop
  pattern, the data-profiling and data-preparation checklists, and the method-planning template.
- `references/data-sourcing.md` — finding open data and the gritty access details:
  OpenAPI paging, encodings (cp949/euc-kr), heterogeneous CSV schemas, keys via
  `.env`, caching aggregates so the notebook stays light.
- `references/notebook-and-report.md` — the cell-per-action discipline, executing a
  notebook with `nbclient` to verify + embed outputs, palette-matched matplotlib,
  and assembling the HTML report.
- `assets/report-template.html` — the Sciencemj **prose-essay** report template:
  design-system tokens, light/dark, **bilingual ko/en** (`.i18n` + `data-ko`/`data-en`,
  `applyLang`), theme + language toggle, `← 포트폴리오` back-link + shared `footer.js`, and
  the essay layout primitives (`.essay`, `figure.figure`, `.midcol`, `.pull`, `.aside`,
  `.srccards`). Copy to `report.html`, replace every `TODO` (fill both languages), keep the
  footer + toggles, delete unused chapters.
- `scripts/data_quality.py` — strict profiler + gate: per-column null-rate, outliers
  (IQR + robust-z), duplicates, constant/all-null columns. Report mode feeds the Stage 4
  profile; `--strict` is the Stage 6 data-prep gate (exits nonzero on violation).
- `scripts/execute_notebook.py` — run a `.ipynb` end-to-end with nbclient and write
  executed outputs back in place (use for the Stage 7 gate).
- `scripts/screenshot_report.py` — render the report to 8 PNGs (viewport × theme ×
  language) with Playwright for the Stage 9 visual-review gate.
- `references/subagents.md` — when subagents are available, delegation patterns for a
  **data-source scout** (Stage 3, preserves context), a **data-quality reviewer** (Stage 6,
  treatment judgment), and a **critic/evaluator** (Stages 5 & 8, adversarial review), with
  ready-to-use prompts and return schemas.

## Scaling with subagents (optional, when available)

If your environment can spawn subagents, three delegations meaningfully raise quality.
They're optional — skip them when subagents aren't available — but high-leverage. This
plugin ships all three as ready subagent types — **`data-source-scout`**, **`data-quality`**,
and **`analysis-critic`** — so you can dispatch them by name; full prompt templates and
return schemas are in `references/subagents.md`.

- **Data-source scout (Stage 3) — protect the main context** (agent: `data-source-scout`). Searching portals and
  probing endpoints produces huge, noisy tool output. Delegate it to a dedicated scout
  subagent that does the messy exploration in *its own* context and returns only a
  **distilled shortlist of candidate sources** (name · access method · key fields/grain ·
  span · license · access difficulty · fit-to-goal). The main thread reads the shortlist,
  picks one, and does the deep probe — its context stays clean for the analysis itself.
- **Data-quality reviewer (Stage 6) — treatment judgment for varied data** (agent: `data-quality`).
  The script (`data_quality.py`) is the deterministic gate; the agent is the judgment the
  script can't give — it reads the profile/report and recommends per-column missing-value /
  outlier / dedup treatment (impute vs drop vs cap vs keep-as-signal) tuned to the data
  shape + goal + method, and flags leakage. Apply its calls, log them, then re-run the
  strict gate until it passes.
- **Critic / evaluator (Stages 5 & 8) — independent adversarial review** (agent: `analysis-critic`). Hand your draft
  to a critic subagent prompted to *find holes, not to praise*. At Stage 5 it stress-tests
  the method plan (does each method fit the goal + data? baseline present? verification
  defined? what's missing or risky?). At Stage 8 it judges each conclusion (do the numbers
  actually support this claim? confounds? overclaiming? does it answer the Stage-2 goal?
  what's unexamined?). Fold the verdicts back in and revise. A fresh, skeptical reader
  catches what the author can't see.

## What "good" looks like

A project that lands: a confirmed problem, a one-sentence measurable goal, data whose
schema you actually verified, a notebook that runs clean with per-cell commentary, a
conclusion honest about whether it answered the question, and an HTML report a
stakeholder could read in five minutes. The reframe — discovering the real problem is
different from the assumed one — is often the most valuable output.
