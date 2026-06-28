# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is **not an application** — it is a Claude Code **plugin** that packages a single
**skill** (`data-analysis-project`). The "product" is the prompt content itself: `SKILL.md`
plus its bundled references, assets, and helper scripts. Editing this repo means editing the
instructions a future Claude instance reads when it runs a data-analysis project. Optimize for
what the *reading* Claude will do, not for runtime behavior of a program.

It is an **opinionated implementation of CRISP-DM** — the nine gated stages map onto CRISP-DM's
six phases (Business Understanding → 1–2, Data Understanding → 3–4, Data Preparation → 6,
Modeling → 5 + 7, Evaluation → 8, Deployment → 9). The README documents this mapping.

**Repo layout (plugin):**

```
.claude-plugin/plugin.json          # plugin manifest (name, version, skills path)
.claude-plugin/marketplace.json     # marketplace entry — makes the plugin installable
agents/                             # plugin subagents: data-source-scout, data-quality, analysis-critic
skills/data-analysis-project/       # the skill payload — paths below are relative to here
  SKILL.md  references/  scripts/  assets/
evals/                              # dev-only: behavioral test cases + fixtures (not shipped)
README.md / README.ko.md            # bilingual docs (EN default + ko language button), CRISP-DM framing
LICENSE                             # MIT
CLAUDE.md                          # this file (repo dev guidance, not shipped)
```

Inside the **Architecture** section below, bare names like `SKILL.md`, `references/*.md`,
`scripts/…`, `assets/…` are all relative to `skills/data-analysis-project/`.

The **`agents/`** dir holds three plugin subagents that promote the delegation patterns in
`skills/data-analysis-project/references/subagents.md` into real, dispatchable subagent
types: **`data-source-scout`** (Stage 3 — returns a distilled dataset shortlist, keeps
search noise out of the main context), **`data-quality`** (Stage 6 — reads the
`data_quality.py` report and recommends per-column missing/outlier/dedup treatment tuned to
the data shape + goal + method), and **`analysis-critic`** (Stages 5 & 8 — adversarial,
default-skeptical review of the method plan and the conclusions). The skill body and
`references/subagents.md` are the source of truth for their prompts; if you change an agent's
behavior, update both the agent file and the reference.

## Architecture — progressive disclosure

The skill is built around context economy: load the minimum upfront, pull detail on demand.

- **`SKILL.md`** — the always-loaded entry point. Frontmatter `description` is the trigger
  (verbose on purpose, bilingual KO/EN, lists trigger phrases). Body holds the **9-stage
  workflow** with a **`→ gate`** after each stage. The gates are the core idea: each stage
  must be verified real before the next begins. The run tracks progress with a **per-stage
  Claude TODO** (in-progress on entry, completed only when the gate passes). Keep `SKILL.md`
  lean — it points to the references rather than inlining their detail.
- **`references/*.md`** — loaded only when the relevant stage is reached. One file per
  concern: `stage-playbook.md` (per-stage tactics + templates, incl. Stage 4 profiling &
  Stage 6 data-prep checklists), `data-sourcing.md` (Stage 3 access details),
  `notebook-and-report.md` (Stages 7 & 9), `subagents.md` (optional scout + data-quality +
  critic delegation). Adding depth → put it in a reference and link from `SKILL.md`.
- **`assets/report-template.html`** — the Stage 9 deliverable template: a **prose-driven
  data essay** (blog × paper hybrid), not a dashboard of cards. Self-contained, Sciencemj
  design system, **bilingual ko/en** (`.i18n` + `data-ko`/`data-en`, `applyLang` swaps
  `innerHTML`), theme + language toggles, essay primitives (`.essay`, `figure.figure`,
  `.midcol`, `.pull`, `.aside`, `.srccards`). `TODO` placeholders the user-facing run fills
  in. Carries a `← 포트폴리오` back-link and an `embed/footer.js` `<script>` that must survive copies.
- **`scripts/data_quality.py`** — the Stage 4 profiler (report mode) and the Stage 6
  data-prep gate (`--strict`): per-column null-rate, outliers (IQR + robust-z), duplicates,
  constant/all-null columns; exits nonzero on violation.
