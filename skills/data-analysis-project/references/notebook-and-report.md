# Notebook build + report assembly

Covers Stage 7 (notebook) and Stage 9 (report).

## Stage 7 — the notebook

The discipline that makes a notebook readable and trustworthy:

- **One cell = one action.** Setup, fetch, clean, join, each chart, each model — its own
  cell. Easy to test, re-run, and reason about.
- **At most one kind of output per cell.** A cell ends in *either* a dataframe preview,
  *or* one plot, *or* a print summary, *or* a map — not several. Single outputs keep the
  narrative legible and make diffs/embeds clean.
- **Test each cell as you write it.** Run it, look at the output, fix before moving on.
  Don't write ten cells then run — errors compound and localize poorly.
- **Markdown between cells.** After a cell produces a result, add a short markdown cell
  that *interprets what you just saw* ("강한 계절성 + 급성장 → 단순 전년반복으론 부족").
  This is the analysis, not decoration.
- **Reproducibility.** Read the **Stage-6 prepared & validated table** from `data/processed/`
  (don't re-clean in the notebook — prep happened in Stage 6); keep secrets in `.env`; set a Korean
  font for matplotlib when labels are Korean (`mpl.rcParams["font.family"]="AppleGothic"`
  on macOS, or a bundled CJK font).

### Modeling execution — only when a predictive (scored) model is fit

Applies when the analysis **fits a predictive model scored against a target metric** (forecast /
classification / regression that predicts a value — *not* regression used only to estimate an
effect/relationship). **Skip for descriptive or unsupervised endpoints** — EDA, clustering (e.g.
K-means), correlation, mapping, effect-estimation — which have no predict-and-score target; give
those their own method-appropriate validity instead (silhouette + k rationale for clustering,
CI / effect size for relationships), not the loop below.

When it does apply:
- **Fit, then tune — don't run once.** Try a few parameter settings and pick the winner on a
  **validation split or CV within the training set**; keep the **holdout/backtest untouched** and
  use it only for the final reported metric (never tune on the number you report). A small grid is
  fine; log what you searched. One untuned run is not a result.
- **Rank baseline vs candidates by the Stage-5 named metric, select a winner.** The ch.07 table
  in the report carries that **named metric** (MAPE/RMSE/AUC/…), not a generic "설명력/Fit".
- **Describe the chosen model** — key coefficients / feature importances / settings — and state
  its **assumptions** (stationarity, independence, no severe multicollinearity) and whether they hold.
- If the technique needs a data shape the table lacks, **loop back to Stage 6** to re-shape, then re-fit.

**For descriptive/unsupervised projects** (no predictive model): **keep the 방법 chapter** in the
report but replace the model-comparison table with a **method-appropriate validity table** (e.g.
silhouette by k, or correlation r + CI) — don't delete the method chapter.

### Verify the whole notebook (Stage 7 gate)

Writing cells isn't proof they run in sequence. Execute the notebook end-to-end and
confirm zero errors + embedded outputs. Use the bundled script:

```bash
python scripts/execute_notebook.py path/to/analysis.ipynb
```

It runs every cell with `nbclient` (live network/keys available, cwd = project root),
writes executed outputs back into the `.ipynb`, and reports any cell that errored or any
cell with more than one rich output (a violation of the one-output rule).

## Palette-matched charts

When the report has a design language, render matplotlib figures in the same palette so
they don't clash with the page. Set `figure.facecolor`/`axes.facecolor` to the canvas,
use the accent color for the primary series, hide top/right spines, color ticks muted,
and save with `bbox_inches="tight", facecolor=...`. Save figures to a `report_assets/`
folder and reference them from the HTML. CSS bars/tables look more "designed" than
matplotlib for simple distributions — use inline HTML for those, images for maps/trends.

## Stage 9 — the HTML report (prose-driven data essay)

Deliver **one self-contained HTML file** that reads like a **long-form data essay — a
hybrid of a blog post and an academic paper.** Flowing prose (줄글) is the backbone that
carries the argument; cards, figures, graphs, pull-quotes and callouts are **visual
accents inserted between prose blocks — never the main content.** Avoid the
"dashboard of cards with one-line captions" look. The essay still covers 목표 · 과정 ·
결과 · 보완할 점, but as a narrative, not a grid of tiles.

Start from `assets/report-template.html` — a neutral-default design-system template (warm
earth-tone tokens, light+dark, Space Grotesk/Hanken/JetBrains Mono, theme + language toggle;
restyle via the CSS variables). Copy it to the project as `report.html`, fill it in, and
**personalize the back-link / repo / footer to the running user — or remove them**.

### Title & structure
- **Title = the project's core question** (e.g. "범죄 적은 동네가 더 비쌀까?"). A strong
  descriptive title is fine for a part-2/follow-up report.
- **Hero** (`.wrap.hero-grid`): eyebrow (`데이터 에세이 · scope · period`), question title
  (`h1.title` with an accent `<em>`), a 2–3 sentence `.lead` stating the bottom line, a
  `.byline` meta row (unit · nature · read-time), `.hero-meta` key stats, and a side
  `.panel` holding the single most striking figure + headline number.
- **Numbered chapters** — each a `<section class="longform">`. Begin every chapter with a
  `.kicker` (`<span class="no">01</span>` + chapter name + `<span class="ln">` rule) and an
  `.essay h2`. Typical arc: 질문 → 데이터 → 데이터 품질·준비 → 배경 → 핵심 검증 → 해석 주의 →
  재프레임 → 방법 → 결론 → 한계. The **데이터 품질·준비** chapter is where Stages 4 & 6 surface
  to the reader: what the profile found (null-rate, outliers, duplicates) and how you treated
  it (impute / drop / cap + why) — keep it honest and brief, it strengthens trust in the result.
- **Conclusion** — open with the **explicit verdict against the goal** (the Stage-2 정량 목표:
  achieved vs threshold → MET / PARTIAL / NOT MET, stated plainly — never dress an unmet
  threshold as success), then lead with flowing prose (`p.lede` then a normal `p`) arguing the
  narrative, *then* summarize with insight cards (`.insights/.ins`). Prose first, cards second —
  never cards alone.
- **Limitations** — prose intro + cards; name what would overturn the conclusion. If the
  analysis reframed the problem, keep a prominent "most important reframe" `.callout` with
  the supporting number.
- **Closing CTA** (`.card.raised`, accent-soft) linking to the sibling/next report — delete
  if there is none.
- **Sources & references** (`.refbox`, two columns: 출처 / 참고 자료). Keep the footer.

### Layout primitives (CSS already in the template)
All flow blocks share one content width (`max-width:1080px`, the section width) so prose,
figures, cards and the title line up on the same left/right edge as the reader scrolls.
- `.essay` — the prose column. Prose `p` is 17.5px / line-height 1.86; bold spans use
  `var(--ink)`. `p.lede` is ~20.5px with an accent `::first-letter`.
- `figure.figure` — figures at the same content width; wrap the `<img>` in `.frame`. Caption
  = `figcaption` with a mono `.ft` tag ("그림 1 / Fig. 1") + an *interpretive* sentence (what
  it shows, not just what it is). `.wide` exists as a hook but currently equals the base width.
- `.midcol` / `.midcol.wide` — center tables, bar charts (`.bars`), or cards between prose
  blocks (same width).
- `.pull` → `blockquote` — accent left-border pull-quote in the display font, for the
  one-line takeaway of a section.
- `.aside .box` — tinted "방법 노트 / Method note" box for methodological caveats.
- `.srccards` — 3-up mini data-source cards.
- Reuse `.callout`, `.insights/.ins`, `.bars`, `.card`, `table`, `.refbox`. Use the viz
  palette (`--viz-*`) for multi-category charts and the mono font for metrics. **Do not
  invent new color/spacing values — use the design-system CSS variables.** CSS bars/tables
  look more "designed" than matplotlib for simple distributions; use inline HTML for those,
  images for maps/trends.

### Hard requirements
- **Bilingual.** Every translatable element has class `i18n` with both `data-ko` and
  `data-en`; the visible text = the `data-ko` version. The `applyLang()` script swaps
  `innerHTML`. Verify parity: count of `.i18n` == count of `data-ko` == count of `data-en`,
  and **zero `TODO`** remaining.
- **Keep the theme toggle** (`ddr-theme`) and the **language toggle** (`ddr-lang`). **Personalize
  the identity:** set the portfolio back-link, the repo link, and the footer to the *running
  user's* — or remove them. **Never ship the template author's URLs / shared footer**; the bundled
  default footer is self-contained.
- **Ground every number in the actual notebook/analysis output — never invent figures.**
  Caption each figure with what it *shows*.
- **Self-contained.** Opens standalone; every `report_assets/*` path resolves. Put figures
  in `report_assets/` next to `report.html` and update each `<img src>`.

### Visual-review gate (do not skip)
Screenshot across viewport × theme × language and actually look at the output:

```bash
python scripts/screenshot_report.py path/to/report.html   # → 8 PNGs in <report_dir>/_review/
```

It drives the report's own toggles and writes `{desktop,mobile} × {light,dark} × {ko,en}`.
Confirm: no element overlap, consistent spacing rhythm, content blocks aligned to one
left/right edge (prose, figures, cards), mobile collapses to a single column, EN (runs
longer than KO) doesn't break layout, both themes legible. **Fix → re-screenshot → repeat
until it passes.**
(Requires Playwright: `uv add --dev playwright && uv run playwright install chromium`.)

If publishing to GitHub, enable Pages (Settings → Pages → branch `main`, `/`) so the HTML
renders online — GitHub shows raw source otherwise — and link it from the README.