- **`scripts/execute_notebook.py`** — the Stage 7 gate (run notebook, write outputs back).
- **`scripts/screenshot_report.py`** — the Stage 9 visual gate (Playwright → 8 PNGs,
  viewport × theme × language).
- **`evals/`** — `evals.json` defines behavioral test cases (assertions are natural-language
  expectations checked by an eval harness, e.g. skill-creator); `evals/files/` holds fixtures.

The three layers stay consistent: a stage in `SKILL.md`, its tactics in a reference, and an
eval assertion that the behavior happens. Change one, check the other two.

## The workflow the skill encodes (the actual content)

Nine gated stages, in order: **1** frame & qualify the problem (3 tests: 구체성/실재성/실행가능성,
loop with user; quick 상황 점검 — data/legal/cost/risk feasibility) → **2** set a measurable goal — business 합격선 + a model-agnostic 정량 목표
(most important; uses `superpowers:brainstorming`) → **3** source & probe real data → **4**
profile the sourced data (null-rate/outliers/dupes via `data_quality.py`; measure
size/grain/seasonality/imbalance) → **5** plan method (baseline-first; finalize the metric +
threshold) → **6** prepare & validate the modeling table (strict missing/outlier checks + the
`data-quality` agent; gate = `data_quality.py --strict`) → **7** build notebook (one cell = one
action, one output per cell, markdown interpretation between cells; when a predictive (scored) model is fit, tune + rank/select by the named metric) → **8** conclude — explicit
MET/PARTIAL/NOT-MET verdict vs the goal + user go/no-go (loop-back or honest-publish) +
self-critique → **9**
self-contained, bilingual, prose-driven HTML **data essay** (numbered chapters; cards/figures are
accents, not the backbone) with a limitations section, passed through the screenshot
visual-review gate.
The skill's thesis: weak analyses fail at framing/goal, not modeling — so the gates and the
"reframe is a valid outcome" stance are load-bearing, not filler.

## Commands

```bash
# Stage-4 profile / Stage-6 strict gate: null-rate, outliers (IQR+z), dupes, constant cols
python skills/data-analysis-project/scripts/data_quality.py path/to/data.csv                  # report mode
python skills/data-analysis-project/scripts/data_quality.py path/to/prepared.csv --strict     # gate (exit≠0 on violation)
uv add pandas numpy                                    # deps

# Stage-7 gate: run a notebook end-to-end, write outputs back, report errors + multi-output cells
python skills/data-analysis-project/scripts/execute_notebook.py path/to/analysis.ipynb [--timeout 900]
uv add --dev nbclient nbformat ipykernel              # deps

# Stage-9 visual gate: render report to 8 PNGs (desktop/mobile × light/dark × ko/en)
python skills/data-analysis-project/scripts/screenshot_report.py path/to/report.html [--out DIR]
uv add --dev playwright && uv run playwright install chromium   # deps
```

There is no build/lint/test runner in this repo. `evals/evals.json` is consumed by an
external eval harness (skill-creator), not run from here.

## Conventions when editing

- **Match the bilingual voice.** Stage names and trigger phrases are Korean + English; keep
  both. Examples are concrete and often domain-specific (따릉이/지하철/카페) — preserve that texture.
- **Keep `SKILL.md` short, push detail down.** If a section grows long, it belongs in a
  `references/` file with a one-line pointer from `SKILL.md`.
- **Preserve the gate structure.** Every stage ends with `→ gate:`. Don't soften gates into
  suggestions — they are the skill's main contribution.
- **The template stays self-contained, bilingual, prose-first.** Don't split
  `report-template.html` into external CSS/JS; keep the footer `<script>`, the theme +
  language toggles, and the `← 포트폴리오` link intact. Every translatable node carries
  `.i18n` + both `data-ko`/`data-en` (keep parity). Prose is the backbone — cards/figures
  are accents; don't regress it into a card dashboard.
- When you change a stage's behavior, update the matching `evals/evals.json` assertion so the
  test still describes the real intended behavior.
